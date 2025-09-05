[[YOCTO]]
## YOCTO Local Source - 
`/home/pratiksha/hdd/pratiksha/kshitij/vicharak-linux-sdk/yocto`

## YOCTO Git repos - 

meta rockchip - `https://github.com/vicharak-in/meta-rockchip`

rockchip-kernel - `https://github.com/vicharak-in/rockchip-linux-kernel-priv` 
	  commit id - `cbb5086a42d23f7d4485c71048a5df4ad1056686`

## Code to be added to build/conf/local.conf
``` bash
MACHINE ??= "qemux86-64"  
MACHINE ?= "rk3399-vaaman"  
INIT_MANAGER = "systemd"  
IMAGE_INSTALL:append = "kernel-modules"  
IMAGE_INSTALL:append = " linux-rockchip-overlays"  
LICENSE_FLAGS_ACCEPTED += "commercial"  
IMAGE_INSTALL:append = " kernel-modules mpg123 alsa-lib alsa-utils alsa-plugins pulseaudio pulseaudio-server pulseaudio-misc v4l-utils"  
DISTRO_FEATURES:append = " alsa"  
PACKAGECONFIG:pn-mpg123 = "pulseaudio"  
ROOTFS_POSTPROCESS_COMMAND += " \  
   install -d ${IMAGE_ROOTFS}/boot/extlinux; \  
   install -m 0644 ${DEPLOY_DIR_IMAGE}/extlinux.conf ${IMAGE_ROOTFS}/boot/extlinux/extlinux.conf; \  
"  
IMAGE_INSTALL:append = " u-boot-rockchip-extlinux"  
  
IMAGE_BOOT_FILES = "Image rk3399-vaaman-linux--6.1-r0-rk3399-vaaman-20250709061407.dtb:rk3399-vaaman.dtb extlinux.conf"
```

## Changed code in u-boot-rockchip.bb
```bash

do_install:append() {
    install -d ${D}/boot/extlinux
    if [ -f ${DEPLOYDIR}/extlinux.conf ]; then
        install -m 0644 ${DEPLOYDIR}/extlinux.conf ${D}/boot/extlinux/extlinux.conf
    fi
}

FILES:${PN} += "/boot/extlinux/extlinux.conf"

```

## Commands to trigger the YOCTO build

1. In the directory -> `/home/pratiksha/hdd/pratiksha/kshitij/vicharak-linux-sdk/yocto`, run the command `source oe-init-build-env `
2. You should automatically `cd` into the `build directory`. Add the lines mentioned above in the `/conf/local.conf` file. 
3. Run the command -> `bitbake core-image-full-cmdline` to automatically trigger the build.
## HOW TO ADD KERNEL PATCHES TO THE YOCTO BUILD

Should you wish to debug something at the kernel level via kernel patches, follow these steps to apply the same to the YOCTO build.

1. Generate your patch in the following format -> `000x-<short_description_of_changes>.patch`
2. place it in the following directory -> `/home/pratiksha/hdd/pratiksha/kshitij/vicharak-linux-sdk/yocto/meta-rockchip/recipes-kernel/linux/files`
3. Add the patch path to the .bb file (like `linux-rockchip_6.1.bb`) ->
``` bash
1.SRC_URI = " \  
       git://github.com/vicharak-in/rockchip-linux-kernel-priv.git;protocol=https;nobranch=1;branch=vicharak-6.1; \  
       file://${THISDIR}/files/rk3399_vaaman.cfg \  
       file://${THISDIR}/files/cgroups.cfg \  
       file://${THISDIR}/files/ext4.cfg \  
file://${THISDIR}/files/0001-ARM64-configs-vaaman-Enable-RTL8822CS-Wi-Fi-driver.patch \  
file://${THISDIR}/files/0002-rk3399-vaaman.dtsi-Added-Support-for-Audio-Codec.patch \  
file://${THISDIR}/files/0003_enable_camera_node_via_dtsi.patch \  
file://${THISDIR}/files/000x-<short_description_of_changes>.patch \  
" 
```

