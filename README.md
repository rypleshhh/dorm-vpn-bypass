# Обход фильтрации в общаге: выборочная маршрутизация на OpenWrt (TCP через VLESS+Reality, UDP через hysteria2)

Личная рабочая конфигурация для сети общежития, где UDP пропускается только на нескольких портах, а TCP фильтруется по DPI. Всё настраивается один раз на роутере OpenWrt, за ним работает один VPS. Устройства в LAN ничего не знают о туннеле.

Это не инсталлятор, а референс. IP, домены, ключи и подсеть LAN подставляются свои.

## Что за сеть

- UDP закрыт белым списком портов, и списки разные в разных сегментах одной сети. В моём случае из сегмента, где стоит WAN роутера, наружу проходили UDP 20, 21, 161, 194, 520–540, 902, 903, а из Wi-Fi-сети общаги (другой сегмент) — 123, 3478, 5001. Остальное режется молча. У тебя будут другие порты, их надо мерить (см. «Как найти открытые порты»).
- TCP наружу проходит на 443 и паре стандартных портов, TCP/443 при этом смотрит DPI.
- Связь с VPS периодически пропадает на несколько минут целиком, вместе с ICMP, и потом возвращается сама. Причину я не нашёл, подробности в `docs/troubleshooting.md`.

Цель: через VPS идут только заблокированные сайты, Telegram и UDP (игры, войс), остальное напрямую. Без настройки на каждом устройстве.

## Архитектура

```
LAN-клиент
  |
  v
dnsmasq резолвит домен --> nft set (russia4 / telegram4)
  |
  v
nftables prerouting:
  TCP/443 в этих сетах        --> REDIRECT --> Xray dokodemo-door :12345
  UDP/443 (QUIC)              --> return   --> fw4 REJECT, браузер уходит на TCP
  UDP/53                      --> мимо, DNS остаётся на роутере
  весь остальной UDP          --> TPROXY   --> Xray dokodemo-door :12346
  |
  v
Xray на роутере
  TCP: outbound "proxy"     VLESS+Reality на VPS:443, mux включён
  UDP: outbound "proxy-udp" SOCKS5 на 127.0.0.1:1080
                               |
                               v
                            sing-box: hysteria2 на VPS:520-540 (port hopping)
  |
  v
VPS
  TCP/443          --> Xray, VLESS+Reality
  UDP/520-540      --> iptables REDIRECT --> hysteria2 на :123
  |
  v
Интернет
```

Собственный трафик роутера (sing-box до VPS) идёт через OUTPUT, а TPROXY висит на prerouting и ловит только `ip saddr` из LAN, так что петли нет.

### Почему UDP идёт через hysteria2, а не через VLESS

Раньше UDP шёл вторым VLESS-outbound'ом, то есть UDP поверх TCP. Это работало, но наследовало head-of-line blocking: один потерянный TCP-сегмент задерживал все UDP-пакеты за ним, игры с кучей коротких потоков (пробинг relay, матчмейкинг) джиттерили. Когда выяснилось, что UDP закрыт не целиком, а белым списком, появилась возможность гнать UDP настоящим UDP-транспортом. hysteria2 работает поверх QUIC, потери на одном потоке не тормозят остальные.

Сам Xray 25.x на роутере не умеет hysteria2-outbound, поэтому стоит мост: Xray отдаёт UDP в локальный SOCKS5, его поднимает sing-box и уже он держит соединение hysteria2. Перехват (TPROXY) и маршрутизация остались прежними, поменялся только outbound `proxy-udp`. Откат на VLESS делается заменой одного outbound'а.

### Port hopping

На сервере hysteria слушает один порт: 123, он открыт из Wi-Fi-сети общаги, и устройства оттуда ходят на него напрямую, без роутера. Для роутера, у которого открыт другой набор портов, на VPS стоит правило REDIRECT: всё, что приходит на UDP 520–540, заворачивается на 123. sing-box на роутере ходит на случайный порт из диапазона и раз в 30 секунд меняет его (`server_ports` + `hop_interval`). Если DPI начнёт резать конкретный порт, соединение переедет на соседний.

### Почему QUIC (UDP/443) не идёт в туннель

Почти все крупные сайты отдают HTTP/3, и браузер ходит к ним по QUIC, то есть по UDP/443. Если этот UDP завернуть в туннель, через VPS пойдут YouTube, Google, всё за Cloudflare, Яндекс и ВК, включая то, что прекрасно работает напрямую. А когда туннель подвисает, браузер продолжает держать живую с его точки зрения QUIC-сессию и не откатывается на TCP, сайт просто не открывается. Поэтому UDP/443 исключён из TPROXY и отклоняется на fw4 с ICMP port unreachable. Браузер мгновенно уходит на TCP, а TCP идёт по обычной выборочной схеме.

### Почему TCP и UDP на разных inbound'ах Xray

Изначально был один `dokodemo-door` на оба протокола. Включение `sockopt.tproxy` на нём (нужно для UDP) ломало `SO_ORIGINAL_DST` для TCP: REDIRECT (DNAT) и TPROXY — разные механизмы ядра, на одном сокете они не уживаются. Xray начинал «туннелировать» запросы сам на себя. Решение: два inbound'а на двух портах, TCP остаётся на REDIRECT, UDP получает свой inbound с TPROXY.

### Mux и `flow: xtls-rprx-vision`

TCP-outbound использует `mux.cool`: много логических потоков в одном соединении до VPS, без повторного TLS+Reality-хендшейка на каждый новый домен. Время первого соединения упало примерно с 0.43 с до 0.11 с.

`xtls-rprx-vision` с mux несовместим: mux заворачивает трафик в свой протокол, vision ожидает сырой TLS, и сервер падает на разборе ответа с EOF. `flow` убран на обеих сторонах (должен совпадать). Это осознанный размен маскировки формы трафика на задержку. Для обычной DPI-блокировки хватает, против активного детекта не рассчитано.

## Как найти открытые порты

В [splicertc](https://github.com/rypleshhh/splicertc) есть `cmd/portprobe`. Сервер слушает список портов и отвечает токеном с номером порта, клиент перебирает порты и проверяет токен, так что видно и закрытые порты, и случаи, когда отвечает кто-то посередине.

```sh
# VPS
ufw allow 1:65535/udp
./portprobe -mode server -tcp none -udp 1-1024,1194,3478,4500,5000-5010,8443,19302,27015,51820

# клиент: запускать там, откуда реально пойдёт туннель
./portprobe -host YOUR_VPS_IP -tcp none -udp 1-1024,1194,3478,4500,5000-5010,8443,19302,27015,51820 -parallel 64 -timeout 2s

# VPS, после теста
ufw delete allow 1:65535/udp
```

Мерить нужно с самого роутера, а не с ПК за ним: UDP с ПК уходит в TPROXY, и тест покажет туннель, а не сеть. Под MT7621 клиент собирается так: `GOOS=linux GOARCH=mipsle GOMIPS=softfloat CGO_ENABLED=0 go build ./cmd/portprobe`. Сегменты одной сети могут отличаться, Wi-Fi и проводной порт стоит проверить отдельно.

## Структура

```
router/etc/xray/config.json             Xray на роутере
router/etc/sing-box/config.json         Клиент hysteria2, SOCKS5 на 127.0.0.1:1080
router/etc/init.d/xray-nft              Таблица TCP-редиректа + обновление IP-листа Telegram
router/etc/init.d/xray-nft-udp          Таблица UDP TPROXY (с исключением QUIC) + policy routing
router/etc/init.d/russia-inside-update  Еженедельное обновление списка доменов (itdoginfo/allow-domains)
router/etc/sysctl.d/99-tcp-fastopen.conf
vps/etc/xray/config.json                Xray на VPS
vps/etc/systemd/system/xray.service     Стандартный unit xray-core
vps/etc/hysteria/config.yaml            Сервер hysteria2
vps/etc/ufw/before.rules.nat            REDIRECT 520-540 -> 123 для port hopping
vps/etc/sysctl.d/99-tcp-fastopen.conf
docs/troubleshooting.md                 Проблемы, на которые наткнулся, и как их разбирал
docs/commands.md                        Шпаргалка команд
```

## Установка

### VPS

1. Поставить xray-core официальным скриптом с [XTLS/Xray-install](https://github.com/XTLS/Xray-install).
2. Сгенерировать ключи Reality (`xray x25519`) и UUID (`xray uuid`), придумать `shortId` (`openssl rand -hex 8`) и домен для маскировки, который отдаёт TLS 1.3 и доступен с VPS. Здесь `www.idealo.de`, проверь свой через `openssl s_client`.
3. Скопировать `vps/etc/xray/config.json`, подставить `YOUR_UUID`, `YOUR_REALITY_PRIVATE_KEY`, `YOUR_SHORT_ID`. Скопировать `vps/etc/systemd/system/xray.service`, `systemctl daemon-reload && systemctl enable --now xray`.
4. Поставить hysteria2 официальным скриптом (`bash <(curl -fsSL https://get.hy2.sh/)`), положить `vps/etc/hysteria/config.yaml` в `/etc/hysteria/config.yaml`, подставить домен, почту и пароль, `systemctl enable --now hysteria-server`. Домен должен указывать на VPS.
5. Фаервол: запретить входящее по умолчанию и открыть только нужное.
   ```sh
   ufw default deny incoming
   ufw default allow outgoing
   ufw allow 22/tcp
   ufw allow 443/tcp
   ufw allow 123/udp
   ufw allow 80/tcp      # ACME для сертификата hysteria
   ```
6. Port hopping: вставить содержимое `vps/etc/ufw/before.rules.nat` в начало `/etc/ufw/before.rules` (интерфейс `ens1` поменять на свой из `ip route`), затем `ufw enable` или `ufw reload`. Проверка: `iptables -t nat -S PREROUTING | grep REDIRECT`.
7. Скопировать `vps/etc/sysctl.d/99-tcp-fastopen.conf`, `sysctl -p`.

Порты, которые Docker публикует через `-p`, ufw не закрывает: Docker пишет свои правила в обход. Если на VPS крутятся контейнеры, проверь `ss -tulpn` на `docker-proxy`.

### Роутер (OpenWrt)

Проверено на OpenWrt 24.10.3, ramips/mt7621 (ASUS RT-AX53U). Если LAN не `192.168.1.0/24`, поправь `LAN_NET` в init-скриптах.

1. Extroot на USB-флешку. На RT-AX53U под overlay около 32 МБ, sing-box туда не влезает. Нужны `kmod-usb-storage block-mount kmod-fs-ext4 e2fsprogs`, дальше по [официальной инструкции](https://openwrt.org/docs/guide-user/additional-software/extroot_configuration). Грабли с флешкой, на которой остался старый extroot, описаны в `docs/troubleshooting.md`.
2. Поставить xray-core под архитектуру роутера, `kmod-nft-tproxy`, `dnsmasq-full` (стоковый `dnsmasq` не умеет `nftset`, его сначала удалить) и `sing-box`.
3. Скопировать `router/etc/xray/config.json`, подставить `YOUR_VPS_IP`, `YOUR_UUID`, `YOUR_REALITY_PUBLIC_KEY`, `YOUR_SHORT_ID` (совпадают с VPS).
4. Скопировать `router/etc/sing-box/config.json` в `/etc/sing-box/config.json`, подставить IP, пароль и домен hysteria. Проверить и включить:
   ```sh
   sing-box check -c /etc/sing-box/config.json
   uci set sing-box.main.enabled='1'
   uci set sing-box.main.conffile='/etc/sing-box/config.json'
   uci commit sing-box
   /etc/init.d/sing-box enable && /etc/init.d/sing-box restart
   curl -s --socks5-hostname 127.0.0.1:1080 https://ifconfig.me   # должен вернуть IP VPS
   ```
5. Прописать каталог для списка доменов: `uci set dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'; uci commit dhcp`.
6. Скопировать `router/etc/init.d/xray-nft`, `xray-nft-udp`, `russia-inside-update` в `/etc/init.d/`, `chmod +x`, `enable` все три.
7. Правило fw4, которое отклоняет QUIC из LAN (без него браузеры ждут таймаута, прежде чем уйти на TCP):
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
8. Скопировать `router/etc/sysctl.d/99-tcp-fastopen.conf`, `sysctl -p`.
9. Первый запуск: `/etc/init.d/russia-inside-update restart`, затем `/etc/init.d/xray-nft restart`, затем Xray и `xray-nft-udp`.
10. Cron (`/etc/crontabs/root`, потом `/etc/init.d/cron restart`):
    ```
    30 7 * * 1 /etc/init.d/russia-inside-update restart >/tmp/russia-inside-update.log 2>&1
    * * * * * ping -c3 -W2 YOUR_VPS_IP >/dev/null 2>&1 || date >> /root/vps-down.log
    ```
    Вторая строка пишет в лог минуты, когда VPS не отвечал, и заодно держит адрес «тёплым» пингом. Зачем, см. `docs/troubleshooting.md`.
11. Перезагрузить роутер и проверить, что всё поднялось само: `nft list tables` (должны быть `xray` и `xray-udp`), `/etc/init.d/xray status`, `/etc/init.d/sing-box status`, `curl` через SOCKS из шага 4.

С ПК за роутером UDP-путь проверяется через NTP: `w32tm /stripchart /computer:time.cloudflare.com /samples:3 /dataonly`. Пришли замеры, значит пакеты прошли TPROXY → Xray → sing-box → hysteria2.

