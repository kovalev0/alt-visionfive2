# Подготовка рабочего места

[На главную](INDEX.md) · [Урок 01 →](lab01-board/README.md)

---

## Содержание

### Обязательно

1. [Подготовка образа ALT](#1-подготовка-образа-alt)
2. [Запись образа на micro-SD карту](#2-запись-образа-на-micro-sd-карту)
3. [Подключение к консоли](#3-подключение-к-консоли)
   - [Через UART](#31-подключение-через-uart-консоль)
   - [Через HDMI](#32-подключение-через-hdmi-консоль)
4. [Проверка IP-адреса и настройка SSH](#4-проверка-ip-адреса-и-настройка-ssh)
5. [Проверка окружения на плате](#5-проверка-окружения-на-плате)
6. [Установка инструментов](#6-установка-инструментов)
7. [Настройка окружения для сборки модулей](#7-настройка-окружения-для-сборки-модулей)

###  Дополнительно

8. [Перенос корневой системы на NVMe SSD](#8-Перенос-корневой-системы-на-NVMe-SSD)
9. [Правка образа (для разработчиков)](#9-правка-образа-для-разработчиков)
10. [Сборка модуля ядра в изолированном окружении](#10-сборка-модуля-ядра-в-изолированном-окружении)
11. [Отладка ядра и модулей через KGDB](#11-отладка-ядра-и-модулей-через-kgdb)
---

## 1. Подготовка образа ALT

Скачайте готовый образ по ссылке:
**https://drive.google.com/file/d/1APTkK_40jZvS-hFhrGV20zVD9wph-mnk/view?usp=drive_link**

Сохраните архив в директорию `~/alt-visionfive2/image` и проверьте
контрольную сумму:

```bash
$ sha256sum regular-mate-latest-riscv64-visionfive2.img.tar.xz
# Ожидаемый результат:
# 28d577b08aea285f63bca7f9e509aab55e25db75278063724ca6feb6f5db5507
```

> Текущая версия ядра - **6.12.77** ([репозиторий](https://git.altlinux.org/people/kovalev/packages/kernel-image.git?p=kernel-image.git;a=shortlog;h=refs/heads/alt-JH7110_VisionFive2_6.12.y_devel-6.12.77)).

> Архив с rpm пакетами: [6.12.77.tar.gz](https://drive.google.com/file/d/1R8HTj-r8UlAaRLnXUTUl8V1OHExCj4qK/view?usp=sharing).

> Вы можете обновить ядро вручную, следуя инструкциям в секции
> [11.2 Установка отладочного ядра на плату](#112-установка-отладочного-ядра-на-плату),
> заменяя в командах суффикс `6.12.kgdb` на `6.12`.
---

## 2. Запись образа на micro-SD карту

Подключите micro-SD карту **объёмом не менее 16 ГБ** к рабочей машине
и определите имя устройства (например, `/dev/sdX`).

> ⚠️ **Внимание:** команды ниже **безвозвратно уничтожат все данные**
> на указанном устройстве. Дважды проверьте имя устройства перед выполнением.

```bash
# Устанавливаем утилиты для работы с образом и разделами, а также консолью
$ sudo apt-get update
$ sudo apt-get install losetup dmsetup kpartx e2fsprogs cloud-utils screen putty

# Указываем устройство SD-карты (замените sdX на ваше)
$ DEVICE=/dev/sdX

# Стираем существующую таблицу разделов
$ sudo wipefs -a $DEVICE

# Переходим в каталог с образом, распаковываем архив и записываем на карту
$ cd ~/alt-visionfive2/image
$ tar -xf regular-mate-latest-riscv64-visionfive2.img.tar.xz
$ sudo dd if=./regular-mate-latest-riscv64-visionfive2.img of=$DEVICE bs=4M status=progress

# Расширяем корневой раздел (4-й) на всё свободное место карты
$ sudo growpart $DEVICE 4

# Проверяем файловую систему и применяем новый размер
$ sudo e2fsck -f ${DEVICE}4
$ sudo resize2fs ${DEVICE}4
```

---

## 3. Подключение к консоли

Перед первым включением питания платы **важно** правильно настроить
dip-переключатели (Boot Mode) - выставить режим **SD** как на картинке
ниже:

![Boot режим](images/dip_micro_sd.jpg)

В этом режиме плата игнорирует содержимое QSPI (встроенная flash-память)
и загружает актуальные версии прошивки с MicroSD.

В противном случае плата может начать загрузку, используя устаревшие
компоненты из QSPI Flash. Это часто приводит к аппаратным ошибкам и
некорректному определению ресурсов платы.

> **Пример из практики:** При загрузке через заводской QSPI система
> может определить только **4 ГБ** оперативной памяти вместо
> положенных **8 ГБ**. Переключение в режим загрузки с **SD-карты**
> заставляет процессор использовать актуальные версии SPL и U-Boot
> из образа ALT Linux, которые корректно инициализируют контроллер
> памяти и весь объем RAM.

### 3.1 Подключение через UART-консоль

![UART подключение](images/Star5_v2_serial_connection.jpg)

Отладочная консоль VisionFive 2 выведена на **40-пиновую гребёнку GPIO**
(UART0).

Используйте пины **6 (GND)**, **8 (TX)**, **10 (RX)**:

```
 Вид на GPIO-разъём сверху (порты USB смотрят на вас):
 ┌──────────────────────────────────────────────────────┐
 │ 2   4  [6]  [8]  [10]  12  14  ...  (внешний ряд)    │
 │ .   .  GND  TXD  RXD   .   .                         │
 │ 1   3   5    7    9    11  13  ...  (внутренний ряд) │
 └──────────────────────────────────────────────────────┘
           │    │    │
           │    │    └─── RX платы (пин 10) → TX адаптера
           │    └──────── TX платы (пин  8) → RX адаптера
           └───────────── GND      (пин  6) → GND адаптера
```

> **Параметры порта:** 115200 8N1 (8 бит данных, без паритета, 1 стоп-бит)

> **Логический уровень:** 3.3 В — адаптеры с уровнем 5 В использовать
> **нельзя**

> **Не подключайте VCC** от адаптера к плате, если она уже питается через USB-C

Открываем терминал на хост-машине:

```bash
# Вариант 1 — через screen
$ screen /dev/ttyUSB0 115200

# Вариант 2 — через putty
$ putty -serial /dev/ttyUSB0 -sercfg 115200,8,n,1,N
```

После включения питания в консоли появится вывод загрузчика U-Boot, затем
ядра. Если экран пуст — скорее всего, TX и RX перепутаны местами.

---

### 3.2 Подключение через HDMI-консоль

Подключите монитор через HDMI и включите питание платы. Лог загрузки ядра
будет отображаться на экране, после чего запустится графический менеджер с
окном приветствия.

---

## 4. Проверка IP-адреса и настройка SSH

### 4.1 Вход в систему

Доступны два пользователя:

| Логин  | Пароль |
|--------|--------|
| `user` | `1`    |
| `root` | `1`    |

Войдите под пользователем `user`.

---

### 4.2 Определение IP-адреса

Вывести список всех сетевых интерфейсов и их адресов:

```bash
$ ip a
```

Если нужен только IP-адрес конкретного интерфейса:

```bash
# Проводной интерфейс end0 (первый порт Ethernet)
$ ip addr show end0 | grep -Po 'inet \K[\d.]+'
# Пример вывода: 192.168.0.104

# Проводной интерфейс end1 (второй порт Ethernet, если подключён)
$ ip addr show end1 | grep -Po 'inet \K[\d.]+'
```

> Если адрес не отображается — проверьте подключение кабеля и работу
> DHCP на роутере.

---

### 4.3 Подключение по SSH

С хост-машины:

```bash
$ ssh user@<IP-адрес>
# или от root
$ ssh root@<IP-адрес>
```

Для удобства сразу добавьте публичный ключ, чтобы не вводить пароль при
каждом подключении:

```bash
$ ssh-copy-id user@<IP-адрес>
```

> Если `ssh-copy-id` недоступен, можно вручную скопировать содержимое
> `~/.ssh/id_rsa.pub` в файл `~/.ssh/authorized_keys` на плате.

---

## 5. Проверка окружения на плате

Убедитесь, что система загружена корректно и готова к работе:

```bash
# Проверяем версию ядра — должно быть 6.12.77-6.12-alt1.forge.rv64
$ uname -r

# Проверяем архитектуру — должно быть riscv64
$ uname -m

# Если `Makefile` существует — заголовки установлены и можно собирать модули ядра.
$ ls /lib/modules/$(uname -r)/build/Makefile

```

---

## 6. Установка инструментов

Устанавливаем компилятор, отладчики и библиотеки для работы с GPIO и I2C:

```bash
$ sudo apt-get update
$ sudo apt-get install -y \
      gcc make git \
      strace gdb \
      i2c-tools \
      gpio-tools \
      libgpiod2 \
      libgpiod-devel \
      trace-cmd \
      busybox \
      dtc
```
---

## 7. Настройка окружения для сборки модулей

Создаём рабочую директорию для лабораторных работ:

```bash
$ mkdir -p ~/labs
$ cd ~/labs
```

Задаём переменную `KDIR`, которая указывает на заголовки текущего ядра.
Она используется в каждом `Makefile` курса:

```bash
$ export KDIR=/lib/modules/$(uname -r)/build

# Проверяем, что путь существует и корректен
$ echo $KDIR
$ ls $KDIR/Makefile
```

Чтобы переменная устанавливалась автоматически при каждом входе в систему:

```bash
$ echo 'export KDIR=/lib/modules/$(uname -r)/build' >> ~/.bashrc
```

---

## 8. Перенос корневой системы на NVMe SSD
Если в плату установлен NVMe SSD, можно перенести на него корневую
файловую систему — это значительно ускорит сборку модулей и работу
с файлами по сравнению с SD-картой.

> В примерах ниже используется расположение из реального стенда:

> — SD-карта: `/dev/mmcblk1p4` (текущий корень `/`)

> — NVMe-раздел: `/dev/nvme0n1p1` (будущий корень)

---

### 8.1 Подготовка NVMe-раздела

Проверяем текущее состояние диска — он должен быть виден как блочное
устройство без разделов:

```bash
$ lsblk /dev/nvme0n1
# Ожидаемый вывод:
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1 259:0    0 238,5G  0 disk
```

Создаём GPT-таблицу разделов и один раздел на весь диск с помощью `sgdisk`:

```bash
# Устанавливаем sgdisk, если не установлен
$ sudo apt-get install gdisk

# Создаём новую GPT-таблицу и единственный раздел на весь диск:
# -Z        — обнуляем любую существующую разметку
# -n 1:0:0  — раздел №1, от первого свободного сектора до последнего
# -t 1:8300 — тип: Linux filesystem
# -c 1:root — имя раздела (необязательно, для наглядности)
$ sudo sgdisk -Z -n 1:0:0 -t 1:8300 -c 1:root /dev/nvme0n1

# Сообщаем ядру о новой таблице разделов
$ sudo partprobe /dev/nvme0n1

# Проверяем результат — должен появиться nvme0n1p1
$ lsblk /dev/nvme0n1
# Ожидаемый вывод:
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 238,5G  0 disk 
└─nvme0n1p1 259:2    0 238,5G  0 part
```

Форматируем созданный раздел в ext4 и присваиваем метку `NVME_ROOT`
для удобной идентификации в `fstab`:

```bash
# ВНИМАНИЕ: команда уничтожит все данные на nvme0n1p1
$ sudo mkfs.ext4 -L NVME_ROOT /dev/nvme0n1p1
```

---

### 8.2 Монтирование NVMe-раздела

```bash
# Создаём точку монтирования
$ sudo mkdir -p /mnt/nvme

# Монтируем NVMe-раздел
$ sudo mount /dev/nvme0n1p1 /mnt/nvme

# Проверяем, что раздел примонтирован
$ df -h /mnt/nvme
# Ожидаемый вывод:
Файловая система Размер Использовано  Дост Использовано% Cмонтировано в
/dev/nvme0n1p1     234G         2,1M  222G            1% /mnt/nvme
```

---

### 8.3 Копирование корневой файловой системы через rsync

Копируем всё содержимое текущего корня на NVMe. Псевдофайловые системы
(`/proc`, `/sys`, `/dev`, `/run`) и точку монтирования самого NVMe
исключаем — их не нужно копировать:

```bash
$ sudo rsync -aAXHv \
      --exclude=/proc \
      --exclude=/sys \
      --exclude=/dev \
      --exclude=/run \
      --exclude=/tmp \
      --exclude=/mnt \
      / /mnt/nvme/
``` 

> Флаги `rsync`:

> `-a` — архивный режим (сохраняет права, владельцев, символьные ссылки, временны́е метки)

> `-A` — сохраняет ACL

> `-X` — сохраняет расширенные атрибуты

> `-H` — сохраняет жёсткие ссылки

> `-v` — подробный вывод прогресса

Копирование займёт несколько минут:
```bash
sent 6.431.074.481 bytes  received 3.252.438 bytes  15.867.637,28 bytes/sec
total size is 7.815.278.366  speedup is 1,21
```

По завершении проверяем, что файлы на месте:

```bash
$ ls /mnt/nvme
# Ожидается:
bin   etc   lib    lost+found  opt   sbin     srv  var
boot  home  lib64  media       root  selinux  usr
```

---

### 8.4 Правка /etc/fstab на NVMe

Открываем `fstab` на **скопированной** системе (не на текущей!):

```bash
$ sudo nano /mnt/nvme/etc/fstab
```

Находим строку с корневым разделом и заменяем метку `ROOT` на `NVME_ROOT`:

```
# Было:
LABEL=ROOT      /       ext4    defaults        1 1

# Стало:
LABEL=NVME_ROOT /       ext4    defaults        1 1
```

Итоговый `/etc/fstab` должен выглядеть примерно так:

```
proc            /proc           proc    nosuid,noexec,gid=proc                        0 0
devpts          /dev/pts        devpts  nosuid,noexec,gid=tty,mode=620,ptmxmode=0666  0 0
tmpfs           /tmp            tmpfs   nosuid                                        0 0
LABEL=NVME_ROOT /               ext4    defaults                                      1 1
UUID=6E3D-DECB  /boot/BOOT/     vfat    defaults                                      0 2
```

---

### 8.5 Правка загрузчика extlinux.conf

Открываем конфигурацию загрузчика:

```bash
$ sudo nano /boot/BOOT/extlinux/extlinux.conf
```

Комментируем старую строку `append` и добавляем новую — с корнем на NVMe:

```
default l0
menu title U-Boot menu
prompt 0
timeout 50

label l00
    menu label ALT Linux 6.12.44-6.12-alt1.forge.rv64
    linux  /6.12.44-6.12-alt1.forge.rv64/vmlinuz
    initrd /6.12.44-6.12-alt1.forge.rv64/initrd.img
    fdtdir /6.12.44-6.12-alt1.forge.rv64/

    # append root=/dev/mmcblk1p4 rw console=ttyS0,115200 console=tty0 earlycon rootwait stmmaceth=chain_mode:1 audit=0 selinux=0
    append root=/dev/nvme0n1p1 rw console=ttyS0,115200 console=tty0 earlycon rootwait stmmaceth=chain_mode:1 audit=0 selinux=0
```

> Старая строка закомментирована (`#`) — её легко восстановить, если
> что-то пойдёт не так.

---

### 8.6 Перезагрузка и проверка

Размонтируем NVMe и перезагружаем плату:

```bash
$ sudo umount /mnt/nvme
$ sudo reboot
```

После загрузки проверяем, что система работает с NVMe:

```bash
# Корневой раздел должен быть /dev/nvme0n1p1
$ findmnt /
# Ожидаемый вывод:
TARGET
  SOURCE         FSTYPE OPTIONS
/ /dev/nvme0n1p1 ext4   rw,relatime

# Дополнительная проверка через lsblk
$ lsblk -o NAME,MOUNTPOINT,LABEL
# Ожидаемый вывод:
NAME        MOUNTPOINT LABEL
mtdblock0              
mtdblock1              
mtdblock2              
mmcblk1                
├─mmcblk1p1            
├─mmcblk1p2            
├─mmcblk1p3 /boot/BOOT BOOT
└─mmcblk1p4            ROOTFS
nvme0n1                
└─nvme0n1p1 /          NVME_ROOT
```

Если загрузка не прошла и плата ушла в петлю — подключитесь через UART,
прервите загрузку в U-Boot и откатитесь на SD-карту, раскомментировав
исходную строку в `extlinux.conf`.

---

## 9. Правка образа (для разработчиков)

Этот раздел описывает, как смонтировать разделы `.img`-образа на
хост-машине, внести правки (например, подправить `extlinux.conf`)
и синхронизировать содержимое с рабочей SD-картой.

### 9.1 Подготовка точек монтирования

```bash
$ mkdir -p mnt_boot mnt_root sd_boot sd_root
```

### 9.2 Монтирование образа через loop-устройство

```bash
# Подключаем образ как loop-устройство
$ sudo losetup -f regular-mate-latest-riscv64-visionfive2.img

# Получаем имя устройства, которое было назначено образу
$ DEVICE=$(sudo losetup -a | grep regular-mate-latest-riscv64-visionfive2.img | awk '{print $1}' | tr -d ':')

# Создаём маппинг разделов через kpartx
$ sudo kpartx -a $DEVICE

# Извлекаем чистое имя устройства (например, loop0)
$ DEV_NAME=$(basename $DEVICE)

# Монтируем загрузочный и корневой разделы образа
$ sudo mount /dev/mapper/${DEV_NAME}p3 mnt_boot/
$ sudo mount /dev/mapper/${DEV_NAME}p4 mnt_root/
```

### 9.3 Монтирование рабочей SD-карты

```bash
# Замените /dev/sdX на фактическое устройство вашей SD-карты
$ sudo mount /dev/sdX3 sd_boot/
$ sudo mount /dev/sdX4 sd_root/
```

### 9.4 Копирование с SD-карты в образ

Используется для обновления образа из рабочей (уже настроенной)
системы на SD-карте.

```bash
# Полностью заменяем содержимое разделов образа содержимым SD-карты
$ sudo rm -rf mnt_boot/* mnt_root/*
$ sudo rsync -aAXvP sd_boot/ mnt_boot/
$ sudo rsync -aAXvP sd_root/ mnt_root/
```

> Флаг `-P` объединяет `--progress` и `--partial` — удобно для больших
> разделов.

Если нужно обновить только загрузочный раздел (например, изменился
`extlinux.conf` или ядро):

```bash
$ sudo rsync -aAXvP sd_boot/ mnt_boot/
```

### 9.5 Точечная правка файлов в образе

Если нужно только подправить конкретный файл без полной синхронизации —
например, параметры загрузчика:

```bash
$ sudo nano mnt_boot/extlinux/extlinux.conf
```

Пример корректного `extlinux.conf` с правильным порядком `console=` для
вывода лога на HDMI:

```
default l0
menu title U-Boot menu
prompt 0
timeout 50

label l0
    menu label ALT Linux 6.12.77-6.12-alt1.forge.rv64
    linux  /6.12.77-6.12-alt1.forge.rv64/vmlinuz
    initrd /6.12.77-6.12-alt1.forge.rv64/initrd.img
    fdtdir /6.12.77-6.12-alt1.forge.rv64/

    append root=/dev/mmcblk1p4 rw console=ttyS0,115200 console=tty0 earlycon rootwait stmmaceth=chain_mode:1 selinux=0 audit=0
```

> **Важно:** `console=tty0` должен стоять **последним** — именно к нему
> привязывается `/dev/console` для systemd и userspace, что обеспечивает
> полный лог загрузки на HDMI.

### 9.6 Размонтирование

```bash
$ sudo umount sd_boot/ sd_root/ mnt_boot/ mnt_root/

# Удаляем все device-mapper маппинги и отсоединяем loop-устройство
$ sudo dmsetup remove_all
$ sudo losetup -d $DEVICE
```

> Кстати, стоит иметь в виду: `dmsetup remove_all` убирает **все**
> device-mapper устройства в системе (включая LVM, если есть).
> Если на хост-машине используется LVM — безопаснее заменить на
> `sudo kpartx -d $DEVICE`, который удаляет только маппинги конкретного
> образа.

---

## 10. Сборка модуля ядра в изолированном окружении

Этот раздел описывает, как собрать отдельный модуль ядра (`.ko`)
в изолированном окружении [gear-hsh-wrapper](https://github.com/kovalev0/gear-hsh-wrapper)
и установить его на рабочей системе. Пример актуален для случаев,
когда нужный модуль не включён в поставляемое ядро — например,
`iptable_raw`, необходимый для работы Docker.

### 10.1 Подготовка окружения

Если `gear-hsh-wrapper` ещё не установлен (*по-умолчанию*) — следуйте
инструкции в его [README](https://github.com/kovalev0/gear-hsh-wrapper),
либо выполните следующие команды:

```bash
# Начальная настройка системы
$ sudo apt-get update
$ sudo apt-get install -y hasher gear qemu-user-binfmt_misc git
$ sudo hasher-useradd user
$ sudo sh -c "echo 'prefix=~:/tmp/.private' >> /etc/hasher-priv/system"
$ sudo sh -c "echo 'allowed_mountpoints=/proc,/dev/pts,/sys,/dev/shm' >> /etc/hasher-priv/system"
$ sudo sh -c "echo 'allowed_devices=/dev/kvm' >> /etc/hasher-priv/system"
$ sudo sh -c "echo 'allow_ttydev=yes' >> /etc/hasher-priv/system"
$ sudo systemctl restart hasher-privd.service
$ mkdir -p ~/.hasher
$ echo "install_resolver_configuration_files=1" > ~/.hasher/config
$ echo "wlimit_time_short=180" >> ~/.hasher/config

# Установка gear-hsh-wrapper
$ git clone https://github.com/kovalev0/gear-hsh-wrapper.git
$ cd gear-hsh-wrapper
$ ./install.sh
```

### 10.2 Установка исходников ядра в окружение

> Перелогиньтесь — hasher-useradd добавляет пользователя в новые группы.

```bash
# Склонируйте репозиторий ядра
$ cd ~
$ git clone --branch alt-JH7110_VisionFive2_6.12.y_devel git://git.altlinux.org/people/kovalev/packages/kernel-image.git
$ cd kernel-image

# Укажите вариант (flavour) ядра
$ git config --add gear.specsubst.kflavour 6.12

# Запустите процесс сборки
$ gear-hsh-wrapper-sisyphus-riscv64 build
```

После выполнения последней команды начнется полная сборка rpm пакета,
включая полную сборку ядра. Однако, нас интересует только сборка
конкретного модуля, поэтому, нужно дождаться начала сборки ядра
(**Executing(%build)**) и остановиться (Ctrl + С):

```
Executing(%build): /bin/sh -e /usr/src/tmp/rpm-tmp.54673
+ umask 022
+ /bin/mkdir -p /usr/src/RPM/BUILD
+ cd /usr/src/RPM/BUILD
+ cd kernel-image-6.12-6.12.77-alt1.forge.rv64/kernel-source-6.12
+ banner build

######   #     #  ###  #        ######   
#     #  #     #   #   #        #     #  
#     #  #     #   #   #        #     #  
######   #     #   #   #        #     #  
#     #  #     #   #   #        #     #  
#     #  #     #   #   #        #     #  
######    #####   ###  #######  ###### 
^C
```

Теперь в нашем hasher-окружении `sisyphus-riscv64` установлены все
зависимости для сборки. Если требуется установить дополнительные
пакеты, например, редактор `nano`, выполните:

```bash
$ gear-hsh-wrapper-sisyphus-riscv64 install nano
```

### 10.3 Получение .config и Module.symvers с платы

Эти два файла привязывают сборку к конкретному запущенному ядру — без
них модули не будут совместимы с системой.

```bash
$ zcat /proc/config.gz > ./config.orig
$ cp /usr/src/linux-$(uname -r)/Module.symvers .

# Копируем файлы внутрь окружения hasher
$ cp ./config.orig    ~/hasher/sisyphus-riscv64/chroot/usr/src/
$ cp ./Module.symvers ~/hasher/sisyphus-riscv64/chroot/usr/src/
```

### 10.4 Вход в окружение и переход к исходникам ядра

```bash
$ gear-hsh-wrapper-sisyphus-riscv64 shell
```

Внутри окружения исходники ядра находятся в:

```
/usr/src/RPM/BUILD/kernel-image-6.12-6.12.77-alt1.forge.rv64/kernel-source-6.12/
```

```bash
[builder@localhost .in]$ cd /usr/src/RPM/BUILD/kernel-image-6.12-*/kernel-source-6.12/
```

### 10.5 Применение конфига и Module.symvers

```bash
[builder@localhost kernel-source-6.12]$ cp ~/config.orig .config
[builder@localhost kernel-source-6.12]$ cp ~/Module.symvers .

[builder@localhost kernel-source-6.12]$ make ARCH=riscv olddefconfig
# Ожидаемый вывод: No change to .config
```

### 10.6 Включение нужного модуля ядра

Проверяем текущее состояние нужной опции:

```bash
[builder@localhost kernel-source-6.12]$ grep IP_NF_RAW .config
# CONFIG_IP_NF_RAW is not set   ← модуль выключен
```

Включаем как модуль (`=m`):

```bash
[builder@localhost kernel-source-6.12]$ ./scripts/config --module CONFIG_IP_NF_RAW
[builder@localhost kernel-source-6.12]$ make ARCH=riscv olddefconfig
```

### 10.7 Сборка модуля ядра

Важный нюанс синтаксиса make: директорию нужно указывать **без** префикса
`modules/` и **без** слеша в конце — именно такой формат принудительно
пересобирает цель:

```bash
# Шаг 1: компилируем объектные файлы
[builder@localhost kernel-source-6.12]$ make ARCH=riscv net/ipv4/netfilter/ -j$(nproc) 

# Шаг 2: линкуем .ko
[builder@localhost kernel-source-6.12]$ make ARCH=riscv M=net/ipv4/netfilter/ modules -j$(nproc)
```

Проверяем результат:

```bash
[builder@localhost kernel-source-6.12]$ find net/ipv4/netfilter/ -name '*.ko'
net/ipv4/netfilter/iptable_raw.ko
net/ipv4/netfilter/ip_tables.ko
net/ipv4/netfilter/iptable_filter.ko
# и другие
```

### 10.8 Установка модуля ядра

Выходим из окружения и копируем `.ko` с изолированного окружения в каталог
модулей ядра. Собранные модули лежат внутри chroot по пути:

```
~/hasher/sisyphus-riscv64/chroot/usr/src/RPM/BUILD/kernel-image-6.12-.../kernel-source-6.12/
```

```bash
# Задаём путь к собранным модулям
$ KO_PATH=~/hasher/sisyphus-riscv64/chroot/usr/src/RPM/BUILD/\
kernel-image-6.12-6.12.77-alt1.forge.rv64/kernel-source-6.12/net/ipv4/netfilter

# Копируем (устанавливаем)
$ sudo cp $KO_PATH/iptable_raw.ko $KO_PATH/ip_tables.ko \
  /lib/modules/$(uname -r)/kernel/net/ipv4/netfilter/

# Обновляем таблицу зависимостей
$ sudo depmod -a

# Загружаем модуль
$ sudo modprobe iptable_raw

# Проверяем
$ lsmod | grep iptable
```

### 10.9 Автозагрузка модуля при старте системы

```bash
$ echo 'iptable_raw' | sudo tee /etc/modules-load.d/docker.conf
```

### 10.10 Проверка Docker

> Установите [docker](https://www.altlinux.org/Docker)

```bash
$ docker run --rm -it registry.altlinux.org/sisyphus/alt
[root@ /]# apt-get update
```

---

## 11. Отладка ядра и модулей через KGDB
 
KGDB позволяет отлаживать ядро и модули удалённо с хост-машины (например, x86_64) через
UART-соединение. GDB на хосте подключается к ядру платы по протоколу GDB remote serial.
 
### 11.1 Необходимые опции ядра
 
Для работы KGDB ядро должно быть собрано со следующими опциями:
 
```
CONFIG_KGDB=y                  # поддержка KGDB (только built-in, не модуль)
CONFIG_KGDB_SERIAL_CONSOLE=y   # транспорт через UART
CONFIG_DEBUG_INFO=y            # символы отладки в vmlinux
CONFIG_FRAME_POINTER=y         # корректные backtrace
CONFIG_MAGIC_SYSRQ=y           # sysrq для остановки ядра (echo g)
 
# Обязательно ОТКЛЮЧИТЬ для работы software breakpoints в модулях:
# CONFIG_STRICT_KERNEL_RWX is not set
# CONFIG_STRICT_MODULE_RWX is not set
```

> **Почему нужно отключить `STRICT_*_RWX`:** эти опции делают `.text`-секции
> ядра и модулей недоступными для записи после загрузки. KGDB вставляет
> `ebreak`-инструкцию прямо в код — без доступа на запись брейкпоинты
> устанавливаются, но никогда не срабатывают. Hardware breakpoints (`hbreak`)
> тоже не помогут — JH7110 поддерживает только 2 hardware trigger, которые
> обычно уже заняты ядром.

Готовые пакеты отладочного ядра (с этими опциями и отдельным flavour `6.12.kgdb`),
собранные из исходного кода ветки [alt-JH7110_VisionFive2_6.12.y_devel.kgdb](https://git.altlinux.org/people/kovalev/packages/kernel-image.git?p=kernel-image.git;a=shortlog;h=refs/heads/alt-JH7110_VisionFive2_6.12.y_devel.kgdb),
можете скачать архивом по ссылке [https://drive.google.com/file/d/1hQls0jE2KZ8LvFeNPLLSxL0ka1TMmwrD/view?usp=sharing].

Архив содержит следующие пакеты:
```
kernel-headers-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm
kernel-headers-modules-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm
kernel-image-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm
kernel-image-6.12.kgdb-debuginfo-6.12.77-alt1.forge.rv64.riscv64.rpm
kernel-modules-drm-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm
kernel-modules-drm-6.12.kgdb-debuginfo-6.12.77-alt1.forge.rv64.riscv64.rpm
```

### 11.2 Установка отладочного ядра на плату

```bash
# Устанавливаем ядро, DRM-модули и заголовки
$ tar -xf 6.12.77.kgdb.tar.gz
$ cd 6.12.77.kgdb
$ sudo apt-get install -y \
    ./kernel-image-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm \
    ./kernel-modules-drm-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm \
    ./kernel-headers-modules-6.12.kgdb-6.12.77-alt1.forge.rv64.riscv64.rpm
```

Перенесите загрузочные файлы ядра (vmlinuz, initrd, .dtb) в каталог зугрузочной sd-карты:

```bash
$ sudo mkdir /boot/BOOT/6.12.77-6.12.kgdb-alt1.forge.rv64
$ sudo /boot/vmlinuz-6.12.77-6.12.kgdb-alt1.forge.rv64 /boot/BOOT/6.12.77-6.12.kgdb-alt1.forge.rv64/vmlinuz
$ sudo cp /boot/initrd-6.12.77-6.12.kgdb-alt1.forge.rv64.img /boot/BOOT/6.12.77-6.12.kgdb-alt1.forge.rv64/initrd.img
$ sudo cp /boot/devicetree/6.12.77-6.12.kgdb-alt1.forge.rv64/starfive/jh7110-starfive-visionfive-2-v1.3b.dtb /boot/BOOT/6.12.77-6.12.kgdb-alt1.forge.rv64/
```
и добавьте запись в extlinux.conf, одновременно выбрав это ядро для загрузки по-умолчанию,
добавив метку `label l0`, а старую запись переименуйте в `label l1`:

```
label l0
    menu label ALT Linux 6.12.77-6.12.kgdb-alt1.forge.rv64
    linux  /6.12.77-6.12.kgdb-alt1.forge.rv64/vmlinuz
    initrd /6.12.77-6.12.kgdb-alt1.forge.rv64/initrd.img
    fdtdir /6.12.77-6.12.kgdb-alt1.forge.rv64/

    append root=/dev/mmcblk1p4 rw console=ttyS0,115200 console=tty0 earlycon rootwait stmmaceth=chain_mode:1 audit=0 selinux=0
```

Перезагрузите плату и проверьте что загружено правильное ядро:
 
```bash
$ uname -r
# Ожидаемый вывод: 6.12.77-6.12.kgdb-alt1.forge.rv64
```

Активируйте KGDB через UART:

```bash
$ echo ttyS0 | sudo tee /sys/module/kgdboc/parameters/kgdboc
```

### 11.3 Подготовка хоста

#### Установка gdb-multiarch

Штатный `gdb` в ALT собран только для ограниченного набора архитектур и не умеет
работать с RISC-V таргетом, поэтому нужно установить новые пакеты. Можете либо
воспользоваться уже собранными пакетами по ссылке [https://drive.google.com/file/d/1sKdbWD4BP411jiBqg1lh2k6QopHkLCX9/view?usp=sharing],
либо самостоятельно пересобрать пакет с флагом `--enable-targets=all`:

- добавьте в `%define configure_opts` в spec-файле пакета `gdb`:
 
```
--enable-targets=all \
```
 
- пересоберите (например, через `gear-hsh-wrapper` на хосте) и установите.
 
Проверка:
 
```bash
$ gdb -ex 'set architecture riscv:rv64' -ex quit
# Не должно быть ошибки "Undefined item"
```
 
#### Извлечение vmlinux из debuginfo-пакета
 
`vmlinux` содержит символы отладки ядра и необходим для KGDB. Извлекаем из
rpm-пакета:
 
```bash
$ cat kernel-image-6.12.kgdb-debuginfo-6.12.77-alt1.forge.rv64.riscv64.rpm \
    | rpm2cpio \
    | cpio -imdv \
    "./usr/lib/debug/lib/modules/6.12.77-6.12.kgdb-alt1.forge.rv64/vmlinux"
```
 
Файл появится по пути `./usr/lib/debug/lib/modules/.../vmlinux`.
 
#### Установка gdb-dashboard (опционально)
 
[gdb-dashboard](https://github.com/cyrus-and/gdb-dashboard) — удобный TUI для GDB с
подсветкой исходного кода, стека и переменных.
 
```bash
# Устанавливаем поддержку подсветки синтаксиса
$ sudo apt-get install python3-module-Pygments
 
# Скачиваем dashboard
$ wget -q -P ~/ https://github.com/cyrus-and/gdb-dashboard/raw/master/.gdbinit
$ mkdir -p ~/.gdbinit.d
```
 
Создайте файл настроек `~/.gdbinit.d/init`:
 
```
dashboard -layout source stack !threads expressions variables !registers !assembly
dashboard stack -style limit 3
dashboard source -style height 30
 
# set substitute-path /home/user/labs ~/labs
set serial baud 115200
```
 
> `set substitute-path` — замена префикса пути к исходникам. Например, модуль собирался
> на плате в `/home/user/labs/...`, а на хосте исходники лежат в другом месте.
> GDB автоматически подставит правильный путь при показе кода.
 
### 11.4 Запуск сессии отладки
 
**Шаг 1. На плате** — остановите ядро для подключения GDB:
 
```bash
$ echo g | sudo tee /proc/sysrq-trigger
# Плата зависнет — это нормально, ядро ждёт GDB
```
 
**Шаг 2. На хосте** — запустите GDB и подключитесь (вместо `(gdb)` будет `>>>`, если
используете gdb dashboard):

```bash
$ gdb ./usr/lib/debug/lib/modules/6.12.77-6.12.kgdb-alt1.forge.rv64/vmlinux
(gdb) set serial baud 115200
(gdb) set architecture riscv:rv64
(gdb) target remote /dev/ttyUSB0
```
 
После подключения GDB покажет текущее местоположение ядра:
 
```
0xffffffff8000eb02 in arch_kgdb_breakpoint () at arch/riscv/kernel/kgdb.c:259
```

Чтобы продолжить выполнение кода ядра, введите `continue` или `c`:
```bash
(gdb) c
```
 
**Шаг 3.** Загрузите символы отлаживаемого модуля.

Например, рассмотрим отладку модуля `my_counter` из урока 4 `lab04-kernel-module` [module3-counter-module.md](module3-counter-module.md).

Пересоберите, затем загрузите модуль на плате и считайте адреса секций:

```bash
# На плате
$ pwd
/home/user/labs/lab04/my_counter
$ make
make -C /lib/modules/6.12.77-6.12.kgdb-alt1.forge.rv64/build M=/home/user/labs/lab04/my_counter modules
make[1]: вход в каталог «/usr/src/linux-6.12.77-6.12.kgdb-alt1.forge.rv64»
  CC [M]  /home/user/labs/lab04/my_counter/my_counter.o
  MODPOST /home/user/labs/lab04/my_counter/Module.symvers
  CC [M]  /home/user/labs/lab04/my_counter/my_counter.mod.o
  CC [M]  /home/user/labs/lab04/my_counter/.module-common.o
  LD [M]  /home/user/labs/lab04/my_counter/my_counter.ko
  BTF [M] /home/user/labs/lab04/my_counter/my_counter.ko
Skipping BTF generation for /home/user/labs/lab04/my_counter/my_counter.ko due to unavailability of vmlinux
make[1]: выход из каталога «/usr/src/linux-6.12.77-6.12.kgdb-alt1.forge.rv64»
$
$ sudo insmod ./my_counter.ko 
$ sudo cat /sys/module/my_counter/sections/.text
0xffffffff02271000
$ sudo cat /sys/module/my_counter/sections/.data
0xffffffff0227d060
```

На хосте:

```bash
# Копируем каталог с исходниками и артефактами в рабочий каталог запуска GDB
$ scp -r riscv:/home/user/labs/lab04/my_counter .
```

В GDB:
 
```
# Сообщаем пути к исходникам (иначе ошибка: Cannot display "my_counter.c")
(gdb) set substitute-path /home/user/labs/lab04/my_counter ./my_counter
# Указываем адреса размещения кода модуля ядра
(gdb) add-symbol-file ./my_counter/my_counter.ko 0xffffffff02271000 \
      -s .data 0xffffffff0227d060
```
 
> **Важно:** адреса секций *иногда* меняются при каждом `insmod`. Всегда берите
> актуальные значения из `/sys/module/<name>/sections/` после загрузки модуля
> и никогда не делайте `rmmod`/`insmod` между получением адресов и сессией GDB.
 
**Шаг 4.** Установите брейкпоинт и продолжите выполнение:
 
```
(gdb) break my_counter_reset_store
(gdb) c
```
 
**Шаг 5. На плате** — триггерите нужную функцию (без `echo g`):
 
```bash
$ echo 1 | sudo tee /sys/kernel/my_counter/reset
```
 
GDB остановится автоматически когда выполнение дойдёт до брейкпоинта:
```gdb
─── Output/messages ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Thread 254 hit Breakpoint 7.2, my_counter_reset_store (kobj=0xffffffd6c289d980, attr=0xffffffff0227d0d0 <my_counter_reset_attr>, 
    buf=0xffffffd6c20d0bc8 "1\n", count=2) at /home/user/labs/lab04/my_counter/my_counter.c:120
120     {
─── Source ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 105      if (val == 1)
 106          my_counter_do_increment();
 107      return count;
 108  }
 109  
 110  /* --- reset --- */
 111  static ssize_t my_counter_reset_show(struct kobject *kobj,
 112                       struct kobj_attribute *attr, char *buf)
 113  {
 114      return sysfs_emit(buf, "write 1 to reset counter\n");
 115  }
 116  
 117  static ssize_t my_counter_reset_store(struct kobject *kobj,
 118                        struct kobj_attribute *attr,
 119                        const char *buf, size_t count)
!120  {
 121      unsigned int val;
 122      int ret = kstrtouint(buf, 10, &val);
 123      if (ret)
 124          return ret;
 125      if (val == 1)
 126          my_counter_do_reset();
 127      return count;
 128  }
 129  
 130  /* --- trigger --- */
 131  static ssize_t my_counter_trigger_show(struct kobject *kobj,
 132                         struct kobj_attribute *attr, char *buf)
 133  {
 134      return sysfs_emit(buf, "%d\n", my_counter_trigger);
─── Stack ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[0] from 0xffffffff02271126 in my_counter_reset_store+0 at /home/user/labs/lab04/my_counter/my_counter.c:120
[1] from 0xffffffff80e672ae in kobj_attr_store+14 at lib/kobject.c:840
[2] from 0xffffffff8030f7f8 in sysfs_kf_write+42 at fs/sysfs/file.c:136
[+]
─── Expressions ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
─── Variables ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
arg kobj = 0xffffffd6c289d980: {name = 0xffffffd6e1579580 "my_counter",entry = {next = 0xffffffd6c289d…, attr = 0xffffffff0227d0d0 <my_counter_reset_attr>: {attr = {name = 0xffffffff022811f8 "reset",mode…, buf = 0xffffffd6c20d0bc8 "1\n": 49 '1', count = 2
loc val = 4294967254, ret = <optimized out>
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
>>>
```
 
### 11.5 Как остановить ядро в произвольный момент
 
Пока ядро работает, GDB ждёт и не принимает команды. Ctrl+C в GDB на RISC-V
не работает. Единственный способ — отправить `sysrq` с платы:
 
```bash
# На плате (в отдельном SSH-сеансе)
$ echo g | sudo tee /proc/sysrq-trigger
```

### 11.6 Оптимизация и отладка модулей

#### Почему нельзя просто `-O0`

Ядро Linux написано с расчётом на компиляцию с оптимизацией. Часть кода
использует `inline`-функции и конструкции которые при `-O0` либо не
компилируются вовсе, либо дают неправильный код — неверно работает стек,
атомарные операции, memory barrier'ы. Поэтому ядро **всегда** собирается
минимум с `-O2`, и это требование нельзя снять.
 
Следствие для отладчика: при `-O2` компилятор агрессивно оптимизирует код —
переменные исчезают (оседают в регистрах или вовсе выбрасываются), строки
исходника перестают соответствовать инструкциям, функции инлайнятся и
пропадают из таблицы символов. GDB показывает `<optimized out>` вместо
значений переменных.
 
#### Флаг `-Og` — компромисс для отладки
 
GCC предоставляет специальный уровень `-Og` введённый именно для отладки.
Он включает только те оптимизации которые не мешают отладчику: переменные
сохраняются, порядок инструкций близок к исходнику, но код остаётся
корректным с точки зрения ядра. Это лучший выбор когда нужна отладка, но
`-O0` неприемлем.
 
#### Флаги для пользовательского модуля
 
Для полного отключения оптимизации в своём модуле добавьте в `Makefile`:
 
```makefile
ccflags-y := -g -Og -fno-inline
```
 
`-fno-inline` важен отдельно — без него GCC инлайнит маленькие функции
даже при `-Og`, и они исчезают из отладчика как самостоятельные символы.
 
Если нужно отключить оптимизацию только для конкретного файла не затрагивая
остальные:
 
```makefile
CFLAGS_my_counter.o := -g -Og -fno-inline
```
 
Проверьте что флаги применились:
 
```bash
$ make V=1 2>&1 | grep 'my_counter.o'
# В выводе должны быть -Og -fno-inline
```
 
> **Итоговая рекомендация:** для отладки модуля используйте `-Og -fno-inline`.
> Для ядра целиком изменить уровень оптимизации нельзя, но можно пересобрать
> конкретный файл ядра с нужными флагами через `CFLAGS_имяфайла.o`.
 
---

[На главную](INDEX.md) · [Урок 01 →](lab01-board/README.md)
