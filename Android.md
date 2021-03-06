Renesas EMEV Open Source Project (ROSP)
========================================

More discussion about this can be found on

https://groups.google.com/forum/?fromgroups#!forum/renesas-emev-osp

Download AOSP source code
-------------------------

Follow the generic steps described in:

 http://source.android.com/source/downloading.html

A version of the AOSP manifest customized for our Renesas EMEV2 project can be found at:

https://github.com/Renesas-EMEV2/Renesas-manifest  (branch emev-4.1)

so, the "repo init" step should be made using this manifest:

	repo init -u git://github.com/Renesas-EMEV2/Renesas-manifest.git -b emev-4.1

to update the entire set projects to the Jingerbread (4.1) version, plus our customization.

In .repo/manifests/deafult.xml, our ROSP custom projects are linked to the corresponding repositories on our github organization (i.e. "Renesas-EMEV2"), while the rest are pulled off the standard google source repositories, as usual in AOSP.

Each custom project has a "emev-4.1" branch, to track ROSP changes for JB (the "MyPad" branch was used to track changes for GB instead).

NOTE - Consider waiting a few hours to downloading the full AOSP source, even on a fast connection, as the complete package is several Gigas.

Preparing the build environment
-------------------------------

### Install the mandatory packages

This is the section hardest to describe in general, as it depends on the host system the build environment is based upon. Luckily, you need to fix this up only once (until at least you decide to migrate the project to a new host):

You need at least to install the AOSP prerequisite packages, as said in:

 http://source.android.com/source/initializing.html

but sometimes that's not enough. For example, a basic Ubuntu setup is normally missing the 'git' tool, which then needs to be installed with:

	sudo apt-get install git

### Setting up JAVA 

We need JDK version 6 to build JB. The Open-JDK is installed with:

	sudo apt-get install openjdk-6-jdk
 
On the other hand, installation of Oracle JDK as suggested on the ASOP Google pagesdoes not work!

If you have both Open-JDK and Oracle-JDK installed, you can choose which one to use executing:

	sudo update-alternatives --config java
	sudo update-alternatives --config javac
	sudo update-alternatives --config jar
	sudo update-alternatives --config javah
	sudo update-alternatives --config javadoc

and select the Open-JDK version for each of these packages. The build using Oracle-JDK failed to boot...

See also here:

 https://help.ubuntu.com/community/Java

### Other missing stuff

Also, for an Ubuntu 12.04 host, even installing all the suggested packages, I got a couple of errors during make, so that I had to install these ones as well:

	sudo apt-get install lib32ncurses5-dev
	sudo apt-get install lib32z1-dev

I found these searching on Google for the error codes + "Ubuntu 12", so let me suggest to try this approach, if you encouter any error. The discussion group https://groups.google.com/forum/?fromgroups#!forum/renesas-emev-osp is always open for questions, of course.

Generating the "Signing Keys"
-----------------------------

I'm not sure what the aim of these steps is, or if they're actually required, but I did them following a tutorial I found about Android porting... (http://marakana.com/static/courseware/android/Remixing_Android/index.html):

	SIGNER="/C=IT/ST=RM/L=Rome/O=ffxx68/OU=Android/CN=Android Platform Signer/emailAddress=ffumi68@gmail.com"
	cd $AOSP
	rm build/target/product/security/*.p*
	echo | development/tools/make_key build/target/product/security/platform "$SIGNER"
	echo | development/tools/make_key build/target/product/security/shared "$SIGNER"
	echo | development/tools/make_key build/target/product/security/media "$SIGNER"
	echo | development/tools/make_key build/target/product/security/testkey "$SIGNER"

This is the command to verify that all went fine:
 
	ls -1 build/target/product/security/*.p*
	openssl x509 -noout -subject -issuer -in  build/target/product/security/platform.x509.pem

I guess these keys are something required to pass the official Android Google certification (though I can't tell for sure).

How to push changes back to GitHub
----------------------------------

Altough it is definitely off topic to discuss here how to use git, or GitHub in general (or which you could find several better docs online), if you change anything, to push it back onto our GitHub repository you would do:

	git add <whatever file you changed>
	git commit -m "Leave your commit comments here"
	git push -u github emev-4.1

This would require you to have an account registered on GitHub, where you may have forked the project to.

Find help oh GitHub, for admin and access details. E.g.

 http://help.github.com/send-pull-requests/

A "Renesas-EMEV2" Organization:

 https://github.com/organizations/Renesas-EMEV2

has been setup on GitHub, in order to allow collaborative development. Members taking part to this organization can "push" or "pull" commits on the various repositories.

Anyone interested in participating can use our ROSP Discussion Group on Google, to ask for membership:

 https://groups.google.com/forum/#!forum/renesas-emev-osp
 
Use the same group if you think anything in the whole process may be improved. Suggestions are always welcome!

Building Android
----------------

Once you have your code, you would build Android with:

	cd $AOSP
	. build/envsetup.sh

	including device/asus/grouper/vendorsetup.sh
	including device/generic/armv7-a-neon/vendorsetup.sh
	...
	including device/renesas/emev/vendorsetup.sh
	...

	lunch renesas_emev-eng

	============================================
	PLATFORM_VERSION_CODENAME=REL
	PLATFORM_VERSION=4.1.1
	TARGET_PRODUCT=renesas_emev
	TARGET_BUILD_VARIANT=eng
	TARGET_BUILD_TYPE=release
	TARGET_BUILD_APPS=
	TARGET_ARCH=arm
	TARGET_ARCH_VARIANT=armv7-a-neon
	HOST_ARCH=x86
	HOST_OS=linux
	HOST_OS_EXTRA=Linux-2.6.38-15-generic-x86_64-with-Ubuntu-11.04-natty
	HOST_BUILD_TYPE=release
	BUILD_ID=JRO03L
	OUT_DIR=out
	============================================

	time make -j2 showcommands 2>&1 | tee make.log
	...
(took about 4 hours to make, on my 2x2.4GHz, 4 Gb RAM Ubuntu)

	...
	Installed file list: out/target/product/emev/installed-files.txt

Note - this error is given at the end (need a fix in partition size):

	error: do_inode_allocate_extents: Failed to allocate 2006 blocks

but it doesn't prevent a succesful build.

Building the kernel
-----------------------

Kernel source code is stored at

 https://github.com/Renesas-EMEV2/Kernel 

and the emev-4.1 branch should be used. Kernel image should be built before building SGX modules and final packaging steps.

See also the corrsponding documentation in

 https://github.com/Renesas-EMEV2/Documentation/Kernel.md

Building SGX interface modules
------------------------------

The kernel modules

 emxxlfb.ko
 pvrsrvkm.ko

should be recompiled manually, before completing the packaging.

See how to in device/renesas/emev/sgx_new/eurasia_km/compile.txt, or build.sh

I wish one day this step is integrated within the main Android build.

About OMX and SGX binaries
--------------------------

The firmware binaries and libraries managing:

 - 3D and graphics accelerator (SGX)
 - the video decoder (OMX)

are released only as pre-compiled libraries, source code being proprietary.

These are stored in the repository as binaries, under

	$AOSP/device/renesas/emev/sgx
	$AOSP/device/renesas/emev/omf  

and the build scripts take care of deploying them to the target locations.

About Broadcom BCM4329 WiFi driver
----------------------------------

The kernel driver for the BCM4329 Wifi device (hardwired on the "Livall" tablet board and interfaced through the SDIO bus), is provided as a pre-compiled binary only:

	device/emev/wifi/dhd.ko

Source code for this is proprietary to Livall and needs recompiling only in case of major changes to the kernel.

Other tablet manufacturers may provide different solutions, like for example a USB WiFi dongle, which may require a different kernel module.

I found this tutorial very useful, about the integration of a WiFi driver in Android: 

 http://blog.linuxconsulting.ro/2010/04/porting-wifi-drivers-to-android.html

Packaging files for a firmware update
-------------------------------------

NOTE - Remeber to re-build Android after SGX and OMF files are completed, as expleined above.

Then, in order to wrap up the final build and prepar it for delivery to the device, the script

	$AOSP/device/renesas/emev/pack.sh <destination> 

collects the complete Android file system and kernel image and put them into destination dir, which could be the Fat-32 formatted SD card home dir for example (see below).

The content of this directory then needs to be transferred into the root folder of an SD-card, (max 2Gb size allowed) formatted as FAT16 or FAT32, and the tablet started with teh Vol+ & Power to replace internal NAND firmware.

NOTE - The script assumes $AOSP and $KERNEL variables are set to the home dir of the AOSP and Kernel code on your host. E.g.:

	export AOSP=~/renesas/jb
	export KERNEL=~/renesas/kernel

In order to finally "flash" all this onto your device, we suggest to review the documentation at:

	https://github.com/Renesas-EMEV2/Documentation/

Check out "Preparing an SD card for device firmware update", or "Preparing a "bootable" SD card".

About Google Apps
-----------------

The (free) AOSP source is missing the most common Google Apps, like Play, GMail, Maps, etc. These are covered by license fees and so are installed on final devices only by manufacturers paying those fees, usually.

Nonetheless, packages can be found on the net. E.g.:

 http://wiki.rootzwiki.com/Google_Apps

If you have downloaded them (but it isn't mandatory for the script to succeed!), the pack.sh script assumes that you have put the packages into a directory named "GoogleApps", under $AOSP/device/renesas/emev.

