# AOSP-Firmware build for Akebi96

Here, we will show how to build the boot firmware (including U-Boot bootloader) for Akebi96 board.

## Preparation

### Prerequisites 
See AOSP build for Akebi96, for required packages and compilers. Also, you must build AOSP for making OP-TEE OS binary (which is built as a part of AOSP.)

## Build firmware

### Download uniphier-bl, firmware, U-Boot and Trusted Firmware

Download required source code and binaries as below.

```
cd ~/aosp/
git clone -b master --single-branch https://github.com/uniphier/uniphier-bl.git
git clone -b master --single-branch https://github.com/96boards-akebi96/akebi96-prebuild.git
git clone -b v2018.09 --single-branch http://git.denx.de/u-boot.git
git clone -b master --single-branch https://github.com/96boards-akebi96/arm-trusted-firmware.git
git clone -b mbedtls-2.4.2 --single-branch https://github.com/ARMmbed/mbedtls
```

### Build uniphier-bl, U-Boot and Trusted Firmware

#### Setup Env

Setup cross-build environmental vars.

```
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

#### Build U-Boot

Build U-Boot from upstream source code.

```
cd ~/aosp/u-boot
make uniphier_v8_defconfig
./scripts/kconfig/merge_config.sh -m ./.config ~/aosp/akebi96-configs/u-boot/akebi96-aosp.config
make olddefconfig
cp ~/aosp/akebi96-prebuild/u-boot/uniphier-ld20-aosp.h include/configs/uniphier-ld20-aosp.h
make DEVICE_TREE=uniphier-ld20-akebi96 CONFIG_SYS_CONFIG_NAME=uniphier-ld20-aosp
cp ./u-boot.bin ~/aosp/images/
```

#### Copy OP-TEE OS from AOSP

After (AOSP build for Akebi96)[build-aosp.md#BuildAOSPforAkebi96], you can find tee-pager.bin under optee directory. So copy it in images directory.

```
cd ~/aosp/android/out/target/product/akebi96/
cp optee/arm-plat-uniphier/core/tee-pager.bin ~/aosp/images/
```

The reason why we reuse the OP-TEE OS from AOSP is the TEE-applications built in the AOSP image are based on this OP-TEE OS image. If we rebuild the OP-TEE OS outside AOSP, the applications may not work.

#### Build Trusted Firmware

Build Trusted Firmware with U-Boot and OP-TEE.

```
cd ~/aosp/arm-trusted-firmware/
make PLAT=uniphier realclean
make PLAT=uniphier BUILD_PLAT=./build SPD=opteed BL32=~/aosp/images/tee-pager.bin BL33=~/aosp/images/u-boot.bin bl2_gzip fip
cp build/fip.bin build/bl2.bin.gz ~/aosp/images/
```

#### Build uniphier-bl

Build uniphier-bl (First-stage boot loader for UniPhier arm64 SoC) with Trusted Firmware BL2.

```
cd ~/aosp/uniphier-bl/
make all
cat bl_ld20_global.bin ~/aosp/images/bl2.bin.gz > ~/aosp/images/uniphier_bl.bin
```

#### Copy Images to TFTP directory
Copy the firmware images to TFTP directory.

```
cd ~/aosp/images/
cp fip.bin uniphier_bl.bin ~/aosp/tftpboot/
```

Also, copy prebuild VOC firmware.
```
cp ~/aosp/akebi96-prebuild/boot_voc_ld20.bin ~/aosp/tftpboot/
```

## Install Firmware
This document will explain how to install firmware image via TFTP. Please follow the TFTP setup and HW preparation at (AOSP build for Akebi96)[build-aosp.md#InstallAOSPtoAkebi96].

### Writing firmware on eMMC
Get the image via TFTP and write it to eMMC boot0 partition.

```
=> mmc dev 0 1
=> tftpboot c0000000 uniphier_bl.bin && mmc write c0000000 0 100
=> tftpboot c0000000 fip.bin && mmc write c0000000 100 d00
=> tftpboot c0000000 boot_voc_ld20.bin && mmc write c0000000 e00 200
```

OK, now we write the new firmware on eMMC.

