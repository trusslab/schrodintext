# SchrodinText

Copyright (c) 2016-2019 University of California, Irvine. All rights reserved.

People: Ardalan Amiri Sani, Nicholas Wei.

See [here](https://www.ics.uci.edu/~ardalan/papers/Amiri_Sani_MobiSys17.pdf) for technical details.

This document is shared under the GNU Free Documentation License WITHOUT ANY WARRANTY. See [https://www.gnu.org/licenses/](https://www.gnu.org/licenses/) for details.

## Prerequisites
- Linux (assumes >= Ubuntu 16.04 LTS) machine with ~300GB of free space.
- Should be x86 64-bit with 8GB+ of RAM
- LeMaker HiKey 620 Development Board w/ Serial Console Access
- See [here](https://github.com/trusslab/schrodintext) for instructions on running using the ARM Juno Development Board

## Organization
SchrodinText is comprised of three main components:
- Android OS
- Xen Hypervisor
- OP-TEE OS/TrustZone

## Build Source
The section below will guide you through downloading and building all components from source. It is recommended to use the same folder/file names as used below if building from source.

## Warnings
SchrodinText for HiKey was developed on AOSP 7.1.2 which was maintained at the time but is now no longer supported and considered obsolete. Please proceed at your own risk and note that instructions may break at anytime without notice. Furthermore, we were unable to program the IOMMUs which results in the text being rendered unsecurely onto the framebuffer. We still provide the code/instructions to those who may be interested in it.

### Build Android w/ OP-TEE
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
# For AOSP 7.1.2
$ mkdir hikey_aosp_nougat && cd hikey_aosp_nougat
# To download source - this will take a while
$ repo init -u https://android-git.linaro.org/git/platform/manifest.git -b android-7.1.2_r33 -g "default,-non-default,-device,hikey,fugu"
$ repo sync
# Apply HiKey related patches for AOSP after sync complete (apply patches in order and upon completion of previous command only)
$ ./android-patchsets/hikey-n-workarounds
$ ./android-patchsets/get-hikey-blobs
$ ./android-patchsets/NOUGAT-RLCR-PATCHSET
$ ./android-patchsets/hikey-optee-n
$ ./android-patchsets/hikey-optee-4.9
$ ./android-patchsets/optee-master-workarounds
$ ./android-patchsets/swg-mods-n
```

Apply SchrodinText changes

Navigate to AOSP 7 directory at ```hikey_aosp_nougat/android```. Then apply the following changes:

```
$ cd frameworks/base
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_android_base_hikey.git
$ git fetch schrod_origin
$ git checkout -b schrodintext schrod_origin/schrod_hikey
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

$ cd kernel/linaro/hisilicon
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_android_kernel_hikey.git
$ git checkout -b schrodintext schrod_origin/schrod_hikey

# Use new grub.cfg to have Xen boot
$ cd device/linaro/hikey/bootloader/EFI/BOOT
$ mv grub.cfg grub.cfg.old
$ wget https://raw.githubusercontent.com/trusslab/schrodintext/master/hikey/files/grub.cfg
$ cd ../../../../../..

# Apply OP-TEE changes since it will be built with AOSP
$ cd optee/optee_os
$ git remote add schrod_origin https://github.com/trusslab/schrodintext_optee.git
$ git fetch schrod_origin
$ git checkout -b schrodintext schrod_origin/schrodintext
# Change shared memory address for HiKey (default is configured for Juno)
$ cd core/arch/arm/tee
$ sed -i 's/0xfee00000/0x3ee00000/' entry_fast.c
$ cd ../../../../../..

# Build - this may take a while depending on your system speed. Do not use sudo.
# You may need to execute the following commented command first if you are on a newer Ubuntu distro:
# export LANG=C 
$ source build/envsetup.sh
$ lunch hikey-userdebug
$ make TARGET_BUILD_KERNEL=true TARGET_BOOTIMAGE_USE_FAT=true \
TARGET_TEE_IS_OPTEE=true TARGET_BUILD_UEFI=true CFG_SECURE_DATA_PATH=y \
CFG_SECSTOR_TA_MGMT_PTA=y
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

Configure shared memory address for HiKey (default is configured for Juno)

```
$ cd schrodintext_xen/xen/common
$ sed -i 's/0xfee00000/0x3ee00000/' memory.c
```

Build Xen

```
$ cd schrodintext_xen
$ make dist-xen XEN_TARGET_ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

## Installation

### Android & OP-TEE

Navigate to ```hikey_aosp_nougat``` folder.

```
$ cp -a out/target/product/hikey/*.img device/linaro/hikey/installer/hikey/
$ cd device/linaro/hikey/installer/hikey
# Turn HiKey on in recovery mode (links 1-2 and 3-4 closed, 5-6 open)
$ sudo ./flash-all.sh /dev/ttyUSBn # n represents corresponding device number, use dmesg to find which number
# Wait until flashing finishes and do not navigate away from this directory or turn off HiKey
```

### Xen
```
# Mount boot image
$ sudo mount -o loop,rw,sync boot_fat.uefi.img boot_tmp
# This modified device tree fixes some Xen boot issues
$ wget https://raw.githubusercontent.com/trusslab/schrodintext/master/hikey/files/hi6220-hikey.dtb
$ sudo cp hi6220-hikey.dtb boot_tmp/
# Xen configuration file
$ wget https://raw.githubusercontent.com/trusslab/schrodintext/master/hikey/files/xen.cfg
$ sudo cp xen.cfg boot_tmp/
# Xen binary
$ sudo cp ~/schrodintext_xen/xen/xen boot_tmp/
$ sudo mv boot_tmp/xen boot_tmp/xen.efi
# Unmount & Flash
$ sudo umount boot_tmp
$ sudo fastboot flash boot boot_fat.uefi.img
```

## Running SchrodinText
There is a test app loaded called SchrodinTextApp that is used to demonstrate SchrodinText.

Power on HiKey Board and select 'AOSP-Xen' boot entry from GRUB using serial console. This launches Xen which will load Android automatically as dom0 after Xen initializes.

Wait until Android is initialized. Plug HDMI cable from a monitor into HDMI port for HiKey. Verify GUI is visible.

Launch SchrodinTextApp

```
# In the serial console Android Linux shell in HiKey
$ su
> chmod 666 /dev/schrobuf
> monkey --pct-syskeys 0 -p edu.uci.ardalan.schrodintextapp 1
# Warning: SchrodinText does not successful program HiKey IOMMUs, therefore text is rendered unsecurely to framebuffer.
> screencap -p /data/local/tmp/screen.png
```

## References
- [https://source.android.com/setup/build/downloading](https://source.android.com/setup/build/downloading)
- [https://source.android.com/setup/build/initializing](https://source.android.com/setup/build/initializing)
- [https://wiki.xenproject.org/wiki/Compiling_Xen_From_Source](https://wiki.xenproject.org/wiki/Compiling_Xen_From_Source)
- [https://github.com/linaro-swg/optee_android_manifest/tree/hikey-n-4.9-master](https://github.com/linaro-swg/optee_android_manifest/tree/hikey-n-4.9-master)

# Acknowledgments

The work was supported in part by NSF Award #1617513.
