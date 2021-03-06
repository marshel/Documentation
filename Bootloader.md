
Renesas EMMA EV2 bootloader
===========================

More discussion about this can be found on

https://groups.google.com/forum/?fromgroups#!forum/renesas-emev-osp

SoC boot architecture
---------------------

If you want to known more about the boot sequence for this platform, you can refer the following:

datasheets/R19UH0036EJ0600_1chip.pdf

* Appendix C: Boot loader in ROM

u-boot configurations
---------------------

This u-boot can be compiled with different configurations, for the Renesas EMMA platforms. E.g.

* emev_emmc_config
* emev_sd_config
* ...

EM/EV ROM boot code exist in 0xFFFF_0000, and it use internal SRAM 0xF000_0000. This ROM boot code will load "mini-boot" to SRAM and jump to 0xF000_0000. For SD boot mode, mini-boot is "sdboot.bin" in the SD card root directory. For eMMC boot, miniboot is the first 8K bytes in mmcblk0p1. Mini-boot will then load the remaining part:

For an SD-card boot:
Load rest of u-boot (uboot-sd.bin to DDR#0x4100_8000);
Load kernel image (uImage to DDR#0x4000_7fc0);
Load ram disk (cramfs to DDR#0x4600_0000), this is optional due to compile option.

For an eMMC boot:
Load rest of u-boot (mmcblk0p1#0x0000_2000-0x0004_0000 to DDR#0x4100_8000);
Load kernel image (mmcblk0p2 to DDR#0x4000_7fc0);
Then jump to u-boot.

Source code of mini-boot
------------------------

File names are hardcoded in board/emxx/emev/mini-boot/mmc/mmc.c :

	static const unsigned char match_name[3][8] = {
	#if defined(CONFIG_EMXX_EMMCBOOT)
		{0x75, 0x62, 0x6F, 0x6F, 0x74, 0x34, 0x20, 0x20},	/* uboot4 */
		{0x75, 0x49, 0x6D, 0x61, 0x67, 0x65, 0x34, 0x20},	/* uImage4 */
	#else
		{0x75, 0x62, 0x6F, 0x6F, 0x74, 0x2D, 0x73, 0x64},	/* uboot-sd */
		{0x75, 0x49, 0x6D, 0x61, 0x67, 0x65, 0x20, 0x20},	/* uImage */
	#endif
		{0x63, 0x72, 0x61, 0x6D, 0x66, 0x73, 0x20, 0x20},	/* cramfs */
	};

There will be different possible boot arguments set up u-boot, whose definitions can be found in include/configs/emev.h. E.g

	...
	#if defined(CONFIG_EMXX_EMMCBOOT) || defined(CONFIG_EMXX_ESDBOOT) 
	#define CONFIG_EXT3_ROOT	"/dev/mmcblk0p3"	/* emmc-boot or esd-boot */
	#elif defined(CONFIG_EMXX_SDTEST)
	#define CONFIG_EXT3_ROOT	"/dev/mmcblk1p3"        /* test-sd boot */
	#elif CONFIG_EMEV_EMMC_1Piece
	#define CONFIG_EXT3_ROOT	"/dev/mmcblk1p3"	/* sd-boot (emmc 1 device) */
	#else
	#define CONFIG_EXT3_ROOT	"/dev/mmcblk2p3"	/* sd-boot (emmc 2 device) */
	#endif
	...
	#if defined(CONFIG_EMXX_MMCBOOT) || defined(CONFIG_EMXX_SDTEST)
	#define CONFIG_BOOTCOMMAND	"run ext3cmd"
	#else
	#define CONFIG_BOOTCOMMAND	"run cramfscmd"
	#endif
	...

Build configurations
--------------------

* emev_emmc_config -> starts "ext3cmd" with root fs set to /dev/mmcblk0p3 (the internal eMMC NAND, partition 3)
* emev_sd_config -> starts "cramfscmd" with root fs set to /dev/mmcblk1p3 (the external SD-card Flash, partition 3)
* emev_sd_line_config -> 
* emev_sdtest_config -> starts "ext3cmd" with root fs set to /dev/mmcblk1p3 (the external SD-card Flash, partition 3)
* emev_emmc_2ddr_config -> same as emev_emmc_configbut for a 2DDR model board (applied to _sd_ and _sdtest_

emev_emmc_config is used for emmc boot mode. The 1st partation include image of uboot-emmc.bin. The 2nd partation include image of kernel. (uImage). 3rd image is default boot image.

emev_sd_config is used for booting from SD card, which must include a MBR and the 1st partation must set partation type 0x6(FAT16) in MBR 1st image should formatted to FAT16 format (use "-F 16" parameter with mkfs.vfat). sdboot.bin, uboot-sd.bin and uImage are placed on root directory of 1st partation and they will be loaded in memory by u-boot. The 2nd partation is used to store u-boot environments, if there is no 2nd partition, device can still boot, but can not save environment variables. By default, the 3rd partation is used as root directory.

emev_sd_line is used for boot from SD card. The difference with emev_sd_config is an extera "cramfs" file is loaded to phyical address
0x46000000-0x4800000000

emev_sdtest_config is used to boot from SD card the complete filesystem, which should be found in 3rd partition. See below for more details.

emev_emmc_2ddr_config is used for older boards, having 2 DDR chips (newer model have 4 chip). The _2ddr_ option can be used along with the other configuratons as well.

Build Examples
--------------

	export CROSS_COMPILE=arm-eabi-  (or whatever build toolchain used)
	cd u-boot

1) for eMMC boot

	make distclean
	make emev_emmc_config
	make
	...
	ls -lrt
	...
	-rw-r--r--   1 ffxx68 ffxx68  131172 2012-09-21 08:39 u-boot-emmc.bin

2) for simple SD boot 
 
	make distclean
	make emev_sd_config
	make
	...
	ls -lrt
	...
	-rw-r--r--   1 ffxx68 ffxx68  122712 2012-09-21 08:41 uboot-sd.bin
	-rwxr-xr-x   1 ffxx68 ffxx68    8477 2012-09-21 08:41 sdboot.bin

3) for SD "line system", including the 'cramfs' File System, e.g. for firmware updates

	make distclean
	make emev_sd_line_config
	Add EMEV SD BOOT Option
	Add LINE SYSTEM Option
	make
	...
	ls -lrt
	...
	-rw-r--r--   1 ffxx68 ffxx68  122728 2012-09-21 08:46 uboot-sd.bin
	-rwxr-xr-x   1 ffxx68 ffxx68    8768 2012-09-21 08:46 sdboot.bin

4) for a bootable SD-card

	make distclean
	make emev_sdtest_config
	Add EMEV test-SD BOOT Option
	make
	...
	ls -lrt
	...
	-rw-r--r--   1 ffxx68 ffxx68  122712 2012-09-21 08:36 uboot-sd.bin
	-rwxr-xr-x   1 ffxx68 ffxx68    8544 2012-09-21 08:36 sdboot.bin

Toolchains
----------

The bootloader have been tested as working when built with:

* Code Sourcery 2009.03
* gcc 4.4.x

Other versions can't be ensured to work, at present.

Firmware "flashing" on the final device
---------------------------------------

Installing a new firmware (i.e. "flashing") on Renesas devices implies having an FAT-32 or FAT-16 SD card, containing at least:

 - sdboot.bin and uboot-sd.bin built from "emev_sd_line_config"
 - a uImage with a kernel providing the basic services for file system access and update the screen buffer (to show installation progress)
 - a cramfs file system, to run the installation
 - an installation script
 - the u-boot-emmc.bin built from "emev_emmc_config" and renamed to uboot4.bin
 - the final kernel image, renamed to uImage4
 - the complete userland file system, renamed to android-fs4.tar.gz

Starting the device with the Volume+ & Power button will trigger the SD card boot, which will in turn start the install routines. The boot from SD-card may change on different tablet models, with different main board makers. The Vol+ & Power pplies to a "livall" model. For a "Smallart" model, a button found on the mainboard should be pushed in place of Vol+. On the

You can find how flashing works by mounting a host Linux sytem the "cramfs" found in a typical stock firmware package and analyzing the init steps. E.g, for a "Livall" tablet:

	sudo mkdir /tmp/cramfs
	sudo mount -o loop cramfs /tmp/cramfs
	su
	cat /tmp/cramfs/etc/init.drcS
	...
        /bin/mount -t vfat -o codepage=932,iocharset=euc-jp,sync /dev/mmcblk1p1 /tmp/sd
	...
    	FF_EXE=`ls /tmp/sd/ff4`
    	if [ "$FF_EXE" = "/tmp/sd/ff4" ]; then
        	/bin/cp /tmp/sd/ff4 /tmp/ff4
    	fi
    	INSTALL_SH=`ls /tmp/sd/install.sh`
    	if [ "$INSTALL_SH" = "/tmp/sd/install.sh" ]; then
		/tmp/sd/install.sh
    	fi
	...

The boot command includes "init=/linuxrc", which points to a standard busybox executable. This will by default invoke the "::sysinit:/etc/init.d/rcS" action. This is would invoke either "ff4 or "install.sh" script, whatever is present. 

ff4 is a menu-driven tool (we have it as a closed source executable only), which will let you choose what to do, including the full internal NAND flashing. This latter option is similar to what the "install.sh" script does, that is:

* recreate NAND partitions
* write bootloaders, and kernel images from SD-card in to the corresponding partition each
* expand the Android file system tar.gz from SD-card onto the corresponding partition

The install.sh script is found in the fwupd/files/ directory

Preparing an SD card for device firmware update
-----------------------------------------------

An example script to prepare an SD card for a firmware update is given:

	fwupd/fwupd.sh <destination>

which will generate all bootlaoder binaries and move files to destination path, which should be the SD card root dir.

A pre-compiled "uImage" is provided here, used with the cramfs to run installation. This uImage may also be rebuild from open source kernel code, found at:

 https://github.com/Renesas-EMEV2/Kernel

on the 'Livall_JB' branch, using the 'emev_Livall_upd_defconfig' build configuration.

NOTE - The android file system (android-fs4.tar.gz) and final kernel image (uImage4) are not included here and should be put into fwupd/files before running the script.

See also the android fs packaging procedure, described at:

https://github.com/Renesas-EMEV2/Documentation/blob/emev-4.1/Android.md

Preparing a "bootable" SD card
------------------------------

Booting from SD card is normally meant for firmware updates, but we could also make the system start the Android on the SD card itself, by properly partitioning and preparing it to be bootable, for faster test cycles during development stages.

The following approach could be used:

1. the SD card should be partitioned and prepared in a way that is similar to the internal NAND
2. the Android init.rc should be corrected to mount the fs to SD card partitions, in place of the NAND ones
3. the uboot-sd.bin should boot with "run ext3cmd" with root fs at "/dev/mmcblk1p3" (the SD card, partition 3)

The following command builds the bootloader as per point 3 above:

	make emev_sdtest_config

The complete script to prepare the SD card in such way is provided:

	testsd/testsd.sh /dev/sdd  (where "/dev/sdd" is the mount point of the SD on the host)

including the uboot make, SD partitioning and creation steps.

IMPORTANT - Before first boot, the init.rc found in the "androidfs4" SD card partition should be patched replacing:

	mount ext3 /dev/block/mmcblk0p6 /data nosuid nodev

with:

	mount ext3 /dev/block/mmcblk1p5 /data nosuid nodev

NOTE - The android file system (android-fs4.tar.gz) and kernel image (uImage) are not included and should be stored in testsd/ before executing the script.

To be noticed also that running Android from the SD-card makes it MUCH slower that from the internal NAND, especially first time it's booted-up (which may take several minutes to complete), but also for every single action made on device.


Using serial console and reloading kernel image
-----------------------------------------------

The device exposes the UART connector as pads on the main board (e.g. see datasheet/SERIAL.png for the "Livall" tablet; pad details may differ for different manufacturers). A common Linux program which implements the Kermit serial protocol is C-Kermit (find more info on-line, about this program). If you use a UART-to-USB adapater to connect the device to your host PC, the following is the Kermit configuration to use on your host:

	set line /dev/ttyUSB0
	set speed 115200
	set carrier-watch off
	set flow-control none

The following alias may also help starting kermit:

 	alias kerm='sudo chmod 666 /dev/ttyUSB0; kermrc'

Once in Kermit, connect to the device with the standard 'connect' command:

	C-Kermit> c [onnect]

Immediately after booting, you'll  will have 3 seconds to hit a key to enter the u-boot prompt. To continue with he usual start-up procedure, use the 'boot' command:

 	EM/EV # boot

If on the other hand you want to load a different kernel image instead, for example if the installed one is corrupted, you should execute the 'loadb' u-boot command:

	EM/EV # loadb

Now escape back to the kermit shell prompt (ctrl+\ followed by ctrl+c by default). Tell kermit to transmit the file you want by doing:

	C-Kermit> robust
	C-Kermit> send /host/path/to/uImage

Text will appear giving you information about the progress of the upload. A 5 Mbyte uImage takes approximately 10 mimutes to complete. Note the 'robust' command preceding the 'send' one, which may be necessary for a reliable communication. See also the U-Boot and Kermit command documentation, for more information.

After the upload has finished, switch back to the U-Boot command line and boot the new kernel image:

	C-Kermit> c

	EM/EV # boot

