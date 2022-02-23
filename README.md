# Introduction

Enable OPTIGA™ TPM 2.0 in U-Boot on Raspberry Pi 4. Extend critical measurements to PCR before transitioning to Linux kernel.

# Table of Contents

- **[Prerequisites](#prerequisites)**
- **[Raspberry Pi 4 Base Image](#raspberry-pi-4-base-image)**
- **[Rebuild Raspberry Pi 4 Kernel (32-bit)](#rebuild-raspberry-pi-4-kernel-32-bit)**
- **[Rebuild Raspberry Pi 4 Kernel (64-bit)](#rebuild-raspberry-pi-4-kernel-64-bit)**
- **[Build U-Boot Binary (64-bit)](#build-u-boot-binary-64-bit)**
- **[Build U-Boot Boot Script (64-bit)](#build-u-boot-boot-script-64-bit)**
- **[Enter U-Boot](#enter-u-boot)**
- **[Raspberry Pi OS](#raspberry-pi-os)**
- **[References](#references)**
- **[License](#license)**

# Prerequisites

- Host machine: 
  ```
  $ lsb_release -a
  No LSB modules are available.
  Distributor ID:	Ubuntu
  Description:	Ubuntu 20.04.2 LTS
  Release:	20.04
  Codename:	focal

  $ uname -a
  Linux ubuntu 5.8.0-59-generic #66~20.04.1-Ubuntu SMP Thu Jun 17 11:14:10 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
  ```
- 8GB SD Card
- Raspberry Pi 4 Model B [[1]](#1) with IRIDIUM9670 TPM2.0 [[2]](#2)
  <br><img src="https://github.com/wxleong/tpm2-uboot/raw/master/media/raspberry-with-tpm.jpg" width="50%">

# Raspberry Pi 4 Base Image

On your host machine.

Install dependencies:
```
$ sudo apt update
$ sudo apt install curl
```

Download the 64-bit *Raspberry Pi OS with desktop* image (~1.2GB)(Linux 5.10.63-v8+ aarch64 GNU/Linux):
```
$ curl https://downloads.raspberrypi.org/raspios_arm64/images/raspios_arm64-2021-11-08/2021-10-30-raspios-bullseye-arm64.zip --output ~/2021-10-30-raspios-bullseye-arm64.zip
$ cd ~
$ unzip 2021-10-30-raspios-bullseye-arm64.zip
```

<del>Download the 32-bit *Raspberry Pi OS with desktop* image (~1.2GB):</del>
```
$ curl https://downloads.raspberrypi.org/raspios_armhf/images/raspios_armhf-2021-11-08/2021-10-30-raspios-bullseye-armhf.zip --output ~/2021-10-30-raspios-bullseye-armhf.zip
$ cd ~
$ unzip 2021-10-30-raspios-bullseye-armhf.zip
```

Connect your microSD card to the Ubuntu machine. Execute command `sudo fdisk -l` to find your microSD (e.g., `/dev/sdc`).
```
$ sudo fdisk -l
Disk /dev/???: 7.3 GiB, 7876902912 bytes, 15384576 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x1fa31320

Device     Boot  Start      End  Sectors  Size Id Type
/dev/???1         8192   532479   524288  256M  c W95 FAT32 (LBA)
/dev/???2       532480 15384575 14852096  7.1G 83 Linux
```

Unmount all partitions.
```
$ sudo umount /dev/???1
$ sudo umount /dev/???2
```

**Warning, following command will write to the path `/dev/???`, selecting the wrong path will most certainly result in data loss or killing your operating system!**
```
$ sudo dd if=~/2021-10-30-raspios-bullseye-arm64.img of=/dev/??? bs=100M status=progress oflag=sync
```

# Rebuild Raspberry Pi 4 Kernel (32-bit)

**Ignore this section, this is for future reference only.**

Rebuild the kernel to obtain an uncompressed flat image. The original kernel image is a gzip formatted image.

On your host machine.

Install dependencies:
```
$ sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev
```

Install 32-bit toolchain:
```
$ sudo apt install crossbuild-essential-armhf
```

Download kernel source:
```
$ git clone https://github.com/raspberrypi/linux ~/linux
$ cd ~/linux
$ git checkout 1.20210928
$ make kernelversion
5.10.63
```

Build:
```
$ KERNEL=kernel7l
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2711_defconfig
$ make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
```

Copy to microSD card:
```
$ mkdir mnt
$ mkdir mnt/fat32
$ mkdir mnt/ext4
$ sudo umount /dev/sd?1
$ sudo umount /dev/sd?2
$ sudo mount /dev/sd?1 mnt/fat32
$ sudo mount /dev/sd?2 mnt/ext4
$ sudo env PATH=$PATH make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=mnt/ext4 modules_install
$ sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
$ sudo cp arch/arm/boot/zImage mnt/fat32/$KERNEL.img
$ sudo cp arch/arm/boot/dts/*.dtb mnt/fat32/
$ sudo cp arch/arm/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
$ sudo cp arch/arm/boot/dts/overlays/README mnt/fat32/overlays/
$ sudo umount mnt/fat32
$ sudo umount mnt/ext4
```

# Rebuild Raspberry Pi 4 Kernel (64-bit)

Rebuild the kernel to obtain an uncompressed flat image. The original kernel image is a gzip formatted image.

On your host machine.

Install dependencies:
```
$ sudo apt install git bc bison flex libssl-dev make libc6-dev libncurses5-dev
```

Install 64-bit toolchain:
```
$ sudo apt install crossbuild-essential-arm64
```

Download kernel source:
```
$ git clone https://github.com/raspberrypi/linux ~/linux
$ cd ~/linux
$ git checkout 1.20210928
$ make kernelversion
5.10.63
```

Build:
```
$ KERNEL=kernel8
$ make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig
$ make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```

Copy to microSD Card:
```
$ mkdir mnt
$ mkdir mnt/fat32
$ mkdir mnt/ext4
$ sudo umount /dev/sd?1
$ sudo umount /dev/sd?2
$ sudo mount /dev/sd?1 mnt/fat32
$ sudo mount /dev/sd?2 mnt/ext4
$ sudo env PATH=$PATH make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/ext4 modules_install
$ sync
$ sudo cp mnt/fat32/$KERNEL.img mnt/fat32/$KERNEL-backup.img
$ sudo cp arch/arm64/boot/Image mnt/fat32/$KERNEL.img
$ sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/fat32/
$ sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/fat32/overlays/
$ sudo cp arch/arm64/boot/dts/overlays/README mnt/fat32/overlays/
$ sudo umount /dev/sd?1
$ sudo umount /dev/sd?2
```

# Build U-Boot Binary (64-bit)

On your host machine.

Install dependencies:
```
$ sudo apt install device-tree-compiler
```

Download U-Boot source:
```
$ git clone https://github.com/u-boot/u-boot ~/u-boot
$ cd ~/u-boot
$ git checkout v2022.01-rc2
```

Build:
```
$ make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu- rpi_4_defconfig
$ make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu- menuconfig
    Boot options  --->
        [*] Enable preboot
        (pci enum; usb start; setenv bootdelay 10) preboot default value
    Library routines  --->
      Security support  --->
        [*] Trusted Platform Module (TPM) Support
    Device Drivers  --->
      [*] SPI Support  --->
        [*]   Enable Driver Model for SPI drivers
        [*]   Soft SPI driver
      TPM support  --->
        [*] TPMv2.x support
        [*]   Enable support for TPMv2.x SPI chips
    Command line interface  --->
      Security commands  --->
        [*] Support 'hash' command
        [*] Enable the 'tpm' command
$ make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu- all
```

Build U-Boot device tree overlay:
```
$ git clone https://github.com/wxleong/tpm2-uboot ~/tpm2-uboot
$ cd ~/tpm2-uboot
$ dtc -O dtb -b 0 -@ uboot-tpm-slb9670-overlay.dts -o uboot-tpm-slb9670.dtbo
```

Copy `u-boot.bin` to your microSD card boot partition, `uboot-tpm-slb9670.dtbo` to the overlays folder, and finally edit the `config.txt` file:
```
enable_uart=1
kernel=u-boot.bin
dtoverlay=uboot-tpm-slb9670
```

# Build U-Boot Boot Script (64-bit)

The example boot scipt `boot.scr` only works with uncompressed kernel image.

Build U-Boot boot script:
```
$ git clone https://github.com/wxleong/tpm2-uboot ~/tpm2-uboot
$ cd ~/tpm2-uboot
$ ~/u-boot/tools/mkimage -A arm64 -T script -C none -n "u-boot script" -d boot.scr boot.scr.uimg
```

Copy `boot.scr.uimg` to your microSD card boot partition.

# Enter U-Boot

Power up the rpi and it will boot into U-Boot. Hit any key to interrupt autoboot. Otherwise, the `boot.scr.uimg` will be executed. Some shell commands to try on U-Boot:
| Command | Info |
|--|--|
| `help` | Print supported commands. |
| `bdinfo` | Print board info. |
| `env print` | Print environment variable. |
| `boot` | Execute `boot.scr.uimg`. |
| `reset` | Reset CPU. |
| `tpm2 init` | Initialize TPM software stack. |
| `tpm2 startup TPM2_SU_CLEAR` | TPM2_Startup command (reset state). |
| `tpm2 clear TPM2_RH_PLATFORM` | TPM2_Clear command. |
| `tpm2 pcr_read 0 20000000` | Read PCR 0 (SHA256 bank) to memory address 0x20000000. |
| `tpm2 pcr_extend 0 20000000` | Extend PCR 0 (SHA256 bank) with digest at memory address 0x20000000. |
| `md.b 20000000 20` | Display 32 bytes of data at address 0x20000000. |
| `fatsize mmc 0:1 kernel8.img` | Measure the size of a file and put the result into variable `${filesize}`. |
| `hash sha256 ${kernel_addr_r} ${filesize} digest` | Compute sha256 digest of an area of memory and put the result into variable `${digest}`. Memory starting address `${kernel_addr_r}` size of `${filesize}` . |
| `hash sha1 ${kernel_addr_r} ${filesize} *20000000` | Compute sha256 digest of an area of memory and put the result to address 0x20000000. Memory starting address `${kernel_addr_r}` size of `${filesize}`. |
| `crc` | Checksum calculation. |

An example to measure device tree overlay and kernel binary:
```
U-Boot> tpm2 init
U-Boot> tpm2 startup TPM2_SU_CLEAR
U-Boot> tpm2 clear TPM2_RH_PLATFORM

U-Boot> fatsize mmc 0:1 overlays/tpm-slb9670.dtbo
U-Boot> fatload mmc 0:1 ${kernel_addr_r} overlays/tpm-slb9670.dtbo
U-Boot> hash sha256 ${kernel_addr_r} ${filesize} *20000000
U-Boot> md.b 20000000 20
U-Boot> tpm2 pcr_extend 0 20000000
U-Boot> tpm2 pcr_read 0 20000000

U-Boot> fatsize mmc 0:1 kernel8.img
U-Boot> fatload mmc 0:1 ${kernel_addr_r} kernel8.img
U-Boot> hash sha256 ${kernel_addr_r} ${filesize} *20000000
U-Boot> md.b 20000000 20
U-Boot> tpm2 pcr_extend 0 20000000
U-Boot> tpm2 pcr_read 0 20000000
```

Manually boot into Raspberry Pi OS:
<pre><code># Load kernel base dtb
<del>U-Boot> fatload mmc 0:1 ${fdt_addr} bcm2711-rpi-4-b.dtb</del> // this is auto loaded by rpi firmware, <a href="https://forums.raspberrypi.com/viewtopic.php?t=314502">read here</a>
U-Boot> fdt addr ${fdt_addr}
U-Boot> fdt resize 0x2000 // resize fdt to size + padding to 4k addr + extra size 0x2000. This is neccessary otherwise "fdt apply" will throw error FDT_ERR_NOSPACE

# Apply overlay
U-Boot> setexpr fdt_overlay_addr ${fdt_addr} + 0xF000
U-Boot> fatload mmc 0:1 ${fdt_overlay_addr} overlays/tpm-slb9670.dtbo
U-Boot> fdt apply ${fdt_overlay_addr}

# Load & boot the kernel image
U-Boot> fdt get value bootargs /chosen bootargs
U-Boot> fatload mmc 0:1 ${kernel_addr_r} kernel8.img
<del>U-Boot> fatsize mmc 0:1 kernel8.img</del> // this is needed only if the kernel image is compressed
<del>U-Boot> setenv kernel_comp_size ${filesize}</del> // this is needed only if the kernel image is compressed
<del>U-Boot> setenv kernel_comp_addr_r 0x0A000000</del> // this is needed only if the kernel image is compressed
U-Boot> booti ${kernel_addr_r} - ${fdt_addr}
</code></pre>

Boot into Raspberry Pi OS with a single command by executing the boot script `boot.scr.uimg` (the example scipt only works with uncompressed kernel image):
```
U-Boot> boot
```

# Raspberry Pi OS

Once you entered the Raspberry Pi OS, check if the TPM is enabled by looking for the device nodes:
```
$ ls /dev | grep tpm
/dev/tpm0
/dev/tpmrm0
```

# References

<a id="1">[1] https://www.raspberrypi.org/products/raspberry-pi-4-model-b/</a><br>
<a id="2">[2] https://www.infineon.com/cms/en/product/evaluation-boards/iridium9670-tpm2.0-linux/</a><br>
<a id="3">[3] https://www.infineon.com/cms/en/product/security-smart-card-solutions/optiga-embedded-security-solutions/optiga-tpm/</a><br>
<a id="4">[4] https://www.raspberrypi.com/documentation/computers/linux_kernel.html#cross-compiling-the-kernel</a><br>
<a id="5">[5] https://github.com/u-boot/u-boot</a><br>
<a id="6">[6] https://u-boot.readthedocs.io/en/latest/build/</a><br>
<a id="7">[7] https://u-boot.readthedocs.io/en/latest/usage/index.html#shell-commands</a><br>
<a id="8">[8] https://github.com/joholl/rpi4-uboot-tpm</a><br>

# License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.