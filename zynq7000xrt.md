# Zynq-7000 XRT for petalinux 2023.1/2023.2

If you try to create a custom Vitis platform for any Zynq-7000 device, or need the xrt rootfs package for some other reason, you are going to find that Xilinx has removed support for xrt from every Zynq-7000 device. Although these devices use the now dated 32-bit armhf architecture, some are still using these FPGA-s, so I sought a solution to this problem.

In this tutorial I am only going to focus on the xrt xompatibility problem from the platform generation process, there are already tutorials on the internet covering the other steps such as [this](https://www.hackster.io/news/microzed-chronicles-microzed-zynq-7000-vitis-platform-creation-df25e1054fb6) or [this](https://www.hackster.io/anujvaishnav20/building-custom-sdsoc-platform-with-petalinux-268bfd).

If you were to try to build a petalinux project with xrt, something similar to the following error message would appear
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
According to the ELFIO documentation, the dynsrt->get_size() returns uint64_t, which it seems cannot be implicitly converted to size_t, so the solution is easy, you just need to open the file zedboard_2023.2/zedboard/build/tmp/work/cortexa9t2hf-neon-xilinx-linux-gnueabi/xrt/202320.2.16.0-r0/git/src/runtime_src/core/common/api/xrt_module.cpp, and edit line 414, cast the get_size() to size_t so that it looks like this:
```
std::string argnm{symname, symname + std::min(strlen(symname), (size_t)dynstr->get_size())};
```
(This might not be the safest way of solving this problem, I didn't really think think through if this could lead to any problems, if you are using this in mission critical applications (don't), make sure that this is not a problem, I do not take any responsibility.)

After making this change the petalinux project should build succesfully.

Note that in Vitis 2023.2 the hw_emu does not work for Zynq-7000 devices, the wonderful team at Xilinx started modifying the code but stopped half way through. (I am not kidding, you can check the Xilinx/Vitis/2023.2/bin/launch_emulator.py file around line 814) If I find a solution, I might write a tutorial.
