# AOSP build for Akebi96

## Prerequisites

### Install Packages

On the Ubuntu 18.04, following packages are needed for basic BSP.

```
apt-get install --fix-missing -y git bc cmake ncurses-dev autoconf bison ccache cscope curl flex gdisk libfdt-dev libglib2.0-dev libpixman-1-dev netcat python-crypto python-serial uuid-dev xz-utils zlib1g-dev gawk wget git-core diffstat unzip texinfo gcc-multilib build-essential chrpath socat libsdl1.2-dev xterm cpio libssl-dev
```

And also followings for AOSP

```
apt-get install -y mtd-utils genromfs sudo stgit device-tree-compiler python3 iputils-ping iasl sparse bsdmainutils u-boot-tools img2simg repo openjdk-8-jdk ccache libgl1-mesa-dev libxml2-utils xsltproc lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev zip dosfstools mtools simg2img connect-proxy locales python-mako python-pycryptopp kmod
```

### Install Cross Compilers

Also, you need to install cross build gcc.

```
cd /opt/
wget http://releases.linaro.org/components/toolchain/binaries/7.3-2018.05/arm-linux-gnueabihf/gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz
tar xf gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf.tar.xz
wget http://releases.linaro.org/components/toolchain/binaries/7.3-2018.05/aarch64-linux-gnu/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
tar xf gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
export PATH="$PATH:/opt/gcc-linaro-7.3.1-2018.05-x86_64_arm-linux-gnueabihf/bin:/opt/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin"
```


## Build the Kernel and Drivers

### Preparation

Set environmental variables and make working directories.

```
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
mkdir -p ~/aosp/kernel-build ~/aosp/android ~/aosp/images ~/aosp/tftpboot
```

### Download Kernel Related Repositories

Download kernel, kconfigs and out-of-tree drivers as below 

```
cd ~/aosp/
git clone -b unph-android-v4.19-testing --single-branch https://github.com/96boards-akebi96/linux.git
git clone -b master --single-branch https://github.com/96boards-akebi96/akebi96-configs.git
git clone -b master --single-branch https://android.googlesource.com/kernel/configs
git clone -b master --single-branch https://github.com/96boards-akebi96/rtl8822bu.git
git clone -b master --single-branch https://github.com/96boards-akebi96/rtk_btusb.git
```

### Download Mali kernel driver

Download mali patches.

```
git clone -b master --single-branch https://github.com/96boards-akebi96/akebi96-mali-patches.git
```

Also get the Mali kernel driver (TX041-SW-99002-r26p0-01rel0.tgz) from the arm web site ( https://developer.arm.com/products/software/mali-drivers/midgard-kernel ).

And prepare the source code.

```
tar xzf TX041-SW-99002-r26p0-01rel0.tgz -C ~/aosp/
mv ~/aosp/TX041-SW-99002-r26p0-01rel0 ~/aosp/mali-midgard
cd mali-midgard/driver/product/kernel/
cat ~/aosp/akebi96-mali-patches/series | while read $p; do \
       patch -p1 < ../akebi96-mali-patches/$p; done
```

### Build ACK-4.19 based kernel

And build the kernel as below

```
cd ~/aosp/linux
export KBUILD=~/aosp/kernel-build
export KCONFIG_CONFIG=$KBUILD/.config
export JOBS=`getconf _NPROCESSORS_ONLN`
make O=$KBUILD  defconfig
./scripts/kconfig/merge_config.sh -m ${KCONFIG_CONFIG} \
        ~/aosp/akebi96-configs/linux/akebi96-base.config \
        ~/aosp/configs/android-4.19/android-base.config \
        ~/aosp/configs/android-4.19/android-recommended.config \
        ~/aosp/configs/android-4.19/android-recommended-arm64.config \
        ~/aosp/akebi96-configs/linux/akebi96-aosp-vendor.config
make O=$KBUILD  olddefconfig
make O=$KBUILD  -j $JOBS  Image socionext/uniphier-ld20-akebi96.dtb
cp ~/aosp/kernel-build/arch/arm64/boot/Image ~/aosp/images/
cp ~/aosp/kernel-build/arch/arm64/boot/dts/socionext/uniphier-ld20-akebi96.dtb ~/aosp/images/
```

And Build out-of-tree drivers

```
KVER=`make O=$KBUILD -s kernelrelease`
cd ~/aosp/rtl8822bu/
make clean
make KSRC=~/aosp/linux KVER=${KVER} O=${KBUILD} -j ${JOBS}
cp 8822bu.ko ~/aosp/images/
cd ~/aosp/rtk_btusb/
make KBUILD=${KBUILD} -j ${JOBS}
cp rtk_btusb.ko ~/aosp/images/
cd ~/aosp/mali-midgard/
make clean
make KERNEL_DIR=~/aosp/linux MAKETOP=~/aosp/images O=${KBUILD} modules -j  $JOBS
cp drivers/gpu/arm/midgard/mali_kbase.ko ~/aosp/images/
```

At this point, ~/aosp/images/ contain the kernel, dtb and modules that we copy over images that come with android sync.

## Build AOSP 9 for Akebi96

### Downlaod AOSP 9 (latest) for Akebi96

Download and sync the AOSP repositories. This may take a long time (depends on your network performance, etc.)

```
cd ~/aosp/android
git clone  -b master --single-branch https://github.com/96boards-akebi96/akebi96-known-good-manifests.git
repo init -u https://android.googlesource.com/platform/manifest -b master
cp akebi96-known-good-manifests/akebi96.xml .repo/manifests/
repo sync -j $JOBS -m akebi96.xml
```

And copy the pre-build binaries

```
cp ~/aosp/images/Image ~/aosp/images/uniphier-ld20-akebi96.dtb \
       ~/aosp/images/8822bu.ko ~/aosp/images/rtk_btusb.ko ~/aosp/images/mali_kbase.ko \
       device/linaro/akebi96/copy/
```

### Build AOSP for Akebi96

This may take a few hours. Setting ARCH=arm is for building OP-TEE.

```
export ARCH=arm
source build/envsetup.sh
lunch akebi96-userdebug
make -j $JOBS
```

### Split system.img into smaller files

Since LD20 reserves the last 0x4b bytes of memory for each channel, we can not touch it. And on Akebi96 board, we have 1GB DDR for each channel (3 channels, 3GB in total). This means we can not use the image file over 1GB. system.img will be bigger than 1GB, so we have to split it into 512MB.

Convert sparse-image to raw image and split it as below.

```
simg2img system.img system-raw.img
split -d -b 512M system-raw.img system.img
rm system-raw.img
```

Since the system raw image size is 1.5GB, you must have 3 files, _system.img00, _system.img01, and _system.img02. Then make those sparse-image and compressed.

```
for img in _system.img0* ; do img2simg $img ${img#_}; rm $img ; gzip ${img#_}; done
```


### Copy image files for TFTP

Copy the image files into tftpboot directory.

```
cp boot_fat_sparse.img system.img*.gz userdata.img vendor.img ~/aosp/tftpboot/
```

Now you have AOSP images under ~/aosp/tftpboot/

## Install AOSP to Akebi96

Here, we will explain how to install AOSP image via TFTP. Of course there are several other ways to do it, but using TFTP will be a solid way to do that (but take a long time to transfer the image). 

To install the image, you may need the latest U-Boot on the board since it supports writing Android Sparse Image to eMMC.

### Start TFTPD
Install TFTPD and start it. This is Ubuntu based system example.

```
apt-get install -y tftpd-hpa
/usr/sbin/in.tftpd --user tftp --address 0.0.0.0:69 --secure ~/aosp/tftpboot
```

### Board Setup

Setup the board to connect cables correctly. 

- Make sure the Power Switch(SW2402) is off (pop-up)
- Connect AC adapter to the DC Jack (J2401)
- Connect Ethernet cable to the ether connector (CN3600) and other side to the ethernet hub (which connect to host PC too)
- Connect micro-USB cable to USB-UART UART#0(CN2501) and other side to host PC
- Make sure the Boot Mode Switch (SW2002) is "BE_BOOT"

Now you are ready to boot up the board.

### Board Bootup

At first, you must start minicom (or other serial console program) on the USB-UART port, which is detected when you connect it to host PC. If it is ttyUSB0, run below command on host PC.

```
minicom -D /dev/ttyUSB0
```

Then, power up the board (push the power switch). Soon you will see U-Boot boot up.

Please interrupt the boot when you see "Hit any key to stop autoboot:", then you will be in the U-Boot console like below.

```
Warning: ethernet@65000000 (eth0) using random MAC address - ee:ec:8b:f9:85:5c
eth0: ethernet@65000000
Hit any key to stop autoboot: 0 
=>
```

### Setup Network Parameters

To use TFTP, we have to setup tftp server IP and netmasks. If you use a class-C (/24) local network, you need to setup following parameters.

|Parameter  | Description                     |    default value|
|:----------|:-------------------------------| ----------------:|
|serverip   |	IP address of TFTP server       |	192.168.11.1 |
|gatewayip  |	IP address of default gateway   |	192.168.11.1 |
|ipaddr     |	IP address of the akebi96 board |	192.168.11.10 |
|netmask    |	Netmask of the local network    |	255.255.255.0 |

You can setup these parameters as below (just an example).

```
=> setenv serverip 192.168.1.2
=> setenv gatewayip 192.168.1.1
=> setenv ipaddr 192.168.1.3
=> setenv netmask 255.255.255.0
```

### Create GPT on eMMC using U-Boot

Please note the partition sizes are hardcoded in Android (BoardConfig.mk) as below.
source in order to generate exact sized images.

|Label   |Start(MB)|Size(MB)| Start Sector|					UUID|
|:-------|--------:|-------:|------------:|--------------------------------------|
|boot    |	   32|	     64|    0x0010000| 49A4D17F-93A3-45C1-A0DE-F50B2EBE2599|
|recovery|	   96|	     64|    0x0030000| 4177C722-9E92-4AAB-8644-43502BFD5506|
|system  |	  160|	   1536|    0x0050000| 38F428E6-D326-425D-9140-6E0EA133647C|
|userdata|	 1696|	   1024|    0x0350000| DC76DDA9-5AC1-491C-AF42-A82591580C0D|
|vendor  |	 2720|	   1024|    0x0550000| C5A0AEEC-13EA-11E5-A1B1-001E67CA0C3C|

On the U-Boot console, run following command to write the GPT partitions on eMMC

```
=> mmc dev 0 0
=> gpt write mmc 0 'name=boot,start=32M,size=64M,uuid=49A4D17F-93A3-45C1-A0DE-F50B2EBE2599;name=recovery,start=96M,size=64M,uuid=4177C722-9E92-4AAB-8644-43502BFD5506;name=system,start=160M,size=1536M,uuid=38F428E6-D326-425D-9140-6E0EA133647C;name=userdata,start=1696M,size=1024M,uuid=DC76DDA9-5AC1-491C-AF42-A82591580C0D;name=vendor,start=2720M,size=1024M,uuid=C5A0AEEC-13EA-11E5-A1B1-001E67CA0C3C;'
```

### Write Android Sparse Images
Download images over TFTP and write it to eMMC. TFTP get will take a long time.

```
=> tftpboot c0000000 boot_fat_sparse.img ; mmc swrite c0000000 10000
=> tftpboot 85000000 system.img00.gz && unzip 85000000 c0000000 && mmc erase  50000 100000 && mmc swrite c0000000 50000
=> tftpboot 85000000 system.img01.gz && unzip 85000000 c0000000 && mmc erase 150000 100000 && mmc swrite c0000000 150000
=> tftpboot 85000000 system.img02.gz && unzip 85000000 c0000000 && mmc erase 250000 100000 && mmc swrite c0000000 250000
=> tftpboot c0000000 userdata.img && mmc erase 350000 200000 && mmc swrite c0000000 350000
=> tftpboot c0000000 vendor.img ; mmc swrite c0000000 550000
```

### Boot Android from U-Boot
Boot the image from eMMC as below.

```
=> setexpr bootm_low 0x82000000 '&' fe000000 
=> setenv bootargs androidboot.hardware=akebi96 androidboot.selinux=permissive earlycon loglevel=15
=> fatload mmc 0 0x82080000 Image
=> fatload mmc 0 0x84a00000 ramdisk
=> fatload mmc 0 0x84100000 uniphier-ld20-akebi96.dtb
=> booti 0x82080000 0x84a00000 0x84100000
```

