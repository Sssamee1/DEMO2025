# Demo2025
## Сетевая топология
Подключаем к каждой машине 
айпишники

## 1. Базовая настройка системы

### Изменение имени хоста
```bash
hostnamectl hostname "имя_машины"
exec bash
```

## 2. Создание пользователя sshuser (на HQ-SRV и BR-SRV)

```bash
# Создание пользователя с UID 1010
useradd -m -u 1010 sshuser

# Установка пароля
passwd sshuser

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
```

## 3. Настройка SSH (на HQ-SRV и BR-SRV)

### Создание баннера
```bash
nano /etc/mybanner
```
Содержимое файла:
```
Authorized access only
```

### Настройка SSH daemon
```bash
nano /etc/openssh/sshd_config
```
Добавить/изменить параметры:
```
Port 2025
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser
```

### Перезапуск службы
```bash
systemctl restart sshd.service
```

## 4. Настройка DHCP сервера (на HQ-RTR)

### Указание интерфейса для DHCP
```bash
nano /etc/sysconfig/dhcpd
```
```
DHCPARGS=ens35
```

### Создание конфигурационного файла
```bash
cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
```

Содержимое файла:
```
option domain-name "au-team.irpo";
option domain-name-servers 172.16.0.2;
default-lease-time 6000;
max-lease-time 72000;
authoritative;

subnet 172.16.0.0 netmask 255.255.255.192 {
    range 172.16.0.3 172.16.0.8;
    option routers 172.16.0.1;
}
```

### Запуск и автозагрузка службы
```bash
systemctl enable --now dhcpd
```

## 5. Настройка маршрутизации

### На ISP - включение IP forwarding
```bash
nano /etc/net/sysctl.conf
```
```
net.ipv4.ip_forward = 1
```

## 6. Настройка GRE туннелей

### На BR-RTR (Branch Router)
**Сетевые настройки:**
- Device: tun1
- Parent: ens34
- Local IP: 172.16.5.2
- Remote IP: 172.16.4.2
- IPv4 Configuration: Manual
  - IP: 192.168.0.2/24
  - Gateway: 192.168.0.1

### На HQ-RTR (Headquarters Router)
**Сетевые настройки:**
- Device: tun1
- Parent: ens34
- Local IP: 172.16.4.2
- Remote IP: 172.16.5.2
- IPv4 Configuration: Manual
  - IP: 192.168.0.1/24
  - Gateway: 192.168.0.2

## 7. Настройка OSPF маршрутизации

### На HQ-SRV и BR-SRV - активация FRR
```bash
nano /etc/frr/daemons
```
Изменить строку:
```
ospfd=yes
```

```bash
systemctl enable --now frr
```

### На HQ-RTR - настройка OSPF
```bash
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.0.0/26 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
```

### Настройка TTL для туннеля
```bash
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
```

### На BR-RTR - аналогичная настройка
```bash
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.6.0/27 area 0  # BR подсеть
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit

# Настройка TTL для туннеля
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
```

### Проверка OSPF
```bash
# Вход в FRR консоль
vtysh

# Проверка соседей OSPF
show ip ospf neighbor

# Проверка таблицы маршрутизации OSPF
show ip ospf route

# Проверка интерфейсов OSPF
show ip ospf interface

# Выход из консоли
exit
```



## Примечания

- IP-адреса соответствуют схеме сети
- Проверить firewall 
