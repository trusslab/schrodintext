# SchrodinText

Copyright (c) 2016-2019 University of California, Irvine. All rights reserved.

People: Ardalan Amiri Sani, Nicholas Wei.

See [here](https://www.ics.uci.edu/~ardalan/papers/Amiri_Sani_MobiSys17.pdf) for technical details.

This document is shared under the GNU Free Documentation License WITHOUT ANY WARRANTY. See https://www.gnu.org/licenses/ for details.
_______________________________

## Prerequisites
- Linux (assumes >= Ubuntu 16.04 LTS) machine with ~300GB of free space.
- Should be x86 64-bit with 8GB+ of RAM
- ARM Development Juno Board w/ serial console access
- Micro SD Card (>=8GB) for the Juno Board

## Organization
SchrodinText is comprised of three main components:
- Android OS
- Xen Hypervisor
- OP-TEE OS/TrustZone

## Build Source
The section below will guide you through downloading and building all components from source. If you would like to skip this step, there are prebuilt binaries provided in the "Install" section. It is recommended to use the same folder/file names as used below if building from source.

### Build Android
Install build dependencies

```
$ sudo apt-get install openjdk-8-jdk 
$ sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev libgl1-mesa-dev libxml2-utils xsltproc unzip python
```
Install ```repo``` tool

```
# Make sure you have a bin/ directory in your home directory and that it is in your PATH.
# If not run the following two commented commands first:
# mkdir ~/bin
# PATH=~/bin:$PATH

$ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$ chmod a+x ~/bin/repo
```

Get Source Code

```
# For AOSP 7.1.1_r13
$ mkdir juno_aosp_nougat && cd juno_aosp_nougat
$ wget http://releases.linaro.org/members/arm/android/juno/17.01/linaro_android_build_cmds.sh
$ chmod a+x linaro_android_build_cmds.sh

# Remove the last three lines in the script because those start building immediately after downloading the source code.
$ sed '$d' linaro_android_build_cmds.sh | sed '$' | sed '$d' >&1 | tee linaro_android_build_cmds.sh

# To download source - this will take a while
$ ./linaro_android_build_cmds.sh -t
```

The kernel provided in the previous source tree does not seem to work well on Juno so we will need to download an AOSP Oreo tree for a working kernel source. Alternatively, to avoid downloading another source tree, download the prebuilt kernel provided in the "Install" section.

```
# For AOSP 8.1.0_r23
$ mkdir juno_aosp_oreo && cd juno_aosp_oreo
$ wget http://releases.linaro.org/members/arm/android/juno/18.04/linaro_android_build_cmds.sh
$ chmod a+x linaro_android_build_cmds.sh

# Remove the last three lines in the script because those start building immediately after downloading the source code.
$ sed '$d' linaro_android_build_cmds.sh | sed '$' | sed '$d' >&1 | tee linaro_android_build_cmds.sh

# To download source - this will take a while
$ ./linaro_android_build_cmds.sh -t
```

Apply SchrodinText changes

Navigate to AOSP 7 directory at ```juno_aosp_nougat/android```. Then apply the following changes:

```
$ cd frameworks/base
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_android_base.git
$ git fetch schrod_origin
$ git checkout -b schrodintext schrod_origin/schrodintext
$ cd ..

$ cd minikin
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_android_minikin.git
$ git fetch schrod_origin
$ git checkout -b schrodintext schrod_origin/schrodintext
$ cd ../..

$ cd external/skia
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_android_skia.git
$ git fetch schrod_origin
$ git checkout -b schrodintext schrod_origin/schrodintext
$ cd ../..

$ cd packages/apps
$ wget https://github.com/trusslab/schrodintext/raw/master/files/SchrodinTextApp.zip && unzip SchrodinTextApp.zip
$ cd ../../build/target/product
$ sed -i 's/^PRODUCT_PACKAGES := \/PRODUCT_PACKAGES := SchrodinTextApp \/" generic_no_telephony.mk
$ cd ../../../../..

# Use new fstab.juno to have Android boot off micro SD card
$ cd device/linaro/juno
$ mv fstab.juno fstab.juno.old
$ wget https://raw.githubusercontent.com/trusslab/schrodintext/master/files/fstab.juno

# Build - this may take a while depending on your system speed.
# You may need to execute the following commented command first if you are on a newer Ubuntu distro:
# export LANG=C 
$ source build/envsetup.sh
$ lunch juno-userdebug
$ make -j4
```

Navigate to AOSP 8 directory at ```juno_aosp_oreo/android```. Then apply the following changes:

```
$ cd kernel/linaro/armlt
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_kernel.git
$ git checkout -b schrodintext schrod_origin/schrodintext

# Build - this may take a while depending on your system speed.
# You may need to execute the following commented command first if you are on a newer Ubuntu distro:
# export LANG=C 
$ source build/envsetup.sh
$ lunch juno-userdebug
$ make -j4
```

### Build Xen
Install aarch64 cross-compiler

```
$ sudo apt-get install gcc-aarch64-linux-gnu
```

Download source code with SchrodinText changes applied

```
$ git clone https://github.com/trusslab/schrodintext_xen.git schrodintext_xen
```

Build Xen

```
$ cd schrodintext_xen
$ make dist-xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

### Build OP-TEE

Download Juno Arm Platform deliverables

```
$ mkdir juno_platform && cd juno_platform
$ wget https://community.arm.com/cfs-file/__key/communityserver-wikis-components-files/00-00-00-00-10/6825.armplat_5F00_1810.py
$ python3 armplat_1810.py
```

Apply SchrodinText changes to OP-TEE

```
$ cd optee/optee_os
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_optee.git
$ git fetch schrod_origin
$ git checkout -b schrodintext schrod_origin/schrodintext
$ cd ../..
```

Build and package OP-TEE

```
$ ./build-scripts/build-all.sh clean
$ ./build-scripts/build-all.sh build
$ ./build-scripts/build-target-bins.sh package
```

## Installation
Prebuilt binaries are available [here](https://github.com/trusslab/schrodintext/releases).

### Install OP-TEE, Xen, and Kernel/DTB
Turn on Juno board and connect USB cable and serial console cable to your PC to copy files. It is highly recommended to save a backup of your current Juno files. It is assumed that you already have a micro SD card for the right slot loaded with Juno firmware USING UEFI to boot. If not, a set of firmware binaries is also provided in the prebuilt binaries.

```
# Access serial console for Juno. Use the following settings:
#    115200 baud
#    8-bit word length
#    No parity
#    1 stop bit
#    No flow control
# Enable usb to mount files
Cmd> usb_on
```

Navigate to ```juno_platform``` or get ```fip.bin``` provided in prebuilt binaries

```
$ cd output/juno/juno
```

Copy OP-TEE build output to Juno board.

```
# Mounted directory will usually be in /media/$user/JUNO
$ mv fip-uefi.bin fip.bin
$ sudo cp fip.bin /media/$user/JUNO/SOFTWARE
```

Navigate to ```schrodintext_xen``` or get ```xen.efi``` provided in prebuilt binaries

```
$ cd xen
$ mv xen xen.efi
$ wget https://raw.githubusercontent.com/trusslab/schrodintext/master/files/xen.cfg
$ sudo cp xen.efi /media/$user/JUNO/SOFTWARE
$ sudo cp xen.cfg /media/$user/JUNO/SOFTWARE
```

Navigate to ```juno_aosp_nougat``` or get ```ramdisk.img``` provided in prebuilt binaries

```
$ cd out/target/product/juno
$ sudo cp ramdisk.img /media/$user/JUNO/SOFTWARE
```

Navigate to ```juno_aosp_oreo``` or get ```Image``` and ```juno.dtb``` provided in prebuilt binaries

```
$ cd out/target/product/juno/obj/kernel/arch/arm64/boot
$ sudo cp Image /media/$user/JUNO/SOFTWARE
$ sudo cp juno.dtb /media/$user/JUNO/SOFTWARE
```

Add Xen files to ```images.txt``` in Juno configuration so it gets flashed to Juno's memory or use the patched Juno firmware provided in the prebuilt binaries (if using patched Juno firmware copy all files to ```/media/$user/juno/``` and overwrite when asked, then move to next step)

```
# Because this file can vary, it is safer to manually add the Xen files.
# The folder containing image.txt is different depending on your Juno board revision number.
$ sudo vi /media/$user/JUNO/SITE1/HB10262B/images.txt

#   Perform the following changes:
#   - Increase 'TOTALIMAGES' counter at top by 2
#   - Add the two entries below at the bottom of images.txt. Change the numbers to fit as appropriate in your images.txt

      NOR10UPDATE: AUTO                 ;Image Update:NONE/AUTO/FORCE
      NOR10ADDRESS: 0x00200000          ;Image Flash Address
      NOR10FILE: \SOFTWARE\xen.efi      ;Image File Name
      NOR10NAME: xen.efi                ;Specify target filename to preserve file extension
      NOR10LOAD: 00000000               ;Image Load Address
      NOR10ENTRY: 00000000              ;Image Entry Point

      NOR11UPDATE: AUTO                 ;Image Update:NONE/AUTO/FORCE
      NOR11ADDRESS: 0x03E00000          ;Image Flash Address
      NOR11FILE: \SOFTWARE\xen.cfg      ;Image File Name
      NOR11NAME: xen.cfg                ;Specify target filename to preserve file extension
      NOR11LOAD: 00000000               ;Image Load Address
      NOR11ENTRY: 00000000              ;Image Entry Point
```

Sync and disconnect USB

```
$ sync
Cmd> usb_off
```

### Install Android

Navigate to ```juno_aosp_nougat``` or use ```android.img``` provided in prebuilt binaries and skip to step 3

```
$ cd out/target/product/juno
$ mkdir fs_images
$ cp {cache.img,system.img,userdata.img} fs_images
$ cd fs_images
$ wget https://raw.githubusercontent.com/trusslab/schrodintext/master/files/android-img.sh
$ chmod a+x android-img.sh
```

Package .imgs into ```android.img```

```
$ ./android-img.sh
```

Mount micro SD card and flash ```android.img``` to micro SD card

```
# Use 'lsblk' after mounting SD card to see which /dev/sdX it is. Replace 'X' with appropriate identifier.
$ lsblk
$ sudo dd if=android.img of=/dev/sdX bs=4M
$ sync
```

Unmount micro SD card and load it into left micro SD slot in Juno board.

## Running SchrodinText
There is a test app loaded called SchrodinTextApp that is used to demonstrate SchrodinText.

Power on Juno Board and navigate to UEFI Shell through serial console.

Launch Xen which will load Android automatically as dom0 after Xen initializes.

```
Shell> fs2:
FS2:> xen.efi
```

Wait until Android is initialized. Plug HDMI cable from a monitor into HDMI1 (2nd) port for Juno. Verify that GUI is visible.

Launch SchrodinTextApp

```
# In the serial console Android Linux shell in Juno
$ su
> chmod 666 /dev/schrobuf
> monkey --pct-syskeys 0 -p edu.uci.ardalan.schrodintextapp 1
# Verify text rendered on display does not show up when dumping framebuffer (taking screenshot).
> screencap -p /sdcard/Pictures/schrod_result.png
```

Power down Juno board and mount SD card with Android into PC. Navigate to ```data/media/0/Pictures``` in mounted SD card to view screenshot. Verify text rendered on display does not show up when dumping framebuffer (taking screenshot).

## References
- [https://source.android.com/setup/build/downloading](https://source.android.com/setup/build/downloading)
- [https://source.android.com/setup/build/initializing](https://source.android.com/setup/build/initializing)
- [http://releases.linaro.org/members/arm/android/juno/](http://releases.linaro.org/members/arm/android/juno/)
- [https://wiki.xenproject.org/wiki/Compiling_Xen_From_Source](https://wiki.xenproject.org/wiki/Compiling_Xen_From_Source)
- [https://community.arm.com/dev-platforms/w/docs/304/arm-reference-platforms-deliverables](https://community.arm.com/dev-platforms/w/docs/304/arm-reference-platforms-deliverables)
- [https://community.arm.com/dev-platforms/w/docs/391/run-the-arm-platforms-deliverables-on-juno](https://community.arm.com/dev-platforms/w/docs/391/run-the-arm-platforms-deliverables-on-juno)
