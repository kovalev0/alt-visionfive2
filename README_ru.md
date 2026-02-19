# ALT Linux для VisionFive2

Этот репозиторий содержит инструкции, конфигурационные файлы и готовую
прошивку для развёртывания ALT Linux на плате StarFive VisionFive2
(архитектура RISC-V).

Инструкции разделены на три основные части:
1.  **[Быстрый старт](#быстрый-старт)**: Используйте готовую прошивку и конфигурационные файлы из этого репозитория для быстрого запуска ALT Linux.
2.  **[Сборка кастомного ядра (опционально)](#сборка-кастомного-ядра-опционально))**: Сборка и установка кастомного ядра для исправления проблем с графикой и полноценной поддержки оборудования VisionFive2.
3.  **[Сборка прошивки из исходников](#сборка-прошивки-из-исходников)**: Скомпилируйте U-Boot и OpenSBI из исходного кода, если вам необходимо настроить загрузчики.

---

## Быстрый старт

В этом разделе описана подготовка SD-карты и развёртывание ALT Linux
с использованием готовых файлов из данного репозитория.

### 1. Подготовка SD-карты

Сначала определите устройство SD-карты (например, `/dev/sdX`).
**Будьте крайне осторожны** — следующие команды уничтожат все данные
на указанном устройстве.

```bash
# Установите пакеты, предоставляющие необходимые утилиты для работы с образом
$ sudo apt-get update && apt-get install gdisk parted dosfstools e2fsprogs kpartx

# Замените /dev/sdb на фактическое устройство вашей SD-карты
$ DEVICE=/dev/sdb

# Очистите существующую таблицу разделов
$ sudo wipefs -a        $DEVICE
$ sudo sgdisk --zap-all $DEVICE
```

Затем создайте необходимые разделы с помощью `sgdisk`. Схема включает
разделы для SPL (загрузчик второго уровня), U-Boot, конфигурации
ядра/загрузчика и корневой файловой системы.

```bash
# Создайте новую таблицу разделов GPT
$ sudo sgdisk --clear \
     --new=1:4096:8191    --change-name=1:"spl"    --typecode=1:2E54B353-1271-4842-806F-E436D6AF6985 \
     --new=2:8192:16383   --change-name=2:"uboot"  --typecode=2:BC13C2FF-59E6-4262-A352-B275FD6F7172 \
     --new=3:16384:614399 --change-name=3:"kernel" --typecode=3:EBD0A0A2-B9E5-4433-87C0-68B6B72699C7 \
     --new=4:614400:0     --change-name=4:"root"   --typecode=4:0FC63DAF-8483-4772-8E79-3D69D8477DE4  \
     $DEVICE

# Перечитайте таблицу разделов
$ sudo partprobe -s $DEVICE
```

### 2. Запись прошивки и создание файловых систем

Запишите готовые файлы прошивки в соответствующие разделы.

```bash
# Запишите прошивку SPL и U-Boot из директории 'firmware/'
$ sudo dd if=./firmware/u-boot-spl.bin.normal.out  of=${DEVICE}1 bs=512 conv=notrunc status=progress
$ sudo dd if=./firmware/visionfive2_fw_payload.img of=${DEVICE}2 bs=512 conv=notrunc status=progress

# Создайте файловые системы для загрузочного и корневого разделов
$ sudo mkfs.vfat -F 32 -n BOOT   ${DEVICE}3
$ sudo mkfs.ext4 -L       ROOTFS ${DEVICE}4
```

### 3. Развёртывание корневой файловой системы ALT Linux

Смонтируйте созданные разделы и распакуйте
[архив rootfs](https://nightly.altlinux.org/sisyphus-riscv64/current/)
ALT Linux.

```bash
# Создайте точки монтирования
$ sudo mkdir -p /mnt/alt_boot /mnt/alt_root

# Смонтируйте разделы
$ sudo mount ${DEVICE}3 /mnt/alt_boot
$ sudo mount ${DEVICE}4 /mnt/alt_root

# Скачайте и распакуйте последний rootfs ALT Linux для riscv64
# Доступны варианты MATE (по умолчанию) и XFCE. Выберите один.
$ wget https://nightly.altlinux.org/sisyphus-riscv64/current/regular-mate-latest-riscv64.tar.xz
# $ wget https://nightly.altlinux.org/sisyphus-riscv64/current/regular-xfce-latest-riscv64.tar.xz

# Распакуйте скачанный архив.
$ sudo tar -Jxf ./regular-mate-latest-riscv64.tar.xz -C /mnt/alt_root
# Если вы выбрали XFCE, используйте команду ниже
# $ sudo tar -Jxf ./regular-xfce-latest-riscv64.tar.xz -C /mnt/alt_root
```

### 4. Настройка системы

Скопируйте ядро, initrd и бинарный файл дерева устройств в загрузочный
раздел. Затем скопируйте конфигурационные файлы загрузчика из данного
репозитория.

```bash
# Скопируйте ядро и initrd
$ sudo cp /mnt/alt_root/boot/{vmlinuz,initrd.img} /mnt/alt_boot/

# Найдите и скопируйте нужный файл дерева устройств (DTB)
$ sudo find /mnt/alt_root -name 'jh7110-starfive-visionfive-2-v1.3b.dtb' -exec cp {} /mnt/alt_boot/ \;

# Скопируйте конфигурационные файлы загрузчика из директории 'boot/'
$ sudo cp -r ./boot/* /mnt/alt_boot/
```

Настройте `fstab`.

```bash
# Создайте точку монтирования для загрузочного раздела внутри rootfs
$ sudo mkdir -p /mnt/alt_root/boot/BOOT

# Добавьте запись в fstab для автоматического монтирования загрузочного раздела
$ UUID=$(sudo blkid -o value -s UUID ${DEVICE}3)
$ echo "UUID=$UUID /boot/BOOT/ vfat defaults 0 2" | sudo tee -a /mnt/alt_root/etc/fstab

# Размонтируйте разделы
$ sudo umount /mnt/alt_boot /mnt/alt_root
```

Теперь вы можете вставить SD-карту в VisionFive2 и загрузиться в ALT Linux.

**Логин по умолчанию**: `root`
**Пароль по умолчанию**: `altlinux`

### 5. Первоначальная настройка

После загрузки обновите систему и создайте нового пользователя.

```bash
# Обновите список пакетов и выполните обновление системы
apt-get update
apt-get dist-upgrade

# Создайте нового пользователя и добавьте его в группу wheel для доступа через sudo
useradd -m user
usermod -aG wheel,proc,sys user
passwd user

# Включите доступ через sudo для группы wheel
control sudowheel enabled
```

---

## Сборка кастомного ядра (опционально)

Стандартное ядро может иметь проблемы с графикой, поэтому вы можете
собрать и установить кастомное ядро
([6.12-forge](https://git.altlinux.org/people/kovalev/packages/kernel-image.git?p=kernel-image.git;a=shortlog;h=refs/heads/alt-JH7110_VisionFive2_6.12.y_devel)) с помощью [gear-hsh-wrapper](https://github.com/kovalev0/gear-hsh-wrapper).

### 1. Сборка ядра

Эти команды можно выполнять как на хост-машине, так и непосредственно
на VisionFive2.

```bash
# Для первоначальной настройки см. документацию gear-hsh-wrapper (раздел Quick Start Example).

# Увеличьте лимит времени простоя для среды сборки
$ echo "wlimit_time_short=180" >> ~/.hasher/config

# Клонируйте репозиторий с исходным кодом ядра
# http://git.altlinux.org/people/kovalev/packages/kernel-image.git
$ git clone --branch alt-JH7110_VisionFive2_6.12.y_devel git://git.altlinux.org/people/kovalev/packages/kernel-image.git
$ cd kernel-image

# Укажите вариант (flavour) ядра
$ git config --add gear.specsubst.kflavour 6.12

# Запустите процесс сборки
$ gear-hsh-wrapper-sisyphus-riscv64 build
```

Скомпилированные RPM-пакеты будут доступны в
`~/hasher/sisyphus-riscv64/repo/riscv64/RPMS.hasher/`.

```bash
# Список собранных пакетов на момент написания
$ ls ~/hasher/sisyphus-riscv64/repo/riscv64/RPMS.hasher/
kernel-doc-6.12-6.12.74-alt1.forge.rv64.noarch.rpm
kernel-headers-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm
kernel-headers-modules-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm
kernel-image-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm
kernel-image-6.12-debuginfo-6.12.74-alt1.forge.rv64.riscv64.rpm
kernel-modules-drm-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm
kernel-modules-drm-6.12-debuginfo-6.12.74-alt1.forge.rv64.riscv64.rpm
```

### 2. Установка кастомного ядра

Скопируйте необходимые RPM-пакеты на плату VisionFive2 и установите их.

```bash
# На VisionFive2 установите новые пакеты ядра
# Обязательно: kernel-image, kernel-modules-drm
# Опционально для разработки: kernel-headers-modules
$ sudo apt-get install /path/to/rpms/{kernel-image-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm,kernel-modules-drm-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm,kernel-headers-modules-6.12-6.12.74-alt1.forge.rv64.riscv64.rpm}
```

### 3. Настройка U-Boot для загрузки нового ядра

Скопируйте файлы нового ядра в отдельную директорию на загрузочном
разделе (смонтированном в `/boot/BOOT/`).

```bash
# Замените на фактическую версию ядра
$ KVER="6.12.74-6.12-alt1.forge.rv64"

# Создайте директорию для нового ядра
$ sudo mkdir -p /boot/BOOT/$KVER

# Скопируйте ядро, initrd и файл дерева устройств
$ sudo cp /boot/vmlinuz-$KVER /boot/BOOT/$KVER/vmlinuz
$ sudo cp /boot/initrd-$KVER.img /boot/BOOT/$KVER/initrd.img
$ sudo cp /boot/devicetree/$KVER/starfive/jh7110-starfive-visionfive-2-v1.3b.dtb /boot/BOOT/$KVER/
```

Добавьте новый пункт меню в `/boot/BOOT/extlinux/extlinux.conf`.

```
# Добавьте это в ваш extlinux/extlinux.conf
label l00
    menu label ALT Linux 6.12.74-6.12-alt1.forge.rv64
    linux  /6.12.74-6.12-alt1.forge.rv64/vmlinuz
    initrd /6.12.74-6.12-alt1.forge.rv64/initrd.img
    fdtdir /6.12.74-6.12-alt1.forge.rv64/

    append root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0
```

Перезагрузите плату. В меню U-Boot выберите новый пункт с кастомным
ядром. Успешная загрузка будет выглядеть примерно так:

```
Retrieving file: /extlinux/extlinux.conf
577 bytes read in 12 ms (46.9 KiB/s)

U-Boot menu
1:      ALT Linux 6.12.74-6.12-alt1.forge.rv64
2:      Alt GNU/Linux
Enter choice: 1
1:      ALT Linux 6.12.74-6.12-alt1.forge.rv64
Retrieving file: /6.12.74-6.12-alt1.forge.rv64/initrd.img
8972696 bytes read in 777 ms (11 MiB/s)
Retrieving file: /6.12.74-6.12-alt1.forge.rv64/vmlinuz
12680659 bytes read in 1092 ms (11.1 MiB/s)
append: root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0
Retrieving file: /6.12.74-6.12-alt1.forge.rv64/jh7110-starfive-visionfive-2-v1.3b.dtb
57011 bytes read in 17 ms (3.2 MiB/s)
   Uncompressing Kernel Image
## Flattened Device Tree blob at 46000000
   Booting using the fdt blob at 0x46000000
   Using Device Tree in place at 0000000046000000, end 0000000046010eb2

Starting kernel ...

[    0.000000] Linux version 6.12.74-6.12-alt1.forge.rv64 (builder@localhost.localdomain) (gcc-14 (GCC) 14.3.1 20250812 (ALT Sisyphus_riscv64 14.3.1-alt0.port), GNU ld (GNU Binutils) 2.43.1.20241025) #1 SMP Mon Sep  1 12:20:06 UTC 2025
```

---

## Сборка прошивки из исходников

Этот раздел предназначен для опытных пользователей, которые хотят
собрать U-Boot и OpenSBI с нуля.

### 1. Настройка среды сборки

Используйте [gear-hsh-wrapper](https://github.com/kovalev0/gear-hsh-wrapper)
для создания среды сборки Sisyphus RISC-V.

```bash
# Инициализируйте среду sisyphus-riscv64
$ gear-hsh-wrapper-sisyphus-riscv64 init

# Установите необходимые зависимости для сборки
$ gear-hsh-wrapper-sisyphus-riscv64 install git libssl-devel python3 dtc flex

# Войдите в оболочку сборки
$ gear-hsh-wrapper-sisyphus-riscv64 shell-net
```

Следующие команды необходимо выполнять внутри оболочки
`gear-hsh-wrapper` (`[builder@localhost ...]$`).

### 2. Сборка U-Boot

```bash
[builder@localhost .in]$ cd
[builder@localhost ~]$ git clone --depth=1 -b JH7110_VisionFive2_devel https://github.com/starfive-tech/u-boot.git
[builder@localhost ~]$ cd u-boot/

# Настройте конфигурацию для платы VisionFive2
[builder@localhost u-boot]$ make -j $(nproc) starfive_visionfive2_defconfig

# Соберите U-Boot. Флаги KCFLAGS используются для подавления отдельных предупреждений компилятора.
[builder@localhost u-boot]$ make -j $(nproc) KCFLAGS="-Wno-int-conversion -Wno-implicit-function-declaration"

# В результате будут получены следующие артефакты:
# - u-boot.bin
# - arch/riscv/dts/starfive_visionfive2.dtb
# - spl/u-boot-spl.bin
```

### 3. Обработка SPL

SPL (Secondary Program Loader) U-Boot требует постобработки
специальным инструментом.

```bash
[builder@localhost ~]$ git clone --depth=1 -b master https://github.com/starfive-tech/Tools
[builder@localhost ~]$ cd Tools/spl_tool

# Соберите spl_tool
[builder@localhost spl_tool]$ make -j$(nproc)

# Обработайте файл u-boot-spl.bin
[builder@localhost spl_tool]$ ./spl_tool -c -f ~/u-boot/spl/u-boot-spl.bin

# Результирующий файл будет создан по пути: ~/u-boot/spl/u-boot-spl.bin.normal.out
```

### 4. Сборка OpenSBI и FIT-образа

OpenSBI (Open Supervisor Binary Interface) — финальный образ прошивки.

```bash
[builder@localhost ~]$ git clone --depth=1 -b master https://github.com/starfive-tech/opensbi.git
[builder@localhost ~]$ cd opensbi/

# Соберите OpenSBI с U-Boot в качестве полезной нагрузки
[builder@localhost opensbi]$ make -j $(nproc) PLATFORM=generic \
    FW_PAYLOAD_PATH=~/u-boot/u-boot.bin \
    FW_FDT_PATH=~/u-boot/arch/riscv/dts/starfive_visionfive2.dtb \
    FW_TEXT_START=0x40000000

# Результирующая прошивка будет находиться по пути:
# build/platform/generic/firmware/fw_payload.bin
```

Наконец, создайте FIT-образ (Flattened Image Tree), который будет
загружаться U-Boot.

```bash
[builder@localhost ~]$ cd ~/Tools/uboot_its/

# Скопируйте образ OpenSBI в рабочую директорию
[builder@localhost uboot_its]$ cp ~/opensbi/build/platform/generic/firmware/fw_payload.bin ./

# Используйте mkimage для создания финального образа прошивки
[builder@localhost uboot_its]$ ~/u-boot/tools/mkimage -f visionfive2-uboot-fit-image.its -A riscv -O u-boot -T firmware visionfive2_fw_payload.img

# Финальный образ будет находиться по пути:
# ~/Tools/uboot_its/visionfive2_fw_payload.img
```

В результате будут получены два файла прошивки:
1.  `~/u-boot/spl/u-boot-spl.bin.normal.out`
2.  `~/Tools/uboot_its/visionfive2_fw_payload.img`

Теперь вы можете использовать эти файлы в разделе
[Быстрый старт](#2-запись-прошивки-и-создание-файловых-систем).
