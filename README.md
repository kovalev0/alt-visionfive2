# ALT Linux for VisionFive2

This repository provides instructions, configuration files, and pre-built firmware to deploy ALT Linux on the StarFive VisionFive2 RISC-V board.

The instructions are divided into two main parts:
1.  **[Quick Start](#quick-start)**: Use pre-built firmware and configuration files from this repository to get ALT Linux running quickly.
2.  **[Building Firmware from Source](#building-firmware-from-source)**: Compile U-Boot and OpenSBI from source if you need to customize the bootloaders.

---

## Quick Start

This section guides you through preparing an SD card and deploying ALT Linux using the pre-built items located in this repository.

### 1. Prepare the SD Card

First, identify your SD card device (e.g., `/dev/sdX`). **Be extremely careful**, as the following commands will erase all data on the specified device.

```bash
# Replace /dev/sdb with your actual SD card device
DEVICE=/dev/sdb

# Wipe existing partition information
sudo wipefs -a        $DEVICE
sudo sgdisk --zap-all $DEVICE
```

Next, create the necessary partitions using `sgdisk`. The layout includes partitions for the SPL (Second-stage Program Loader), U-Boot, kernel/bootloader configs, and the root filesystem.

```bash
# Create a new GPT partition table
sudo sgdisk --clear \
     --new=1:4096:8191    --change-name=1:"spl"    --typecode=1:2E54B353-1271-4842-806F-E436D6AF6985 \
     --new=2:8192:16383   --change-name=2:"uboot"  --typecode=2:BC13C2FF-59E6-4262-A352-B275FD6F7172 \
     --new=3:16384:614399 --change-name=3:"kernel" --typecode=3:EBD0A0A2-B9E5-4433-87C0-68B6B72699C7 \
     --new=4:614400:0     --change-name=4:"root"   --typecode=4:0FC63DAF-8483-4772-8E79-3D69D8477DE4  \
     $DEVICE

# Reread the partition table
sudo partprobe -s $DEVICE
```

### 2. Flash Firmware and Create Filesystems

Write the pre-built firmware files to the corresponding partitions.

```bash
# Write the SPL and U-Boot firmware from the 'firmware/' directory
sudo dd if=./firmware/u-boot-spl.bin.normal.out  of=${DEVICE}1 bs=512 conv=notrunc status=progress
sudo dd if=./firmware/visionfive2_fw_payload.img of=${DEVICE}2 bs=512 conv=notrunc status=progress

# Create filesystems for the boot and root partitions
sudo mkfs.vfat -F 32 -n BOOT   ${DEVICE}3
sudo mkfs.ext4 -L       ROOTFS ${DEVICE}4
```

### 3. Deploy ALT Linux Root Filesystem

Mount the newly created partitions and extract the ALT Linux [rootfs tarball](https://nightly.altlinux.org/sisyphus-riscv64/current/).

```bash
# Create mount points
mkdir -p /mnt/alt_boot /mnt/alt_root

# Mount the partitions
sudo mount ${DEVICE}3 /mnt/alt_boot
sudo mount ${DEVICE}4 /mnt/alt_root

# Download and extract the latest ALT Linux rootfs for riscv64
# You can use MATE (default) or XFCE. Choose one.
wget https://nightly.altlinux.org/sisyphus-riscv64/current/regular-mate-latest-riscv64.tar.xz
# wget https://nightly.altlinux.org/sisyphus-riscv64/current/regular-xfce-latest-riscv64.tar.xz

# Extract the downloaded archive.
sudo tar -Jxf ./regular-mate-latest-riscv64.tar.xz -C /mnt/alt_root
# If you chose XFCE, use the command below instead
# sudo tar -Jxf ./regular-xfce-latest-riscv64.tar.xz -C /mnt/alt_root
```

### 4. Configure the System

Copy the kernel, initrd, and device tree blob to the boot partition. Then, copy the bootloader configuration files from this repository.

```bash
# Copy kernel and initrd
sudo cp /mnt/alt_root/boot/{vmlinuz,initrd.img} /mnt/alt_boot/

# Find and copy the correct device tree blob
sudo find /mnt/alt_root -name 'jh7110-starfive-visionfive-2-v1.3b.dtb' -exec cp {} /mnt/alt_boot/ \;

# Copy the bootloader configuration files from the 'boot/' directory
sudo cp -r ./boot/* /mnt/alt_boot/
```

Configure `apt` repositories and `fstab`.

```bash
# Set up APT repositories
sudo tee /mnt/alt_root/etc/apt/sources.list <<EOF
rpm [http://ftp.altlinux.org/pub/distributions/ALTLinux/ports/riscv64](http://ftp.altlinux.org/pub/distributions/ALTLinux/ports/riscv64) Sisyphus/riscv64 classic
rpm [http://ftp.altlinux.org/pub/distributions/ALTLinux/ports/riscv64](http://ftp.altlinux.org/pub/distributions/ALTLinux/ports/riscv64) Sisyphus/noarch  classic
EOF

# Create a mount point for the boot partition inside the rootfs
sudo mkdir -p /mnt/alt_root/boot/BOOT

# Add an fstab entry to auto-mount the boot partition
UUID=$(sudo blkid -o value -s UUID ${DEVICE}3)
echo "UUID=$UUID /boot/BOOT/ vfat defaults 0 2" | sudo tee -a /mnt/alt_root/etc/fstab

# Unmount partitions
sudo umount /mnt/alt_boot /mnt/alt_root
```

You can now insert the SD card into your VisionFive2 and boot into ALT Linux.

**Default login**: `root`
**Default password**: `altlinux`

### 5. First-Time Setup

After booting, update the system and create a new user.

```bash
# Update package lists and upgrade the system
apt-get update
apt-get dist-upgrade

# Create a new user and add them to the wheel group for sudo access
useradd -m user
usermod -aG wheel,proc,sys user
passwd user

# Enable sudo access for the wheel group
control sudowheel enabled
```

---

## Building a Custom Kernel (Optional)

The default kernel might have issues with graphics. For a better experience, you can build and install a custom kernel ([6.12-forge](https://git.altlinux.org/people/kovalev/packages/kernel-image.git?p=kernel-image.git;a=shortlog;h=refs/heads/alt-JH7110_VisionFive2_6.12.y_devel))  using the [gear-hsh-wrapper](https://github.com/kovalev0/gear-hsh-wrapper).

### 1. Build the Kernel

These commands can be executed on your host machine or directly on the VisionFive2.

```bash
# For initial setup, see the gear-hsh-wrapper documentation (Quick Start Example section).

# Increase the idle time limit for the build environment
echo "wlimit_time_short=180" >> ~/.hasher/config

# Clone the kernel source git repository
git clone --branch alt-JH7110_VisionFive2_6.12.y_devel http://git.altlinux.org/people/kovalev/packages/kernel-image.git
cd kernel-image

# Specify the kernel flavour
git config --add gear.specsubst.kflavour 6.12

# Start the build process
gear-hsh-wrapper-sisyphus-riscv64 build
```

The compiled RPM packages will be available in `~/hasher/sisyphus-riscv64/repo/riscv64/RPMS.hasher/`.

```bash
# List of built packages at the time of writing
ls ~/hasher/sisyphus-riscv64/repo/riscv64/RPMS.hasher/
kernel-doc-6.12-6.12.44-alt1.forge.rv64.noarch.rpm
kernel-headers-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm
kernel-headers-modules-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm
kernel-image-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm
kernel-image-6.12-debuginfo-6.12.44-alt1.forge.rv64.riscv64.rpm
kernel-modules-drm-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm
kernel-modules-drm-6.12-debuginfo-6.12.44-alt1.forge.rv64.riscv64.rpm
```


### 2. Install the Custom Kernel

Copy the required RPM packages to your VisionFive2 board and install them.

```bash
# On the VisionFive2, install the new kernel packages
# Required: kernel-image, kernel-modules-drm
# Optional for development: kernel-headers-modules
sudo apt-get install /path/to/rpms/{kernel-image-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm,kernel-modules-drm-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm,kernel-headers-modules-6.12-6.12.44-alt1.forge.rv64.riscv64.rpm}
```

### 3. Configure U-Boot to Load the New Kernel

Copy the new kernel files to a new directory on the boot partition (mounted at `/boot/BOOT/`).

```bash
# Replace with the actual kernel version
KVER="6.12.44-6.12-alt1.forge.rv64"

# Create a directory for the new kernel
sudo mkdir -p /boot/BOOT/$KVER

# Copy the kernel, initrd, and device tree blob
sudo cp /boot/vmlinuz-$KVER /boot/BOOT/$KVER/vmlinuz
sudo cp /boot/initrd-$KVER.img /boot/BOOT/$KVER/initrd.img
sudo cp /boot/devicetree/$KVER/starfive/jh7110-starfive-visionfive-2-v1.3b.dtb /boot/BOOT/$KVER/
```

Add a new menu entry to `/boot/BOOT/extlinux/extlinux.conf`.

```
# Append this to your extlinux/extlinux.conf
label l00
    menu label ALT Linux 6.12.44-6.12-alt1.forge.rv64
    linux  /6.12.44-6.12-alt1.forge.rv64/vmlinuz
    initrd /6.12.44-6.12-alt1.forge.rv64/initrd.img
    fdtdir /6.12.44-6.12-alt1.forge.rv64/

    append root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0
```

Reboot the board. At the U-Boot menu, select the new kernel entry. A successful boot will look similar to this:

```
Retrieving file: /extlinux/extlinux.conf
577 bytes read in 12 ms (46.9 KiB/s)

U-Boot menu
1:      ALT Linux 6.12.44-6.12-alt1.forge.rv64
2:      Alt GNU/Linux
Enter choice: 1
1:      ALT Linux 6.12.44-6.12-alt1.forge.rv64
Retrieving file: /6.12.44-6.12-alt1.forge.rv64/initrd.img
8972696 bytes read in 777 ms (11 MiB/s)
Retrieving file: /6.12.44-6.12-alt1.forge.rv64/vmlinuz
12680659 bytes read in 1092 ms (11.1 MiB/s)
append: root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0
Retrieving file: /6.12.44-6.12-alt1.forge.rv64/jh7110-starfive-visionfive-2-v1.3b.dtb
57011 bytes read in 17 ms (3.2 MiB/s)
   Uncompressing Kernel Image
## Flattened Device Tree blob at 46000000
   Booting using the fdt blob at 0x46000000
   Using Device Tree in place at 0000000046000000, end 0000000046010eb2

Starting kernel ...

[    0.000000] Linux version 6.12.44-6.12-alt1.forge.rv64 (builder@localhost.localdomain) (gcc-14 (GCC) 14.3.1 20250812 (ALT Sisyphus_riscv64 14.3.1-alt0.port), GNU ld (GNU Binutils) 2.43.1.20241025) #1 SMP Mon Sep  1 12:20:06 UTC 2025
```

---

## Building Firmware from Source

This section is for advanced users who want to build U-Boot and OpenSBI from scratch.

### 1. Set Up the Build Environment

Use [gear-hsh-wrapper](https://github.com/kovalev0/gear-hsh-wrapper) to create a Sisyphus RISC-V build environment.

```bash
# Initialize the sisyphus-riscv64 environment
gear-hsh-wrapper-sisyphus-riscv64 init

# Install necessary build dependencies
gear-hsh-wrapper-sisyphus-riscv64 install git libssl-devel python3 dtc flex

# Enter the build shell
gear-hsh-wrapper-sisyphus-riscv64 shell-net
```

The following commands should be run inside the `gear-hsh-wrapper` shell (`[builder@localhost ...]$`).

### 2. Build U-Boot

```bash
[builder@localhost .in]$ cd
[builder@localhost ~]$ git clone --depth=1 -b JH7110_VisionFive2_devel https://github.com/starfive-tech/u-boot.git
[builder@localhost ~]$ cd u-boot/

# Configure for the VisionFive2 board
[builder@localhost u-boot]$ make -j $(nproc) starfive_visionfive2_defconfig

# Build U-Boot. KCFLAGS are used to suppress specific compiler warnings.
[builder@localhost u-boot]$ make -j $(nproc) KCFLAGS="-Wno-int-conversion -Wno-implicit-function-declaration"

# Required artifacts are now built:
# - u-boot.bin
# - arch/riscv/dts/starfive_visionfive2.dtb
# - spl/u-boot-spl.bin
```

### 3. Process the SPL

The U-Boot SPL (Secondary Program Loader) requires post-processing with a special tool.

```bash
[builder@localhost ~]$ git clone --depth=1 -b master https://github.com/starfive-tech/Tools
[builder@localhost ~]$ cd Tools/spl_tool

# Build the spl_tool
[builder@localhost spl_tool]$ make -j$(nproc)

# Process the u-boot-spl.bin file
[builder@localhost spl_tool]$ ./spl_tool -c -f ~/u-boot/spl/u-boot-spl.bin

# The output file is generated at: ~/u-boot/spl/u-boot-spl.bin.normal.out
```

### 4. Build OpenSBI and FIT Image

OpenSBI (Open Supervisor Binary Interface) is the final firmware payload.

```bash
[builder@localhost ~]$ git clone --depth=1 -b master https://github.com/starfive-tech/opensbi.git
[builder@localhost ~]$ cd opensbi/

# Build OpenSBI with U-Boot as the payload
[builder@localhost opensbi]$ make -j $(nproc) PLATFORM=generic \
    FW_PAYLOAD_PATH=~/u-boot/u-boot.bin \
    FW_FDT_PATH=~/u-boot/arch/riscv/dts/starfive_visionfive2.dtb \
    FW_TEXT_START=0x40000000

# The resulting firmware is located at:
# build/platform/generic/firmware/fw_payload.bin
```

Finally, create the FIT (Flattened Image Tree) image that U-Boot will load.

```bash
[builder@localhost ~]$ cd ~/Tools/uboot_its/

# Copy the OpenSBI payload to the working directory
[builder@localhost uboot_its]$ cp ~/opensbi/build/platform/generic/firmware/fw_payload.bin ./

# Use mkimage to create the final firmware image
[builder@localhost uboot_its]$ ~/u-boot/tools/mkimage -f visionfive2-uboot-fit-image.its -A riscv -O u-boot -T firmware visionfive2_fw_payload.img

# The final image is located at:
# ~/Tools/uboot_its/visionfive2_fw_payload.img
```

The two resulting firmware files are:
1.  `~/u-boot/spl/u-boot-spl.bin.normal.out`
2.  `~/Tools/uboot_its/visionfive2_fw_payload.img`

You can now use these files in the [Quick Start](#2-flash-firmware-and-create-filesystems) section.
