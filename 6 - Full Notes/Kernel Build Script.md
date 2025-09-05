export CROSS_COMPILE=aarch64-linux-gnu-

make O=out ARCH=arm64 rockchip_linux_defconfig

cat arch/arm64/configs/rk3588_axon.config >> out/.config

make O=out ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc --all)