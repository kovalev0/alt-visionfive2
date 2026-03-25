# module4 · U-Boot: extlinux.conf, переменные окружения, FIT-образ

← [Назад](module3-opensbi.md) · [На главную](../INDEX.md) · [Следующий →](module5-devicetree.md)

> **Документация U-Boot:** [docs.u-boot.org](https://docs.u-boot.org/en/latest/)

> **extlinux/syslinux формат:** [docs.u-boot.org/en/latest/develop/distro.html](https://docs.u-boot.org/en/latest/develop/distro.html)

---

## Роль U-Boot в цепочке загрузки

U-Boot (Universal Bootloader) — загрузчик который работает в S-mode после
того как OpenSBI передал управление. Его задачи:

1. Инициализация оставшейся периферии (MMC, сеть, USB)
2. Поиск и загрузка ядра Linux, initrd, DTB из файловой системы
3. Формирование строки аргументов ядра (`bootargs`)
4. Передача управления ядру

U-Boot содержит собственную маленькую файловую систему FAT32 — именно она
позволяет читать `extlinux.conf` с загрузочного раздела без поддержки ext4.

---

## Структура разделов SD-карты

```
  SD-карта (GPT)
  ┌─────────────────────────────────────────────────────┐
  │  Раздел 1: spl    (2E54B353-...) ~2 MiB             │
  │  u-boot-spl.bin.normal.out                          │
  │  Читается BootROM напрямую                          │
  ├─────────────────────────────────────────────────────┤
  │  Раздел 2: uboot  (BC13C2FF-...) ~4 MiB             │
  │  visionfive2_fw_payload.img (FIT: OpenSBI + U-Boot) │
  │  Читается SPL                                       │
  ├─────────────────────────────────────────────────────┤
  │  Раздел 3: kernel (FAT32) ~300 MiB                  │
  │  vmlinuz, initrd.img, *.dtb                         │
  │  extlinux/extlinux.conf                             │
  │  Читается U-Boot                                    │
  ├─────────────────────────────────────────────────────┤
  │  Раздел 4: root   (ext4)                            │
  │  Корневая файловая система ALT Linux                │
  └─────────────────────────────────────────────────────┘
```

```bash
# Посмотреть разделы в работающей системе
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mtdblock0    31:0    0   960K  0 disk 
mtdblock1    31:1    0    64K  0 disk 
mtdblock2    31:2    0    15M  0 disk 
mmcblk1     179:0    0 116.5G  0 disk 
├─mmcblk1p1 179:1    0     2M  0 part 
├─mmcblk1p2 179:2    0     4M  0 part 
├─mmcblk1p3 179:3    0   292M  0 part /boot/BOOT
└─mmcblk1p4 179:4    0 116.2G  0 part /

$ sudo fdisk -l /dev/mmcblk1 2>/dev/null
Disk /dev/mmcblk1: 116.48 GiB, 125069950976 bytes, 244277248 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 6F1A47D9-B7FB-471F-AFE1-62E3BB7CDC0D

Device          Start       End   Sectors   Size Type
/dev/mmcblk1p1   4096      8191      4096     2M HiFive BBL
/dev/mmcblk1p2   8192     16383      8192     4M Linux extended boot
/dev/mmcblk1p3  16384    614399    598016   292M Microsoft basic data
/dev/mmcblk1p4 614400 244277214 243662815 116.2G Linux filesystem

# Точки монтирования
$ mount | grep mmcblk
/dev/mmcblk1p4 on / type ext4 (rw,relatime)
/dev/mmcblk1p3 on /boot/BOOT type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro)
```

---

## extlinux.conf

extlinux.conf — конфигурационный файл загрузчика в формате syslinux/pxelinux,
поддерживаемый U-Boot через механизм "distro boot". U-Boot ищет его по
нескольким путям, в том числе `/extlinux/extlinux.conf` на FAT-разделе.

```bash
# Посмотреть текущий конфиг (раздел kernel смонтирован в /boot/BOOT)
$ sudo cat /boot/BOOT/extlinux/extlinux.conf
```

Пример содержимого:

```
## /boot/BOOT/extlinux/extlinux.conf

default l0
timeout 30

label l0
    menu label Alt GNU/Linux
    linux  /vmlinuz
    initrd /initrd.img
    fdtdir /

    append root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 \
           earlycon rootwait stmmaceth=chain_mode:1 selinux=0
```

Разбор директив:

| Директива | Значение |
|-----------|----------|
| `default l0` | Метка записи загружаемой по умолчанию |
| `timeout 30` | Таймаут в десятых долях секунды (3 секунды) |
| `linux` | Путь к ядру (vmlinuz) на FAT-разделе |
| `initrd` | Путь к initial ramdisk |
| `fdtdir` | Каталог для поиска DTB (U-Boot ищет по имени платы) |
| `append` | Строка аргументов ядра (`/proc/cmdline`) |

### Аргументы ядра

```
root=/dev/mmcblk1p4    — корневая ФС на разделе 4 SD-карты
rw                     — монтировать root read-write
console=tty0           — вывод на HDMI
console=ttyS0,115200   — вывод на UART0 (отладочная консоль)
earlycon               — ранний вывод до инициализации UART драйвера
rootwait               — ждать появления root-устройства
stmmaceth=chain_mode:1 — параметр драйвера Ethernet
selinux=0              — отключить SELinux
```

```bash
# Проверить аргументы с которыми загружено текущее ядро
$ cat /proc/cmdline
```

---

## Переменные окружения U-Boot

U-Boot хранит переменные окружения в QSPI Flash (или SD-карте). Просмотр
и изменение возможны из интерактивной консоли U-Boot.

Для доступа к консоли U-Boot нужно нажать любую клавишу в 3-секундный
интервал при загрузке (виден только через UART-консоль).

```
StarFive # printenv
# вывод всех переменных

StarFive # printenv bootcmd
# команда запускаемая по умолчанию

StarFive # printenv bootargs
# текущие аргументы ядра (могут быть переопределены extlinux.conf)
```

Важные переменные:

```
bootcmd        — что выполнять при автозагрузке
kernel_addr_r  — адрес в RAM куда загружается ядро
fdt_addr_r     — адрес в RAM куда загружается DTB
ramdisk_addr_r — адрес в RAM куда загружается initrd
fdtfile        — имя файла DTB (автоопределяется или задаётся вручную)
```

### Добавление второй записи загрузки (кастомное ядро)

Из Урока 0 (README основного репозитория) известно что кастомное ядро 6.12
устанавливается отдельно. Для него добавляется запись в `extlinux.conf`:

```
label l00
    menu label ALT Linux 6.12.77-6.12-alt1.forge.rv64
    linux  /6.12.77-6.12-alt1.forge.rv64/vmlinuz
    initrd /6.12.77-6.12-alt1.forge.rv64/initrd.img
    fdtdir /6.12.77-6.12-alt1.forge.rv64/

    append root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 \
           earlycon rootwait stmmaceth=chain_mode:1 selinux=0
```

---

## Практика: наблюдение процесса загрузки U-Boot

```bash
# После перезагрузки через UART-консоль будет виден полный лог.
# В работающей системе можно посмотреть что U-Boot передал ядру:

# Аргументы ядра
$ cat /proc/cmdline

# Информация о DTB переданном U-Boot
$ dtc -I fs /proc/device-tree 2>/dev/null | less
$ cat /sys/firmware/devicetree/base/compatible
# вывод: starfive,visionfive-2-v1.3bstarfive,jh7110
```

---

## Команды U-Boot для диагностики

Эти команды выполняются в интерактивной консоли U-Boot (нужен UART):

```
# Информация о системе
bdinfo                          — параметры платы, адреса памяти
version                         — версия U-Boot

# Работа с памятью
md.l 0x40000000 0x10            — дамп памяти (32-bit) с адреса 0x40000000
mw.l 0x40000000 0xdeadbeef 1    — запись 32-bit слова в память

# Работа с MMC
mmc list                        — список MMC устройств

StarFive # mmc list
sdio0@16010000: 0
sdio1@16020000: 1

mmc dev 1                       — выбрать SD-карту (обычно устройство 1)

StarFive # mmc dev 1
switch to partitions #0, OK
mmc1 is current devic

mmc part                        — таблица разделов

StarFive # mmc part

Partition Map for MMC device 1  --   Partition Type: EFI

Part    Start LBA       End LBA         Name
        Attributes
        Type GUID
        Partition GUID
  1     0x00001000      0x00001fff      "spl"
        attrs:  0x0000000000000000
        type:   2e54b353-1271-4842-806f-e436d6af6985
        guid:   dd8f4de1-d7f1-4539-ac9b-3f75eae577bb
  2     0x00002000      0x00003fff      "uboot"
        attrs:  0x0000000000000000
        type:   bc13c2ff-59e6-4262-a352-b275fd6f7172
        guid:   e2688d2a-b5bd-4b23-81ad-255784bde316
  3     0x00004000      0x00095fff      "kernel"
        attrs:  0x0000000000000000
        type:   ebd0a0a2-b9e5-4433-87c0-68b6b72699c7
        type:   data
        guid:   26ced9ab-dc03-46f0-9ac0-712334a0099b
  4     0x00096000      0x0e8f5fde      "root"
        attrs:  0x0000000000000000
        type:   0fc63daf-8483-4772-8e79-3d69d8477de4
        type:   linux
        guid:   a626c830-7a3a-4184-86a5-f2598101138c

fatls mmc 1:3                   — список файлов на FAT-разделе (раздел 3)

StarFive # fatls mmc 1:3
 13649241   vmlinuz
  9371629   initrd.img
    58144   jh7110-starfive-visionfive-2-v1.3b.dtb
            extlinux/
      400   uEnv.txt
            6.12.77-6.12-alt1.forge.rv64/

4 file(s), 2 dir(s)

fatload mmc 1:3 $kernel_addr_r vmlinuz  — загрузить файл в память

# Загрузка ядра вручную
mmc dev 1
fatload mmc 1:3 $kernel_addr_r vmlinuz
fatload mmc 1:3 $ramdisk_addr_r initrd.img
setenv initrd_size ${filesize}
fatload mmc 1:3 $fdt_addr_r jh7110-starfive-visionfive-2-v1.3b.dtb
setenv bootargs "root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0"
booti $kernel_addr_r $ramdisk_addr_r:${initrd_size} $fdt_addr_r
```

---

← [Назад](module3-opensbi.md) · [На главную](../INDEX.md) · [Следующий →](module5-devicetree.md)
