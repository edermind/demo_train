# Модуль 1. Настройка сетевой инфраструктуры

**Демонстрационный экзамен — Сетевое и системное администрирование**
*Шпаргалка: задание → готовое решение на всех устройствах*

![Топология информационно-коммуникационной системы](topology.png)

*Рисунок 1. Топология информационно-коммуникационной системы*

---

## Как пользоваться и условные обозначения

- Все команды Linux выполняются от `root` (или через `sudo`). Метки устройств: **ISP**, **HQ** (HQ-RTR / HQ-SRV / HQ-CLI), **BR** (BR-RTR / BR-SRV), **Все устройства**.
- Имена интерфейсов (`ens18`, `ens19` …) — условные. На своих ВМ выполните `ip -br a` и подставьте реальные имена.
- Документ ориентирован на Linux (Debian/Astra). Отличия для ALT Linux вынесены в Приложение А. Клиент HQ-CLI — Windows (получает настройки по DHCP).
- Решение целиком соответствует пунктам задания 1–9. DNS-записи и часовой пояс уточняйте по «таблице 3» и месту проведения экзамена.

### Допущения по интерфейсам

| Устройство | Интерфейс | Назначение |
|---|---|---|
| ISP | ens18 / ens19 / ens20 | Uplink (DHCP) / к HQ-RTR / к BR-RTR |
| HQ-RTR | ens18 / ens19 / ens20 / ens21 | к ISP / HQ-SRV-Net / HQ-CLI-Net / MGMT |
| BR-RTR | ens18 / ens19 | к ISP / BR-SRV-Net |
| HQ-SRV / BR-SRV | ens18 | ЛВС офиса |

---

## Сводная таблица IP-адресации

Расчёт по требованиям задания (п. 2). Стыки с провайдером заданы условием; внутренние сети — частные адреса RFC1918.

| Сеть | Подсеть / маска | Требование | Адресов | Шлюз / хост |
|---|---|---|---|---|
| ISP ↔ HQ-RTR | 172.16.1.0/28 | задано | 16 | ISP .1 / HQ-RTR .2 |
| ISP ↔ BR-RTR | 172.16.2.0/28 | задано | 16 | ISP .1 / BR-RTR .2 |
| HQ-SRV-Net | 192.168.0.0/27 | ≤ 32 | 32 | HQ-RTR .1 / HQ-SRV .2 |
| HQ-CLI-Net | 192.168.0.32/28 | ≥ 16 | 16 | HQ-RTR .33 / HQ-CLI DHCP |
| Сеть управления (MGMT) | 192.168.0.48/29 | ≤ 8 | 8 | HQ-RTR .49 |
| BR-SRV-Net | 192.168.1.0/28 | ≤ 16 | 16 | BR-RTR .1 / BR-SRV .2 |
| Туннель HQ ↔ BR | 172.16.0.0/30 | для GRE | 4 | HQ-RTR .1 / BR-RTR .2 |

> **Примечание.** Логика масок: «не более 32» → /27 (32 адреса); «не менее 16» → /28 (минимальная сеть, вмещающая 16); «не более 8» → /29; «не более 16» → /28. Все внутренние блоки взяты из 192.168.0.0/16 (RFC1918) и не пересекаются.

---

## 1. Имена устройств (FQDN)

**Задание:** задать имена согласно топологии, используя полное доменное имя (домен `sirius-exam.org`).

**Все Linux-устройства (выполнить на каждом своё имя):**

```bash
# ISP
hostnamectl set-hostname isp.sirius-exam.org
# HQ-RTR
hostnamectl set-hostname hq-rtr.sirius-exam.org
# BR-RTR
hostnamectl set-hostname br-rtr.sirius-exam.org
# HQ-SRV
hostnamectl set-hostname hq-srv.sirius-exam.org
# BR-SRV
hostnamectl set-hostname br-srv.sirius-exam.org

# Прописать FQDN в /etc/hosts (пример для HQ-SRV):
echo '127.0.1.1  hq-srv.sirius-exam.org hq-srv' >> /etc/hosts
# применить приглашение в текущей сессии:
exec bash
```

**HQ-CLI (Windows, PowerShell от администратора):**

```powershell
Rename-Computer -NewName "HQ-CLI" -Force -Restart
# FQDN hq-cli.sirius-exam.org формируется DNS-суффиксом sirius-exam.org,
# который машина получит по DHCP (см. п.8).
```

> **Примечание.** Если HQ-CLI на Linux: `hostnamectl set-hostname hq-cli.sirius-exam.org`.

---

## 2. Адресация IPv4

**Задание:** настроить IPv4 на всех устройствах согласно сводной таблице. Ниже — `/etc/network/interfaces` (Debian/Astra).

**ISP:**

```ini
# /etc/network/interfaces
auto ens18
iface ens18 inet dhcp          # uplink к магистральному провайдеру

auto ens19
iface ens19 inet static        # к HQ-RTR
    address 172.16.1.1/28

auto ens20
iface ens20 inet static        # к BR-RTR
    address 172.16.2.1/28

# применить:
# systemctl restart networking
```

> **Примечание.** Маршрут по умолчанию ISP получает автоматически по DHCP, ручная настройка не требуется.

**HQ-RTR:**

```ini
# /etc/network/interfaces
auto ens18
iface ens18 inet static        # к ISP
    address 172.16.1.2/28
    gateway 172.16.1.1

auto ens19
iface ens19 inet static        # HQ-SRV-Net
    address 192.168.0.1/27

auto ens20
iface ens20 inet static        # HQ-CLI-Net
    address 192.168.0.33/28

auto ens21
iface ens21 inet static        # сеть управления (MGMT)
    address 192.168.0.49/29

# systemctl restart networking
```

> **Примечание.** Если внутренние сети идут одним транком — оформите их как VLAN-субинтерфейсы: `iface ens19.100 / ens19.200 / ens19.999` (пакет `vlan`, модуль `8021q`).

**BR-RTR:**

```ini
auto ens18
iface ens18 inet static        # к ISP
    address 172.16.2.2/28
    gateway 172.16.2.1

auto ens19
iface ens19 inet static        # BR-SRV-Net
    address 192.168.1.1/28

# systemctl restart networking
```

**HQ-SRV:**

```ini
auto ens18
iface ens18 inet static
    address 192.168.0.2/27
    gateway 192.168.0.1
    dns-nameservers 127.0.0.1   # сам является DNS-сервером (п.9)
```

**BR-SRV:**

```ini
auto ens18
iface ens18 inet static
    address 192.168.1.2/28
    gateway 192.168.1.1
    dns-nameservers 192.168.0.2  # DNS = HQ-SRV
```

**HQ-CLI (Windows):** адрес/шлюз/DNS получаются автоматически — оставить адаптер на «Получить IP-адрес автоматически» (DHCP настраивается в п.8).

---

## 3. Доступ в Интернет на ISP, PAT и учётные записи

### 3.1–3.2. Маршрутизация и PAT на ISP

**Задание:** магистральный интерфейс — по DHCP; динамический PAT для выхода HQ-RTR и BR-RTR (сети 172.16.1.0/28 и 172.16.2.0/28) в Интернет.

**ISP:**

```bash
# включить маршрутизацию
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-router.conf

# динамический PAT (overload) в сторону Интернета (ens18):
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens18 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens18 -j MASQUERADE

# сохранить правила:
apt install -y iptables-persistent
netfilter-persistent save
```

> **Примечание.** Маршруты до внутренних сетей офисов на ISP не нужны: HQ-RTR и BR-RTR сами делают NAT в сторону ISP (п.8), поэтому трафик к провайдеру приходит уже с адресов 172.16.1.2 / 172.16.2.2.

### 3.3. Пользователь sshuser на серверах

**Задание:** на HQ-SRV и BR-SRV создать `sshuser`, пароль `P@ssw0rd`, UID 2026, sudo без пароля.

**HQ-SRV и BR-SRV (одинаково):**

```bash
useradd -m -u 2026 -s /bin/bash sshuser
echo 'sshuser:P@ssw0rd' | chpasswd
# sudo без ввода пароля:
echo 'sshuser ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/sshuser
chmod 440 /etc/sudoers.d/sshuser
# проверка:
id sshuser
```

### 3.4–3.7. Пользователь net_admin на маршрутизаторах

**Задание:** на HQ-RTR и BR-RTR создать `net_admin`, пароль `P@ssw0rd`; в Linux — sudo без пароля; в не-Linux ОС — максимальные привилегии.

**HQ-RTR и BR-RTR (Linux):**

```bash
useradd -m -s /bin/bash net_admin
echo 'net_admin:P@ssw0rd' | chpasswd
echo 'net_admin ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/net_admin
chmod 440 /etc/sudoers.d/net_admin
```

> **Примечание.** Если маршрутизатор не на Linux: создайте `net_admin` с паролем `P@ssw0rd` и назначьте максимальный уровень привилегий (например, в Cisco — privilege 15: `username net_admin privilege 15 secret P@ssw0rd`).

---

## 4. Маршрутизация трафика на HQ-RTR

**Задание:** реализовать на HQ-RTR маршрутизацию (включить пересылку пакетов между интерфейсами).

**HQ-RTR (а также BR-RTR и ISP):**

```bash
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward = 1' > /etc/sysctl.d/99-router.conf
sysctl -p /etc/sysctl.d/99-router.conf
# проверка:
cat /proc/sys/net/ipv4/ip_forward    # должно быть 1
```

---

## 5. Безопасный удалённый доступ (SSH) на HQ-SRV и BR-SRV

**Задание:** порт 2026; доступ только `sshuser`; не более двух попыток входа; баннер «Authorized access only».

**HQ-SRV и BR-SRV (одинаково):**

```bash
# 1) баннер
echo 'Authorized access only' > /etc/ssh/banner
```

```ini
# 2) /etc/ssh/sshd_config — задать/изменить параметры:
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/banner
```

```bash
# 3) применить:
systemctl restart sshd     # в некоторых ОС: systemctl restart ssh

# проверка с клиента:
ssh -p 2026 sshuser@<ip-сервера>
```

> **Примечание.** `MaxAuthTries 2` ограничивает число попыток аутентификации до двух. Если включён межсетевой экран, откройте порт `2026/tcp`. `AllowUsers` гарантирует подключение только пользователю `sshuser`.

---

## 6. IP-туннель между HQ-RTR и BR-RTR (GRE)

**Задание:** настроить туннель (GRE или IP-in-IP) между офисами; сведения занести в отчёт. Подложка туннеля — WAN-адреса (172.16.1.2 и 172.16.2.2), они достижимы друг для друга через ISP.

**HQ-RTR:**

```ini
# в /etc/network/interfaces:
auto tun1
iface tun1 inet static
    address 172.16.0.1/30
    pre-up ip tunnel add tun1 mode gre local 172.16.1.2 remote 172.16.2.2 ttl 64
    post-down ip tunnel del tun1

# поднять:
# ifup tun1
```

**BR-RTR:**

```ini
auto tun1
iface tun1 inet static
    address 172.16.0.2/30
    pre-up ip tunnel add tun1 mode gre local 172.16.2.2 remote 172.16.1.2 ttl 64
    post-down ip tunnel del tun1

# ifup tun1
# проверка связности по туннелю:
# ping 172.16.0.1
```

> **Примечание (для отчёта).** Технология — GRE; конечные точки 172.16.1.2 ↔ 172.16.2.2; адресация туннеля 172.16.0.0/30 (HQ .1, BR .2). Вариант IP-in-IP: заменить `mode gre` на `mode ipip`.

---

## 7. Динамическая маршрутизация (OSPF, link-state)

**Задание:** link-state-протокол; работает только на интерфейсах туннеля; маршрутами обмениваются только два маршрутизатора; защита паролем; сведения — в отчёт. Используем OSPF (FRRouting).

**HQ-RTR и BR-RTR — включить демон OSPF:**

```bash
apt install -y frr
# в /etc/frr/daemons:
#   ospfd=yes
systemctl restart frr
```

**HQ-RTR — vtysh:**

```text
vtysh
configure terminal
router ospf
 ospf router-id 1.1.1.1
 network 172.16.0.0/30 area 0       # туннель
 network 192.168.0.0/27 area 0      # HQ-SRV-Net
 network 192.168.0.32/28 area 0     # HQ-CLI-Net
 network 192.168.0.48/29 area 0     # MGMT
 passive-interface default          # запрет OSPF на всех интерфейсах
 no passive-interface tun1          # ...кроме туннеля
exit
interface tun1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd   # парольная защита
exit
do write memory
```

**BR-RTR — vtysh:**

```text
vtysh
configure terminal
router ospf
 ospf router-id 2.2.2.2
 network 172.16.0.0/30 area 0       # туннель
 network 192.168.1.0/28 area 0      # BR-SRV-Net
 passive-interface default
 no passive-interface tun1
exit
interface tun1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 P@ssw0rd
exit
do write memory
```

```bash
# проверка соседства и маршрутов:
vtysh -c 'show ip ospf neighbor'
vtysh -c 'show ip route ospf'
```

> **Примечание.** `passive-interface default` + `no passive-interface tun1` обеспечивают работу OSPF ТОЛЬКО на туннеле, поэтому соседство образуется лишь между HQ-RTR и BR-RTR. Внутренние сети при этом анонсируются и доступны из соседнего офиса. Для отчёта: протокол OSPF, area 0, аутентификация MD5 (key 1 = `P@ssw0rd`), активный интерфейс — `tun1`.

---

## 8. NAT на офисных маршрутизаторах и DHCP для HQ-CLI

### 8.1. Динамический NAT (PAT) в сторону ISP

**Задание:** все устройства офисов должны иметь доступ в Интернет. Каждый офисный маршрутизатор транслирует свои внутренние сети в свой WAN-интерфейс (ens18).

**HQ-RTR:**

```bash
iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o ens18 -j MASQUERADE
netfilter-persistent save
```

**BR-RTR:**

```bash
iptables -t nat -A POSTROUTING -s 192.168.1.0/28 -o ens18 -j MASQUERADE
netfilter-persistent save
```

> **Примечание.** MASQUERADE привязан к WAN-интерфейсу (`-o ens18`), поэтому трафик между офисами через туннель (`tun1`) НЕ транслируется — межофисная связность по OSPF сохраняется.

### 8.2. DHCP-сервер на HQ-RTR для сети HQ-CLI

**Задание:** подсеть HQ-CLI; сервер — HQ-RTR; клиент — HQ-CLI; исключить адрес маршрутизатора; шлюз — HQ-RTR; DNS — HQ-SRV; суффикс `sirius-exam.org`.

**HQ-RTR:**

```bash
apt install -y isc-dhcp-server
```

```text
# /etc/dhcp/dhcpd.conf
option domain-name "sirius-exam.org";
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.0.32 netmask 255.255.255.240 {
    range 192.168.0.34 192.168.0.46;        # .33 (шлюз/роутер) исключён
    option routers 192.168.0.33;             # шлюз = HQ-RTR
    option domain-name-servers 192.168.0.2;  # DNS = HQ-SRV
    option domain-name "sirius-exam.org";
}
```

```bash
# указать интерфейс выдачи в /etc/default/isc-dhcp-server:
#   INTERFACESv4="ens20"
systemctl restart isc-dhcp-server
```

**HQ-CLI (Windows) — проверка:**

```powershell
ipconfig /release
ipconfig /renew
ipconfig /all      # адрес из 192.168.0.34-46, шлюз .33, DNS .2, суффикс sirius-exam.org
```

---

## 9. DNS-инфраструктура и часовой пояс

### 9.1–9.3. DNS-сервер на HQ-SRV (BIND9)

**Задание:** основной DNS — HQ-SRV; прямое и обратное разрешение (по таблице 3); пересылка на публичный DNS (77.88.8.7 / 77.88.8.3).

**HQ-SRV — установка и опции:**

```bash
apt install -y bind9
mkdir -p /etc/bind/zones
```

```text
# /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { any; };
    forwarders { 77.88.8.7; 77.88.8.3; };
    dnssec-validation no;
    listen-on { any; };
};
```

**HQ-SRV — описание зон:**

```text
# /etc/bind/named.conf.local
zone "sirius-exam.org" {
    type master;
    file "/etc/bind/zones/db.sirius-exam.org";
};
zone "0.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.0";
};
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
};
```

**HQ-SRV — прямая зона:**

```text
# /etc/bind/zones/db.sirius-exam.org
$TTL 604800
@   IN  SOA hq-srv.sirius-exam.org. admin.sirius-exam.org. (
        2026053101 604800 86400 2419200 604800 )
@           IN  NS   hq-srv.sirius-exam.org.
hq-srv      IN  A    192.168.0.2
hq-rtr      IN  A    192.168.0.1
hq-cli      IN  A    192.168.0.34
br-rtr      IN  A    192.168.1.1
br-srv      IN  A    192.168.1.2
isp         IN  A    172.16.1.1
```

**HQ-SRV — обратные зоны:**

```text
# /etc/bind/zones/db.192.168.0
$TTL 604800
@ IN SOA hq-srv.sirius-exam.org. admin.sirius-exam.org. (
        2026053101 604800 86400 2419200 604800 )
@   IN  NS   hq-srv.sirius-exam.org.
1   IN  PTR  hq-rtr.sirius-exam.org.
2   IN  PTR  hq-srv.sirius-exam.org.
34  IN  PTR  hq-cli.sirius-exam.org.

# /etc/bind/zones/db.192.168.1
$TTL 604800
@ IN SOA hq-srv.sirius-exam.org. admin.sirius-exam.org. (
        2026053101 604800 86400 2419200 604800 )
@   IN  NS   hq-srv.sirius-exam.org.
1   IN  PTR  br-rtr.sirius-exam.org.
2   IN  PTR  br-srv.sirius-exam.org.
```

**HQ-SRV — проверка и запуск:**

```bash
named-checkconf
named-checkzone sirius-exam.org /etc/bind/zones/db.sirius-exam.org
systemctl restart bind9     # в части ОС: systemctl restart named
```

> **Примечание.** Состав A/PTR-записей приведите в точное соответствие «таблице 3» из задания. Клиенты должны указывать DNS = 192.168.0.2: HQ-CLI получает его по DHCP, BR-SRV — статически (`dns-nameservers 192.168.0.2`), доступ к HQ-SRV — через туннель/OSPF.

**Проверка разрешения имён (с любого узла):**

```bash
nslookup hq-srv.sirius-exam.org 192.168.0.2
nslookup 192.168.1.2 192.168.0.2     # обратное разрешение
nslookup ya.ru 192.168.0.2           # проверка пересылки
```

### 9.4. Часовой пояс на всех устройствах

**Задание:** настроить часовой пояс по месту проведения экзамена (виртуальный коммутатор не настраивается).

**Все Linux-устройства:**

```bash
timedatectl set-timezone Europe/Moscow   # подставьте зону места экзамена
timedatectl                               # проверка
# список зон при необходимости: timedatectl list-timezones
```

**HQ-CLI (Windows):**

```powershell
Set-TimeZone -Id "Russian Standard Time"   # пример для Europe/Moscow
Get-TimeZone
```

---

## Приложение А. Отличия для ALT Linux

Если устройства развёрнуты на ALT Linux, изменяются способ настройки сети и часть имён пакетов/служб; логика решения та же.

**Сеть (etcnet) — пример для статического адреса:**

```bash
# каталог интерфейса:
mkdir -p /etc/net/ifaces/ens19
# /etc/net/ifaces/ens19/options
BOOTPROTO=static
TYPE=eth
DISABLED=no
# /etc/net/ifaces/ens19/ipv4address
192.168.0.1/27
# шлюз (на WAN-интерфейсе): /etc/net/ifaces/ens18/ipv4route
default via 172.16.1.1
# применить:
systemctl restart network
```

**Пакеты и службы:**

| Назначение | Debian/Astra | ALT Linux |
|---|---|---|
| Сеть | /etc/network/interfaces, `systemctl restart networking` | /etc/net/ifaces/*, `systemctl restart network` |
| DHCP-сервер | isc-dhcp-server | dhcp-server (`apt-get install dhcp-server`) |
| DNS | bind9 | bind (служба `named`) |
| Маршрутизация OSPF | frr | frr |
| NAT | iptables + iptables-persistent | iptables / nftables, служба iptables save |

> **Примечание.** На ALT включение форвардинга, GRE-туннель (`ip tunnel`), настройка `sshd_config`, `sudoers`, OSPF в `vtysh` и `timedatectl` выполняются идентично примерам выше.

---

## Финальная проверка

- **Имена:** `hostnamectl` на каждом узле показывает FQDN вида `*.sirius-exam.org`.
- **Адресация:** `ip -br a` соответствует сводной таблице; ping между всеми сетями.
- **Интернет:** с HQ-SRV/HQ-CLI/BR-SRV проходит `ping 77.88.8.8` (двойной NAT офис→ISP).
- **Туннель:** `ping 172.16.0.2` с HQ-RTR; `show ip ospf neighbor` — один сосед.
- **OSPF:** в таблице маршрутов HQ-RTR видны сети BR и наоборот; работает только на `tun1`.
- **DHCP:** HQ-CLI получает адрес .34-.46, шлюз .33, DNS .2, суффикс `sirius-exam.org`.
- **DNS:** `nslookup` прямого/обратного имени и внешнего домена через 192.168.0.2 успешен.
- **SSH:** подключение только `sshuser` на порт 2026, баннер «Authorized access only», 2 попытки.
- **Время:** `timedatectl` на всех Linux и `Get-TimeZone` на HQ-CLI — зона места экзамена.
