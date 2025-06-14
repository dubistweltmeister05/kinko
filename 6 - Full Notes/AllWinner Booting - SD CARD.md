[[Allwinner A20]]

The boot is a simple 3-part process - 
 - U-Boot
 - Kernel
 - RFS

Everything is handled via the SD card. Below is a guide on what worked for each step of the process.

## U-Boot
Remember that we are working on the mainline U-Boot, and NOT on the legacy version. Get the mainline U-Boot from  ->
```
git clone git://git.denx.de/u-boot.git
```

Install the following dependencies ->
  - ```sudo apt-get install swig```
  - `sudo apt-get install python3-dev`
  - `sudo apt-get install device-tree-compiler`

Change into the u-boot folder. Remember, the defconfig file that we use for this u-boot is - `A20-OLinuXino-Lime2_defconfig`

Run the command `make CROSS_COMPILE=arm-linux-gnueabihf- A20-OLinuXino-Lime2_defconfig`, to get a .config file for the u-boot.

#### OUTPUT - `u-boot-sunxi-with-spl.bin`
Remember the full path for this file.

## Kernel
Remember, since we are working with the mainline U-boot, the KERNEL has to be mainline as well. DO NOT INSTALL THE LEGACY KERNEL OPTION. 

If it is your first time working with the A20 (Welcome to the Linux Kernel Team at Vicharak, btw), then a shallow clone of the main Linux repo will work just fine.

Clone a working shadow copy of the kernel repository via the following command - 
`git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git --depth=1`

Change into the kernel directory and run the following commands - 
 - defconfig - `make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- sunxi_defconfig`
 - zImage - `ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j4 zImage`
 - devicetree(DTB) - `ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j4 dtbs`
 - modules - `ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j4 modules`
#### OUTPUT - zImage and dtb (`sun7i-a20-olinuxino-lime2.dtb`).

## RFS 
install the Ubuntu noble server cloud image. NOT the entire image, but just the rootfs image as a tarball. 

Get it from here `https://cloud-images.ubuntu.com/noble/20250403/`

#### OUTPUT - tarball of rootfs


# SD-CARD Prep
Get a 16GB SD card and a card reader.

Back the SD card up in a known folder of your machine, as a .img file (just in case).

Run `lsblk` on a terminal to get the name of the SD Card (sdX, where X can be a,b,c,...etc).

Next, sort out the environment variables 
```
export card=/dev/sdX
export p=""
```

Format (Clean) the SD card via the command - `sudo dd if=/dev/zero of=${card} bs=1M count=1`

Then, change into the directory that holds the u-boot that you compiled. Run - `dd if=u-boot-sunxi-with-spl.bin of=${card} bs=1024 seek=8` to copy the u-boot SPL into the SD card.

Next, we have to create 2 partitions on the SD card. Run - 
```
sudo blockdev --rereadpt ${card}
sudo sfdisk ${card} <<EOF
1M,16M,c
,,L
EOF
```

```
sudo mkfs.vfat ${card}${p}1
sudo mkfs.ext4 ${card}${p}2
cardroot=${card}${p}2
```

Now, mount the dtb and the zImage on your SD Card. Run - 
```
sudo mount ${card}${p}1 /mnt/
sudo cp <path-to-your-zImage> /mnt/
sudo cp <path-to-the-dtb> /mnt/
sudo umount /mnt/
```

Next, change into the directory of your SD card's directory and create a boot.cmd file. Write the following in the file - 
```
setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait panic=10
load mmc 0:1 0x43000000 ${fdtfile} || load mmc 0:1 0x43000000 boot/${fdtfile}
load mmc 0:1 0x42000000 zImage || load mmc 0:1 0x42000000 boot/zImage
bootz 0x42000000 - 0x43000000
```

Finally, to wrap the boot.cmd around a header, run the following command - 
```
mkimage -C none -A arm -T script -d boot.cmd boot.scr
```

## KNOWN ERRORS
 - **ISSUE 1** -> The legacy kernel refuces to compile when it compiels using GCC-13. 
 **FIX** -> The repository has the gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf toolchain files. Install it and specify it's usage during the compiling of the kernel via the followig commands - 
 ```
 export TOOLCHAIN_PATH=/home/ronin/Downloads/gcc-linaro-4.9-2017.01/bin
export PATH=$TOOLCHAIN_PATH:$PATH
export CROSS_COMPILE=arm-linux-gnueabihf-

TO VERIFY - $ arm-linux-gnueabihf-gcc --version
 ```
That's all!
