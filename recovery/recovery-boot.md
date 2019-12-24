Akebi96 Recovery Boot Mode
==========================

# Akebi96 Recovery System

Akebi96 has an eMMC on the board, which has 2 Boot Areas, Boot-Area1 and Boot-Area2. Boot-Area1 is open for users. You can install and update your own bootloaders like Trusted Firmware, OP-TEE, U-Boot, and maybe EDK-2 etc. Boot-Area2 is dedicated for recovery firmware, which is a customized boot firmware including Trusted Firmware and U-Boot.
The board has an Jumper pin 'J1000', which controls which Boot-Area is used for boot. If it is open (default), the board boots from Boot-Area1. So your own boot loader boots. If J1000 is closed, the board boots from Boot-Area2 into recovery mode.

## Recovery Mode

The Recovery Mode is a custom U-Boot. This tries to boot up the device with following order.

1. Wait 3 second for autoboot
2. Enable USB and tries to search 'recovery.scr' U-Boot script from USB flash drive on USB ports. If the script exists, execute it for recovery.
3. Run Fastboot server on the USB gadget port. Wait for the port even if the recovery in 2. succeeds or fails.

Of course, you can interrupt at 1. and run U-Boot commands manually for your custom recovery method, like TFTP boot.
And you can reset the board if the recovery in 2. is done successfully.

## Recovery from USB Flash Drive

If you breaks your boot loaders on Boot Area1, recoverying from USB Flash Drive is the standard way. (Of course, you can use TFTP etc. by manual, but not automatically as this method is.)
This method uses a USB flash drive format with VFAT filesystem. It is OK whether the USB flash drive has a partition table or not, but if it has a partition table, recovery script and images must be in the 1st partition.

We will provide the recovery images at [TBD](). You can download this pre-build recovery images and script, extract all files and put them into your USB flash drive, and plug the drive on USB (Type-A) port.
After that, close 'J1000' jumper pin and boot the board. You can see recovery process on HDMI display.

### Using Akebi96-tools for Recovery from USB Flash Drive

You can find the original recovery script in [akebi96-tools](https://github.com/96boards-akebi96/akebi96-tools). See [Akebi96 USB Recovery Script](https://github.com/96boards-akebi96/akebi96-tools/blob/master/usbflash/README.md) for instructions.

## Recovery from Fastboot

[TBD]

