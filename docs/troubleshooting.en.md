# Troubleshooting log

Real problems hit while building this, kept here because the fixes
aren't obvious from the configs alone.

## TCP breaks after enabling UDP on the same inbound

Symptom: after adding `sockopt.tproxy` to the single `dokodemo-door`
inbound that also served TCP via REDIRECT, everything that used to work
over TCP (Telegram, proxied sites) started failing — Xray logs showed it
tunneling requests to `<router-ip>:<xray-port>` (itself) instead of the
real destination.

Cause: REDIRECT (DNAT) recovers the original destination via
`getsockopt(SO_ORIGINAL_DST)`. TPROXY makes the socket "transparent"
(`IP_TRANSPARENT`) and just trusts the socket's local address as the
destination — it does no such lookup. Putting both behaviors on one
socket breaks the TCP path's destination resolution.

Fix: split into `transparent-in-tcp` (port 12345, `network: tcp`, plain
REDIRECT, no tproxy sockopt) and `transparent-in-udp` (port 12346,
`network: udp`, `sockopt.tproxy: tproxy`). Route both to `proxy` via
`routing.rules[0].inboundTag` as an array.

## `nft ... tproxy to :PORT` fails with "conflicting protocols"

```
Error: conflicting protocols specified: ip vs. unknown. You must specify
ip or ip6 family in tproxy statement
```

In an `inet` (dual-stack) table, `tproxy` needs an explicit family:

```
... tproxy ip to :12346
```

## `nft ... tproxy ...` fails with "No such file or directory"

Not a syntax error — the `nft_tproxy` kernel module isn't loaded (and on
OpenWrt, isn't installed by default). Install it:

```sh
opkg update
opkg install kmod-nft-tproxy
```

TPROXY also requires policy routing, or the kernel won't deliver locally
marked packets to the listening process:

```sh
ip rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
```

## TPROXY rule must live in `mangle`, not `nat`

The existing TCP redirect table uses a `nat`/`dstnat` hook. TPROXY does
not work there — it needs its own table with a `filter` hook at
`mangle - 10` priority. Keep it as a **separate** nftables table
(`xray-udp` here) rather than trying to add it to the existing `nat`
table.

## `telegram4` nft set goes empty after a reboot

The IP set that back the Telegram redirect rule is populated by
downloading a list on start — if the router's WAN/DHCP isn't up yet when
the init script runs (`S21`, early in boot), `wget` fails once and the
set is left empty (a fresh, valid, but unpopulated set — not an old
cached one, since this is the very first boot with the script).

Fix: retry the download a handful of times with a delay instead of
failing after one attempt (see `download_list()` in
`router/etc/init.d/xray-nft`).

A second, unrelated cause of the same symptom showed up later: setting
`"loglevel": "debug"` in the Xray config makes it log a line per UDP
packet. Under sustained UDP load (e.g. an active game session) this
appears to have overloaded the router (high load average, syslog buffer
overflow, at least one apparent spontaneous reboot) — which also wipes
`telegram4` again. Keep `loglevel` at `warning` or higher in normal use;
only turn on `debug` while actively diagnosing something, and turn it
back off afterward.

## `mux.cool` breaks when combined with `flow: xtls-rprx-vision`

```
common/mux: failed to handler mux client connection > proxy/vless/outbound:
failed to decode response header > failed to read response version > EOF
```

`vision` needs to see and manipulate the raw TLS stream (padding,
splicing). `mux` wraps traffic in its own framed protocol
(`v1.mux.cool`) before it ever reaches the TLS layer, which vision can't
parse correctly. They are mutually exclusive. Pick one:

- Keep `vision`, skip `mux` — better anti-fingerprinting, but every new
  TCP connection to the VPS pays a full TLS+Reality handshake.
- Drop `flow` entirely (both client and server, they must match), enable
  `mux` — connections get reused, first-connection latency drops
  drastically, but you lose vision's traffic-shape obfuscation.

## Mux'd UDP flows get jittery (games)

If both the TCP and UDP transparent inbounds route through the *same*
muxed outbound, game UDP traffic (which is really TCP-tunneled) shares
one physical connection with web browsing traffic. A single dropped
packet head-of-line-blocks everything multiplexed on that connection —
worse for something like game matchmaking that opens lots of short-lived
flows than the pre-mux setup, where each UDP "session" got its own
independent tunnel.

Fix: two separate VLESS outbounds to the same VPS — one with `mux`
enabled for the TCP inbound, one without `mux` for the UDP inbound. See
`routing.rules` in `router/etc/xray/config.json`.

UDP later moved off VLESS to hysteria2 altogether, see "hysteria2 on the router" below. The `proxy-udp` outbound is now a socks outbound to sing-box.

## Config files with a markdown-link artifact in string fields

If you ever see a field like `serverName` or `dest` come out looking
like `"[www.idealo.de](https://www.idealo.de)"` instead of a plain
domain — that's a copy-paste artifact from a markdown-rendering source,
not something Xray needs. Strip it down to the plain domain string.
Xray/Reality seems to tolerate it silently in some cases, but don't rely
on that.

## TCP Fast Open didn't help

Enabled on both ends (`net.ipv4.tcp_fastopen` sysctl +
`sockopt.tcpFastOpen: true` in Xray's `streamSettings`). No measurable
difference in first-connection latency. TFO only speeds up the TCP SYN
round-trip; on this setup essentially all first-connection latency was
in the Reality/TLS 1.3 handshake, a different protocol layer that TFO
doesn't touch. Left enabled since it's harmless, but don't expect it to
fix handshake-bound latency.

## "All UDP is blocked" turned out to be a port whitelist

For a long time I assumed dorm UDP was fully blocked: QUIC, WireGuard, and hysteria2 on standard ports didn't get through. Probing with `portprobe` from [splicertc](https://github.com/rypleshhh/splicertc) showed a whitelist, and a different one for different segments of the same network. From the dorm Wi-Fi UDP 123, 3478, 5001 are open; from the segment where the router's WAN sits, 20, 21, 161, 194, 520–540, 902, 903. No overlap.

Two takeaways. Measure from the device and segment the tunnel will actually originate from: testing from a PC behind the router is useless, its UDP goes into TPROXY. And on the server it's convenient to keep a single hysteria port and REDIRECT the others to it (see `vps/etc/ufw/before.rules.nat`).

`nslookup ya.ru 8.8.8.8` and `w32tm /stripchart /computer:pool.ntp.org` only show that 53 and 123 are open to those particular servers. They say nothing about the whitelist towards your own VPS.

## Sites that aren't in the lists stop loading

Symptom: ordinary sites (not in `russia4`/`telegram4`) intermittently fail to load and come back after toggling a VPN on the PC. `curl` over TCP works at the same moment.

Cause: all LAN UDP went into TPROXY, QUIC (UDP/443) included. Browsers opened HTTP/3 sites (`Alt-Svc: h3=":443"` in the response) through the tunnel. When the tunnel stalled, the QUIC session still looked alive and the browser didn't fall back to TCP. Toggling a VPN changed the network, the browser dropped its sessions and temporarily marked QUIC as broken, hence "the VPN helps".

Fix: `udp dport 443 return` as the first rule of the `xray-udp` chain (inserted in `router/etc/init.d/xray-nft-udp`) plus an fw4 rule `Reject-QUIC` (lan → wan, udp/443, REJECT) so the browser gets an immediate refusal instead of waiting for a timeout. Live check: `nft list chain inet xray-udp prerouting`, the counter on `return` grows while YouTube is open.

Along the way it turned out about 12 GB went through the UDP tunnel in a month, mostly QUIC video, in the same queue as game traffic.

## Connectivity to the VPS drops entirely for a few minutes

Symptom: the VPS stops answering everything, ping included, from the PC and the router at once, sometimes from mobile data too. A few minutes later it all comes back on its own. Meanwhile the server is alive and reachable through a VPN that exits on that same server. A second server at the same hoster behaves similarly.

Checked and ruled out:

- fail2ban: my IP never showed up in bans, and the `sshd` jail only blocks 22/tcp.
- ufw on the server: rules in place, `ufw-before-input` accepts `echo-request`.
- DNS and `ipconfig /renew`: unrelated, things break after resolution.
- tcpdump on the VPS during an outage: nothing arrives from our IP, neither ICMP nor UDP. It's dropped before the server.

One capture of an outage also showed that new TLS connections from the router to the VPS stalled at the handshake (ServerHello left the server and was never acknowledged), while already established long-lived connections worked in the same second. That looks like a stateful filter deciding per new flow. Root cause not found. The router runs a cron job that pings the VPS every minute and writes unanswered minutes to `/root/vps-down.log`, to see whether outages are periodic.

Diagnostic traps that cost time:

- `pathping` stops route discovery at the first hop that doesn't answer and never sends anything to the server. Transit routers rate-limit ICMP time-exceeded, so a break at hop 11–12 means nothing. Use WinMTR for per-hop loss, `tracert -d` for the route.
- With a VPN in TUN mode, ping to the server is answered by the TUN stack itself and the server shows up "at hop one". That ping measures nothing.
- `Test-NetConnection -Port 443` only checks the TCP handshake. The freeze happens later, during TLS, and shows up with `curl -v`.

## hysteria2 on the router: Xray 25.x can't do it

Xray-core 25.1.30 from the OpenWrt repo doesn't know a `hysteria` outbound; the `"protocol": "hysteria"` syntax appeared in later versions. Instead of upgrading Xray on the router, `sing-box` (packaged for 24.10) is installed: it serves SOCKS5 on `127.0.0.1:1080` and holds hysteria2 to the VPS, and `proxy-udp` in Xray became a socks outbound to that port. Interception doesn't change.

If sing-box logs `timeout: no recent network activity`, not a single QUIC packet came back from the server. A wrong password would give an auth error, not a timeout. Run `tcpdump -ni <iface> udp and host <router-public-ip>` on the VPS: empty means the port is dropped on the way.

## sing-box doesn't fit on the router: extroot

The RT-AX53U has about 32 MB of overlay, 17 MB were free, and full `sing-box` unpacked doesn't fit. Fix: extroot on a USB stick. The stick was physically plugged in but the system didn't see it: a factory reset had dropped `kmod-usb-storage`, `block-mount`, and `fstab`, and `dmesg` only had `usb 1-2: new high-speed USB device` and `mount_root: switching to ubifs overlay`.

What matters:

- The stick still held the overlay from the old install. Don't attach it as is, the router would come up with stale configs. I formatted it (`mkfs.ext4`) and copied the current `/overlay` onto it.
- Once `block-mount` is installed, hotplug mounts the stick at `/mnt/sda1` by itself, so a manual `mount` returns `Resource busy`. That's fine, data is written to the stick.
- The `fstab.extroot` entry must be created before copying the overlay, otherwise the new overlay won't have it.
- If the stick dies, the router simply boots from internal flash.

## VPS: Docker publishes ports around ufw

After `ufw default deny incoming`, container ports published with `-p` (e.g. `0.0.0.0:8080->8080`) stay open: Docker writes its own iptables rules that ufw doesn't see. Check `ss -tulpn` for `docker-proxy`. Bind to `127.0.0.1:8080:8080` in compose to close one. A container with `network_mode: host`, on the other hand, is subject to ufw.

Also: stopped containers with `restart: always` come back after `systemctl restart docker` and grab their ports again. Remove unneeded ones instead of just stopping them.
