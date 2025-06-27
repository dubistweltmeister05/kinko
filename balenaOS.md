```bash
 LIC_FILES_CHKSUM = "file://COPYING;md5=6bc538ed5bd9a7fc9398086aedcd7e46"
  
  KERNEL_VERSION_SANITY_SKIP = "1"
  LINUX_VERSION ?= "5.10"
  EXCLUDE_FROM_WORLD = "1"
  
  SRC_URI:append = " ${@bb.utils.contains('IMAGE_FSTYPES', 'ext4', \
             'file://${THISDIR}/files/ext4.cfg', \
             '', \
             d)}"
  
  do_compile_append() {
      oe_runmake dtbs
  }
  
  do_install_append() {
      # Get the kernel version string from the build system
      VER=$(grep "^VERSION =" ${S}/Makefile | awk '{print $3}')
      PATCH=$(grep "^PATCHLEVEL =" ${S}/Makefile | awk '{print $3}')
      SUB=$(grep "^SUBLEVEL =" ${S}/Makefile | awk '{print $3}')
      LOCAL=$(grep "^CONFIG_LOCALVERSION=" ${B}/.config | cut -d'"' -f2)
  
      KERNEL_VERSION="${VER}.${PATCH}.${SUB}${LOCAL}"
  
      # Fallback if STAGING_DIR lookup fails
      if [ -z "$KERNEL_VERSION" ]; then
          KERNEL_VERSION="${KERNEL_VERSION ?? 5.10.233-rockchip-standard}"
      fi
  
      install -d ${D}/boot/overlays-${KERNEL_VERSION}
      install -m 0644 ${B}/arch/arm64/boot/dts/rockchip/overlays/* ${D}/boot/overlays-${KERNEL_VERSION}
  }
  
  do_deploy_append() {
      install -d ${DEPLOYDIR}/overlays
      install -m 644 ${B}/arch/arm64/boot/dts/rockchip/overlays/* ${DEPLOYDIR}/overlays
   }
   
   do_patch:append() {
       sed -i 's/-I\($(BCMDHD_ROOT)\)/-I$(srctree)\/\1/g' \
           ${S}/drivers/net/wireless/rockchip_wlan/rkwifi/bcmdhd/Makefile
   }
   
   PACKAGES += "${PN}-overlays"
   FILES_${PN}-overlays = "/boot/overlays*/*"

```