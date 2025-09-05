[[YOCTO]]

Here is the structure of the metadata directories (the general ones, anyway)

- 1. `meta/classes`
  These contain the .bbclass files. These files are used to abstract common code so that it can be reused by multiple packages. The base file is inherited by all packages that wish to use. 
- `2. meta/conf/`
  This contains a core set of config files that start from `bitbake.conf` , from which all configuration files are included. There are a few sub-directories that lie under this - 
  - 1.` meta/conf/machine/`
    This contains the machine config files that we set in the local.conf files. If you set MACHINE = "qemux86", the OpenEmbedded build system looks for a qemux86.conf file in this directory. The include directory contains various data common to multiple machines. If you want to add support for a new machine to the Yocto Project, look in this directory
  - 2. `meta/conf/distro/`
    This controls any distro-specific configurations that need to be done according to the user
  - 3.` meta/conf/machine-sdk/`
    The OpenEmbedded build system searches this directory for configuration files that correspond to the value of SDKMACHINE. By default, 32-bit and 64-bit x86 files ship with the Yocto Project that support some SDK hosts. However, it is possible to extend that support to other SDK hosts by adding additional configuration files in this subdirectory within another layer.
- 3. `meta/files/ `
  This directory contains common license files and several text files used by the build system. The text files contain minimal device information and lists of files and directories with known permissions.
- 4.` meta/lib/`
  This directory contains OpenEmbedded Python library code used during the build process. It is enabled via the `addpylib` directive in `meta/conf/local.conf.`
- 5. `meta/recipes-bsp/`
  This directory contains anything linking to specific hardware or hardware configuration information such as “u-boot” and “grub”.
- 6. `meta/recipes-connectivity/`
  This directory contains libraries and applications related to communication with other devices.
- 7. `meta/recipes-core/`
  This directory contains what is needed to build a basic working Linux image including commonly used dependencies.
- 8. `meta/recipes-devtools/`
  This directory contains tools that are primarily used by the build system. The tools, however, can also be used on targets.
- 9.` meta/recipes-extended/`
  This directory contains non-essential applications that add features compared to the alternatives in core. You might need this directory for full tool functionality.
- 10. `meta/recipes-gnome/`
  This directory contains all things related to the GTK+ application framework.
- 11. `meta/recipes-graphics/`
  This directory contains X and other graphically related system libraries.
- 12. `meta/recipes-kernel/`
  This directory contains the kernel and generic applications and libraries that have strong kernel dependencies.
- 13. `meta/recipes-multimedia/`
  This directory contains codecs and support utilities for audio, images and video.
- 14. `meta/recipes-rt/`
  This directory contains package and image recipes for using and testing the `PREEMPT_RT` kernel.
- 15. `meta/recipes-sato/`
  This directory contains the Sato demo/reference UI/UX and its associated applications and configuration data.
- 16. `meta/recipes-support/`
  This directory contains recipes used by other recipes, but that are not directly included in images (i.e. dependencies of other recipes).
- 17.` meta/site/`
  This directory contains a list of cached results for various architectures. Because certain “autoconf” test results cannot be determined when cross-compiling due to the tests not able to run on a live system, the information in this directory is passed to “autoconf” for the various architectures.
- 18. `meta/recipes.txt`