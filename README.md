# Cora Z7-07S Petalinux BSP Project

## Built for Petalinux 2017.4

This petalinux project targets the Vivado block diagram project found here: https://github.com/Digilent/Cora-Z7-07S-base-linux.

#### Warning: You should only use this repo when it is checked out on a release tag

## BSP Features

The project includes the following features by default:

* Ethernet with Unique MAC address and DHCP support (see known issues)
* USB Host support
* UIO drivers for onboard buttons and LEDs
* SSH server
* Build essentials package group for on-board compilation using gcc, etc. 
* U-boot environment variables can be overriden during SD boot by including uEnv.txt
  in the root directory of the SD card (see u-boot documentation).

### Digilent Petalinux Apps 

This project includes the Digilent-apps repository, a set of linux libraries, utilities and demos that are packaged as Petalinux 
apps so they can easily be included with Petalinux projects. These apps add various board specific funtionality, such as controlling
GPIO devices and RGB LEDs from the command line. For complete documentation on these apps, see the repository documentation: 
https://github.com/Digilent/digilent-apps.

## Known Issues

* MACHINE_NAME is currently still set to "template". Not sure the ramifications of changing this, but I don't think our boards
  our supported. For now just leave this as is until we have time to explore the effects of changing this value.
* We have experienced issues with petalinux when it is not installed to /opt/pkg/petalinux/. Digilent highly recommends installing petalinux
  to that location on your system.
* Netboot address and u-boot text address may need to be modified when using initramfs and rootfs is too large. The ramifications of this
  need to be explored and notes should be added to this guide. If this is causing a problem, then u-boot will likely crash or not successfully
  load the kernel. The workaround for now is to use SD rootfs.
* Ethernet PHY reset is not indicated in the device tree. This means it will not be used by the linux and u-boot drivers. This should not 
  cause any problems at runtime, the reset is only used on boot. See commented device tree lines for how to enable. When uncommented, ethernet 
  was not functional, likely due to a mismatch in the GPIO polarity. 
* To support using the generic UIO driver we have to override the bootargs. This is sloppy, and we should explore modifying our
  demos/libraries to use modprobe to load the uio driver as a module and set the of_id=generic-uio parameter at load time. Then
  we could stop overriding the bootargs in the device tree and also keep the generic uio driver as a module (which is petalinux's
  default) instead of building it into the kernel.
* The Cora Z7 does not have onboard non-volatile memory to store the MAC address in, so it must be set another way. A globally unique MAC is
  provided for each Cora Z7 on a sticker next to the Zynq-7000 chip. The system can be configured to use this address two ways: (1) by creating
  a file called uEnv.txt on the FAT partition of the microSD card that contains the following line (replacing x's with the MAC address):

ethaddr=xx:xx:xx:xx:xx:xx

  or (2), by setting the MAC address in the petalinux-config menu and then rebuilding the binaries.
* It seems likely the XADC Wizard is no longer needed due to improvements made to the Xilinx XADC driver and petalinux. In a future release 
  it should be removed from the block diagram, and system-user.dtsi currently should be modified to target the node that is now generated 
  for the hard silicon XADC controller found in the Zynq PS. 
* In order to fix Petalinux compatibility with the 07S device, a patch needs to be applied to the device-tree-generator recipe. The patch has
  been added to the project as described in the Xilinx AR here: https://www.xilinx.com/support/answers/70402.html . This should not affect 
  build process or runtime experience, however it is likely this patch can be removed once this project gets updated to a newer version of 
  Petalinux.

## Quick-Start Guide

This guide will walk you through some basic steps to get you booted into Linux and rebuild the Petalinux project. After completing it, you should refer
to the Petalinux Reference Guide (UG1144) from Xilinx to learn how to do more useful things with the Petalinux toolset. Also, refer to the Known Issues 
section above for a list of problems you may encounter and work arounds.

This guide assumes you are using Ubuntu 16.04.3 LTS. Digilent highly recommends using Ubuntu 16.04.x LTS, as this is what we are most familiar with, and 
cannot guarantee that we will be able to replicate problems you encounter on other Linux distributions. Virtual machines with 150-200GB allocated are a good
way to manage the OS and library dependencies of Petalinux rather than going through the difficult process of setting up dual-boot on physical machines.

### Install the Petalinux tools

Digilent has put together this quick installation guide to make the petalinux installation process more convenient. Note it is only tested on Ubuntu 16.04.3 LTS. 

First install the needed dependencies by opening a terminal and running the following:

```
sudo -s
apt-get install tofrodos gawk xvfb git libncurses5-dev tftpd zlib1g-dev zlib1g-dev:i386  \
                libssl-dev flex bison chrpath socat autoconf libtool texinfo gcc-multilib \
                libsdl1.2-dev libglib2.0-dev screen pax 
reboot
```

Next, install and configure the tftp server (this can be skipped if you are not interested in booting via TFTP):

```
sudo -s
apt-get install tftpd-hpa
chmod a+w /var/lib/tftpboot/
reboot
```

Create the petalinux installation directory next:

```
sudo -s
mkdir -p /opt/pkg/petalinux
chown <your_user_name> /opt/pkg/
chgrp <your_user_name> /opt/pkg/
chgrp <your_user_name> /opt/pkg/petalinux/
chown <your_user_name> /opt/pkg/petalinux/
exit
```

Finally, download the petalinux installer from Xilinx and run the following (do not run as root):

```
cd ~/Downloads
./petalinux-v2017.4-final-installer.run /opt/pkg/petalinux
```

Follow the onscreen instructions to complete the installation.

### Source the petalinux tools

Whenever you want to run any petalinux commands, you will need to first start by opening a new terminal and "sourcing" the Petalinux environment settings:

```
source /opt/pkg/petalinux/settings.sh
```

### Download the petalinux project

There are two ways to obtain the project. If you plan on version controlling your project you should clone this repository using the following:

```
git clone --recursive https://github.com/Digilent/Petalinux-Cora-Z7-07S.git
```
If you are not planning on version controlling your project and want a simpler release package, go to https://github.com/Digilent/Petalinux-Cora-Z7-07S/releases/
and download the most recent .bsp file available there for the version of Petalinux you wish to use.


### Generate project

If you have obtained the project source directly from github, then you should simply _cd_ into the Petalinux project directory. If you have downloaded the 
.bsp, then you must first run the following command to create a new project.

```
petalinux-create -t project -s <path to .bsp file>
```

This will create a new petalinux project in your current working directory, which you should then _cd_ into.


### Run the pre-built image from SD

#### Note: The pre-built images are only included with the .bsp release. If you cloned the project source directly, skip this section. 

1. Obtain a microSD card that has its first partition formatted as a FAT filesystem.
2. Copy _pre-built/linux/images/BOOT.BIN_ and _pre-built/linux/images/image.ub_ to the first partition of your SD card.
3. Create a new file on the microSD card called uEnv.txt. Add the following line to the file, replacing the MAC address as found on the board's sticker, then save and close it:
```
ethaddr=00:0a:35:00:1e:53
```
4. Eject the SD card from your computer and insert it into the Cora Z7.
5. Short the MODE jumper (JP2).
6. Attach a power source and select it with JP3 (note that using USB for power may not provide sufficient current)
7. If not already done to provide power, attach a microUSB cable between the computer and the Cora Z7
8. Open a terminal program (such as minicom) and connect to the Cora Z7 with 115200/8/N/1 settings (and no Hardware flow control). The Cora Z7 UART typically shows up as /dev/ttyUSB1
9. Optionally attach the Cora Z7 to a network using ethernet.
10. Press the SRST button to restart the Cora Z7. You should see the boot process at the terminal and eventually a root prompt.

### Build the petalinux project

Run the following commands to build the petalinux project with the default options:

```
petalinux-build
petalinux-package --boot --force --fsbl images/linux/zynq_fsbl.elf --fpga images/linux/system_wrapper.bit --u-boot
```

### Boot the newly built files from SD 

Follow the same steps as done with the pre-built files, except use the BOOT.BIN and image.ub files found in _images/linux_.

### Configure SD rootfs 

This project is initially configured to have the root file system (rootfs) existing in RAM. This configuration is referred to as "initramfs". A key 
aspect of this configuration is that changes made to the files (for example in your /home/root/ directory) will not persist after the board has been reset. 
This may or may not be desirable functionality.

Another side affect of initramfs is that if the root filesystem becomes too large (which is common if you add many features with "petalinux-config -c rootfs)
 then the system may experience poor performance (due to less available system memory). Also, if the uncompressed rootfs is larger than 128 MB, then booting
 with initramfs will fail unless you make modifications to u-boot (see note at the end of the "Managing Image Size" section of UG1144).

For those that want file modifications to persist through reboots, or that require a large rootfs, the petalinux system can be configured to instead use a 
filesystem that exists on the second partition of the microSD card. This will allow all 512 MiB of memory to be used as system memory, and for changes that 
are made to it to persist in non-volatile storage. To configure the system to use SD rootfs, write the generated root fs to the SD, and then boot the system, 
do the following:

Start by running petalinux-config and setting the following option to "SD":

```
 -> Image Packaging Configuration -> Root filesystem type
```

Next, open project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi in a text editor and locate the "bootargs" line. It should read as follows:

`
		bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio";
`

Replace that line with the following before saving and closing system-user.dtsi:

`
		bootargs = "console=ttyPS0,115200 earlyprintk uio_pdrv_genirq.of_id=generic-uio root=/dev/mmcblk0p2 rw rootwait";
`

#### Note: If you wish to change back to initramfs in the future, you will need to undo this change to the bootargs line.

Then run petalinux-build to build your system. After the build completes, your rootfs image will be at images/linux/rootfs.ext4.

Format an SD card with two partitions: The first should be at least 500 MB and be FAT formatted. The second needs to be at least 1.5 GB (3 GB is preferred) and 
formatted as ext4. The second partition will be overwritten, so don't put anything on it that you don't want to lose. If you are uncertain how to do this in 
Ubuntu, gparted is a well documented tool that can make the process easy.

Copy _images/linux/BOOT.BIN_ and _images/linux/image.ub_ to the first partition of your SD card.

Identify the /dev/ node for the second partition of your SD card using _lsblk_ at the command line. It will likely take the form of /dev/sdX2, where X is 
_a_,_b_,_c_,etc.. Then run the following command to copy the filesystem to the second partition:

#### Warning! If you use the wrong /dev/ node in the following command, you will overwrite your computer's file system. BE CAREFUL

```
sudo umount /dev/sdX2
sudo dd if=images/linux/rootfs.ext4 of=/dev/sdX2
sync
```

The following commands will also stretch the file system so that you can use the additional space of your SD card. Be sure to replace the
block device node as you did above:

```
sudo resize2fs /dev/sdX2
sync
```

#### Note: It is possible to use a third party prebuilt rootfs (such as a Linaro Ubuntu image) instead of the petalinux generated rootfs. To do this, just copy the prebuilt image to the second partition instead of running the "dd" command above. Please direct questions on doing this to the Embedded linux section of the Digilent forum.

Eject the SD card from your computer, then do the following:

1. Insert the microSD into the Cora Z7
2. Attach a power source and select it with JP3 (note that using USB for power may not provide sufficient current)
3. If not already done to provide power, attach a microUSB cable between the computer and the Cora Z7
4. Open a terminal program (such as minicom) and connect to the Cora Z7 with 115200/8/N/1 settings (and no Hardware flow control). The Cora Z7 UART typically shows up as /dev/ttyUSB1
5. Optionally attach the Cora Z7 to a network using ethernet.
6. Press the SRST button to restart the Cora Z7. You should see the boot process at the terminal and eventually a root prompt.

### Prepare for release

This section is only relevant for those who wish to upstream their work or version control their own project correctly on Github.
Note the project should be released configured as initramfs for consistency, unless there is very good reason to release it with SD rootfs.

```
petalinux-package --prebuilt --clean --fpga images/linux/system_wrapper.bit -a images/linux/image.ub:images/image.ub
echo "ethaddr=00:0a:35:00:1e:53" > pre-built/linux/images/uEnv.txt 
petalinux-build -x distclean
petalinux-build -x mrproper
petalinux-package --bsp --force --output ../releases/Petalinux-Cora-Z7-07S-20XX.X-X.bsp -p ./
cd ..
git status # to double-check
git add .
git commit
git push
```
Finally, open a browser and go to github to push your .bsp as a release.


