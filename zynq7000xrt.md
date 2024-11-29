---
title: Zynq-7000 XRT for petalinux version 2023.1 and above
layout: page
permalink: /zynq7000xrt.md
theme: minima
---
# Zynq-7000 XRT for petalinux version 2023.1 and above

If you try to create a custom Vitis platform for any Zynq-7000 device, or need the xrt rootfs package for some other reason, you are going to find that Xilinx has removed support for xrt from every Zynq-7000 device. Although these devices use the now dated 32-bit armhf architecture, many[^1] are still using these FPGA-s, so I sought a solution to this problem.
[^1]: nobody

In this tutorial I am only going to focus on the xrt xompatibility problem from the platform generation process, as there are already tutorials on the internet covering the other steps such as [this](https://www.hackster.io/news/microzed-chronicles-microzed-zynq-7000-vitis-platform-creation-df25e1054fb6) or [this](https://www.hackster.io/anujvaishnav20/building-custom-sdsoc-platform-with-petalinux-268bfd), here follows a very condensed recap of the aforementioned tutorials: 

```
. /opt/Xilinx/petalinux/2021.2/settings.sh
petalinux-create -t project -s zedboard-2021.2.bsp
. /opt/Xilinx/Vivado/2024.1/settings.sh
vivado
#open the project from the hardware folder of the petalinux project, upgrade everything generate bitstream export hardware
#new terminal
. /opt/Xilinx/petalinux/2024.1/settings.sh
petalinux-create -t project --template zynq --name zedboard
petalinux-config --get-hw-description=../zedboard.2024.1.xsa
#in the menuconfig change filesystem to ext4, bootargs to earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk0p2 rw rootwait cma=256M
```
edit project-spec/meta-user/conf/user-rootfsconfig, add
```
CONFIG_xrt
CONFIG_xrt-dev
CONFIG_zocl
CONFIG_opencl-clhpp-dev
CONFIG_opencl-headers-dev
CONFIG_packagegroup-petalinux-opencv
```
edit project-spec/meta-user/system-user.dtsi, add (i don't know why or if this is necesary)
```
&amba {
 zyxclmm_drm {
 		compatible = “xlnx,zocl”;
 		status = “okay”;
 	};
 };
```
```
petalinux-config -c rootfs # select everything in user packages
petalinux-build
```

For version 2024.2 this build will suceed, but on earlier versions, if you were to try to build a petalinux project with xrt, something similar to the following error message would appear
```
ERROR: Nothing RPROVIDES 'xrt' (but components/yocto/layers/meta-petalinux/recipes-core/images/petalinux-image-minimal.bb RDEPENDS on or otherwise requires it)

xrt was skipped: incompatible with machine zynq-generic-7z020 (not in COMPATIBLE_MACHINE)
```
To trick petalinux into believing that it can actually build these dependencies, you need to modify the bitbake files for xrt and zocl at ```components/yocto/layers/meta-xilinx/meta-xilinx-core/recipes-xrt/xrt/xrt_git.bb``` and ```components/yocto/layers/meta-xilinx/meta-xilinx-core/recipes-xrt/zocl/zocl_git.bb```, respectively. Add the folowing line to each file:
```
COMPATIBLE_MACHINE:zynq = ".*"
```
The files might not exist before running the petlainux-build command, if you can't find them, run petalinux-build until you get the error message above.

This almost fixes the problem, but the petalinux tool still gives a very long error message (a few thousand warnings and just one error to be precise). The error message is the following:
```
zedboard_2023.2/zedboard/build/tmp/work/cortexa9t2hf-neon-xilinx-linux-gnueabi/xrt/202320.2.16.0-r0/git/src/runtime_src/core/common/api/xrt_module.cpp:414:54: error: no matching function for call to 'min(size_t, ELFIO::Elf_Xword)'
  414 |         std::string argnm{symname, symname + std::min(strlen(symname), dynstr->get_size())};
      |                                              ~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
According to the ELFIO documentation, the dynsrt->get_size() returns uint64_t, which it seems cannot be implicitly converted to size_t, so the solution is easy, you just need to open the file ```build/tmp/work/cortexa9t2hf-neon-xilinx-linux-gnueabi/xrt/202320.2.16.0-r0/git/src/runtime_src/core/common/api/xrt_module.cpp``` (```build/tmp/work/cortexa9t2hf-neon-xilinx-linux-gnueabi/xrt/202410.2.17.0-r0/git/src/runtime_src/core/common/api/xrt_module.cpp``` on version 2024.1), and edit line 414 (lines 565 and 634 for 2024.1), cast the get_size() to size_t so that it looks like this:
```
std::string argnm{symname, symname + std::min(strlen(symname), (size_t)dynstr->get_size())};
```
(This might not be the safest way of solving this problem, I didn't really think think through if this could lead to any problems, if you are using this in mission critical applications (don't), make sure that this is not a problem, I do not take any responsibility.)

After making this change the petalinux project should build succesfully.

Note that in Vitis 2023.2 the hw_emu does not work for Zynq-7000 devices, the wonderful team at Xilinx started modifying the code but stopped half way through. (I am not kidding, you can check the Xilinx/Vitis/2023.2/bin/launch_emulator.py file around line 814) If I find a solution, I might write a tutorial.

In version 2024.1, petalinux-build --sdk also fails. I only encountered these errors in Ubuntu 20.04, but not in 22.04. I have no idea what is happening and why. Edit ```build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-libxcrypt/4.4.30-r0/git/build-aux/scripts/BuildCommon.pm``` and modify it according to [this commit](https://github.com/besser82/libxcrypt/pull/171/commits/ce562f4d33dc090fcd8f6ea1af3ba32cdc2b3c9c). Several perl modules are also missing, I installed these (Text::Tabs, Text::Template?, open, utf8, lib, bin, if, File::Spec::Functions) in a directory in my home folder (download package from cpan, tar -xf, perl Makefile.PL INSTALL_BASE=/home/username/perl, make install) and add BEGIN {push @INC, '/home/username/perl/lib/perl5'} above the required lines in the files 
```
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-libxcrypt/4.4.30-r0/git/build-aux/scripts/expand-selected-hashes
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-libxcrypt/4.4.30-r0/git/build-aux/scripts/gen-crypt-hashes-h
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-libxcrypt/4.4.30-r0/git/build-aux/scripts/gen-crypt-symbol-vers-h
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-libxcrypt/4.4.30-r0/git/build-aux/scripts/gen-crypt-h
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-libxcrypt/4.4.30-r0/git/build-aux/scripts/gen-libcrypt-map
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-automake/1.16.5-r0/automake-1.16.5/doc/help2man
build/tmp/work/x86_64-nativesdk-petalinux-linux/nativesdk-autoconf/2.71-r0/recipe-sysroot-native/usr/bin/help2man
```

