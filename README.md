# Лабораторная работа № 3. KVM и Docker

**Студент:** Фролкин Никита  
**Группа:** 6411-100503D

---

## 1. Среда выполнения

* **Хост KVM:** Ubuntu Server 24.04 (виртуальная машина Hyper‑V)  
* **Версии ПО:**  
  * `qemu-kvm`: 1:8.2.2+ds-0ubuntu1.16  
  * `libvirt-daemon`: 8.2.2  
* **Гостевая ВМ:** Ubuntu Server 24.04  
  * Имя ВМ: `ubuntu-vm1`  
  * Ресурсы: 2 vCPU, 2 ГБ ОЗУ  
  * Диски:  
    * vdb — 20 ГБ (дополнительный, raw)  
  * Сеть: NAT (default), IP гостя: **192.168.122.137**

---

## 2. Задание 1. Подготовка хоста KVM

**Теория:**  
KVM (Kernel-based Virtual Machine) — это модуль ядра Linux, превращающий ОС в гипервизор. Совместно с библиотеками libvirt и эмулятором QEMU он позволяет создавать и управлять виртуальными машинами с аппаратной поддержкой виртуализации (Intel VT-x / AMD-V). Утилита `virsh` — основной интерфейс командной строки для libvirt.

**Установка пакетов:**
```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst bridge-utils
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

**Проверка статуса libvirtd:**
```bash
$ systemctl status libvirtd
● libvirtd.service - Virtualization daemon
     Active: active (running) since ...
```
[Скриншот 1: systemctl status libvirtd]

**Версия virsh и список ВМ:**
```bash
$ virsh version
Compiled against library: libvirt 8.2.2
Using library: libvirt 8.2.2
Using API: QEMU 8.2.2
Running hypervisor: QEMU 8.2.2

$ virsh list --all
 Id   Name           State
----------------------------
 1    guest-ubuntu   running
 2    ubuntu-vm      running
```
[Скриншот 2: virsh version и virsh list --all]

---

## 3. Задание 2. Гостевая ВМ

**Теория:**  
`virt-install` — утилита для создания ВМ из командной строки. Параметр `--cdrom` подключает ISO‑образ как виртуальный CD‑ROM, а `--graphics vnc` позволяет наблюдать за установкой через удалённый рабочий стол. Формат образа диска `qcow2` экономит место за счёт динамического выделения блоков (Copy‑On‑Write).

**Создание ВМ командой virt-install с VNC:**
```bash
sudo virt-install \
  --name ubuntu-vm1 \
  --memory 2048 \
  --vcpus 2 \
  --disk size=10,format=qcow2 \
  --cdrom /tmp/ubuntu24.iso \
  --os-variant ubuntu24.04 \
  --network network=default \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole
```

**Подключение к установщику через VNC:**
```bash
vncviewer localhost:5901
```
[Скриншот 3: окно VNC с началом установки Ubuntu Server]

По завершении установки ВМ отображается в списке:
```bash
$ virsh list --all
 Id   Name         State
----------------------------
 4    ubuntu-vm1   running
```

---

## 4. Задания 3–4. Пользователь user и SSH по ключу

**Теория:**  
Аутентификация по SSH-ключам безопаснее парольной: используется пара «открытый ключ (на сервере) – закрытый ключ (у клиента)». Отключение `PasswordAuthentication` запрещает вход по паролю, оставляя только ключевую проверку.

**Создание пользователя:**
```bash
$ sudo adduser user
Adding user `user' ...
Adding new group `user' (1001) ...
Adding new user `user' (1001) with group `user' ...
...
$ id user
uid=1001(user) gid=1001(user) groups=1001(user)
```

**Настройка SSH по ключу и отключение парольной аутентификации.**

На хосте сгенерирован ключ и скопирован на гостя:
```bash
ssh-keygen -t ed25519
ssh-copy-id user@192.168.122.137
```

В гостевой ВМ в файле `/etc/ssh/sshd_config` установлены:
```
PubkeyAuthentication yes
PasswordAuthentication no
```
[Скриншот 4: фрагмент sshd_config с PasswordAuthentication no]

После перезапуска `sshd` вход по ключу работает, попытка входа без ключа отклоняется.

**Проверка входа:**
```bash
$ ssh user@192.168.122.137
Welcome to Ubuntu 24.04.1 LTS ...
user@user:~$
```
[Скриншот 5: окно терминала с успешным входом по SSH без запроса пароля]

---

## 5. Задание 5. Конфигурация гостевой ВМ

**Теория:**  
`lscpu` показывает архитектуру процессора и количество ядер. `free -h` — использование оперативной памяти. `lsblk` отображает блочные устройства, а `df -h` — занятое место на файловых системах. Сетевые настройки диагностируются через `ip a` (адреса интерфейсов) и `ip r` (таблица маршрутизации).

Выполнены команды сбора информации:

**lscpu:**
```
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  CPU(s):                 2
  ...
```
**free -h:**
```
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       166Mi       1.5Gi       1.0Mi       293Mi       1.7Gi
Swap:          1.0Gi          0B       1.0Gi
```
**lsblk:**
```
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda                       253:0    0   10G  0 disk
├─vda1                    253:1    0    1M  0 part
├─vda2                    253:2    0  1.8G  0 part /boot
└─vda3                    253:3    0  8.2G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0  8.2G  0 lvm  /
vdb                       253:16   0   20G  0 disk  ← доп. диск
```
**df -h:**
```
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  8.0G  2.3G  5.8G  29% /
/dev/vda2                          1.7G   96M  1.5G   6% /boot
```
**ip a:**
```
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP ...
    inet 192.168.122.137/24 metric 100 brd 192.168.122.255 scope global dynamic ens3
```
**ip r:**
```
default via 192.168.122.1 dev ens3 proto dhcp src 192.168.122.137 metric 100
192.168.122.0/24 dev ens3 proto kernel scope link src 192.168.122.137 metric 100
```
[Скриншот 6: коллаж из терминала с выводами lscpu, free -h, lsblk и ip a]

**Описание ресурсов ВМ:**  
Гостевая система оснащена 2 виртуальными процессорами, 2 ГБ оперативной памяти, системным диском на 10 ГБ (используется LVM), дополнительным диском vdb на 20 ГБ (пока не смонтирован). Сеть построена через NAT, интерфейс ens3 получил IP‑адрес из подсети 192.168.122.0/24, шлюз по умолчанию — 192.168.122.1.

---

## 6. Задание 6. Дополнительный диск

**Теория:**  
Формат `raw` предоставляет «сырой» байтовый образ диска фиксированного размера, что исключает проблемы с восприятием размера гостевой ОС. Напротив, `qcow2` динамически расширяется, но иногда требует пересканирования шины. Подключение диска через `virsh attach-disk` с флагом `--config` сохраняет настройки после перезагрузки.

**На хосте KVM:**
```bash
# Создание диска (использован raw для совместимости)
sudo qemu-img create -f raw /var/lib/libvirt/images/test-disk.img 20G

# Подключение к ВМ
virsh attach-disk ubuntu-vm1 /var/lib/libvirt/images/test-disk.img vdb --targetbus virtio --cache none --config
```

**После перезапуска гостя:**
```bash
user@user:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    253:0    0   10G  0 disk ...
vdb    253:16   0   20G  0 disk            ← диск обнаружен
```
[Скриншот 7: lsblk с vdb размером 20 ГБ]

---

## 7. Задание 7. Разметка GPT, файловая система ext4 и монтирование в /disk

**Теория:**  
GPT (GUID Partition Table) — современный стандарт разметки диска, поддерживающий диски большого объёма. Файловая система ext4 — журналируемая, стабильная и широко используемая в Linux. Запись в `/etc/fstab` обеспечивает автоматическое монтирование при загрузке.

В гостевой ВМ выполнены следующие команды:
```bash
sudo parted /dev/vdb mklabel gpt
sudo parted /dev/vdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/vdb1
sudo mkdir /disk
sudo mount /dev/vdb1 /disk
echo '/dev/vdb1 /disk ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

**Проверка монтирования:**
```bash
$ findmnt /disk
TARGET SOURCE    FSTYPE OPTIONS
/disk  /dev/vdb1 ext4   rw,relatime
```
[Скриншот 8: findmnt /disk]

**Фрагмент /etc/fstab:**
```
/dev/vdb1  /disk  ext4  defaults  0  2
```
[Скриншот 9: последняя строка fstab]

---

## 8. Задание 8. Права на /disk для пользователя user

**Теория:**  
В Linux каждый файл и каталог имеют владельца и группу. Команда `chown` меняет владельца, а `chmod` устанавливает права доступа (чтение, запись, исполнение). Без корректных прав пользователь не сможет создавать или изменять файлы.

```bash
sudo chown -R user:user /disk
sudo chmod 755 /disk
```

**Проверка создания файла:**
```bash
user@user:~$ touch /disk/test_file
user@user:~$ ls -l /disk
total 16
drwxr-xr-x 2 user user 4096 Apr 25 05:59 .
drwxr-xr-x 3 root root 4096 Apr 25 05:58 ..
drwxr-xr-x 2 user user 4096 Apr 25 06:00 nginx-site
-rw-rw-r-- 1 user user    0 Apr 25 06:01 test_file
```
[Скриншот 10: ls -l /disk с файлом test_file, принадлежащим user]

---

## 9. Задание 9. Docker

**Теория:**  
Docker — платформа контейнеризации, позволяющая упаковывать приложения со всеми зависимостями в изолированные контейнеры. Контейнеры используют ядро хостовой ОС, но работают в собственном пространстве имён, что обеспечивает лёгкость и переносимость.

**Установка Docker и добавление пользователя в группу docker:**
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker user
```

**Применение прав группы (вход заново):**
```bash
user@user:~$ newgrp docker
```

**Проверка:**
```bash
$ docker --version
Docker version 24.0.7, build 24.0.7-0ubuntu4

$ systemctl status docker
● docker.service - Docker Application Container Engine
   Active: active (running) since ...

$ docker run hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```
[Скриншот 11: вывод docker --version, systemctl status docker и docker run hello-world от имени user]

---

## 10. Задание 10. Nginx в контейнере с фамилией

**Теория:**  
Nginx — высокопроизводительный веб-сервер. Флаг `-v` (volume) в Docker монтирует каталог хоста внутрь контейнера, позволяя отделить код сайта от образа. Параметр `:ro` делает монтирование доступным только для чтения, что повышает безопасность.

**Подготовка контента на /disk:**
```bash
mkdir -p /disk/nginx-site
echo '<html><body><h1>Фамилия: Фролкин</h1></body></html>' > /disk/nginx-site/index.html
```

**Запуск контейнера:**
```bash
docker run -d --name mynginx -p 80:80 -v /disk/nginx-site:/usr/share/nginx/html:ro nginx
```

**Проверка с хоста KVM:**
```bash
$ curl http://192.168.122.137
<html><body><h1>Фамилия: Фролкин</h1></body></html>
```
[Скриншот 12: веб-браузер с открытой страницей или терминал с выводом curl]

---

## 11. Выводы

В ходе лабораторной работы на хосте KVM была создана гостевая виртуальная машина Ubuntu Server 24.04.  
* Настроен SSH-доступ по ключу с отключением парольной аутентификации.  
* Изучена конфигурация гостя (процессор, память, диски, сеть).  
* К гостю подключён дополнительный виртуальный диск на 20 ГБ, на нём создан GPT-раздел с ext4 и настроено постоянное монтирование в `/disk`.  
* Пользователь `user` получил полные права на этот раздел.  
* В гостевой системе установлен Docker, пользователь `user` включён в группу `docker` и успешно запускает контейнеры.  
* В контейнере nginx развёрнут веб-сервер, отдающий HTML-страницу с фамилией студента; статические файлы размещены на дополнительном диске `/disk`.  

**Трудности:**  
- При правке размера диска гостевая ОС не сразу увидела новый объём, потребовалось пересоздание ВМ с импортом существующего диска.  
- Тонкие диски qcow2 вызывали несоответствие отображаемого размера в гостевой системе, поэтому для дополнительного диска был выбран формат raw.  
