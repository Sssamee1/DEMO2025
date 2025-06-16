# если все можно http://xn---220-43dg5d.xn--p1ai/ru/demo-2025/modul-1/modul-1-1
# DM25

# Основное
## 1. Базовая настройка системы
### Изменение имени хоста

```bash
hostnamectl hostname "имя_машины"; exec bash
```

#### НЕ ЗАБУДЬТЕ ОТКЛЮЧИТЬ  firewall  командой systemctl disable firewalld --now
#### (если не отключить gre и ospf могут не работать) на  машинах где tunnel и ospf

### IP-адресса
#### Пример (айпи могут измениться)
```bash
ISP:
isp-hq - 172.16.4.1/28 gateway оставляем пустым
isp-br - 172.16.5.1/28 gateway оставляем пустым

HQ-R
isp-hq - 172.16.4.2/28 gateway 172.16.4.1  (айпи машины isp)

BR-R
isp-br - 172.16.5.2/28 gateway 172.16.5.1  (айпи машины isp)
```

## 2. Настройка маршрутизации
### На ISP - включение IP forwarding
```bash
nano /etc/net/sysctl.conf
```
```
net.ipv4.ip_forward = 1

```
## после этого нужнно перезагрузить ISP
```
reboot
```

## 3. Настройка GRE туннелей
### На BR-RTR (Branch Router)
**Сетевые настройки:**
- Profile name: tun1
- Device: tun1
- Mode: GRE
- Parent: ens34
- Local IP: 172.16.5.2
- Remote IP: 172.16.4.2
- IPv4 Configuration: Manual
  - IP: 192.168.0.2/24
  - Gateway: 192.168.0.1

### На HQ-RTR (Headquarters Router)
**Сетевые настройки:**
- Profile name: tun1
- Device: tun1
- Mode: GRE
- Parent: ens34
- Local IP: 172.16.4.2
- Remote IP: 172.16.5.2
- IPv4 Configuration: Manual
  - IP: 192.168.0.1/24
  - Gateway: 192.168.0.2

### Перезагрузить машины

**Если GRE тунель не работает пропинговать с HQ-R BR-R по его айпи, если пинги не проходят то перезаагрузить машину ISP, также если это не помогло то можно попробовать временно отключить другие сетевые интерфейсы (hqin и brin) команда1 - ip -c a, команда2 - ping -c4 "ip другой машины"**


## 4. Создание пользователя net_admin (HQ-RTR и BR-RTR)
```bash
# Создание пользователя
adduser net_admin

# Установка пароля 
passwd net_admin

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
net_admin ALL=(ALL:ALL)NOPASSWD:ALL

```

## 5. Создание пользователя sshuser (на HQ-SRV и BR-SRV)

```bash
# Создание пользователя
useradd -m -u 1010 sshuser

# Установка пароля 
passwd sshuser

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
sshuser ALL=(ALL:ALL)NOPASSWD:ALL

```

## 6. Настройка SSH (на HQ-SRV и BR-SRV)

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
после чего пингуем
```

## 7. Настройка OSPF маршрутизации

### На HQ-R и BR-R - активация FRR
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
network 172.16.0.0/26 area 0 # указать HQ-IN подсеть
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
exit

# Настройка TTL для туннеля
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
network 172.16.6.0/27 area 0  # указать BR-IN подсеть
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
exit

# Настройка TTL для туннеля
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr

#ТЕПЕРЬ ПЕРЕЗАГРУЗИТЕ HQ-R и BR-R
```

### Проверка OSPF
```bash
# Вход в FRR консоль
vtysh

# Проверка соседей OSPF, если ничего не вывело то забиваем и переходим к следующему заданию
show ip ospf neighbor

# Выход из консоли
exit
```
## 8. Настройка DHCP сервера (на HQ-RTR)

### Указание интерфейса для DHCP
```bash
nano /etc/sysconfig/dhcpd
```
```
DHCPARGS=ens35
```

### Создание конфигурационного файла
```bash
# в файле /etc/dhcp/dhcpd.conf.example показан пример dhcp

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
### RAID 5 на HQ-SRV:
```bash
#команда ниже должна вывести 4 диска sda, sdb, sdc, sdd 
lsblk

#Создание raid5
sudo mdadm --zero-superblock /dev/sd[b-d]

mkfs -t ext4 /dev/md0

mkdir /etc/mdadm

echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

mdadm --detail --scan | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.con

mkdir /mnt/raid5

nano /etc/fstab
#добавить эту строчку в fstab
/dev/md0  /mnt/raid5  ext4  defaults  0  0

mount -a
```

Проверяем монтирование:
```yml
df -h | grep /mnt/raid5
```
> Вывод должен быть:
> ```yml
> /dev/md0  2.0G  24K  1.9G  1%  /mnt/raid5
> ```


# Дополниетльно
## 1.Настройка NFS SERVER на HQ-SRV
```
mkdir /mnt/raid5/nfs

chmod 766 /mnt/raid5/nfs

nano /etc/exports
#добавляем строчку ниже | ВАЖНО ВМЕСТО 172.16.0.0/28 указывайте подсеть которая будет на демо
/mnt/raid5/nfs 172.16.0.0/28(rw,no_root_squash)

exportfs -arv

systemctl enable --now nfs-server
```
## 2.Настройка NFS CLIENT (вроде на hq-cli смотрите по заданию кому надо монтировать)
```
mkdir /mnt/nfs

chmod 777 /mnt/nfs

nano /etc/fstab
#добавляем строчку где 172.16.0.2 айпи nfs сервера 
172.16.0.2:/mnt/raid5/nfs  /mnt/nfs  nfs  defaults  0  0 

mount -a
```
Проверяем монтирование:
```yml
df -h | grep /mnt/nfs
```
> Вывод должен быть:
> ```yml
> 172.16.0.2:/mnt/raid5/nfs  2,0G  0  1,9G  0%  /mnt/nfs
> ```

## 3.Настройка chrony на HQ srv

Правим файл командой  **`nano /etc/chrony.conf`** :
```yml
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (https://www.pool.ntp.org/join.html
#pool pool.ntp.org iburst

server 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0
```

Запускаем и добавляем в автозагрузку утилиту **chronyd**:
```yml
systemctl enable --now chronyd
```

Проверяем работоспособность (вывод должен быть как в примере)
```yml
chronyc sources
```
> Вывод:
> ```yml
> MS Name/IP address        Stratum  Poll  Reach  LastRx  Last  sample
> =============================================================================
> ^/ localhost.localdomain  0        8     377    -       +0ns[  +0ns] +/-  0ns
> ```
```yml
chronyc tracking | grep Stratum
```
> Вывод:
> ```yml
> Stratum: 5
> ```

## 4.Яндекс браузер
### Найти на какой машине загрушен установщик, там и устанавливаем браузер
#### Вводим в терминале следующую команду
```
wget -q -O — https://repo.yandex.ru/yandex-browser/YANDEX-BROWSER-KEY.GPG | sudo apt-key add —
```
#### Добавляем репозиторий
```
sudo sh -c ‘echo «deb [arch=amd64] http://repo.yandex.ru/yandex-browser/deb beta main» > /etc/apt/sources.list.d/yandex-browser-beta.list’
```
#### Обновляем системные библиотеки
```
sudo apt update
```
#### Устанавливаем пакет
```
sudo apt install yandex-browser-beta
```
