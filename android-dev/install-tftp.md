# Akebi96 Image Installation via TFTP

This document explains how to install build image on Akebi96 board via TFTP. Note that this method requires to setup network service etc. If you unsure, use [recovery mode](../recovery/recovery-boot.md) to install images.

## Prerequisites

You need to install tftpd (TFTP-daemon) and setup wired network. This TFTP boot method doesn't work on wireless network (WIFI).

### Setup Network

You need to setup wired network at first. The easiest way is using your wired LAN router. If you don't have the router, you can use LAN cable to connect the board and host PC. (Since this setting is highly depending on your distro, please check your distro support site how to setup wired connection)

While setting up the network, following information is needed.

- Network Address and Mask
- Free IP address on the network
- Host PC's IP address

### Setup TFTPD

Next, you need to install and setup tftpd. Please check your distro how to install tftp-daemon package. On ubuntu, you may need to run;

```
 $ sudo apt-get install -y tftpd-hpa
```

And prepare tftpboot directory under ~/aosp/

```
 $ mkdir ~/aosp/tftpboot
```

All binaries under this directory can be accessed via TFTP.
And run the tftp daemon as below.

```
 $ sudo /usr/sbin/in.tftpd --user tftp --address 0.0.0.0:69 --secure ~/aosp/tftpboot
```

### Setup Board

Setup the board to connect cables correctly.

- Make sure the Power Switch(SW2402) is off (pop-up)
- Connect AC adapter to the DC Jack (J2401)
- Connect Ethernet cable to the ether connector (CN3600) and other side to the ethernet hub (which connect to host PC too)
- Connect micro-USB cable to USB-UART UART#0(CN2501) and other side to host PC
- Make sure the Boot Mode Switch (SW2002) is "BE-BOOT"

### Board Bootup

At first, you must start minicom (or other serial console program) on the USB-UART port connected to CN2501, which is detected when you connect it to host PC. If it is ttyUSB0, run below command on host PC.

```
minicom -D /dev/ttyUSB0
```

Then, power up the board (push the power switch). Soon, you will see U-Boot boot up.

Please interrupt the boot when you see "Hit any key to stop autoboot:", then you will be in the U-Boot console like below.

```
Warning: ethernet@65000000 (eth0) using random MAC address - ee:ec:8b:f9:85:5c
eth0: ethernet@65000000
Hit any key to stop autoboot: 0
=>
```

### Setup Network Parameters of U-Boot

To use TFTP, you have to setup tftp server IP and netmasks on U-Boot.

If you use a class-C (/24) local network, you need to setup following parameters.

|Parameter  | Description                     |    default value|
|:----------|:-------------------------------| ----------------:|
|serverip   |	IP address of TFTP server       |	192.168.11.1 |
|gatewayip  |	IP address of default gateway   |	192.168.11.1 |
|ipaddr     |	IP address of the akebi96 board |	192.168.11.10 |
|netmask    |	Netmask of the local network    |	255.255.255.0 |

You can setup these parameters as below (just an example) on U-Boot console.

```
=> setenv serverip 192.168.1.2
=> setenv gatewayip 192.168.1.1
=> setenv ipaddr 192.168.1.3
=> setenv netmask 255.255.255.0
```

Now, you are ready to install images.


## Install Firmware

This section describes how to install firmware (bootloaders) part. If you want to install AOSP images, please goto [Install AOSP](#installaosp).

#### Copy Images to TFTP directory

At first, copy the firmware images under tftpboot directory on host PC.

```
cd ~/aosp/images/
cp fip.bin uniphier_bl.bin ~/aosp/tftpboot/
```

Also, copy prebuild VOC firmware.
```
cp ~/aosp/akebi96-prebuild/boot_voc_ld20.bin ~/aosp/tftpboot/
```

### Writing firmware on eMMC

Then, on U-Boot console, get the image via TFTP and write it to eMMC boot0 partition.

```
=> mmc dev 0 1
=> tftpboot c0000000 uniphier_bl.bin && mmc write c0000000 0 100
=> tftpboot c0000000 fip.bin && mmc write c0000000 100 d00
=> tftpboot c0000000 boot_voc_ld20.bin && mmc write c0000000 e00 200
```

OK, now we write the new firmware on eMMC.


## Install AOSP

This section describes how to install AOSP part. If you want to install bootloader, please goto [Install Firmware](#installfirmware).


### Copy image files for TFTP

Copy the image files into tftpboot directory on Host PC.

```
cd ~/aosp/android/out/target/product/akebi96
cp boot_fat_sparse.img system.img0?.gz userdata.img vendor.img ~/aosp/tftpboot/
```

Now you have AOSP images under ~/aosp/tftpboot/

### Create GPT on eMMC using U-Boot

Please note the partition sizes are hardcoded in Android (BoardConfig.mk) as below.
source in order to generate exact sized images.

|Label   |Start(MB)|Size(MB)|Start Sector|				   UUID|
|:-------|--------:|-------:|-----------:|-------------------------------------|
|boot    |       32|      64|   0x0010000| 49A4D17F-93A3-45C1-A0DE-F50B2EBE2599|
|recovery|       96|      64|   0x0030000| 4177C722-9E92-4AAB-8644-43502BFD5506|
|system  |      160|    1536|   0x0050000| 38F428E6-D326-425D-9140-6E0EA133647C|
|userdata|     1696|    1024|   0x0350000| DC76DDA9-5AC1-491C-AF42-A82591580C0D|
|vendor  |     2720|    1024|   0x0550000| C5A0AEEC-13EA-11E5-A1B1-001E67CA0C3C|

On the U-Boot console, run following command to write the GPT partitions on eMMC

```
=> mmc dev 0 0
=> gpt write mmc 0 'name=boot,start=32M,size=64M,uuid=49A4D17F-93A3-45C1-A0DE-F50B2EBE2599;name=recovery,start=96M,size=64M,uuid=4177C722-9E92-4AAB-8644-43502BFD5506;name=system,start=160M,size=1536M,uuid=38F428E6-D326-425D-9140-6E0EA133647C;name=userdata,start=1696M,size=1024M,uuid=DC76DDA9-5AC1-491C-AF42-A82591580C0D;name=vendor,start=2720M,size=1024M,uuid=C5A0AEEC-13EA-11E5-A1B1-001E67CA0C3C;'
```

### Write Android Sparse Images

Download images over TFTP and write it to eMMC. Note that ```tftpboot``` command may take a long time.

```
=> tftpboot c0000000 boot_fat_sparse.img ; mmc swrite c0000000 10000
=> tftpboot 85000000 system.img00.gz && unzip 85000000 c0000000 && mmc erase  50000 100000 && mmc swrite c0000000 50000
=> tftpboot 85000000 system.img01.gz && unzip 85000000 c0000000 && mmc erase 150000 100000 && mmc swrite c0000000 150000
=> tftpboot 85000000 system.img02.gz && unzip 85000000 c0000000 && mmc erase 250000 100000 && mmc swrite c0000000 250000
=> tftpboot c0000000 userdata.img && mmc erase 350000 200000 && mmc swrite c0000000 350000
=> tftpboot c0000000 vendor.img ; mmc swrite c0000000 550000
```

Now, you can reboot the board to boot.

