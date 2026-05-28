# Модуль 1 — Пошаговая инструкция (хост за хостом)

**Стенд:** ISP / HQ-SRV / BR-SRV / HQ-CLI — РедОС · HQ-RTR / BR-RTR — Eltex ESR
**Домен:** `sirius-exam.org` · **Пароль:** `P@ssw0rd` · **SSH-порт:** `2026`
**РедОС:** IP и hostname задаём через **nmtui**.

> Веди отчёт по ходу (скриншоты по пунктам с пометкой «в отчёт»). Итог — `ФамилияМодуль1`.

## План адресации (держи под рукой)

| Сеть | Подсеть | Шлюз | Хост |
|------|---------|------|------|
| HQ-SRV (≤32) | `192.168.0.0/27` | `192.168.0.1` | HQ-SRV `192.168.0.2` |
| HQ-CLI (≥16) | `192.168.0.32/27` | `192.168.0.33` | DHCP |
| Управление (≤8) | `192.168.0.64/29` | `192.168.0.65` | — |
| BR-SRV (≤16) | `192.168.1.0/28` | `192.168.1.1` | BR-SRV `192.168.1.2` |
| ISP↔HQ-RTR | `172.16.1.0/28` | ISP `.1` | HQ-RTR `.2` |
| ISP↔BR-RTR | `172.16.2.0/28` | ISP `.1` | BR-RTR `.2` |
| Туннель | `10.10.10.0/30` | HQ `.1` / BR `.2` | — |

---

# ХОСТ 1 — ISP (РедОС)

Делаем полностью, дальше к нему не возвращаемся.

### 1.1. Hostname через nmtui
```
sudo nmtui
```
→ **Set system hostname** → ввести `isp.sirius-exam.org` → OK.
Применить: `sudo hostnamectl set-hostname isp.sirius-exam.org` (или перелогиниться).

### 1.2. Определить интерфейсы
```
ip -br a
```
- `enp7s1` — WAN (адрес `10.x` по DHCP), оставляем как есть.
- `enp7s2` / `enp7s3` — к HQ-RTR и BR-RTR. **Сверь по MAC в Proxmox**, какой куда.
  Ниже: enp7s2 → HQ, enp7s3 → BR (поправь при необходимости).

### 1.3. IP на внутренние интерфейсы через nmtui
```
sudo nmtui
```
→ **Edit a connection** → выбрать **enp7s2** → Edit:
- IPv4 CONFIGURATION → **Manual** → Show → Addresses: `172.16.1.1/28`
- OK.

Снова **Edit a connection** → **enp7s3** → Edit:
- IPv4 → **Manual** → Addresses: `172.16.2.1/28` → OK.

Выйти из nmtui. Применить (поднять соединения):
```
sudo nmcli con up enp7s2
sudo nmcli con up enp7s3
ip -br a       # проверь адреса
```

### 1.4. Форвардинг + NAT (PAT) для офисов
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-fwd.conf
sudo sysctl -p /etc/sysctl.d/99-fwd.conf

sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --permanent --direct --add-rule ipv4 nat POSTROUTING 0 -o enp7s1 -j MASQUERADE
sudo firewall-cmd --reload
```
Проверка: `ip r` — должен быть default через WAN. `ping 8.8.8.8` — есть интернет.

✅ ISP готов.

---

# ХОСТ 2 — HQ-RTR (Eltex ESR) — ЧАСТЬ 1

Делаем всё, что НЕ зависит от BR-RTR. Туннель/OSPF — позже (Часть 2).

Логин (обычно `admin`/`admin`), затем:
```
enable
configure
```

### 2.1. Hostname
```
hostname HQ-RTR
```

### 2.2. Интерфейс к ISP + маршрут по умолчанию
> Имена портов: `show interfaces status`. Подставь свои `gi1/0/N`.
```
interface gigabitethernet 1/0/1
  ip address 172.16.1.2/28
  exit
ip route 0.0.0.0/0 172.16.1.1
```

### 2.3. Локальные интерфейсы HQ (отдельный порт на сеть)
> Подставь свои порты `gi1/0/N` для каждой локальной сети.
```
interface gigabitethernet 1/0/2
  ip address 192.168.0.1/27
  exit
interface gigabitethernet 1/0/3
  ip address 192.168.0.33/27
  exit
interface gigabitethernet 1/0/4
  ip address 192.168.0.65/29
  exit
```

### 2.4. Пользователь net_admin
```
username net_admin
  password P@ssw0rd
  privilege 15
  exit
```

### 2.5. NAT в сторону ISP
```
nat source
  ruleset INTERNET
    rule 1
      match source-address any
      action source-nat interface
      enable
      exit
    exit
  exit
```
> Синтаксис NAT зависит от прошивки — если не принимает, набери начало + `?`.

### 2.6. DHCP для HQ-CLI
```
ip dhcp-server
ip dhcp-server pool HQ-CLI
  network 192.168.0.32/27
  address-range 192.168.0.34-192.168.0.62
  default-router 192.168.0.33
  dns-server 192.168.0.2
  domain-name sirius-exam.org
  exit
```
> `.33` (роутер) исключён — диапазон с `.34`.

### 2.7. Часовой пояс + сохранить
```
clock timezone gmt +3
exit
commit
confirm
```
⏸️ HQ-RTR Часть 1 готова. Туннель и OSPF — после BR-RTR.

---

# ХОСТ 3 — BR-RTR (Eltex ESR)

Можно сделать целиком (туннель настроим, ответная сторона поднимется, когда вернёмся на HQ).

```
enable
configure
hostname BR-RTR
exit
```

### 3.1. Интерфейс к ISP + маршрут
```
interface gigabitethernet 1/0/1
  ip address 172.16.2.2/28
  exit
ip route 0.0.0.0/0 172.16.2.1
```

### 3.2. Интерфейс в сторону BR-SRV
```
interface gigabitethernet 1/0/2
  ip address 192.168.1.1/28
  exit
```

### 3.3. net_admin + NAT
```
username net_admin
  password P@ssw0rd
  privilege 15
  exit
nat source
  ruleset INTERNET
    rule 1
      match source-address any
      action source-nat interface
      enable
      exit
    exit
  exit
```

### 3.4. Туннель GRE (сторона BR)
```
tunnel gre 1
  ip address 10.10.10.2/30
  local address 172.16.2.2
  remote address 172.16.1.2
  enable
  exit
```

### 3.5. OSPF (только на туннеле, с паролем)
```
router ospf 1
  area 0.0.0.0
  enable
  exit
interface tunnel gre 1
  ip ospf instance 1
  ip ospf area 0.0.0.0
  ip ospf authentication message-digest
  ip ospf authentication-key ascii-text P@ssw0rd
  exit
```
> Анонсируй локальную сеть BR в OSPF (на интерфейсе к BR-SRV: `ip ospf instance 1` + `ip ospf area`).

### 3.6. Часовой пояс + сохранить
```
clock timezone gmt +3
exit
commit
confirm
```
✅ BR-RTR готов.

---

# ХОСТ 2 — HQ-RTR (Eltex ESR) — ЧАСТЬ 2

Возвращаемся: поднимаем туннель и OSPF (теперь BR-сторона уже есть).
```
enable
configure
```

### 4.1. Туннель GRE (сторона HQ)
```
tunnel gre 1
  ip address 10.10.10.1/30
  local address 172.16.1.2
  remote address 172.16.2.2
  enable
  exit
```

### 4.2. OSPF
```
router ospf 1
  area 0.0.0.0
  enable
  exit
interface tunnel gre 1
  ip ospf instance 1
  ip ospf area 0.0.0.0
  ip ospf authentication message-digest
  ip ospf authentication-key ascii-text P@ssw0rd
  exit
```
> Анонсируй локальные сети HQ в OSPF (на локальных интерфейсах: `ip ospf instance 1` + `ip ospf area`).

### 4.3. Сохранить + проверить
```
exit
commit
confirm
show ip ospf neighbors      # сосед должен стать Full
```
Проверка с HQ-RTR: `ping 10.10.10.2`.
✅ Магистраль между офисами поднята.

---

# ХОСТ 4 — HQ-SRV (РедОС)

### 5.1. Hostname + IP через nmtui
```
sudo nmtui
```
- **Set system hostname** → `hq-srv.sirius-exam.org`
- **Edit a connection** → интерфейс → IPv4 **Manual**:
  - Address: `192.168.0.2/27`
  - Gateway: `192.168.0.1`
  - DNS: `127.0.0.1`
  - OK → выйти.
```
sudo nmcli con up <соединение>
sudo hostnamectl set-hostname hq-srv.sirius-exam.org
```

### 5.2. Пользователь sshuser (UID 2026, sudo без пароля)
```bash
sudo useradd -u 2026 -m -s /bin/bash sshuser
echo 'sshuser:P@ssw0rd' | sudo chpasswd
echo 'sshuser ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/sshuser
sudo chmod 440 /etc/sudoers.d/sshuser
```

### 5.3. Безопасный SSH (порт 2026, 2 попытки, баннер)
```bash
sudo tee /etc/ssh/sshd_config.d/exam.conf >/dev/null <<'EOF'
Port 2026
MaxAuthTries 2
AllowUsers sshuser
Banner /etc/ssh/banner
EOF
echo 'Authorized access only' | sudo tee /etc/ssh/banner
sudo semanage port -a -t ssh_port_t -p tcp 2026 2>/dev/null || sudo semanage port -m -t ssh_port_t -p tcp 2026
sudo firewall-cmd --permanent --add-port=2026/tcp
sudo firewall-cmd --reload
sudo systemctl restart sshd
ss -tlnp | grep 2026
```

### 5.4. DNS-сервер (bind) — основной для офисов
```bash
sudo dnf install -y bind bind-utils
```
- В `/etc/named.conf`: `listen-on { any; };`, `allow-query { any; };`,
  `forwarders { 77.88.8.7; 77.88.8.3; };`
- Создать прямую зону `sirius-exam.org` и обратные зоны по Таблице 3:
  hq-rtr(A,PTR), br-rtr(A), hq-srv(A,PTR), hq-cli(A,PTR), br-srv(A),
  docker→ISP(HQ-сторона), web→ISP(BR-сторона).
```bash
sudo systemctl enable --now named
dig @127.0.0.1 hq-rtr.sirius-exam.org      # проверка
```

### 5.5. Часовой пояс
```bash
sudo timedatectl set-timezone Europe/Moscow   # по месту экзамена
```
✅ HQ-SRV готов.

---

# ХОСТ 5 — BR-SRV (РедОС)

### 6.1. Hostname + IP через nmtui
```
sudo nmtui
```
- Hostname → `br-srv.sirius-exam.org`
- IPv4 Manual: Address `192.168.1.2/28`, Gateway `192.168.1.1`, DNS `192.168.0.2`.
```
sudo nmcli con up <соединение>
sudo hostnamectl set-hostname br-srv.sirius-exam.org
```

### 6.2. Пользователь sshuser
```bash
sudo useradd -u 2026 -m -s /bin/bash sshuser
echo 'sshuser:P@ssw0rd' | sudo chpasswd
echo 'sshuser ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/sshuser
sudo chmod 440 /etc/sudoers.d/sshuser
```

### 6.3. Безопасный SSH
```bash
sudo tee /etc/ssh/sshd_config.d/exam.conf >/dev/null <<'EOF'
Port 2026
MaxAuthTries 2
AllowUsers sshuser
Banner /etc/ssh/banner
EOF
echo 'Authorized access only' | sudo tee /etc/ssh/banner
sudo semanage port -a -t ssh_port_t -p tcp 2026 2>/dev/null || sudo semanage port -m -t ssh_port_t -p tcp 2026
sudo firewall-cmd --permanent --add-port=2026/tcp
sudo firewall-cmd --reload
sudo systemctl restart sshd
```

### 6.4. Часовой пояс
```bash
sudo timedatectl set-timezone Europe/Moscow
```
✅ BR-SRV готов.

---

# ХОСТ 6 — HQ-CLI (РедОС)

### 7.1. Hostname + IP по DHCP через nmtui
```
sudo nmtui
```
- Hostname → `hq-cli.sirius-exam.org`
- **Edit a connection** → интерфейс → IPv4 CONFIGURATION → **Automatic (DHCP)** → OK.
```
sudo nmcli con up <соединение>
sudo hostnamectl set-hostname hq-cli.sirius-exam.org
```

### 7.2. Проверка
```bash
ip a                     # получил адрес из 192.168.0.32/27
ip r                     # шлюз 192.168.0.33
cat /etc/resolv.conf     # DNS 192.168.0.2, suffix sirius-exam.org
ping 8.8.8.8             # интернет
ping hq-srv.sirius-exam.org   # DNS работает
```

### 7.3. Часовой пояс
```bash
sudo timedatectl set-timezone Europe/Moscow
```
✅ HQ-CLI готов.

---

# Финальная проверка

- ISP ↔ оба роутера (`ping 172.16.1.2`, `172.16.2.2`)
- Туннель: `show ip ospf neighbors` = Full, `ping 10.10.10.2`
- HQ-CLI: DHCP-адрес, интернет, резолв имён
- SSH на серверы: порт 2026, только sshuser, баннер «Authorized access only»
- Из HQ пингуется BR-SRV и наоборот (динамическая маршрутизация)

> 🔴 Eltex: синтаксис NAT/DHCP/tunnel/OSPF зависит от прошивки. Не принимает строку — набери начало + `?`. Сверяйся с Прил_2.
> 🔴 Заполни Таблицу 2 (адреса) и Таблицу 3 (DNS), сделай скриншоты по пунктам «в отчёт».
