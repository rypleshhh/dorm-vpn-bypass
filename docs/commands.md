# Router

/etc/init.d/xray restart
/etc/init.d/xray-nft restart
/etc/init.d/xray-nft-udp restart
/etc/init.d/russia-inside-update restart
/etc/init.d/sing-box restart

/etc/init.d/xray status
/etc/init.d/sing-box status
logread -e xray
logread -e sing-box

xray run -test -c /etc/xray/config.json
sing-box check -c /etc/sing-box/config.json

# туннель hysteria2 жив: должен вернуть IP VPS
curl -s --max-time 10 --socks5-hostname 127.0.0.1:1080 https://ifconfig.me

nft list ruleset
nft list set inet xray telegram4
nft list chain inet xray-udp prerouting

ip rule
ip route show table 100

tcpdump -ni any udp
cat /root/vps-down.log
df -h /overlay

# VPS

systemctl status xray hysteria-server
systemctl restart xray
systemctl restart hysteria-server
journalctl -u xray -f
journalctl -u hysteria-server -n 50

xray run -test -config /usr/local/etc/xray/config.json
ss -lntp | grep :443
ss -ulpn | grep ':123 '
iptables -t nat -S PREROUTING | grep REDIRECT
ufw status verbose

# приходит ли что-то от роутера (подставить внешний IP общаги)
tcpdump -ni ens1 'host DORM_PUBLIC_IP and (icmp or udp portrange 520-540 or udp port 123)'

# Windows, ПК за роутером: UDP через туннель
w32tm /stripchart /computer:time.cloudflare.com /samples:3 /dataonly
