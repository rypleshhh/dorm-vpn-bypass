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
