# module1 · От подачи питания до ядра

← [Назад (README)](README.md) · [На главную](../INDEX.md) · [Следующий →](module2-privilege-levels.md)

---

## Общая цепочка загрузки

```
  Подача питания
       │
       ▼
  BootROM (M-mode)
  Зашит в SoC. Не изменяется.
  Определяет источник загрузки (QSPI Flash / SD / UART).
  Загружает SPL с SD карты в SRAM.
       │
       ▼
  SPL — Secondary Program Loader (M-mode)
  u-boot-spl.bin.normal.out на разделе spl SD-карты.
  Инициализирует DDR-контроллер и LPDDR4.
  Загружает FIT-образ (OpenSBI + U-Boot) из раздела uboot в RAM.
       │
       ▼
  OpenSBI (M-mode → передаёт управление в S-mode)
  visionfive2_fw_payload.img содержит OpenSBI + U-Boot как payload.
  Инициализирует SBI-интерфейс для вышестоящего ПО.
  Остаётся резидентным в M-mode, передаёт управление U-Boot в S-mode.
       │
       ▼
  U-Boot (S-mode)
  Читает extlinux.conf с раздела kernel (FAT32).
  Загружает ядро (vmlinuz), initrd, DTB в RAM.
  Передаёт управление ядру через адрес точки входа.
       │
       ▼
  Linux Kernel (S-mode)
  Получает DTB от U-Boot через регистр a1.
  Инициализирует подсистемы, монтирует rootfs.
  Запускает init/systemd.
       │
       ▼
  Userspace
```

---

## BootROM

**TRM ref:** [Boot Process](https://doc-en.rvspace.org/JH7110/TRM/JH7110_TRM/boot_process.html)

BootROM — неизменяемый код зашитый в SoC на этапе производства. При подаче
питания процессор (точнее, S7 monitor core) стартует с адреса указанного в
reset vector и начинает исполнять BootROM.

Задачи BootROM:
1. Минимальная инициализация SoC (тактирование, базовые клоки)
2. Определение источника загрузки по состоянию BOOT-пинов на плате
3. Загрузка SPL из выбранного источника в SRAM (256 KiB внутри SoC)
4. Передача управления SPL

На VisionFive2 по умолчанию загрузка идёт с QSPI Flash. Там хранится SPL.
Переключение режима загрузки (например на UART для восстановления) делается
двумя DIP-переключателями на плате.

---

## SPL (Secondary Program Loader)

SPL — минимальная версия U-Boot, собранная с флагом `CONFIG_SPL`. Размер
ограничен объёмом SRAM (256 KiB на JH7110), поэтому в нём нет большинства
возможностей полного U-Boot.

Файл на плате: `u-boot-spl.bin.normal.out` (суффикс `.normal.out` добавляет
утилита `spl_tool` от StarFive — она добавляет заголовок для BootROM).

Задачи SPL:
1. Инициализация DDR-контроллера LPDDR4
2. Загрузка FIT-образа (`visionfive2_fw_payload.img`) с раздела `uboot`
   SD-карты в DDR
3. Передача управления OpenSBI

---

## FIT-образ и его структура

FIT (Flattened Image Tree) — формат образа U-Boot, содержащий несколько
компонентов описанных в `.its`-файле (Image Tree Source). На VisionFive2
FIT-образ содержит OpenSBI и U-Boot как payload.

Структура `visionfive2_fw_payload.img`:

```
  visionfive2_fw_payload.img (FIT)
  ├── OpenSBI firmware (fw_payload.bin)
  │   └── U-Boot binary встроен как payload OpenSBI
  └── DTB для U-Boot (starfive_visionfive2.dtb)
```

---

## U-Boot: что видно в UART-консоли

При загрузке через UART-консоль будет виден примерно такой вывод:

```
U-Boot SPL 2021.10 (Sep 01 2025 - 11:14:19 +0000)
LPDDR4: 8G version: g8ad50857.
Trying to boot from MMC2

OpenSBI v1.6
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name               : StarFive VisionFive V2
Platform Features           : medeleg
Platform HART Count         : 5
...
Runtime SBI Version         : 2.0
Standard SBI Extensions     : time,rfnc,ipi,base,hsm,srst,pmu,dbcn,legacy
Experimental SBI Extensions : fwft,dbtr,sse
...
U-Boot 2021.10 (Sep 01 2025 - 11:14:19 +0000)

CPU:   rv64imacu_zba_zbb
Model: StarFive VisionFive V2
DRAM:  8 GiB
MMC:   sdio0@16010000: 0, sdio1@16020000: 1
Loading Environment from SPIFlash... SF: Detected gd25lq128 with page size 256 Bytes, erase size 4 KiB, total 16 MiB
...
Model: StarFive VisionFive V2
Net:   eth0: ethernet@16030000, eth1: ethernet@16040000
Hit any key to stop autoboot:  0 
switch to partitions #0, OK
mmc1 is current device
Try booting from MMC1 ...
Failed to load 'vf2_uEnv.txt'
## Info: input data size = 366 = 0x16E
## Error: "boot2" not defined
Tring booting distro ...
switch to partitions #0, OK
mmc1 is current device
Try booting from MMC1 ...
400 bytes read in 8 ms (48.8 KiB/s)
Retrieving file: /extlinux/extlinux.conf
592 bytes read in 13 ms (43.9 KiB/s)
1:      ALT Linux 6.12.74-6.12-alt1.forge.rv64
Retrieving file: /6.12.74-6.12-alt1.forge.rv64/initrd.img
9371629 bytes read in 456 ms (19.6 MiB/s)
Retrieving file: /6.12.74-6.12-alt1.forge.rv64/vmlinuz
13649241 bytes read in 659 ms (19.8 MiB/s)
append: root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0~
Retrieving file: /6.12.74-6.12-alt1.forge.rv64/jh7110-starfive-visionfive-2-v1.3b.dtb
58144 bytes read in 17 ms (3.3 MiB/s)
   Uncompressing Kernel Image
## Flattened Device Tree blob at 46000000
   Booting using the fdt blob at 0x46000000
   Using Device Tree in place at 0000000046000000, end 000000004601131f

Starting kernel ...
```

Если нажать любую клавишу в 2-секундный таймер — откроется интерактивная
консоль U-Boot:

```
StarFive # printenv          # все переменные окружения
```
Аргументы ядра
```
StarFive # printenv bootargs
bootargs=console=tty1 console=ttyS0,115200  debug rootwait  earlycon=sbi
```
Информация о плате
```
StarFive # bdinfo
boot_params = 0x0000000000000000
DRAM bank   = 0x0000000000000000
-> start    = 0x0000000040000000
-> size     = 0x0000000200000000
flashstart  = 0x0000000000000000
flashsize   = 0x0000000000000000
flashoffset = 0x0000000000000000
baudrate    = 115200 bps
relocaddr   = 0x00000000f7f0e000
reloc off   = 0x00000000b7d0e000
Build       = 64-bit
current eth = ethernet@16030000
ethaddr     = 6c:cf:39:00:ba:ac
IP addr     = 192.168.120.230
fdt_blob    = 0x00000000f7fac8a0
new_fdt     = 0x0000000000000000
fdt_size    = 0x0000000000000000
Video       = dc8200@29400000 active
FB base     = 0x00000000fe000000
FB size     = 0x0x32
lmb_dump_all:
 memory.cnt  = 0x1
 memory[0]      [0x40000000-0x23fffffff], 0x200000000 bytes flags: 0
 reserved.cnt  = 0x1
 reserved[0]    [0x40000000-0x4007ffff], 0x00080000 bytes flags: 4
```

---

## Ручная загрузка из U-Boot

Из логов выше видно следующее:

**MMC**: mmc1 (mmcblk1), загрузочный раздел — партиция 3

**Адреса загрузки** из [**uEnv.txt**](../../boot/uEnv.txt):
```
kernel_addr_r=0x40200000
fdt_addr_r=0x46000000
ramdisk_addr_r=0x46100000
```

**Пути к файлам** из [**extlinux.conf**](../../boot/extlinux/extlinux.conf):
```
	linux  /vmlinuz
	initrd /initrd.img
	fdtdir /

	append  root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0
```

Команды загрузки будут следующие:
```
# 1. Выбрать устройство mmc1
StarFive # mmc dev 1
switch to partitions #0, OK
mmc1 is current device

# 2. Загрузить ядро
StarFive # load mmc 1:3 0x40200000 /vmlinuz
13649241 bytes read in 653 ms (19.9 MiB/s)

# 3. Загрузить DTB
StarFive # load mmc 1:3 0x46000000 /jh7110-starfive-visionfive-2-v1.3b.dtb
58144 bytes read in 10 ms (5.5 MiB/s)

# 4. Загрузить initrd и сохранить его размер
StarFive # load mmc 1:3 0x46100000 /initrd.img
9371629 bytes read in 431 ms (20.7 MiB/s)
StarFive # setenv initrd_size ${filesize}

# 5. Задать параметры ядра
StarFive # setenv bootargs "root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0"

# 6. Запустить ядро
StarFive # booti 0x40200000 0x46100000:${initrd_size} 0x46000000
   Uncompressing Kernel Image
## Flattened Device Tree blob at 46000000
   Booting using the fdt blob at 0x46000000
   Using Device Tree in place at 0000000046000000, end 000000004601131f

Starting kernel ...
```
`mmc 1:3` — mmc1 = mmcblk1, раздел 3 = /boot/BOOT

`booti` — команда загрузки Linux Image (не uImage). Перед вызовом U-Boot:
1. Записывает адрес DTB в регистр `a1`
2. Записывает `0` в регистры `a0` (hart ID передаётся OpenSBI)
3. Передаёт управление на адрес начала ядра

Ядро при старте читает DTB из `a1` — именно так оно узнаёт обо всей
периферии платы. Без DTB ядро не знает что на плате есть GPIO, I2C, SPI
и т.д.

```bash
# Адреса загрузки в RAM можно посмотреть в U-Boot:
StarFive # printenv kernel_addr_r
kernel_addr_r=0x40200000

StarFive # printenv fdt_addr_r
fdt_addr_r=0x46000000

StarFive # printenv ramdisk_addr_r 
ramdisk_addr_r=0x46100000

# Или в работающей системе — через /proc/device-tree
$ ls /proc/device-tree/
$ cat /proc/device-tree/model 2>/dev/null || \
  cat /sys/firmware/devicetree/base/model
# вывод: StarFive VisionFive 2 v1.3B
```

---

## Наблюдение полной цепочки через dmesg

После загрузки системы в `dmesg` виден след инициализации ядра:

```bash
$ dmesg | head -50
```

Первые строки показывают:
```
[    0.000000] Linux version 6.12.x ... riscv64
[    0.000000] Machine model: StarFive VisionFive 2 v1.3B
[    0.000000] earlycon: uart0 at MMIO32 ...
[    0.000000] OF: reserved mem: ...
```

`OF:` — Open Firmware: Flattened Device Tree. Ядро при старте сразу
парсит DTB переданный U-Boot.

```bash
# Инициализация конкретной периферии
dmesg | grep -E "i2c|spi|gpio|pinctrl"

# OpenSBI оставляет след в dmesg
dmesg | grep -i "sbi\|opensbi"
```

---

← [Назад (README)](README.md) · [На главную](../INDEX.md) · [Следующий →](module2-privilege-levels.md)
