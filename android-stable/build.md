#  AOSP build for Akebi96 (stable)

## Prerequisities

### Install Packages

On the Ubuntu 16.04, the following packages are needed for basic BSP.

```
apt-get install --fix-missing -y git bc cmake ncurses-dev autoconf bison ccache cscope curl flex gdisk libfdt-dev libglib2.0-dev libpixman-1-dev netcat python-crypto python-serial uuid-dev xz-utils zlib1g-dev gawk wget git-core diffstat unzip texinfo gcc-multilib build-essential chrpath socat libsdl1.2-dev xterm cpio libssl-dev rsync
```

And also the followings for AOSP.

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


## Build BSP for Akebi96

### Preparation

Set environmental variables and make working directories.

```
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export JOBS=`getconf _NPROCESSORS_ONLN`
mkdir -p ~/aosp/bsp ~/aosp/android
```

### Download BSP

Download BSP as below.

```
cd ~/aosp/bsp
git clone -b 2018.02.6 --single-branch https://github.com/buildroot/buildroot
git clone -b master --single-branch https://github.com/96boards-akebi96/buildroot-configs.git
```

### Download Mali Related Archive

Download an archive from the following site.

* Open Source Mali Midgard GPU Kernel Drivers
	* https://developer.arm.com/tools-and-software/graphics-and-gaming/mali-drivers/midgard-kernel
	* TX041-SW-99002-r28p0-01rel0.tgz

And copy it under ~/aosp/bsp/kmod-mali.

```
cd ~/aosp/bsp
mkdir kmod-mali
cp TX041-SW-99002-r28p0-01rel0.tgz kmod-mali/
```

### Build BSP

And build BSP as below.

```
cd ~/aosp/bsp/buildroot
make BR2_EXTERNAL=../buildroot-configs akebi96_defconfig
make clean
make
```

## Build AOSP 9 for Akebi96

### Download AOSP 9 for Akebi96

```
cd ~/aosp/android
repo init -u https://android.googlesource.com/platform/manifest -b android-9.0.0_r34
[FIXME] git clone -b proprietary --single-branch https://github.com/96boards-akebi96/akebi96-manifests.git .repo/local_manifests
repo sync -j $JOBS
```


### Extract Mali Related Archive

Download an archive from the following site.

* Open Source Mali GPUs Android Gralloc Module
	* https://developer.arm.com/tools-and-software/graphics-and-gaming/mali-drivers/android-gralloc-module
	* BX304L01B-SW-99005-r16p0-01rel0.tgz

And copy it, and apply patches as below.

```
cd ~/aosp/android/vendor/arm/gralloc
cp BX304L01B-SW-99005-r16p0-01rel0.tgz .
./apply_patch.sh
```

### Build AOSP for Akebi96

This may take a few hours.

```
cd ~/aosp/android
source build/envsetup.sh
lunch akebi96-userdebug
make -j $JOBS
./make_romimage.sh
```

### Copy image to USB memory

Set up USB memory formatted with FAT32 on PC, and find device file from dmesg.
(ex. /dev/sdc1)

```
sudo mount /dev/sdc1 /mnt
sudo mkdir -p /mnt/usb
sudo cp ~/aosp/bsp/buildroot/out/images/* /mnt/usb/
sudo umount /mnt
```

### Write image to eMMC

Insert USB memory to USB port of Akebi96, and execute the following command on U-boot.

```
=> run update_from_usb
```
