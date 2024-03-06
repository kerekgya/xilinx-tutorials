# PYNQ v3.0.1 on ZedBoard
Since the ZedBoard is not one of the officially supported boards for PYNQ, there is no official image for it. In order to use PYNQ, an image has to be built manually.

PYNQ 3.0 uses Vivado/VItis/Petalinux version 2022.1. Unfortunately, the latest board support package for ZedBoard is for verison 2021.2. This is not a huge problem, as you can create a BSP yourself.

To do this, you need to extract the hardware platform from the latest available BSP, open the project on the /hardware/ directory from the archive in Vivado, update it to version 2022.1, and export it as an xsa.
```bash
wget https://www.xilinx.com/member/forms/download/xef.html?filename=avnet-digilent-zedboard-v2021.2-final.bsp
tar -xf avnet-digilent-zedboard-v2021.2-final.bsp
source /opt/Xilinx/Vivado/2022.1/settings64.sh
vivado
```
When Vivado asks, answer update automatically, then refresh the IPs and upgrade all of them.

After this we need to export the hardware (include bitstream) into an xsa file (File-Export-Export Hardware).

The exported hardware can be used to create a Petalinux project, which can be packaged into a BSP. (You might be able to export a hdf file from Vivado, and use that directly for the PYNQ project without the need for the bsp packaging, but I am yet to test that solution)

```bash
source /opt/petalinux/2022.1/settings.sh
petalinux-create -t project --template zynq --name zedboard
cd zedboard
petalinux-config --get-hw-description=path_to_exported_hardware.xsa #use the xsa just exported
petalinux-build
petalinux-package --bsp -o zedboard_2022.1.bsp
```
To build the PYNQ image, the PYNQ git repo is required.
```bash
git clone https://github.com/Xilinx/PYNQ.git
```
To avoid having to enter the password for sudo mid-build, modify the sudoers file to allow passwordless root:
```sh
sudo visudo
```
In the bottom of the file, type the following:

```your_username ALL=(ALL) NOPASSWD: ALL```

Install package:
```sh
sudo apt-get install liblzma-dev
```
To set up the system, run
```
PYNQ/sdbuild/scripts/setup_host.sh
```
Download the rootfs and source distribution files:

https://bit.ly/pynq_arm_v3_1

https://github.com/Xilinx/PYNQ/releases/download/v3.0.1/pynq-3.0.1.tar.gz

Create a boards directory and a .spec file for the ZedBoard,
```sh
mkdir boards
cd boards
mkdir Pynq-ZED
nano Pynq-ZED/Pynq-ZED.spec
```
with the following content:
```
ARCH_Pynq-ZED := arm
BSP_Pynq-ZED := zedboard_2022.1.bsp
BITSTREAM_Pynq-ZED := base/base.bit
FPGA_MANAGER_Pynq-ZED := 1

STAGE4_PACKAGES_ZED := pynq ethernet xrt uart pandas opencv jupyter ssl
```
Set up the envirnomnet and start compiling:
```sh
source /opt/Xilinx/petalinux/2022.1/settings.sh
source /opt/Xilinx/Vitis/2022.1/settings64.sh
export PATH=/opt/qemu/bin:/opt/crosstool-ng/bin:$PATH
export BOARD_REPO=path/to/boards
export PYNQ_SDIST_PATH=path/to/pynq-3.0.1.tar.gz
cd PYNQ/sdbuild
export BOARD_TO_BUILD=Pynq-ZED
export PREBUILT_PATH=path/to/jammy.arm.3.0.1.tar.gz
make PYNQ_SDIST=${PYNQ_SDIST_PATH} PREBUILT=${PREBUILT_PATH} BOARDDIR=${BOARD_REPO} BOARDS=${BOARD_TO_BUILD}
```
After this the build should run successfully, and you will have a PYNQ v3.0.1 image.
