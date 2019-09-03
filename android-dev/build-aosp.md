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
git clone -b master --single-branch https://github.com/96boards-akebi96/mali-midgard.git
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

### Downlaod AOSP 9 for Akebi96

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
cd ~/aosp/android/out/target/product/akebi96/
simg2img system.img system-raw.img
split -d -b 512M system-raw.img system.img
rm system-raw.img
```

Since the system raw image size is 1.5GB, you must have 3 files, ```_system.img00```, ```_system.img01```, and ```_system.img02```. Then make those sparse-image and compressed.

```
for img in _system.img0* ; do img2simg $img ${img#_}; rm $img ; gzip ${img#_}; done
```

## Check Images

Now you should have following image files in ```~/aosp/android/out/target/product/akebi96/```

- boot_fat_sparse.img
- system.img*.gz
- userdata.img
- vendor.img

These are the files which is required to be installed on the board.
