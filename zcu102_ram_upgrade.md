---
title: Upgrade RAM on Xilinx zcu102
---
# Upgrade RAM on Xilinx zcu102
# Disclamier: this guide kind of works, there are some parts where I don't understand what is happening and I only tested this with the basic vadd example
The Xilinx zcu102 FPGA comes with a 4 GB DDR4 SODIMM on board. If you need more RAM, you can easily replace it, but reconfiguring the software to actually be able to use the capacity is a bit tricky.
The way the memory controller on the board works is that the there is a lower range(ddr_low) with 32 bit addresses, out of which 2 GB can be used for the OS, and a ddr_high range, which is 36 bits longs and can support up to 32 GB memory, as it is described [here](https://docs.xilinx.com/r/en-US/ug1085-zynq-ultrascale-trm/System-Addresses).
Using 36 bit memory is a bit slower than 32 bits.
I do not actually understand whether the lower 2 GB of the DIMM is always mapped to the 32 bit adress space, and the remaining to the 36 bit space, or you can map all of the DIMM to the 32 bit space. Based on my experimenting, the latter is more probable, though I have not found any documentation about this.

First, create a hardware platform with the correct settings for the new RAM module. The easiest is to extract the default bsp, open the hardware project in Vivado, open block design, customize the Zynq processing system block, and modify the values under DDR configuration according to yor needs. After you are done, export hardware as xsa.

To create a bootable petalinux image, you need a petalinux project.
```sh
source /opt/Xilinx/petalinux/2023.2/settings.sh
petalinux-create -t project --template zynpmp
petalinux-config --get-hw-description=exported_hardware.xsa
```
In the petalinux menuconfig, change the following:
* Set DTG Settings -> machine name to zcu102-rev1.0 (your board revision might be different).
* Change the rootfs type to ext4 in the Image Packaging Configuration -> Root fs type
* Change kernel bootargs in DTG Settings -> Kernel Bootargs to ```earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait cma=1526M```. You can set the cma to any value, 1526 MB is default. It seems there is a 4 GB upper limit, that is not documented anywhere...
* Change memory settings under Subsystem AUTO Hadrware Settings: set the Primary Memory to psu_ddr_1. After this, the size will automatically change to the largest possible, in my case 0x380000000, which is 14 GB.
After these changes, you can build the petalinux project:
```sh
petalinux-build
```
After the build succeeds, you need to create a new Vitis platform. To do this select Create new platform in Vitis, and create a linux platform on psu_cortexa53 using the xsa file. Once generated, open platform settings, under Linux On psu_cortexa53 set the pre-built image directory to the images/linux directory of the petalinux-project and the dtb file to the system.dtb in said directory. You also need to create a bif file with the following contents:
```
/* linux */
the_ROM_image:
{
  [fsbl_config] a53_x64
  [bootloader] <zynqmp_fsbl.elf>
  [pmufw_image] <pmufw.elf>
  [destination_device=pl] <bitstream>
  [destination_cpu=a53-0, exception_level=el-3, trustzone] <atf,bl31.elf>
  [load=0x800100000] <dtb,system.dtb>
  [destination_cpu=a53-0, exception_level=el-2] <uboot,u-boot.elf>
}
```
Note that the load adress of the device tree start with 8, which is crucial when using the ddr_high adress range, otherwise petalinux won't boot.
After this you can build the platform and use it for application acceleration projects.

After booting, I found that instead of the expected 14 GB of available RAM, all 16 GB was available, which I don't understand as it contradicts the config of petalinux...
Also when trying to set CMA to 8192 MB I got an error saying that 4 GB is the upper limit, and no CMA was allocated, but I was able to run the vector add example regardless...
????????????????
I have no idea what is happening??
[this](https://support.xilinx.com/s/article/000034737?language=en_US) might help. Or not.
