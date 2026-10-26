# Dorm network bypass: selective routing on OpenWrt (TCP over VLESS+Reality, UDP over hysteria2)

A personal, working setup for a dormitory network where UDP only gets out on a handful of ports and TCP is DPI-filtered. Everything is configured once on an OpenWrt router backed by a single VPS. Devices on the LAN know nothing about the tunnel.

Not an installer, a reference. Bring your own IPs, domains, keys, and LAN subnet.

## The network

- UDP is a port whitelist, and the whitelist differs between segments of the same network. In my case the segment where the router's WAN sits lets out UDP 20, 21, 161, 194, 520–540, 902, 903, while the dorm Wi-Fi (a different segment) lets out 123, 3478, 5001. Everything else is silently dropped. Your ports will differ, measure them (see "Finding open ports").
- Outbound TCP gets through on 443 and a couple of standard ports, and TCP/443 is DPI-inspected.
- Connectivity to the VPS periodically drops entirely for a few minutes, ICMP included, then comes back on its own. Root cause not found, details in `docs/troubleshooting.en.md`.

Goal: only blocked sites, Telegram, and UDP (games, voice) go through the VPS, everything else goes direct, with no per-device setup.

## Architecture

```
LAN client
  |
  v
dnsmasq resolves domain --> nft set (russia4 / telegram4)
  |
  v
nftables prerouting:
  TCP/443 matching those sets --> REDIRECT --> Xray dokodemo-door :12345
  UDP/443 (QUIC)              --> return   --> fw4 REJECT, browser falls back to TCP
  UDP/53                      --> bypass, DNS stays on the router
  all other UDP               --> TPROXY   --> Xray dokodemo-door :12346
  |
  v
Xray on the router
  TCP: outbound "proxy"     VLESS+Reality to VPS:443, mux on
  UDP: outbound "proxy-udp" SOCKS5 to 127.0.0.1:1080
                               |
                               v
                            sing-box: hysteria2 to VPS:520-540 (port hopping)
  |
  v
VPS
  TCP/443          --> Xray, VLESS+Reality
  UDP/520-540      --> iptables REDIRECT --> hysteria2 on :123
  |
  v
Internet
```

The router's own traffic (sing-box to the VPS) goes through OUTPUT, while TPROXY sits on prerouting and only matches `ip saddr` from the LAN, so there's no loop.

### Why UDP goes over hysteria2 instead of VLESS

UDP used to go through a second VLESS outbound, i.e. UDP over TCP. It worked but inherited head-of-line blocking: one lost TCP segment stalled every UDP packet behind it, and games with lots of short flows (relay probing, matchmaking) got jittery. Once it turned out UDP wasn't blocked outright but whitelisted by port, real UDP transport became an option. hysteria2 runs over QUIC, so loss on one flow doesn't stall the others.

Xray 25.x on the router has no hysteria2 outbound, so there's a bridge: Xray hands UDP to a local SOCKS5, sing-box serves it and holds the hysteria2 connection. Interception (TPROXY) and routing are unchanged, only the `proxy-udp` outbound changed. Rolling back to VLESS means swapping one outbound.

### Port hopping

On the server hysteria listens on a single port: 123, which is open from the dorm Wi-Fi, so devices there connect to it directly without the router. For the router, whose segment has a different set of open ports, the VPS has a REDIRECT rule: anything arriving on UDP 520–540 is redirected to 123. sing-box on the router picks a random port from that range and switches every 30 seconds (`server_ports` + `hop_interval`). If DPI starts dropping one port, the connection moves to a neighbouring one.

### Why QUIC (UDP/443) stays out of the tunnel

Most large sites serve HTTP/3, so the browser talks QUIC to them over UDP/443. Tunnel that UDP and YouTube, Google, everything behind Cloudflare, Yandex, and VK all go through the VPS, including what works fine direct. Worse, when the tunnel stalls the browser keeps a QUIC session it considers alive and doesn't fall back to TCP, so the site just doesn't load. UDP/443 is therefore excluded from TPROXY and rejected by fw4 with ICMP port unreachable. The browser immediately switches to TCP, and TCP follows the normal selective path.

### Why TCP and UDP have separate Xray inbounds

Originally one `dokodemo-door` handled both. Enabling `sockopt.tproxy` on it (needed for UDP) broke `SO_ORIGINAL_DST` for TCP: REDIRECT (DNAT) and TPROXY are different kernel mechanisms that don't mix on one socket, and Xray started "tunneling" requests to itself. Fix: two inbounds on two ports, TCP keeps REDIRECT, UDP gets its own TPROXY inbound.

### Mux and `flow: xtls-rprx-vision`

The TCP outbound uses `mux.cool`: many logical streams over one connection to the VPS, no fresh TLS+Reality handshake per new domain. First-connection time dropped from about 0.43 s to 0.11 s.

`xtls-rprx-vision` doesn't work with mux: mux wraps traffic in its own framing, vision expects raw TLS, and the server fails decoding the response with EOF. `flow` is removed on both ends (they must match). A deliberate trade of traffic-shape masking for latency: enough for ordinary DPI blocking, not built to resist active probing.

## Finding open ports

[splicertc](https://github.com/rypleshhh/splicertc) has `cmd/portprobe`. The server listens on a list of ports and answers with a token carrying the port number; the client walks the ports and checks the token, so you see both closed ports and cases where something in the middle answered.

```sh
# VPS
ufw allow 1:65535/udp
./portprobe -mode server -tcp none -udp 1-1024,1194,3478,4500,5000-5010,8443,19302,27015,51820

# client: run it where the tunnel will actually originate
./portprobe -host YOUR_VPS_IP -tcp none -udp 1-1024,1194,3478,4500,5000-5010,8443,19302,27015,51820 -parallel 64 -timeout 2s

# VPS, afterwards
ufw delete allow 1:65535/udp
```

Measure from the router itself, not from a PC behind it: the PC's UDP goes into TPROXY and you'd be testing the tunnel, not the network. For MT7621 build the client with `GOOS=linux GOARCH=mipsle GOMIPS=softfloat CGO_ENABLED=0 go build ./cmd/portprobe`. Segments of the same network can differ, so check Wi-Fi and the router's uplink separately.

## Layout

```
router/etc/xray/config.json             Xray on the router
router/etc/sing-box/config.json         hysteria2 client, SOCKS5 on 127.0.0.1:1080
router/etc/init.d/xray-nft              TCP redirect table + Telegram IP list updater
router/etc/init.d/xray-nft-udp          UDP TPROXY table (QUIC excluded) + policy routing
router/etc/init.d/russia-inside-update  Weekly domain-list updater (itdoginfo/allow-domains)
router/etc/sysctl.d/99-tcp-fastopen.conf
vps/etc/xray/config.json                Xray on the VPS
vps/etc/systemd/system/xray.service     Standard xray-core unit
vps/etc/hysteria/config.yaml            hysteria2 server
vps/etc/ufw/before.rules.nat            REDIRECT 520-540 -> 123 for port hopping
vps/etc/sysctl.d/99-tcp-fastopen.conf
docs/troubleshooting.en.md              Problems hit along the way and how they were diagnosed
docs/commands.md                        Command cheat sheet
```

## Setup

### VPS

1. Install xray-core with the official script from [XTLS/Xray-install](https://github.com/XTLS/Xray-install).
2. Generate Reality keys (`xray x25519`) and a UUID (`xray uuid`), pick a `shortId` (`openssl rand -hex 8`) and a masquerade domain that serves TLS 1.3 and is reachable from the VPS. `www.idealo.de` is used here, verify yours with `openssl s_client`.
3. Copy `vps/etc/xray/config.json`, fill in `YOUR_UUID`, `YOUR_REALITY_PRIVATE_KEY`, `YOUR_SHORT_ID`. Copy `vps/etc/systemd/system/xray.service`, `systemctl daemon-reload && systemctl enable --now xray`.
4. Install hysteria2 with the official script (`bash <(curl -fsSL https://get.hy2.sh/)`), put `vps/etc/hysteria/config.yaml` at `/etc/hysteria/config.yaml`, fill in domain, email, and password, `systemctl enable --now hysteria-server`. The domain must point at the VPS.
5. Firewall: deny inbound by default and open only what's needed.
   ```sh
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow 22/tcp
   ufw allow 443/tcp
   ufw allow 123/udp
   ufw allow 80/tcp      # ACME for the hysteria certificate
   ```
6. Port hopping: prepend `vps/etc/ufw/before.rules.nat` to `/etc/ufw/before.rules` (replace `ens1` with your interface from `ip route`), then `ufw enable` or `ufw reload`. Check with `iptables -t nat -S PREROUTING | grep REDIRECT`.
7. Copy `vps/etc/sysctl.d/99-tcp-fastopen.conf`, `sysctl -p`.

Ports Docker publishes with `-p` are not closed by ufw: Docker writes its own rules around it. If the VPS runs containers, check `ss -tulpn` for `docker-proxy`.

### Router (OpenWrt)

Tested on OpenWrt 24.10.3, ramips/mt7621 (ASUS RT-AX53U). If your LAN isn't `192.168.1.0/24`, change `LAN_NET` in the init scripts.

1. Extroot on a USB stick. The RT-AX53U has about 32 MB of overlay and sing-box doesn't fit. Install `kmod-usb-storage block-mount kmod-fs-ext4 e2fsprogs` and follow the [official guide](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration). Pitfalls with a stick that still holds an old extroot are in `docs/troubleshooting.en.md`.
2. Install xray-core for the router's architecture, `kmod-nft-tproxy`, `dnsmasq-full` (stock `dnsmasq` lacks `nftset`, remove it first), and `sing-box`.
3. Copy `router/etc/xray/config.json`, fill in `YOUR_VPS_IP`, `YOUR_UUID`, `YOUR_REALITY_PUBLIC_KEY`, `YOUR_SHORT_ID` (matching the VPS).
4. Copy `router/etc/sing-box/config.json` to `/etc/sing-box/config.json`, fill in the hysteria IP, password, and domain. Check and enable:
   ```sh
   sing-box check -c /etc/sing-box/config.json
   uci set sing-box.main.enabled='1'
   uci set sing-box.main.conffile='/etc/sing-box/config.json'
   uci commit sing-box
   /etc/init.d/sing-box enable && /etc/init.d/sing-box restart
   curl -s --socks5-hostname 127.0.0.1:1080 https://ifconfig.me   # should print the VPS IP
   ```
5. Point dnsmasq at the domain-list directory: `uci set dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'; uci commit dhcp`.
6. Copy `router/etc/init.d/xray-nft`, `xray-nft-udp`, `russia-inside-update` into `/etc/init.d/`, `chmod +x`, `enable` all three.
7. fw4 rule that rejects QUIC from the LAN (without it browsers wait for a timeout before falling back to TCP):
   ```sh
   uci add firewall rule
   uci set firewall.@rule[-1].name='Reject-QUIC'
   uci set firewall.@rule[-1].src='lan'
   uci set firewall.@rule[-1].dest='wan'
   uci set firewall.@rule[-1].proto='udp'
   uci set firewall.@rule[-1].dest_port='443'
   uci set firewall.@rule[-1].target='REJECT'
   uci commit firewall
   /etc/init.d/firewall reload
   ```
8. Copy `router/etc/sysctl.d/99-tcp-fastopen.conf`, `sysctl -p`.
9. First run: `/etc/init.d/russia-inside-update restart`, then `/etc/init.d/xray-nft restart`, then Xray and `xray-nft-udp`.
10. Cron (`/etc/crontabs/root`, then `/etc/init.d/cron restart`):
    ```
    30 7 * * 1 /etc/init.d/russia-inside-update restart >/tmp/russia-inside-update.log 2>&1
    * * * * * ping -c3 -W2 YOUR_VPS_IP >/dev/null 2>&1 || date >> /root/vps-down.log
    ```
    The second line logs the minutes when the VPS didn't answer and keeps the address "warm" with pings. Why, see `docs/troubleshooting.en.md`.
11. Reboot and check that everything comes back on its own: `nft list tables` (should list `xray` and `xray-udp`), `/etc/init.d/xray status`, `/etc/init.d/sing-box status`, the `curl` through SOCKS from step 4.

From a Windows PC behind the router, the UDP path can be checked with NTP: `w32tm /stripchart /computer:time.cloudflare.com /samples:3 /dataonly`. If time samples come back, packets went TPROXY → Xray → sing-box → hysteria2.

## Unsolved

- Periodic total outages to the VPS (all protocols, ICMP included, for a few minutes). The cron log is there to find out whether they're periodic.
- All LAN UDP exits in Amsterdam, including calls from Russian services. If one of them misbehaves, its subnets can be excluded with an `ip daddr ... return` rule in the `xray-udp` chain.

## Not included

- `russia-inside.conf` for dnsmasq: `russia-inside-update` downloads and generates it, it's not hand-edited or committed.
- Real keys, passwords, IPs, and domains. The repo only has `YOUR_*` placeholders.
