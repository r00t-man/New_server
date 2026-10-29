# 🖥️ New server — пошаговая настройка чистого сервера (Ubuntu 24.04 LTS)

[![OS Linux](https://img.shields.io/badge/OS-Linux-blue?logo=linux&logoColor=white)](https://www.linux.org/)
[![Tested on](https://img.shields.io/badge/tested%20on-Ubuntu%2024.04-orange?style=flat-square)]()
[![Docker](https://img.shields.io/badge/Docker-optional-blue?logo=docker)](https://www.docker.com/)

Копипаст-гайд: **что делать на только что купленном VPS**, по порядку, команда за командой.
Собрано из опыта реального парка VPN-нод (основной источник — раздел `Server_Security` и `wiki`
репозитория [r00t-man/MZT](https://github.com/r00t-man/MZT)), плюс то, чего там не было, но ставить
стоит (Docker CE из официального репозитория, logrotate-гигиена, лимит nofile под VPN-нагрузку).

Типовой сценарий использования сервера — **VPN-нода Remnawave** (Xray в Docker), но нода ставится
не всегда: поэтому Docker выделен в отдельный факультативный шаг, который «помешает редко, а
пригодится часто». Всё остальное — базовая безопасность — обязательно на **любом** сервере.

> [!IMPORTANT]
> Целевая ОС — **Ubuntu 24.04 LTS**. Это важно: в 24.04 OpenSSH перешёл на **socket-activation**
> (`ssh.socket`), и старые инструкции по смене SSH-порта из интернета на ней **приводят к потере
> доступа**. Шаг 4 разбирает это подробно — не пропускайте.

## 📋 Карта шагов

| # | Шаг | Обязательный? |
|---|-----|---------------|
| 0 | Первое подключение, разведка сервера | ✅ |
| 1 | Обновление системы + автообновления безопасности | ✅ |
| 2 | Пакеты для администрирования и диагностики | ✅ |
| 3 | Отдельный sudo-пользователь | ✅ |
| 4 | SSH: ключи, отключение пароля, смена порта (грабли 24.04!) | ✅ |
| 5 | Fail2ban (sshd + recidive + portscan) | ✅ |
| 6 | Firewall nftables + open-list портов | ✅ |
| 7 | sudo — проверка CVE-2025-32463 | ✅ |
| 8 | sysctl: базовая гигиена + BBR/fq | ✅ |
| 9 | Время (UTC) и DNS (DoT) | ✅ |
| 10 | Swap (если RAM ≤ 2 ГБ) | ✅ |
| 11 | ulimits / лимиты файловых дескрипторов | ✅ |
| 12 | Docker Engine + compose plugin | 🔶 опционально, но рекомендуется |
| 13 | Полезные скрипты: MOTD, audit-history, dockermon, Ultra Clean | 🔶 по желанию |
| 14 | Мониторинг: node_exporter / Beszel | 🔶 по желанию |
| 15 | Remnawave-нода (если этот сервер — нода) | 🔶 если нужен VPN |
| 16 | Финальная проверка + чек-лист | ✅ |

Полезные переменные — задайте один раз в начале каждой сессии, дальше блоки можно копировать целиком:

```bash
SSH_PORT=22132          # ваш будущий нестандартный SSH-порт (конвенция: 22 + последний октет IP)
SERVER_IP="203.0.113.132"
USERNAME="admin"        # ваш sudo-пользователь
```

---

## 0. Первое подключение и разведка

Зашли как root (пароль дал хостер) — сразу смотрим, что за машина:

```bash
cat /etc/os-release && uname -r
nproc && free -h && df -h /
ip -br a
swapon --show            # пусто = свопа нет
systemctl list-units --failed
```

Если у провайдера есть **внешний firewall в панели** (Aeza, Hetzner Cloud, Oracle…) — убедитесь,
что он разрешит ваш будущий SSH-порт, иначе шаг 4 отрежет доступ на уровне хостера.

> [!TIP]
> Все долгие установки (apt, компиляция, docker build) запускайте внутри `tmux`/`screen` или через
> transient systemd-юнит, чтобы обрыв SSH не прервал процесс:
> ```bash
> systemd-run --unit=install-job --collect -- /path/to/script.sh
> journalctl -u install-job -f
> ```

---

## 1. Обновление системы + автообновления безопасности

```bash
apt update && apt full-upgrade -y && apt autoremove -y && apt autoclean -y
reboot
```

После перезагрузки:

```bash
cat /etc/os-release
uname -r
```

Автообновления безопасности (ставьте сразу — на серверах без постоянного присмотра это главный
защитный механизм):

```bash
apt install -y unattended-upgrades needrestart
dpkg-reconfigure -plow unattended-upgrades
```

Проверка, что реально включено:

```bash
systemctl status apt-daily-upgrade.timer apt-daily.timer
unattended-upgrade --dry-run --debug 2>&1 | tail   # подробный разбор — см. заметки MZT по unattended-upgrades
```

> [!NOTE]
> В Ubuntu 24.04 `needrestart` в связке с unattended-upgrades умеет сам перезапускать сервисы после
> обновления библиотек; автоперезагрузку лучше оставить ручной (см. статью
> `wiki/unattended-upgrades.md` в MZT — там полный разбор расписаний и окон).

---

## 2. Пакеты: администрирование и диагностика

```bash
apt install -y \
  curl wget ca-certificates gnupg lsb-release \
  git nano mc vim htop tmux screen jq unzip \
  lsof dos2unix \
  openssl rsync socat \
  dnsutils mtr-tiny ncdu whois \
  logrotate cron \
  fail2ban nftables
```

| Пакет | Зачем |
|---|---|
| `curl`/`wget` | скачивание установочных скриптов |
| `ca-certificates`/`gnupg`/`lsb-release` | подключение сторонних apt-репозиториев по HTTPS/GPG (нужны для Docker CE в шаге 12) |
| `git` | клонирование репозиториев |
| `nano`/`mc`/`vim` | редакторы + файловый менеджер |
| `htop` | мониторинг процессов/нагрузки |
| `tmux`/`screen` | не терять сессию при обрыве SSH |
| `jq` | разбор JSON — любые API (панели, боты, балансировка remna) |
| `lsof` | какой процесс держит порт/файл |
| `dos2unix` | чинит CRLF в скриптах, скопированных из Windows/браузера |
| `openssl` | ручная работа с сертификатами/TLS-хендшейком |
| `rsync` | синхронизация файлов между серверами |
| `socat` | проброс/отладка сетевых соединений |
| `dnsutils`/`mtr-tiny`/`whois` | диагностика DNS и сети |
| `ncdu` | «кто съел диск» |
| `logrotate` | ротация логов (в т.ч. будущих логов remnanode — см. шаг 15) |
| `fail2ban`/`nftables` | защита от брутфорса и фильтрация — шаги 5–6 |

Опционально, если планируете тестировать канал до загрузки трафиком (каскадный VPN между двумя
серверами): `iperf3`.

---

## 3. Отдельный sudo-пользователь

Не работайте на сервере постоянно под `root`:

```bash
adduser $USERNAME
usermod -aG sudo $USERNAME
id $USERNAME
```

Проверка:

```bash
su - $USERNAME
sudo -v
```

Дальше все действия — от этого пользователя (или осознанно от root, если так привычнее для нод).

---

## 4. SSH — ключи, отказ от пароля, смена порта

> [!CAUTION]
> Самый опасный шаг. Не закрывайте текущую сессию, пока новый вход не проверен из **отдельного**
> окна. Если у провайдера есть VNC/консоль восстановления — знайте заранее, как в неё зайти.

### 4.1 Ключ

На своём ПК (если ключа ещё нет): `ssh-keygen -t ed25519 -C "my-server"`. Приватный ключ никогда
не попадает на сервер. На сервере (под нужным пользователем) кладём **публичный**:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/authorized_keys <<'EOF'
ssh-ed25519 AAAA... ваш_публичный_ключ
EOF
chmod 600 ~/.ssh/authorized_keys
```

Из другого окна проверяем, что ключ принят (пока порт 22 и пароль ещё работают):

```bash
ssh -o PreferredAuthentications=publickey $USERNAME@$SERVER_IP
```

### 4.2 Каталог privilege separation (обязательно до проверки конфига)

```bash
mkdir -p /run/sshd && chmod 0755 /run/sshd
```

Без него `sshd -t` падает с `Missing privilege separation directory: /run/sshd`, а вместе с ним
падает `ExecStartPre` юнита ssh — сервис не стартует вообще.

### 4.3 Конфиг sshd — только drop-in, не сам `sshd_config`

Имя `00-a-…` — чтобы файл сортировался **раньше** `50-cloud-init.conf`: у sshd действует правило
«**первая** встреченная директива побеждает», и cloud-init с `PasswordAuthentication yes` иначе
перебьёт ваши настройки.

```bash
cat > /etc/ssh/sshd_config.d/00-a-hardening.conf <<EOF
Port ${SSH_PORT}
Port 22
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
EOF

sshd -t && echo "=== config OK ==="
```

Вторая строка `Port 22` — временная страховка на время перехода. **Если `sshd -t` вывел любую
ошибку — остановитесь и почините файл, не делайте restart.**

> [!WARNING]
> Раскомментируйте/пропишите `Port ${SSH_PORT}` **числом** (переменная в heredoc подставится, если
> задавали её в этой же сессии). Убедитесь, что в файле именно число, а не слово `SSH_PORT`.

### 4.4 Применить порт — ОДИН из двух вариантов

В Ubuntu 24.04 слушающим сокетом SSH по умолчанию управляет **не sshd, а systemd** (`ssh.socket`).
Просто `systemctl restart ssh` новый порт **не подхватывает**.

**Вариант A — отключить socket-activation (надёжно везде):**

```bash
systemctl disable --now ssh.socket
systemctl unmask ssh.service 2>/dev/null || true
systemctl enable --now ssh.service
systemctl restart ssh.service
```

**Вариант B — оставить socket-activation (дефолт 24.04.1+):** работает, если в системе есть
генератор `sshd-socket-generator`, который при `daemon-reload` сам читает `Port` из `sshd_config.d/`
и настраивает dual-stack сокет:

```bash
test -x /usr/lib/systemd/system-generators/sshd-socket-generator \
  && { systemctl daemon-reload && systemctl restart ssh.socket; echo "socket-activation: OK"; } \
  || echo "!!! генератора НЕТ — используйте Вариант A"
```

> [!CAUTION]
> **Никогда не создавайте и не редактируйте `/etc/systemd/system/ssh.socket.d/*` вручную.** Голый
> `ListenStream=<порт>` под унаследованную `BindIPv6Only=ipv6-only` поднимает только IPv6-сокет;
> `ss -tlnp` это не показывает, а снаружи — `Connection refused`. Реальный инцидент закончился
> переустановкой ОС.

### 4.5 Проверка (до закрытия первой сессии!)

```bash
sshd -T | grep -E '^(port|passwordauthentication|permitrootlogin|pubkeyauthentication) '
ss -tlnp | grep -E ":($SSH_PORT|22)\b"
systemctl is-active ssh.service
```

Ожидаем: оба порта, `passwordauthentication no`, сервис `active`. Затем из **нового** окна:

```bash
ssh -p $SSH_PORT $USERNAME@$SERVER_IP
```

Зашло по ключу — первая сессия ещё жива, пароль уже отключён. Теперь можно открыть новый порт в
firewall (шаг 6) и убрать страховку:

```bash
sed -i '/^Port 22$/d' /etc/ssh/sshd_config.d/00-a-hardening.conf
sshd -t && systemctl restart ssh.service      # вариант A
# или:  sshd -t && systemctl daemon-reload && systemctl restart ssh.socket   # вариант B
```

---

## 5. Fail2ban

### 5.1 Базовый jail `sshd` (+ `recidive`)

```bash
cat > /etc/fail2ban/jail.local <<EOF
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled  = true
port     = $SSH_PORT
filter   = sshd
backend  = systemd
findtime = 600
maxretry = 3
bantime  = 1h
action   = nftables[name=SSH, port=$SSH_PORT, protocol=tcp]

[recidive]
enabled  = true
filter   = recidive
logpath  = /var/log/fail2ban.log
findtime = 86400
maxretry = 3
bantime  = -1
action   = nftables[name=RECIDIVE, port=$SSH_PORT, protocol=tcp]
EOF

grep -E 'port|action' /etc/fail2ban/jail.local    # убедиться, что везде ЧИСЛА, не SSH_PORT

systemctl enable --now fail2ban
fail2ban-client status sshd
```

> [!CAUTION]
> В `action` — **явный номер порта**, а не алиас `port=ssh`: алиас резолвится через `/etc/services`
> в жёсткий 22, и бан молча не срабатывает (так «работали» правила на 9 из 14 нод реального парка).
> После смены порта не забудьте обновить `jail.local` и `systemctl restart fail2ban`.

Быстрая проверка, что бан применяется (правило создаётся лениво, при первом бане):

```bash
fail2ban-client set sshd banip 203.0.113.1
nft list ruleset | grep -A3 'set addr-set-sshd'   # должен быть ВАШ порт
fail2ban-client set sshd unbanip 203.0.113.1
```

### 5.2 Ловушка `portscan` — бан сканеров редких портов

Кто постучался в FTP/RDP/СУБД — на VPN-сервере таких портов нет в принципе → бан по всем портам
на сутки. Три файла:

`/etc/nftables-portscan.conf`:

```nft
#!/usr/sbin/nft -f
table inet portscan
delete table inet portscan

table inet portscan {
    chain input {
        type filter hook input priority filter - 5; policy accept;
        iifname "lo" return
        tcp dport { 21, 23, 25, 110, 143, 445, 1433, 1434, 3306, 3389,
                    5432, 5900, 5901, 6379, 7001, 8081, 8888, 9200,
                    11211, 27017 } ct state new limit rate 4/minute \
                    log prefix "PORTSCAN: " drop
    }
}
```

`/etc/systemd/system/nft-portscan-trap.service`:

```ini
[Unit]
Description=Load portscan trap nftables rules
After=network.target fail2ban.service
Wants=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/nft -f /etc/nftables-portscan.conf
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

`/etc/fail2ban/filter.d/portscan.conf`:

```ini
[Definition]
failregex = ^.*PORTSCAN: .*SRC=<HOST> .*$
ignoreregex =
```

`/etc/fail2ban/jail.d/portscan.conf`:

```ini
[portscan]
enabled = true
filter = portscan
backend = systemd
journalmatch = _TRANSPORT=kernel
findtime = 3600
maxretry = 1
bantime = 86400
banaction = %(banaction_allports)s
```

Применение:

```bash
systemctl daemon-reload
systemctl enable --now nft-portscan-trap.service
systemctl restart fail2ban
fail2ban-client status          # sshd, recidive, portscan
```

> [!WARNING]
> Перед включением сверьте список trap-портов с реальными слушающими:
> `ss -tlnp | grep -E ':(21|23|25|110|143|445|1433|1434|3306|3389|5432|5900|5901|6379|7001|8081|8888|9200|11211|27017)\b'`
> Порт 8080 в список **не** включать — на нём легитимно живут панели/боты/XHTTP-инбаунды.
> `banaction = %(banaction_allports)s` — именно banaction (шаблона `action_allports` нет, fail2ban упадёт).
> `backend = systemd` обязателен: минимальные образы Ubuntu идут без rsyslog, journald-лог ядра — единственный источник.

---

## 6. Firewall (nftables)

> [!IMPORTANT]
> Главный факт продакшена: на ноде **нет единого «файла файрвола»**. `nft list ruleset` покажет
> несколько независимых таблиц от разных владельцев: `inet f2b-table` (fail2ban), `inet portscan`
> (наша ловушка), `ip nat`/`ip filter` (**Docker через iptables-nft — никогда не трогать**),
> `ip remnanode`/`ip6 remnanode6` (сам контейнер Xray — тоже не трогать). Свои правила — своя
> таблица, свой oneshot-юнит; штатный `nftables.service` остаётся disabled.

Сначала выпишите **реальные** слушающие порты — allow-list строится по факту, а не по примеру из
гайда:

```bash
ss -tulpn
```

Для чистой ноды обычно достаточно SSH + 443/tcp + 443/udp (для VPN-инбаундов добавите свои).
`/etc/nftables-fw.conf`:

```nft
#!/usr/sbin/nft -f
table inet fw
delete table inet fw

table inet fw {
    chain input {
        type filter hook input priority 0; policy drop;

        iif lo accept
        ct state established,related accept
        ct state invalid drop

        ip6 nexthdr icmpv6 accept
        ip protocol icmp accept          # НЕ блокировать ICMP — ломает мониторинг доступности

        tcp dport { 22132, 80, 443 } accept   # ← ваш SSH-порт числом + то, что реально слушаете
        udp dport { 443 } accept              # VPN-порты (Hysteria2/WireGuard/XHTTP-over-QUIC)
    }
    chain forward { type filter hook forward priority 0; policy drop; }
    chain output  { type filter hook output priority 0; policy accept; }
}
```

`/etc/systemd/system/nft-fw.service`:

```ini
[Unit]
Description=Custom nftables firewall
After=network-pre.target
Before=fail2ban.service

[Service]
Type=oneshot
ExecStart=/usr/sbin/nft -f /etc/nftables-fw.conf
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now nft-fw.service
nft list tables
```

> [!CAUTION]
> 1. **Откройте новый SSH-порт в allow-list ДО применения** `policy drop` — и сначала проверьте его
>    в отдельном окне, чем чинить недоступность через консоль хостера.
> 2. Если у провайдера внешний firewall — продублируйте правила и там.
> 3. После включения Docker (шаг 12) **перепроверьте проброс портов контейнеров** (`curl` на
>    опубликованный порт извне): forward-политика `drop` может мешать NAT-трафику Docker. Если
>    отвалилось — не подгоняйте правила Docker, а добавьте в свою цепочку `forward` разрешение
>    для `docker0`/bridge-сетей или снимите drop с forward (для VPN-ноды без маршрутизации
>    клиентов форвард обычно не нужен вовсе).
> 4. Если ставите `node_exporter` (шаг 14) — порт 9100 наружу не открывать, закрывать allow-list
>    по IP Prometheus (разбор — в MZT, раздел «Firewall»).

---

## 7. sudo — проверить CVE-2025-32463

```bash
apt-get changelog sudo | grep -B15 'CVE-2025-32463' | grep -E '^sudo \('
apt-cache policy sudo
```

Если версия в changelog **≤** установленной/candidate — патч уже есть (на актуальной 24.04 из
`noble-security` он давно бэкпортирован), делать ничего не нужно. `sudo --version` смотреть
бессмысленно — Debian/Ubuntu не меняют видимый апстрим-номер при бэкпорте.

Если патча вдруг нет — компиляция из исходников возможна, но помнит: `make install` кладёт бинарь
в `/usr/local/bin/sudo`, перекрывающий apt-версию, и **будущие CVE перестанут закрываться через
apt** (полный разбор — MZT, раздел «sudo — CVE-2025-32463»).

---

## 8. sysctl: базовая гигиена + BBR

### 8.1 Гигиена (обязательно)

```bash
cat > /etc/sysctl.d/98-hygiene.conf <<'EOF'
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
EOF
sysctl --system
```

`rp_filter = 2` (loose), а не strict: на Xray-ноде разницы нет, а при WireGuard-каскаде со
статическим routing strict-режим молча роняет асимметричные пакеты.

### 8.2 BBR + fq — профиль под объём RAM (для VPN-трафика)

```bash
modprobe tcp_bbr
echo tcp_bbr > /etc/modules-load.d/bbr.conf
free -h   # сначала смотрим объём RAM — профиль подбирается под него
```

Small (~1 ГБ RAM, большинство бюджетных VPS) — `/etc/sysctl.d/99-vpn-tuning.conf`:

```ini
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

net.core.rmem_max = 8388608
net.core.wmem_max = 8388608
net.ipv4.tcp_rmem = 4096 131072 8388608
net.ipv4.tcp_wmem = 4096 131072 8388608
net.ipv4.tcp_mem  = 131072 262144 393216

net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1

net.core.somaxconn = 4096
net.core.netdev_max_backlog = 4096
net.ipv4.tcp_max_syn_backlog = 4096
```

Medium (~2 ГБ) — буферы `16777216`, `tcp_mem = 262144 524288 786432`. Не копировать «VPN sysctl»
на несколько гигабайт на VPS с 1 ГБ — это прямой путь к OOM.

```bash
sysctl --system
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
```

> [!CAUTION]
> Если на этом сервере будет **MTProxy/Telegram-прокси** — BBR + большие буферы ломают
> MTProto-хендшейк за DPI. Для такой ноды добавить переопределение, которое грузится последним по
> алфавиту — `/etc/sysctl.d/99-zz-mtproxy.conf`:
> ```ini
> net.core.default_qdisc = fq_codel
> net.ipv4.tcp_congestion_control = cubic
> ```

Чего **не** трогать: `icmp_echo_ignore_all` (ломает мониторинг, от атак не защищает), массовое
отключение IPv6 через GRUB (может сломать провижининг хостера), `tcp_window_scaling/sack/timestamps`
(уже включены 15 лет).

---

## 9. Время и DNS

```bash
timedatectl
timedatectl set-timezone UTC
resolvectl query github.com
```

Точное время критично для TLS. Шифрование исходящего DNS сервера (DoT) — drop-in
`systemd-resolved`, без удаления системных файлов:

`/etc/systemd/resolved.conf.d/10-dot.conf`:

```ini
[Resolve]
DNS=1.1.1.1#one.one.one.one 1.0.0.1#one.one.one.one
FallbackDNS=9.9.9.9#dns.quad9.net
DNSOverTLS=yes
DNSSEC=no
```

```bash
systemctl restart systemd-resolved
resolvectl status    # "encrypted transport: yes"
```

> [!NOTE]
> Для российских серверов альтернатива — Яндекс DNS DoT (`77.88.8.8#ycdn.ru` / `77.88.8.1#ycdn.ru`)
> или готовый скрипт `ya-dns-dot.sh` из MZT. Если DHCP/cloud-init перебивает конфиг — добавить в
> netplan `dhcp4-overrides: {use-dns: false}` и `network: {config: disabled}` в
> `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` (не удаляя существующие файлы).

---

## 10. Swap (если RAM ≤ 2 ГБ и свопа нет)

```bash
free -h
```

Если swap = 0:

```bash
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
swapon --show
```

> [!NOTE]
> На некоторых образах Ubuntu 24.04 уже крутится **zram** (`swapon --show` покажет `/dev/zram0`) —
> это нормально, zram и swapfile могут жить вместе. Полная диагностика «почему своп не меняется» —
> в MZT, статья «Базовые команды Ubuntu 24 для подготовки VPN-ноды», раздел 7.2.

---

## 11. Лимиты файловых дескрипторов (под VPN/Docker-нагрузку)

Дефолтный `nofile` мал для сотен одновременных клиентов; в стеке Remnawave лимиты задаются и в
compose (`ulimits.nofile: 1048576`), но на хосте их стоит поднять глобально:

```bash
cat >> /etc/security/limits.d/99-nofile.conf <<'EOF'
* soft nofile 1048576
* hard nofile 1048576
root soft nofile 1048576
root hard nofile 1048576
EOF

# для сервисов, запускаемых systemd (например dockerd):
mkdir -p /etc/systemd/system/docker.service.d
cat > /etc/systemd/system/docker.service.d/limits.conf <<'EOF'
[Service]
LimitNOFILE=1048576
EOF
systemctl daemon-reload
```

Проверка после ребута/рестарта сервиса: `ulimit -n`, `systemctl show docker -p LimitNOFILE`.

---

## 12. Docker Engine (опционально, но рекомендуется)

Даже если Remnawave-нода здесь не планируется, Docker удобно иметь сразу (боты, мониторинг,
разовые задачи). Ставим **Docker CE из официального репозитория**, а не `docker.io` из universe:
свежее версии, предсказуемые апгрейды, тот же подход, что в скриптах MZT (`get.docker.com`).

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  > /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io \
               docker-buildx-plugin docker-compose-plugin

systemctl enable --now docker
docker --version && docker compose version
docker run --rm hello-world
```

Доступ без sudo (если работаете под `$USERNAME`):

```bash
usermod -aG docker $USERNAME
newgrp docker   # или перезайти по SSH
```

Ротация логов контейнеров (json-логи растут бесконтрольно — на нагруженных нодах гоняйте и
`Ultra_Clean_VPS.sh` из MZT):

```bash
cat > /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
systemctl restart docker
```

> [!WARNING]
> Docker пишет свои NAT-правила через `iptables-nft` в таблицы `ip nat`/`ip filter` — не удаляйте
> пакет `iptables` и не переключайте `update-alternatives` на legacy-версию: сломаете проброс
> портов контейнеров и совместимость с nftables-правилами fail2ban. После включения Docker
> перепроверьте firewall из шага 6 (пункт 3 caution).

---

## 13. Полезные скрипты из MZT (по желанию)

Установка одной командой, все лежат в [r00t-man/MZT/files](https://github.com/r00t-man/MZT/tree/main/files):

| Скрипт | Что делает | Команда |
|---|---|---|
| 🖥️ MOTD | красивая сводка сервера при входе по SSH | `bash <(curl -Ls https://raw.githubusercontent.com/r00t-man/MZT/main/files/server-stat-modt.sh)` |
| 🔎 Audit History | логирование всех команд (auditd + bash-hook + IP SSH) | `bash <(curl -fsSL https://raw.githubusercontent.com/r00t-man/MZT/refs/heads/main/files/audit-history.sh)` |
| 🐳 Dockermon | интерактивное меню управления Docker | `bash <(curl -Ls https://raw.githubusercontent.com/r00t-man/MZT/main/files/dockermon.sh)` |
| 🧹 Ultra Clean VPS | глубокая очистка (APT/journald/nginx/Docker-логи, 1–8 ГБ места) | `bash <(curl -Ls https://raw.githubusercontent.com/r00t-man/MZT/main/files/Ultra_Clean_VPS.sh)` |

Рекомендую на новом сервере сразу: **MOTD** и **Audit History**; Dockermon — если ставили Docker;
Ultra Clean — держать в памяти как средство «разъесть диск» позже.

---

## 14. Мониторинг (по желанию)

**Вариант 1 — лёгкий Beszel** (Hub + Agent через Docker, минимум усилий): конспект — в MZT,
статья «Мониторинг Beszel — быстрый старт».

**Вариант 2 — Prometheus + node_exporter** (если есть центральный сервер мониторинга):

```bash
bash <(curl -Ls https://raw.githubusercontent.com/r00t-man/MZT/main/files/install_node_exporter_agent.sh)
```

> [!CAUTION]
> `node_exporter` слушает **9100 для кого угодно в интернете** и отдаёт CPU/RAM/диски/список ФС.
> Наружу в firewall его не открывать — закрывать allow-list по IP скрейпера (готовая таблица
> `exporter_guard` — в MZT, раздел «Firewall»). И не забыть: после смены SSH-порта обновить
> таргеты внешнего мониторинга (blackbox_ssh и т.п.) — они статичны и сами не синхронизируются.

---

## 15. Remnawave-нода (если этот сервер — VPN-нода)

Полный деплой стека (панель + бот + кабинет + nginx) — большая отдельная тема, см. MZT:
[`Remnawave/VPN-help-remna-full.md`](https://github.com/r00t-man/MZT/blob/main/Remnawave/VPN-help-remna-full.md).
Здесь — только специфичные для **новой ноды** пункты:

1. Docker уже готов (шаг 12). Каталоги:

```bash
mkdir -p /opt/remnanode /var/log/remnanode && chmod 755 /var/log/remnanode
```

2. `/opt/remnanode/docker-compose.yml` (стандартная нода; `NODE_PORT` — API-порт агента ноды,
   `SECRET_KEY` — из панели; логи монтируем сразу — они нужны TrafGuard и отладке):

```yaml
services:
  remnanode:
    container_name: remnanode
    hostname: remnanode
    image: remnawave/node:latest
    network_mode: host
    restart: always
    cap_add:
      - NET_ADMIN
    ulimits:
      nofile:
        soft: 1048576
        hard: 1048576
    environment:
      - NODE_PORT=2222
      - SECRET_KEY="..."
    volumes:
      - /var/log/remnanode:/var/log/remnanode
```

```bash
cd /opt/remnanode && docker compose up -d
docker logs --tail=100 remnanode
```

3. В панели Remna добавить ноду (IP, порт `NODE_PORT`, секрет). Инбаунды (VLESS/REALITY/XHTTP/
   Hysteria2) создавать в панели; **каждый новый порт инбаунда — в allow-list firewall шага 6**.
4. Логирование Xray — добавить в JSON-профиль ноды:

```json
"log": {
  "error": "/var/log/remnanode/error.log",
  "access": "/var/log/remnanode/access.log",
  "loglevel": "warning"
}
```

5. Ротация логов ноды:

```bash
cat > /etc/logrotate.d/remnanode <<'EOF'
/var/log/remnanode/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
EOF
logrotate -d /etc/logrotate.d/remnanode
```

6. Дальше по задаче: WARP для ноды (`files/warp-remnanode.sh`), TrafGuard/mobile443-filter
   (`Remnawave/Install TaffGuard nftables.md`), обновление панели и нод
   (`Remnawave/Remnawave — обновление панели и нод.md`) — всё в MZT.

---

## 16. Финальная проверка

```bash
ip -br a
ip r
ss -tulpen
sudo systemctl --failed
sudo journalctl -p err -b --no-pager

sshd -T | grep -E '^(port|passwordauthentication|permitrootlogin) '
fail2ban-client status
nft list tables
docker info --format '{{.ServerVersion}}' 2>/dev/null
sysctl net.ipv4.tcp_congestion_control net.core.default_qdisc
swapon --show
```

### ✅ Чек-лист

- [ ] Система обновлена, `unattended-upgrades` включён.
- [ ] Поставлены пакеты администрирования/диагностики.
- [ ] Создан sudo-пользователь, root-логин по паролю больше не используется.
- [ ] SSH: только ключи (`sshd -T | grep passwordauthentication` → `no`), вход по ключу проверен
      из новой сессии с другой машины, порт сменён корректным для 24.04 способом (A или B),
      страховочный `Port 22` убран.
- [ ] Fail2ban: `sshd` + `recidive` с **явным номером порта** в `action`; `portscan`-ловушка
      активна (или осознанно пропущена после сверки trap-портов).
- [ ] Firewall nftables: allow-list построен по реальным `ss -tulpn`, новый SSH-порт открыт,
      ICMP не заблокирован, таблицы Docker/fail2ban/remnanode не тронуты.
- [ ] `sudo` проверен на CVE-2025-32463 (changelog, не `--version`).
- [ ] sysctl: гигиена применена; BBR/fq по профилю RAM (или cubic для MTProxy-ноды).
- [ ] Время UTC, DNS шифруется (DoT) или осознанно нет.
- [ ] Swap есть (если RAM ≤ 2 ГБ).
- [ ] `nofile` поднят глобально и для dockerd.
- [ ] Docker CE + compose установлены (или осознанно не нужны); json-логи контейнеров ограничены.
- [ ] Мониторинг/MOTD/audit подключены (по желанию) — таргеты мониторинга учитывают новый SSH-порт.
- [ ] (Для ноды) remnanode поднята, добавлена в панель, логи пишутся и ротируются, порты инбаундов
      открыты в firewall.

---

*Собрано на основе [r00t-man/MZT](https://github.com/r00t-man/MZT) (разделы `Server_Security`,
`wiki`, `Remnawave`, `files`). Добавлено сверх MZT: установка Docker CE из официального репозитория
(MZT использовал `docker.io`/`get.docker.com` только внутри скриптов), лимиты `nofile`, ротация
Docker-логов, `needrestart`, явная привязка порядка «SSH-порт → firewall → fail2ban → мониторинг».*
