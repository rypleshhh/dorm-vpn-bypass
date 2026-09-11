# Dorm VPN Bypass — Selective Routing over VLESS+Reality (UDP included)

Personal setup to get around dormitory DPI filtering that blocks **all UDP**
and does deep packet inspection on TCP. Runs on an OpenWrt router doing
selective routing, backed by a single VPS running Xray-core.

Not a turnkey installer — this is a real, working, but personal
configuration, published as a reference. You will need to adapt IPs,
domains, and LAN subnet to your own setup.

## Why this exists

- The dorm network blocks UDP entirely (no QUIC, no WireGuard, no
  hysteria2 — none of it gets through).
- TCP/443 is also DPI-filtered, so plain traffic gets blocked/throttled
  by SNI or protocol fingerprint.
- The goal: only route specific traffic (Russian-market-blocked sites,
  Telegram, and now UDP-based apps like Steam/Discord) through a VPS,
  while everything else goes direct — with minimal added latency and
  without touching every device individually (do it once, at the router).

## Architecture

```
LAN client
  |
  v
dnsmasq resolves domain --> nft set (russia4 / telegram4)
  |
  v
nftables prerouting:
  - TCP/443 matching those sets --> REDIRECT --> Xray dokodemo-door :12345
  - ALL UDP (except port 53)    --> TPROXY   --> Xray dokodemo-door :12346
  |
  v
Xray (router)
  - TCP path  --> VLESS+Reality outbound "proxy"     (mux enabled)
  - UDP path  --> VLESS+Reality outbound "proxy-udp" (no mux — see below)
  |
  v
VPS (Xray inbound, VLESS+Reality on TCP/443)
  |
  v
Internet
```

Both outbounds point at the same VPS/port — Xray just opens two
independent TCP+Reality tunnels to it.

### Why TCP and UDP have separate Xray inbounds

Originally there was one `dokodemo-door` inbound handling both. Enabling
`sockopt.tproxy` on it (required for UDP) broke `SO_ORIGINAL_DST` lookup
for the TCP path, because REDIRECT (DNAT) and TPROXY are different kernel
mechanisms that don't mix on the same socket. Symptom: Xray "tunneled"
requests to itself instead of the real destination. Fix: two inbounds on
two ports, TCP keeps REDIRECT, UDP gets its own TPROXY-based inbound.

### Why TCP and UDP have separate outbounds (and only one has mux)

`mux.cool` multiplexes many logical streams over one physical TCP
connection to the VPS — great for cutting repeated TLS/Reality handshake
overhead on web browsing (see below). But it also means a single lost
packet head-of-line-blocks *every* stream sharing that connection. Fine
for a browser loading a dozen assets; bad for something like game
UDP traffic that constantly opens and closes short-lived flows. So web
(TCP) traffic goes through a muxed outbound, and UDP (games/voice) goes
through a second, un-muxed outbound to keep flows independent.

### Flow: `xtls-rprx-vision` vs `none`

`xtls-rprx-vision` gives TLS padding/splicing that helps hide XTLS traffic
patterns from advanced DPI, but it does **not** work together with
`mux.cool` (mux wraps traffic in its own protocol, which breaks vision's
raw-stream assumptions — server-side response decode fails with EOF
errors). We removed `flow` entirely (on both router and VPS — it must
match on both ends) to allow mux, trading away that specific
anti-fingerprinting property for a large latency win: first-connection
handshake time dropped from ~0.43s to ~0.11s once mux started reusing an
already-warmed connection instead of doing a fresh TCP+TLS+Reality
handshake per new domain. This is a deliberate trade-off — fine for
evading ordinary DPI blocking, not meant to resist active adversarial
probing.

### UDP tunneling (Steam/Deadlock/Discord etc.)

All LAN UDP (except DNS on port 53, which is explicitly excluded so local
resolution isn't tunneled) is caught by TPROXY and tunneled through
VLESS to the VPS, which sends it out via a normal `freedom` outbound.
No per-app/per-IP allowlisting — Steam Datagram Relay IPs are dynamic
and not worth trying to enumerate.

Known limitation: this is UDP-over-TCP by construction (VLESS itself
tunnels UDP frames inside its TCP stream), so it inherits TCP's
head-of-line blocking. A game like Deadlock that churns through many
short UDP flows (relay probing, matchmaking) will show more jitter than
a single long-lived UDP stream (e.g. a WireGuard tunnel) would, purely
because of the tunneling mechanism — not a router misconfiguration.
Also tried and ruled out as fixes:
- **TCP Fast Open** (both ends, `sockopt.tcpFastOpen`) — no measurable
  improvement, since nearly all latency lives in the Reality/TLS
  handshake, not the TCP SYN.
- **`udp-over-tcp`-style wrapping** — same fundamental limitation as
  above, not worth the extra hop.

The only real remaining lever for lower/steadier game latency would be a
VPS geographically closer to the game servers actually used, which is an
infrastructure decision, not a config one.

## Layout

```
router/etc/xray/config.json           Xray config (OpenWrt side)
router/etc/init.d/xray-nft            TCP redirect table + Telegram IP list auto-updater
router/etc/init.d/xray-nft-udp        UDP TPROXY table + policy routing, persisted across reboot
router/etc/init.d/russia-inside-update  Weekly domain-list updater (itdoginfo/allow-domains)
router/etc/sysctl.d/99-tcp-fastopen.conf
vps/etc/xray/config.json              Xray config (VPS side)
vps/etc/systemd/system/xray.service   Standard xray-core systemd unit
vps/etc/sysctl.d/99-tcp-fastopen.conf
docs/troubleshooting.md               Problems hit during setup and how they were diagnosed
```

## Setup

### VPS

1. Install xray-core (official install script from
   [github.com/XTLS/Xray-install](https://github.com/XTLS/Xray-install)).
2. Generate a Reality keypair: `xray x25519` (gives you a private+public
   key pair) and a UUID: `xray uuid`.
3. Pick a `shortId` (any short hex string, e.g. `openssl rand -hex 8 | cut -c1-16`)
   and a masquerade domain that actually serves TLS 1.3 and isn't
   geo-blocked for your VPS's IP (`www.idealo.de` was used here, pick
   your own and verify with `openssl s_client`).
4. Copy `vps/etc/xray/config.json`, fill in `YOUR_UUID`,
   `YOUR_REALITY_PRIVATE_KEY`, `YOUR_SHORT_ID`.
5. Copy `vps/etc/systemd/system/xray.service`, `systemctl daemon-reload`,
   `systemctl enable --now xray`.
6. Copy `vps/etc/sysctl.d/99-tcp-fastopen.conf`, `sysctl -p` it.
7. Open port 443 in your firewall.

### Router (OpenWrt)

Tested on OpenWrt 24.10.3, ramips/mt7621 (ASUS RT-AX53U). Adjust
`LAN_NET` in the init scripts if your LAN isn't `192.168.1.0/24`.

1. Install xray-core for your router's architecture, `kmod-nft-tproxy`,
   and `dnsmasq-full` (stock `dnsmasq` lacks `nftset` support — remove
   plain `dnsmasq` first).
2. Copy `router/etc/xray/config.json`, fill in `YOUR_VPS_IP`,
   `YOUR_UUID`, `YOUR_REALITY_PUBLIC_KEY`, `YOUR_SHORT_ID` (must match
   the VPS's values).
3. Set `confdir='/etc/dnsmasq.d'` in dnsmasq's UCI config
   (`uci set dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'; uci commit dhcp`).
4. Copy `router/etc/init.d/xray-nft`, `xray-nft-udp`, and
   `russia-inside-update` into `/etc/init.d/`, `chmod +x` them, and
   `enable` all three (`/etc/init.d/<name> enable`).
5. Copy `router/etc/sysctl.d/99-tcp-fastopen.conf`, `sysctl -p` it.
6. Run `/etc/init.d/russia-inside-update restart` once to populate the
   domain list, then `/etc/init.d/xray-nft restart` to populate the
   Telegram IP set, then start Xray and both nftables scripts.
7. Set up a weekly cron job for the domain list:
   `30 7 * * 1 /etc/init.d/russia-inside-update restart >/tmp/russia-inside-update.log 2>&1`
8. Reboot once and verify everything comes back on its own — see
   `docs/troubleshooting.md` for what to check.

## Not included

- The actual `russia-inside.conf` dnsmasq file — it's downloaded and
  generated by `russia-inside-update`, not meant to be hand-edited or
  committed.
- Real secrets (UUID, Reality keys, VPS IP) — never commit these. Keep
  your filled-in `config.json` files out of git (see `.gitignore`) or in
  a separate private location.
