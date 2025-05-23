# Инструкция по настройке сетевой инфраструктуры и администрированию ОС

> **Примечание:** В документе описаны задания для лабораторных работ, разбитых на 3 модуля.

## Оглавление

1. [Модуль №1: Настройка сетевой инфраструктуры](#модуль-№1-настройка-сетевой-инфраструктуры)
   - [1. Базовая настройка устройств](#1-базовая-настройка-устройств)
   - [2. Настройка ISP](#2-настройка-isp)
   - [3. Создание локальных учетных записей](#3-создание-локальных-учетных-записей)
   - [4. Настройка виртуального коммутатора и VLAN](#4-настройка-виртуального-коммутатора-и-vlan)
   - [5. Настройка безопасного удалённого доступа](#5-настройка-безопасного-удалённого-доступа)
   - [6. Настройка IP-туннеля между офисами](#6-настройка-ip-туннеля-между-офисами)
   - [7. Динамическая маршрутизация](#7-динамическая-маршрутизация)
   - [8. Динамическая трансляция адресов (NAT)](#8-динамическая-трансляция-адресов-nat)
   - [9. Настройка DHCP-сервера](#9-настройка-dhcp-сервера)
   - [10. Настройка DNS-сервера](#10-настройка-dns-сервера)
   - [11. Настройка часового пояса](#11-настройка-часового-пояса)

2. [Модуль №2: Организация сетевого администрирования ОС](#модуль-№2-организация-сетевого-администрирования-ос)
   - [1. Доменный контроллер Samba](#1-доменный-контроллер-samba)
   - [2. Настройка файлового хранилища](#2-настройка-файлового-хранилища)
   - [3. Настройка службы сетевого времени (chrony)](#3-настройка-службы-сетевого-времени-chrony)
   - [4. Настройка Ansible](#4-настройка-ansible)
   - [5. Развертывание MediaWiki в Docker](#5-развертывание-mediawiki-в-docker)
   - [6. Статическая трансляция портов](#6-статическая-трансляция-портов)
   - [7. Настройка Moodle](#7-настройка-moodle)
   - [8. Обратный прокси на nginx](#8-обратный-прокси-на-nginx)
   - [9. Установка Яндекс.Браузера](#9-установка-яндексбраузера)

---

## Модуль №1: Настройка сетевой инфраструктуры

### 1. Базовая настройка устройств

- **Имена устройств**  
  - **На роутерах HQ-RTR и BR-RTR:**  
    ```bash
    hostname {hq-rtr, br-rtr}
    ```
  - **На серверах и клиентах (HQ-SRV, HQ-CLI, BR-SRV):**  
    ```bash
    hostnamectl set-hostname {hq-srv, hq-cli, br-srv}.au-team.irpo; exec bash
    ```

- **IPv4 адресация**  
  - Настройка через `nmtui`.  
  - Адресация должна использовать приватные диапазоны (RFC1918):
    - 10.0.0.0/8  
    - 172.16.0.0/12  
    - 192.168.0.0/16

- **Локальные сети и маски:**
  - **HQ-SRV (VLAN100):** до 64 адресов, маска `/26` (255.255.255.192)
  - **HQ-CLI (VLAN200):** до 16 адресов, маска `/28` (255.255.255.240)
  - **BR-SRV:** до 32 адресов, маска `/27` (255.255.255.224)
  - **Управляющая сеть (VLAN999):** до 8 адресов, маска `/29` (255.255.255.248)

- **Пример таблицы адресации:**

  | Имя Устройства | IPv4                    | Интерфейс | NIC     | Шлюз        |
  |----------------|-------------------------|-----------|---------|-------------|
  | ISP            | NAT (inet)              | ens3      | Internet|             |
  |                | 172.16.4.1/28           | ens4      | ISP_HQ  |             |
  |                | 172.16.5.1/28           | ens5      | ISP_BR  |             |
  | HQ-RTR         | 172.16.4.2/28           | te0       | ISP_HQ  | 172.16.4.1  |
  |                | 192.168.1.1/26          | te1.100   |         |             |
  |                | 192.168.2.1/28         | te1.200   |         |             |
  |                | 192.168.0.1/29         | te1.999   | HQ_NET  |             |
  |                | 172.16.0.1/30             | GRE       | TUN     |             |
  | HQ-SW          | 192.168.0.2/29         | ens3      | HQ_NET  |             |
  |                | -                       | ens4      | SRV_NET |             |
  |                | -                       | ens5      | CLI_NET |             |
  | HQ-SRV         | 192.168.1.2/26          | ens3      | SRV_NET | 192.168.1.1 |
  | HQ-CLI         | 192.168.2.2/28 (DHCP)   | ens3      | CLI_NET | 192.168.2.1|
  | BR-RTR         | 172.16.5.2/28           | te0       | ISP_BR  | 172.16.5.1  |
  |                | 192.168.3.1/27          | te1       | BR_NET  |             |
  |                | 172.16.0.2/30             | GRE       | TUN     |             |
  | BR-SRV         | 192.168.3.2/27          | ens3      | BR_NET  | 192.168.3.1 |

---

### 2. Настройка ISP

- **IP-адресация интерфейсов:**
  - Интерфейс, подключённый к магистральному провайдеру – получает адрес по DHCP.
  - Маршруты по умолчанию:
    - **HQ-RTR:**  
      ```bash
      ip route 0.0.0.0/0 172.16.4.1
      ```
    - **BR-RTR:**  
      ```bash
      ip route 0.0.0.0/0 172.16.5.1
      ```

- **Пример конфигурации на EcoRouter для HQ-RTR:**
  ```bash
  en
  conf t
  int ISP
  ip add 172.16.4.2/28
  port te0
  service-instance toISP
  encapsulation untagged
  connect ip interface ISP
  wr mem
  ```

- **Аналогичная настройка для BR-RTR:**
  ```bash
  en
  conf t
  int ISP
  ip add 172.16.5.2/28
  port te0
  service-instance toISP
  encapsulation untagged
  connect ip interface ISP
  wr mem
  ```

- **Настройка динамической NAT на ISP:**
  ```bash
  echo net.ipv4.ip_forward=1 > /etc/sysctl.conf
  dnf install iptables-services -y
  iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
  iptables-save > /etc/sysconfig/iptables
  systemctl enable --now iptables
  ```

---

### 3. Создание локальных учетных записей

- **На серверах (HQ-SRV, BR-SRV):**
  - Создание пользователя `sshuser`:
    ```bash
    useradd -m -u 1010 sshuser
    echo "sshuser:P@ssw0rd" | sudo chpasswd
    ```
  - Настройка sudo без пароля:
    ```bash
    nano /etc/sudoers
    ```
    Добавить строку:
    ```
    sshuser ALL=(ALL) NOPASSWD:ALL
    ```

- **На маршрутизаторах (HQ-RTR, BR-RTR):**
  - Создание пользователя `net_admin` на EcoRouter:
    ```plaintext
    username net_admin
    password P@$$word
    role admin
    ```

---

### 4. Настройка виртуального коммутатора и VLAN

#### Создание подсети управления (VLAN 999)

- **На HQ-RTR:**
  ```bash
  int te1.999
  ip add 192.168.0.1/29
  description toSW
  port te1
  service-instance toSW
  encapsulation dot1q 999 exact
  rewrite pop 1
  connect ip interface te1.999
  ```

- **На HQ-SW:**  
  *(Перед настройкой убедитесь, что интерфейс `ens3` отключён через nmtui)*
  ```bash
  ovs-vsctl add-br ovs0
  ovs-vsctl add-port ovs0 ens3
  ovs-vsctl set port ens3 vlan_mode=native-untagged tag=999 trunks=999,100,200
  ovs-vsctl add-port ovs0 ovs0-vlan999 tag=999 -- set Interface ovs0-vlan999 type=internal
  ifconfig ovs0-vlan999 inet 192.168.0.2/29 up
  ```

#### Настройка VLAN для серверов и клиентов

- **VLAN 100 для HQ-SRV:**
  - **На HQ-RTR:**
    ```bash
    int te1.100
    ip add 192.168.1.1/26
    port te1
    service-instance te1.100
    encapsulation dot1q 100 exact
    rewrite pop 1
    connect ip interface te1.100
    ```
  - **На HQ-SW:**  
    *(Убедитесь, что `ens4` отключён в nmtui)*
    ```bash
    ovs-vsctl add-port ovs0 ens4
    ovs-vsctl set port ens4 tag=100 trunks=100
    ovs-vsctl add-port ovs0 ovs0-vlan100 tag=100 -- set Interface ovs0-vlan100 type=internal
    ifconfig ovs0-vlan100 up
    ```

- **VLAN 200 для HQ-CLI:**
  - **На HQ-RTR:**
    ```bash
    int te1.200
    ip add 192.168.2.1/28
    port te1
    service-instance te1.200
    encapsulation dot1q 200 exact
    rewrite pop 1
    connect ip interface te1.200
    ```
  - **На HQ-SW:**  
    *(Убедитесь, что `ens5` отключён в nmtui)*
    ```bash
    ovs-vsctl add-port ovs0 ens5
    ovs-vsctl set port ens5 tag=200 trunks=200
    ovs-vsctl add-port ovs0 ovs0-vlan200 tag=200 -- set Interface ovs0-vlan200 type=internal
    ifconfig ovs0-vlan200 up
    ```

> **Отчёт:** Сведения по настройке коммутатора и выбору реализации разделения на VLAN занесите в отчёт.

---

### 5. Настройка безопасного удалённого доступа

- **Изменение SSH порта и SELinux:**
  - Перевод SELinux в режим `permissive`:
    ```bash
    setenforce 0
    ```
    *(Также изменить в файле `/etc/selinux/config`)*
  - Изменение порта в `/etc/ssh/sshd_config`:
    ```
    Port 2024
    AllowUsers sshuser
    MaxAuthTries 2
    Banner /etc/ssh/sshd_banner
    ```
  - Создание баннера:
    ```bash
    echo "Authorized access only" > /etc/ssh/sshd_banner
    systemctl restart sshd
    ```

---

### 6. Настройка IP-туннеля между офисами

- **На HQ-RTR:**
  ```bash
  interface tunnel.1
  ip add 172.16.0.1/30
  ip ospf network broadcast
  ip ospf mtu-ignore
  ip tunnel 172.16.4.2 172.16.5.2 mode gre
  exit
  conf t
  router ospf 1
    ospf router-id 172.16.0.1
    network 172.16.0.0 0.0.0.3 area 0
    network 192.168.0.0 0.0.0.255 area 0
    network 192.168.1.0 0.0.0.63 area 0
    network 192.168.2.0 0.0.0.15 area 0
    passive-interface default
    no passive-interface tunnel.1
  ```

- **На BR-RTR:**
  ```bash
  interface tunnel.1
  ip add 172.16.0.2/30
  ip ospf mtu-ignore
  ip ospf network broadcast
  ip tunnel 172.16.5.2 172.16.4.2 mode gre
  exit
  conf t
  router ospf 1
    ospf router-id 172.16.0.2
    network 10.0.0.0 0.0.0.3 area 0
    network 192.168.3.0 0.0.0.31 area 0
    passive-interface default
    no passive-interface tunnel.1
  ```

> **Примечание:** Выбор технологии – GRE или IP-in-IP – производится по усмотрению.

---

### 7. Динамическая маршрутизация

- **Цель:** Обеспечить доступ ресурсов одного офиса к другому посредством протокола link-state (например, OSPF).

- **Настройка аутентификации OSPF на EcoRouter:**

  **На HQ-RTR:**
  ```bash
  router ospf 1
    area 0 authentication
    ex
  interface tunnel.1
    ip ospf authentication-key ecorouter
  wr mem
  ```

  **На BR-RTR:**
  ```bash
  router ospf 1
    area 0 authentication
    ex
  interface tunnel.1
    ip ospf authentication-key ecorouter
  wr mem
  ```

> **Отчёт:** Занесите сведения о настройке и защите протокола в отчёт.

---

### 8. Динамическая трансляция адресов (NAT)

- **На EcoRouter HQ-RTR:**
  ```bash
  ip nat pool nat1 192.168.0.2-192.168.0.6
  ip nat source dynamic inside-to-outside pool nat1 overload interface ISP
  ip nat pool nat1 192.168.1.2-192.168.1.64
  ip nat source dynamic inside-to-outside pool nat1 overload interface ISP
  ip nat pool nat1 192.168.2.2-192.168.2.16
  ip nat source dynamic inside-to-outside pool nat1 overload interface ISP
  ```
- **На EcoRouter BR-RTR:**
  ```bash
  ip nat pool nat3 192.168.3.2-192.168.3.31
  ip nat source dynamic inside-to-outside pool nat3 overload interface ISP
  ```

- **Настройка NAT на интерфейсах:**

  **HQ-RTR:**
  ```bash
  en
  conf t
  int ISP
    ip nat outside
  exit
  int te1.999
    ip nat inside
  exit
  int te1.100
    ip nat inside
  exit
  int te1.200
    ip nat inside
  ```

  **BR-RTR:**
  ```bash
  en
  conf t
  int ISP
    ip nat outside
  exit
  int SRV
    ip nat inside
  exit
  ```

- **Настройка шлюзов на серверах:**
  - **HQ-SRV:** Шлюз – `192.168.1.1/26`
  - **BR-SRV:** Шлюз – `192.168.3.1/27`

---

### 9. Настройка DHCP-сервера

- **Для офиса HQ (на HQ-RTR):**
  ```bash
  ip pool dhcpHQ 192.168.2.2-192.168.2.16
  en
  conf t
  dhcp-server 1
    pool dhcpHQ 1
      domain-name au-team.irpo
      mask 255.255.255.240
      gateway 192.168.2.1
      dns 192.168.1.2
  exit
  wr mem
  ```
- **Клиентом является HQ-CLI:**  
  На интерфейсе `te1.200` добавьте:
  ```bash
  dhcp-server 1
  ```

> **Примечания:**
> - Исключите из выдачи адрес маршрутизатора.
> - DNS-сервер для HQ-CLI – HQ-SRV.
> - DNS-суффикс – `au-team.irpo`.

---

### 10. Настройка DNS-сервера

- **Установка и базовая настройка:**
  ```bash
  dnf install bind -y
  nano /etc/named.conf
  ```
  ![named1.png](https://github.com/taharakuro/DEMO25/blob/main/named1.png)
  ![named2.png](https://github.com/taharakuro/DEMO25/blob/main/named2.png)

- **Настройка зон:**
  - Создаём директорию и копируем шаблоны:
    ```bash
    mkdir /var/named/master
    cp /var/named/named.localhost /var/named/master/au-team.irpo
    cp /var/named/named.loopback /var/named/master/0.168.192.zone
    chown -R root:named /var/named/master
    chmod -R 750 /var/named/master
    ```
  - Пример файла зоны `au.team.irpo`:
    ```zone
    $TTL    1D
    @       IN      SOA     au-team.irpo. root.au-team.irpo. (
                              0          ; serial
                              1D         ; refresh
                              1H         ; retry
                              1W         ; expire
                              3H )       ; minimum
            IN      NS      @
            IN      A       127.0.0.1
    hq-rtr  IN      A       192.168.0.1
    br-rtr  IN      A       192.168.3.1
    hq-srv  IN      A       192.168.1.2
    hq-cli  IN      A       192.168.2.2
    br-srv  IN      A       192.168.3.2
    moodle  IN      CNAME   hq-rtr
    wiki    IN      CNAME   hq-rtr
    _ldap._tcp   IN   SRV   0   100   389   br-srv.au-team.irpo.
    _kerberos   IN   SRV   0 100 88   br-srv.au-team.irpo.
    _kerberos   IN   TXT   "AU-TEAM.IRPO"
    ```
  - Пример файла зоны `168.192.zone`:
    ```zone
    $TTL    1D
    @       IN      SOA     au-team.irpo. root.au-team.irpo. (
                              0          ; serial
                              1D         ; refresh
                              1H         ; retry
                              1W         ; expire
                              3H )       ; minimum
            IN      NS      @
    1.0     IN      PTR     hq-rtr.au-team.irpo.
    2.1     IN      PTR     hq-srv.au-team.irpo.
    2.2     IN      PTR     hq-cli.au-team.irpo.
    ```
- **Запуск сервиса:**
  ```bash
  systemctl enable --now named
  named-checkconf -z
  ```

- **Настройка клиентов:**  
  На HQ-CLI и HQ-RTR установить DNS-сервер: `192.168.1.2` (при необходимости удалить `8.8.8.8`).

- **Таблица записей (см. Таблицу 2 в исходном файле):**
  | Устройство | Запись                   | Тип    |
  |------------|--------------------------|--------|
  | HQ-RTR     | hq-rtr.au-team.irpo      | A, PTR |
  | BR-RTR     | br-rtr.au-team.irpo      | A      |
  | HQ-SRV     | hq-srv.au-team.irpo      | A, PTR |
  | HQ-CLI     | hq-cli.au-team.irpo      | A, PTR |
  | BR-SRV     | br-srv.au-team.irpo      | A      |
  | HQ-RTR     | moodle.au-team.irpo      | CNAME  |
  | HQ-RTR     | wiki.au-team.irpo        | CNAME  |

- **DNS пересылки:**  
  Используйте общедоступный DNS-сервер в качестве форвардера (например, 8.8.8.8).

---

### 11. Настройка часового пояса

- **На HQ-SRV:**
  ```bash
  timedatectl set-timezone <Ваш_часовой_пояс>
  ```

---

## Модуль №2: Организация сетевого администрирования ОС

### 1. Доменный контроллер Samba

#### Подготовка сервера

- Перевод SELinux в `permissive`:
  ```bash
  setenforce 0
  nano /etc/selinux/config  # замените режим с enforcing на permissive
  ```
- Настройка сети через `nmtui`:  
  - Добавьте второй DNS-сервер: `192.168.1.1/26`  
  - Укажите домен поиска: `au-team.irpo`  
  - Перезапустите сетевой интерфейс.

- Установка необходимых пакетов:
  ```bash
  dnf install samba* krb5* -y
  ```

#### Создание домена через `samba-tool`

- Удалите старый файл конфигурации:
  ```bash
  rm /etc/samba/smb.conf
  ```
- Запустите:
  ```bash
  samba-tool domain provision --use-rfc2307 --interactive
  ```
  *(Следуйте интерактивным подсказкам для задания realm, доменного имени и проч.)*

- **Запуск служб:**
  ```bash
  systemctl enable samba --now
  systemctl status samba
  ```

#### Управление пользователями и группами

- **Создание пользователей для офиса HQ:**  
  Пользователи с именами формата `user№.hq`.
  - Создание пользователя:
    ```bash
    samba-tool user create user#.hq
    smbpasswd -a user#.hq
    smbpasswd -e user#.hq
    ```
- **Создание группы и добавление пользователей:**
  ```bash
  samba-tool group add hq
  samba-tool group addmembers hq user#.hq
  ```
- **Проверка подключения:**
  ```bash
  smbclient -L localhost -U%
  ```

- **Присоединение машины HQ-CLI к домену:**  
  *(Смотрите инструкцию по ссылке: [RedOS Samba Domain](https://redos.red-soft.ru/base/redos-7_3/7_3-administation/7_3-domain-redos/7_3-domain-config/7_3-redos-in-samba/?nocache=1730793368537))*

- **Импорт пользователей из файла `users.csv`:**  
  Файл располагается на BR-SRV в папке `/opt`.

---

### 2. Сконфигурируйте файловое хранилище:
 ### Настройка проивзодится на HQ-SRV:
  Перед тем как начать проверяем, что установлены следюущие пакеты
  dnf isntall mdadm nfs-utils -y
## •	При помощи трёх дополнительных дисков, размером 1Гб каждый, на HQ-SRV сконфигурируйте дисковый массив уровня 5  
    mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd     
 ![mdadmcreate](https://github.com/dizzamer/DEMO2025/blob/main/mdadm_create.png)    
 ![mdaddetail](https://github.com/dizzamer/DEMO2025/blob/main/mdadm_detail.png)  
## •	Имя устройства – md0, конфигурация массива размещается в файле /etc/mdadm.conf  
    mdadm --detail --scan >> /etc/mdadm.conf      
 ## •	Обеспечьте автоматическое монтирование в папку /raid5  
    Добавляем в /etc/fstab:    
    nano /etc/fstab  
    /dev/md0 /raid5 ext4 defaults 0 0  
![fstab](https://github.com/dizzamer/DEMO2025/blob/main/fstab.png)  
 ## •	Создайте раздел, отформатируйте раздел, в качестве файловой системы используйте ext4  
     mkfs.ext4 /dev/md0  
   ![mkfs](https://github.com/dizzamer/DEMO2025/blob/main/mkfs.png)   
 ## •	Создаем точку монтирования и примонтируемся     
    mkdir -p /raid5   
    mount -a   
  ![mount](https://github.com/dizzamer/DEMO2025/blob/main/mount.png)   
 ### •	Настройте сервер сетевой файловой системы(nfs), в качестве папки общего доступа выберите /raid5/nfs, доступ для чтения и записи для всей сети в сторону HQ-CLI   
  ## •	Создаем папку для NFS  
    mkdir -p /raid5/nfs  
    chmod 777 /raid5/nfs  
 ![mkdir_nfs](https://github.com/dizzamer/DEMO2025/blob/main/mkdir_nfs.png)  
 ## Настройка экспорта  
    Добавляем в /etc/exports:  
    nano /etc/exports  
    /raid5/nfs 192.168.1.64/28(rw,sync,insecure,nohide,all_squash,no_subtree_check)
![exports](https://github.com/dizzamer/DEMO2025/blob/main/etcexports.png)  
  ## Применяем изменения и перезагружаем службу
    exportfs -rav  
    systemctl restart nfs-server  
 ### Настройка проивзодится на HQ-CLI:
  ## •	На HQ-CLI настройте автомонтирование в папку /mnt/nfs  
      Добавляем в /etc/fstab:    
      nano /etc/fstab  
      hq-srv:/raid5/nfs /mnt/nfs nfs defaults 0 0
  ![fstab_hqcli](https://github.com/dizzamer/DEMO2025/blob/main/fstab_hqcli.png)  
  ## Создаем точку монтирования и примонтируемся  
    mkdir -p /mnt/nfs  
    mount -a 
  ![mountdir_hqcli](https://github.com/dizzamer/DEMO2025/blob/main/nount_cli.png)  
  ## Проверка монтирования
      После этого при создании файла на клиенте, он должен появляться и на сервере
   •	Основные параметры сервера отметьте в отчёте  

---

### 3. Настройка службы сетевого времени (chrony)

- **На HQ-RTR:**
  ```bash
  en
  conf t
  ntp server 172.16.14.1
  ntp timezone UTC+3
  end
  wr mem
  ```
•	На HQ-RTR настройте сервер chrony, выберите стратум 5  
•	В качестве клиентов настройте HQ-SRV, HQ-CLI, BR-RTR, BR-SRV  
## 4.	Сконфигурируйте ansible на сервере BR-SRV  
  ### Настройка подключения по ssh BR-RTR | HQ-RTR
     Настройка производится на HQ-RTR:  
     en  
     conf  
     security-profile 1  
     rule 1 permit tcp any eq 22 any  
     end  
     wr mem  
     configure
     ip vrf vrf0  
     transport input ssh    
     security 1 vrf vrf0  
     end  
     wr mem  
     conf
     no security default  
     Настройка производится на BR-RTR: 
     conf  
     security-profile 1  
     rule 1 permit tcp any eq 22 any  
     end  
     wr mem  
     configure
     ip vrf vrf0  
     transport input ssh    
     security 1 vrf vrf0  
     end  
     wr mem  
     conf
     no security default
  ## •	Сформируйте файл инвентаря, в инвентарь должны входить HQ-SRV, HQ-CLI, HQ-RTR и BR-RTR  
   ### Настройка производится на BR-SRV:  
    • Рабочий каталог ansible должен располагаться в /etc/ansible  
      dnf install ansible -y  
      1) В файле можно прописывать как ip адреса так и имена хостов, сделаем следующим образом.  
      Так как у нас порт для покдлючения к серверам и клиентам 2024, укажим необходимые переменные для подключения  
      Для роутера так же указываем переменные, для подключения к роутерам будем использовать пользователя net_admin
      nano /etc/ansible/inventory.ini  
      [clients]
      hq-cli ansible_host=192.168.1.65
        
      [servers]
      hq-srv ansible_host=192.168.0.2
         
      [routers]
      hq-rtr ansible_host=192.168.0.62
      br-rtr ansible_host=172.16.5.1
         
      [clients:vars]
      ansible_port=2024
      ansible_user=sshuser
         
      [servers:vars]
      ansible_port=2024
      ansible_user=sshuser
 
      [routers:vars]
      ansible_user=net_admin
      ansible_password=P@$$word
   ![inventory](https://github.com/dizzamer/DEMO2025/blob/main/inventoryini.png) 
   ###  Настройка подключения по ключам на BR-SRV
    2) Подключение к хостам осуществляется по протоколу ssh с помощью rsa ключей.  
    Для начала переходим в ранее созданно пользователя sshuser командой:
    su sshuser
    Сгенерировать серверный ключ можно командой ниже. При её выполнении везде нажмите Enter.
    ssh-keygen
    3) Далее нужно распространить ключ на все подключенные хосты.  
    Распространить ключи на хосты можно командой:  
    На роутеры ключи пробрасывать не нужно, для них подключение по паролю!
    ssh-copy-id sshuser@{hq-srv, hq-cli} 
    где:  
    sshuser - это пользователь, от имени которого будут выполняться плейбуки;  
    server - IP-адрес хоста.  
  ### •	Все указанные машины должны без предупреждений и ошибок отвечать pong на команду ping в ansible посланную с BR-SRV  
    Пингуем удаленные хосты с помощью Ansible находясь в пользователе sshuser:  
    ansible -i /etc/ansible/inventory.ini all -m ping 
    В результате под каждым хостом должно быть написано "ping": "pong".  
![inventory](https://github.com/dizzamer/DEMO2025/blob/main/ansubleping.png) 
## 5.	Развертывание приложений в Docker на сервере BR-SRV. 
    Установка необходимых пакетов:  
    dnf install docker-ce docker-ce-cli docker-compose -y  
    systemctl enable docker --now
    Добавляем текущего пользователя в группу докер, текущий пользователь - student   
    usermod -aG docker $USER  
### •	Создайте в домашней директории пользователя файл wiki.yml для приложения MediaWiki.  
     •	Средствами docker compose должен создаваться стек контейнеров с приложением MediaWiki и базой данных.  
     •	Используйте два сервиса  
     •	Основной контейнер MediaWiki должен называться wiki и использовать образ mediawiki  
     •	Файл LocalSettings.php с корректными настройками должен находиться в домашней папке пользователя и автоматически 
      монтироваться в образ.  
     •	Контейнер с базой данных должен называться mariadb и использовать образ mariadb.  
     •	Он должен создавать базу с названием mediawiki, доступную по стандартному порту, пользователя wiki с паролем   
     WikiP@ssw0rd должен иметь права доступа к этой базе данных  
     •	MediaWiki должна быть доступна извне через порт 8080.  
     Для того, чтобы MediaWiki была доступна извен через порт 8080, нужно в ports сначала указывать 8080  
     Развертывание производится на сервере BR-SRV:   
     touch /home/student/wiki.yml  
     nano /home/student/wiki.yml  
      services:
      MediaWiki:
        container_name: wiki
        image: mediawiki
        restart: always
        ports: 
          - 8080:80
        links:
          - database
        volumes:
          - images:/var/www/html/images
          # - ./LocalSettings.php:/var/www/html/LocalSettings.php
      database:
        container_name: mariadb
        image: mariadb
        environment:
          MYSQL_DATABASE: mediawiki
          MYSQL_USER: wiki
          MYSQL_PASSWORD: WikiP@ssw0rd
          MYSQL_RANDOM_ROOT_PASSWORD: 'yes'
        volumes:
          - dbvolume:/var/lib/mysql
    volumes:
      dbvolume:
          external: true
      images:
      Поднимаем стек контейнеров с помощью команды: 
      docker compose -f wiki.yml up -d  
 ![wikiyml](https://github.com/dizzamer/DEMO2025/blob/main/wikiyml.png)  
### Настройка mediawiki после успешного поднятия контейнеров  
  Переходим по доменному имени или адреса нашего сервера, должны увидеть такую картину:  
 ![wikistart](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki.png)  
  Далее настройка выглядит следуюшим образом:  
  На скриншоте ниже здесь этап проверки всего необходимого для работы MediaWiki, проверка должна пройти успешно  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki2.png)  
  Далее необходимо указать в хосте базы данных то, как у вас называется контнейнер на сервере, у меня mariadb  
  Проверить можно зайдя на сервер и выполнить команду docker ps:  
  ![hostwiki](https://github.com/dizzamer/DEMO2025/blob/main/hostwiki.png)  
  Указываем, все как по заданию и жмем далее:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki3.png)  
  Далее будет такое у вас как снизу, жмем далее:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki4.png)  
  Настройка дальше на скриншоте снизу:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki5.png)  
  На след скриншоте жмем далее:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki6.png)  
  Настройка базы данных должна быть выполнена успешно, как на скриншоте ниже:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki77.png)  
  Далее у вас должен скачаться файл LocalSettings.php автомтаически:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/mediawiki8.png)  
  Потом переходим в директории загрузки:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettings.png)  
  Открывем директорию через терминал:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingsterm.png)  
  Перекидываем его через scp следующим образом:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingsterm2.png)  
  Переходим на сервер и убеждаем, что файл находится рядом с нашим wiki.yml:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingsterm3.png)  
  Далее раскоменчиваем строку в файле wiki.yml:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/localsettingswikiyml.png)   
  Запускаем стек контейнеров командой docker-compose -f wiki.yml up -d    
  Далее переходим по доменному имени или адреса нашего сервера и наблюдаем вот это:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/wikidemo.png)  
  Для проверки того, что все получилось входим под админской учеткой Wiki:WikiP@ssw0rd:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/wikidemologin.png)  
  Должно получиться вот так:  
  ![wikinext](https://github.com/dizzamer/DEMO2025/blob/main/wikiuser.png)  
### 6. Статическая трансляция портов

- **На BR-RTR (для сервиса wiki):**
  ```bash
  ip nat destination static tcp 192.168.3.2 80 192.168.3.2 8080
  ```
- **На HQ-RTR (для сервиса SSH на HQ-SRV):**
  ```bash
  ip nat destination static tcp 192.168.1.2 2024 192.168.1.2 2024
  ```
- **На BR-RTR (для сервиса SSH на BR-SRV):**
  ```bash
  ip nat destination static tcp 192.168.3.2 2024 192.168.3.2 2024
  ```

---

### 7. Настройка Moodle

- **Требования:**
  - Веб-сервер: Apache.
  - СУБД: MariaDB.
  - Создать базу данных `moodledb`.
  - Создать пользователя `moodle` с паролем `P@ssw0rd` и предоставить права.
  - Пользователю `admin` задать пароль `P@ssw0rd`.
  - На главной странице отобразить номер рабочего места (арабская цифра).

> **Отчёт:** Основные параметры внесите в отчёт.

---

### 8. Обратный прокси на nginx

- **Для перенаправления запросов к `moodle.au-team.irpo` на HQ-SRV:**
  - **На HQ-RTR:**
    ```bash
    en
    conf t
    filter-map policy ipv4 moodle 1
      match 80 172.16.4.1/28 192.168.1.2/26 dscp 0
      set redirect hq-rtr.moodle.au-team.irpo
    end
    wr mem

    en
    conf t
    redirect-url SITEREDIRECT
      url hq-rtr.moodle.au-team.irpo
    end
    wr mem
    ```
- **Для перенаправления запросов к `wiki.au-team.irpo` на BR-SRV:**  
  *(Настройка аналогична, с учётом нужного порта и IP-адреса.)*

---

### 9. Установка Яндекс.Браузера

- **Требование:**  
  Установить Яндекс.Браузер для организаций удобным способом и зафиксировать результат в отчёте.
