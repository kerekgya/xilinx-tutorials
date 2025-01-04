---
title: Creating a Vitis platform to boot the ZCU102 from SATA
layout: default
last_modified_at: 2024-01-05
---
# Creating a Vitis platform to boot the ZCU102 from SATA
To prepare the SATA drive, create a single partition on the disk, and write the petalinux rootfs to it.
```sh
cat rootfs.ext4 > /dev/sda1
```
If you do not have the rootfs, you can get it by creating a petalinux project from the default bsp and building it, the rootfs file will be in the ```images/linux``` folder.
```sh
source /opt/petalinux/2023.2/settings.sh
petalinux-create -t default_bsp -s <path-to-bsp>
cd default_bsp
petalinux-build
cat images/linux/rootfs.ext4 > /dev/sda1
```
To create a custom platform, you need the Vitis platforms repo from github.
```sh
git clone https://github.com/Xilinx/Vitis_Embedded_Platform_Source.git
git checkout 2023.2 #make sure you are on the right version
```
Choose the folder for the ZCU102:
```sh
cd Vitis_Embedded_Platform_Source/Xilinx_Official_Platforms/xilinx_zcu102_base/
```
Modify the bootargs in the user dts file sw/prebuilt_linux/user_dts/system-user.dtsi to use /dev/sda1 as the rootfs:
```
chosen {
bootargs = "earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/sda1 rw rootwait cma=1536M";
stdout-path = "serial0:115200n8";
};
```
Build the platform using the prebuilt images:

```sh
make all PREBUILT_LINUX_PATH=/opt/Xilinx/platforms/2023.2/xilinx-zynqmp-common-v2023.2/
```
After the build, the platform will be available in the platform_repo directory.

To boot the FPGA, create a Vitis acceleration project using the new platform and build hardware for the project (to save space and compile time, you can check the skip sd packaging option, though hardware emulation won't work). Copy the output of the Vitis compilation (BOOT.BIN, boot.scr, Image, binary_container1.xclbin, host_exe) to an SD card (you still need an SD card for the FSBL) and boot the ZCU102 using the SD card, the rootfs will be loaded from the SATA disk.

