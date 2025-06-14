[[Allwinner A20]]
##### DATE(Start)- 2/6/25

##### DATE (End) - 3/6/25

to list all serial devices - 
```
sudo dmesg | grep -i tty
```


make CROSS_COMPILE=arm-linux-gnueabihf- sunxi_defconfig


```

make CROSS_COMPILE=arm-linux-gnueabihf- A20-OLinuXino-Lime2_defconfig
```

```
sudo blockdev --rereadpt ${card}
cat <<EOT | sudo tee /dev/null | sudo sfdisk ${card}
1M,16M,c
,,L
EOT
```

```
sudo mkfs.vfat ${card}${p}1
sudo mkfs.ext4 ${card}${p}2
```


```
sudo mount ${card}${p}1 /mnt/
sudo cp /home/ronin/Desktop/Vicharak/linux/arch/arm/boot/zImage /mnt/
sudo cp /home/ronin/Desktop/Vicharak/linux/arch/arm/boot/dts/allwinner/sun7i-a20-olinuxino-lime2.dtb /mnt/
sudo umount /mnt/
```


```
sudo mount ${card}${p}2 /mnt/
sudo tar -C /mnt/ -xJpf /home/ronin/Desktop/Vicharak/RFS/noble-server-cloudimg-armhf-root.tar.xz
sudo umount /mnt
```

sudo mount ${card}${p}1 /mnt/
sudo cp /home/ronin/Desktop/Vicharak/linux-sunxi/arch/arm/boot/uImage /mnt/
sudo cp /home/ronin/Desktop/Vicharak/toolchains/sunxi-tools/script.bin /mnt/
sudo umount /mnt/





sudo cp /home/ronin/Desktop/Vicharak/boot.cmd /mnt/






/home/user/dir/sunxi-tools/fex2bin a20-olinuxino_lime2.fex script.bin


sudo sh -c 'echo "deb http://archive.ubuntu.com/ubuntu xenial main universe" > /etc/apt/sources.list.d/xenial.list'
sudo apt update





#### LINKS - 
A20  wiki - https://linux-sunxi.org/A20

A20 U-Boot - https://linux-sunxi.org/U-Boot

TOOLCHAIN - https://linux-sunxi.org/Toolchain

