# Router

/etc/init.d/xray restart
/etc/init.d/xray-nft restart
/etc/init.d/xray-nft-udp restart
/etc/init.d/russia-inside-update restart

/etc/init.d/xray status
logread -e xray

xray run -test -config /etc/xray/config.json

nft list ruleset
nft list set inet xray telegram4

ip rule
ip route show table 100

tcpdump -ni any udp

# VPS

systemctl status xray
systemctl restart xray
journalctl -u xray -f

xray run -test -config /etc/xray/config.json
ss -lntp | grep :443
