# QEMU CPUFreq Zynq

## Frequency scaling (CPUFreq) on Xilinx Zynq Platform Baseboard for Cortex-A9 emulated by QEMU

In this tutorial, we configure a Linux Kernel to allow the CPU frequency scaling on the Zynq board emulated by QEMU. We also create dedicated Device Tree Source (DTS) and Root file system (uramdisk.image.gz) to be able to control dynamically the frequency of our CPU from Linux environment. It must be noted that we need Xilinx tools to build correctly the Linux image and Rootfs. We provide equally an example of QEMU command how to emulate the xilinx-zynq-a9.

Some useful configuration files are also provided for Kernel and u-boot building. 

Note: Our aim is to emulate the xilinx-zynq-a9 on QEMU and using a small size Linux Kernel and Rootfs.


### Prebuilt_functional

If you don't want to build your own Linux Kernel, root file system and DTS, in `Prebuilt_functional` you can find the Kernel image, root file system and DTS -> DTB that can be used to QEMU emuation. 


### Compile tools

```bash
sudo apt-get install gcc g++ git qt4-dev-tools flex bison patch libncurses5-dev libssl-dev gparted device-tree-compiler
sudo dpkg -i force-architecture i386 glibc
```
### Cross-compile tools
For comiling Xilinx tools the version of gcc sould be >= 6.O, I recommande to use gcc ARM tools, that you can find [Here] (https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)

### Create a work environment 
In the following steps we download Xilinx tools and the Xilinx-Linux kernel. The ${CUSTOMDIR_WORKENV} variable is the path to your work environment (that you can initialize how you wish).

```bash
mkdir resource
cd resource
mkdir boot_loader
mkdir device_tree
mkdir file_system
mkdir linux_kernel
```
### Download gcc ARM tools: 
Go to [Here] (https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads) and download this version gcc-arm-8.2-2018.11-x86_64-arm-linux-gnueabi.tar.xz, copy this file to your work environment.

Extract the tools (gcc-arm-8.2-2018.11-x86_64-arm-linux-gnueabi.tar.xz)

```bash
tar -xf gcc-arm-8.2-2018.11-x86_64-arm-linux-gnueabi.tar.xz
```
In gcc-arm-8.2-2018.11-x86_64-arm-linux-gnueabi/bin you can find all of the compiler tools that we will use in the nex steps.

### Download boot-loader tool and build it: 

```bash
cd boot_loader
git clone https://github.com/Xilinx/u-boot-xlnx.git
cd u-boot-xlnx
cp ../../my_zynq_defconfig ./configs
make distclean
make my_zynq_defconfig
make
```

### Download Linux Kernel and cross-compile Linux for ARM architecture to obtain a uImage:

You have two choices you can use the standard Linux Kernel or the Linux Xilinx Kernel, it is up to you..

Dedicated configuration file is provided in this repository (my_device_defconfig) that includes all the necessary parameters for frequency scaling (cpufreq), and for supporting of a RAM filesystem and RAM disk (initramfs/initrd), this latter option is needed for the QEMU emulation.

## Download Linux Xilinx Kernel

```bash
mkdir linux_kernel
cd linux_kernel
git clone https://github.com/Xilinx/linux-xlnx.git
cd linux-xlnx
cp ../../my_device_defconfig arch/arm/configs
```
It can be noted that this Kernel is almost similar to the standard Linux. In Linux Xilinx Kernel we can found more device drivers to meet better the requirements of Xilinx devices. 

## Download Standard Linux Kernel
```bash
mkdir linux_kernel
cd linux_kernel 
git clone https://github.com/torvalds/linux.git
cd linux
cp ../../my_device_defconfig arch/arm/configs
```

Add path to u-boot tools:

```bash
export PATH=${CUSTOMDIR_WORKENV}/boot_loader/u-boot-xlnx/tools:$PATH
```

Cross-compile Linux to obtain a uImage:
```bash

cd linux_kernel/linux
make ARCH=arm CROSS_COMPILE=${CUSTOMDIR_WORKENV}/gcc-arm-8.2-2018.11-x86_64-arm-linux-gnueabi/bin/arm-linux-gnueabi- my_device_defconfig
make ARCH=arm CROSS_COMPILE=${CUSTOMDIR_WORKENV}/gcc-arm-8.2-2018.11-x86_64-arm-linux-gnueabi/bin/arm-linux-gnueabi- UIMAGE_LOADADDR=0x8000 uImage
cp /arch/arm/boot/uImage ..
```

After compiling you find the uImage at arch/arm/boot/uImage that we copy to the linux_kernel repertory (to make easier the access to your uImage)

### Create Root File System:

In this step we create a uramdisk.image.gz containing our rootfs and uImage, this uramdisk.image.gz would be used by QEMU(--initrd parameter). The basic file system was taken from a prebuilt ZedBoard project of Xilinx, you can download this project [Here] (https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842326/Zynq+2016.4+Release) 

```bash
mkdir file_system
cp -R ramdisk.image_FILES/ file_system
cp linux_kernel/linux-xlnx/arch/arm/boot/uImage file_system/ramdisk.image_FILES/boot
cd file_system 
mkdir tmp_mnt
dd if=/dev/zero of=my_ramdisk.image bs=1024 count=16384
mke2fs -F my_ramdisk.image -L "ramdisk" -b 1024 -m 0
tune2fs my_ramdisk.image -i 0
chmod a+rwx my_ramdisk.image
sudo mount -o loop my_ramdisk.image tmp_mnt/
sudo cp -R ramdisk.image_FILES/* tmp_mnt/
sudo umount tmp_mnt/
gzip my_ramdisk.image
${CUSTOMDIR_WORKENV}/boot_loader/u-boot-xlnx/tools/mkimage -A arm -T ramdisk -C gzip -d my_ramdisk.image.gz umy_ramdisk.image.gz
```
Now you have a ramdisk image (umy_ramdisk.image.gz) that is wraped with the U-Boot header.  

### Create Device Tree Source (DTS)

Personally I discourage you to build a DTS with the standard Xilinx SDK, the generated DTS can make issues durring the QEMU emulation. In this git repository you find a DTS (my_devicetree.dts) that you can modify easily and compile it to obtain a DTB. 

Notes about the my_devicetree.dts:
As you can see in the line 27 at bootargs there is a root=/dev/ram rw to tell that we are using a ramsdisk to boot.   

```diff
	chosen {
+		bootargs = "console=ttyPS0, 115200 root=/dev/ram rw";
		stdout-path = "serial0:115200n8";
	};
```

Compile DTS:

```bash
dtc -O dtb -I dts -o my_devicetree.dtb my_devicetree.dts
cp my_devicetree.dtb device_tree
```

## Download QEMU
I encourage you to download the newest version of QEMU and make a small patch on it before compiling. You can download the standard QEMU or even the QEMU from Xilinx

Download QEMU to your work environment:
```bash
mkdir QEMU
cd QEMU
git clone git://github.com/Xilinx/qemu.git
```
or

```bash
git clone https://github.com/qemu/qemu.git
```



### Set the Patch

```diff
diff --git a/hw/misc/zynq_slcr.c b/hw/misc/zynq_slcr.c
-    s->regs[R_ARM_PLL_CTRL]   = 0x0001A008;
+    s->regs[R_ARM_PLL_CTRL]   = 0x00014008;
```


### Compile Xilinx QEMU:
```bash
cd qemu
./configure --target-list="arm-softmmu,aarch64-softmmu,microblazeel-softmmu" --enable-fdt --disable-kvm --disable-xen
make -j4
```

### Compile standard QEMU:

```bash
cd qemu
git submodule init
git submodule update --recursive
./configure
make -j4
```

### Emulation of xilinx-zynq-a9 on QEMU

```bash
cd ${CUSTOMDIR_WORKENV}
QEMU/qemu/arm-softmmu/qemu-system-arm -M xilinx-zynq-a9 -serial /dev/null -serial mon:stdio -display none -kernel linux_kernel/uImage -dtb device_tree/my_devicetree.dtb --initrd file_system/umy_ramdisk.image.gz
```



### CPUfreq commands

Set frequency scailing governor as userspace
```bash
echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```
Show available frequencies
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
```

Show current frequency
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

Set userspace frequency
```bash
echo $freqValue > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed


### Recommended projects and documents

Installing Embedded Linux on ZedBoard by Associate Professor Clément Foucher

(https://homepages.laas.fr/cfoucher/drupal/sites/homepages.laas.fr.cfoucher/files/u93/Installing_Embedded_Linux_on_ZedBoard.pdf)


Git project 

Firmware required to boot & operate the apertus° AXIOM Beta Camera.
Detailed instructions on how to use the Firmware & operate the camera can be found in the wiki. In this project authors implemented 
the frequency scaling (cpufreq) for QEMU.  

(https://github.com/apertus-open-source-cinema/axiom-firmware)


