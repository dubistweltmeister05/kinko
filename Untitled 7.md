
make O=out ARCH=arm64 LOCALVERSION=-rockchip-standard CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc --all)

make O=out ARCH=arm64 modules_install INSTALL_MOD_PATH=modules LOCALVERSION=-rockchip-standard -j$(nproc --all)

lib/modules/6.1.75-rockchip-standard/kernel/drivers/net/wireless/rockchip_wlan/rtl8822cs/88x2cs.ko

file://${THISDIR}/0001-ARM64-configs-vaaman-Enable-RTL8822CS-Wi-Fi-driver.patch

/home/pratiksha/hdd/pratiksha/kshitij/vicharak-linux-sdk/yocto/build/tmp/work/rk3399_vaaman-poky-linux/core-image-full-cmdline/1.0-r0/rootfs


/home/vicharak/nvme/axon-yocto

/home/vicharak/nvme/axon-yocto/meta-openembedded/meta-oe \
/home/vicharak/nvme/axon-yocto/meta-openembedded/meta-python \
/home/vicharak/nvme/axon-yocto/meta-openembedded/meta-networking \
/home/vicharak/nvme/axon-yocto/meta-openembedded/meta-filesystems \
/home/vicharak/nvme/axon-yocto/meta-openembedded/meta-multimedia \
/home/vicharak/nvme/axon-yocto/meta-openembedded/meta-xfce
/home/vicharak/nvme/axon-yocto/meta-qt5 \
/home/vicharak/nvme/axon-yocto/meta-clang \
/home/vicharak/nvme/axon-yocto/meta-rockchip \




4daa014098ca253279e38a89bfe6957b0687aae0