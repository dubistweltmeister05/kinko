[[3 - Tags/YOCTO]]![[Pasted image 20250609092857.png]]


 At the heart of the build system is BitBake, the task executor. In general, the build’s workflow consists of several functional areas:

- User Configuration: metadata you can use to control the build process.

- Metadata Layers: Various layers that provide software, machine, and distro metadata.

- Source Files: Upstream releases, local projects, and SCMs.

- Build System: Processes under the control of BitBake. This block expands on how BitBake fetches source, applies patches, completes compilation, analyzes output for package generation, creates and tests packages, generates images, and generates cross-development tools.

- Package Feeds: Directories containing output packages (RPM, DEB or IPK), which are subsequently used in the construction of an image or Software Development Kit (SDK), produced by the build system. These feeds can also be copied and shared using a web server or other means to facilitate extending or updating existing images on devices at runtime if runtime package management is enabled.

- Images: Images produced by the workflow.

- Application Development SDK: Cross-development tools that are produced along with an image or separately with BitBake.


## USER CONFIGURATION

This helps define the build being undertaken. Through this, the BItBake tool understands the target architecture for which we are building the image, where to store the downloaded source, and other build properties. 
![[Pasted image 20250609093543.png]]

The `.conf` files are the basic config files that bit bake needs to start the build. he minimally necessary ones reside as example files in the build/conf directory of the **Source Directory**.

The`meta-poky` layer inside Poky contains a conf directory that has example configuration files. These example files are used as a basis for creating actual configuration files when you source `oe-init-build-env,` which is the build environment script. Source the  script and you get a `build` directory in the `poky` folder (if one doesn't exist already).

The Build Directory has a conf directory that contains default versions of your `local.conf`and `bblayers.conf`configuration files. These default configuration files are created only if versions do not already exist in the Build Directory at the time you source the build environment setup script.

The local.conf file provides many basic variables that define a build environment. Here is a list of a few. To see the default configurations in a local.conf file created by the build environment script, see the local.conf.sample in the meta-poky layer:

- Target Machine Selection: Controlled by the MACHINE variable.

- Download Directory: Controlled by the DL_DIR variable.

- Shared State Directory: Controlled by the SSTATE_DIR variable.

- Build Output: Controlled by the TMPDIR variable.

- Distribution Policy: Controlled by the DISTRO variable.

- Packaging Format: Controlled by the PACKAGE_CLASSES variable.

- SDK Target Architecture: Controlled by the SDKMACHINE variable.

- Extra Image Packages: Controlled by the EXTRA_IMAGE_FEATURES variable.


%%
Configurations set in the `conf/local.conf` file can also be set in the `conf/site.conf` and `conf/auto.conf` configuration files.
 %%

The `bblayers.conf` file tells BitBake what layers you want considered during the build. By default, the layers listed in this file include layers minimally needed by the build system. However, you must manually add any custom layers you have created.


##  METADATA, MACHINE CONFIG AND POLICY

Usually, there are 3 layers of input that are used to define the BitBake build and control it in a lot more detail. They are 
 - _Metadata (.bb + Patches):_ Software layers containing user-supplied recipe files, patches, and append files.
 - _Machine BSP Configuration:_ Board Support Package (BSP) layers (i.e. “BSP Layer” in the following figure) providing machine-specific configurations. This type of information is specific to a particular target architecture.
 - _Policy Configuration:_ Distribution Layers (i.e. “Distro Layer” in the following figure) providing top-level or general policies for the images or SDKs being built for a particular distribution.
![[Pasted image 20250609100241.png]]

**BitBake uses the `conf/bblayers.conf` file, which is part of the user configuration, to find what layers it should be using as part of the build.**


#### 1. Distro Layer

The distribution layer provides policy configurations for your distribution. Best practices dictate that you isolate these types of configurations into their own layer. Settings you provide in `conf/distro/distro.conf` override similar settings that BitBake finds in your `conf/local.conf` file in the [Build Directory](https://docs.yoctoproject.org/ref-manual/terms.html#term-Build-Directory).

This is what you may find in a distro layer - 
 - _classes:_ Class files (`.bbclass`) hold common functionality that can be shared among recipes in the distribution. When your recipes inherit a class, they take on the settings and functions for that class.
 - _conf:_ This area holds configuration files for the layer (`conf/layer.conf`), the distribution (`conf/distro/distro.conf`), and any distribution-wide include files.
 - - _recipes-_:* Recipes and append files that affect common functionality across the distribution. This area could include recipes and append files to add distribution-specific configuration, initialization scripts, custom image recipes, and so forth. Examples of `recipes-*` directories are `recipes-core` and `recipes-extra`. Hierarchy and contents within a `recipes-*` directory can vary. Generally, these directories contain recipe files (`*.bb`), recipe append files (`*.bbappend`), directories that are distro-specific for configuration files, and so forth.
   
#### 2. BSP Layer

The BSP Layer provides machine configurations that target specific hardware. Everything in this layer is specific to the machine for which you are building the image or the SDK. A common structure or form is defined for BSP layers. You can learn more about this structure in the [Yocto Project Board Support Package Developer’s Guide](https://docs.yoctoproject.org/bsp-guide/index.html).

Note

In order for a BSP layer to be considered compliant with the Yocto Project, it must meet some structural requirements.

The BSP Layer’s configuration directory contains configuration files for the machine (`conf/machine/machine.conf`) and, of course, the layer (`conf/layer.conf`).

The remainder of the layer is dedicated to specific recipes by function: `recipes-bsp`, `recipes-core`, `recipes-graphics`, `recipes-kernel`, and so forth. There can be metadata for multiple formfactors, graphics support systems, and so forth.

Note

While the figure shows several recipes-* directories, not all these directories appear in all BSP layers.

#### 3. Software Layer

The software layer provides the Metadata for additional software packages used during the build. This layer does not include Metadata that is specific to the distribution or the machine, which are found in their respective layers.

This layer contains any recipes, append files, and patches, that your project needs.

## SOURCES

The open-embedded system needs access to the source files in order to create an image or a target. The method of configuring these sources is a function of the project itself. 

Say, if a project is targeting a release, it can keep the file as a tarball, guaranteeing that the state of a release is statically represented. For a project that is more dynamic in it's nature, it can put these into a git repo. This ensures that updates are periodically pulled and teh project maintains compatibility with the new versions of the source files. 

BitBake uses the [SRC_URI](https://docs.yoctoproject.org/ref-manual/variables.html#term-SRC_URI) variable to point to source files regardless of their location. Each recipe must have a [SRC_URI](https://docs.yoctoproject.org/ref-manual/variables.html#term-SRC_URI) variable that points to the source.

Another area that plays a significant role in where source files come from is pointed to by the [DL_DIR](https://docs.yoctoproject.org/ref-manual/variables.html#term-DL_DIR) variable. This area is a cache that can hold previously downloaded source. You can also instruct the OpenEmbedded build system to create tarballs from Git repositories, which is not the default behavior, and store them in the [DL_DIR](https://docs.yoctoproject.org/ref-manual/variables.html#term-DL_DIR) by using the [BB_GENERATE_MIRROR_TARBALLS](https://docs.yoctoproject.org/ref-manual/variables.html#term-BB_GENERATE_MIRROR_TARBALLS) variable.

![[Pasted image 20250609112335.png]]

