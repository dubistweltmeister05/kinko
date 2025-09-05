[[3 - Tags/YOCTO]]

LINK:- https://www.youtube.com/watch?v=zNLYanJAQ3s

The YOCTO project is an umbrella project which is basically when the entire industry and OSS community is coming together to put together a set of tools and best practices when it comes to building custom linux projects. 

# YOCTO BUILD SYSTEM

### Poky = BitBaake + metadata
 - Poky is a build system that is used by the Yocto Proejct.
 - BitBake is a task executor and a scheduler
 - Metadata is a set of task definitions that includes the .**conf** (Global configurations), the .**bbclass**(encapsulators and inheritors of build logic, packaging eyc), .**bb**(logical units of the software/images to build)
# Key Concepts 
 - The yocto project provides tools and metadata to create custom linux images.
 - These images are created form a repo of "BAKED" recipies.
 - A recipe is a set of instructions for building packages, including - 
	 - Where to obtain upstream sources
	 - Dependencies (on libraries or other recipes)
	 - Configuration/compilation options
	 - Define what files go into what output packages
	A recipe is NOT a makefile
 ![[Pasted image 20250607004417.png]]

Here is an idea of what the actual FUCK goes on during the build process - 
1. Fetch the sources from the remotes. Upstream releases, Local projects, SCMs, whatever.
2. The downloaded sources are then taken to a known location, where any and every needed patch is then applied to them.
3. Then, the config is completed, and according to the selected package creation algo, the necessary output packages are then generated and shipped the fuck out. 
4. There are a whole lotta QA and sanity checks done too. 
5. This is then used to generate a binary package feed. This is helpful when we are building for an embedded application. This stream can be taken onto a, say, network server that is then fed into the fucking board that we are working with, setting up what I am thinking is a network boot scenario. I may be wrong, of course.

Since yocto is NOT a linux distro, there is not much uniformity in it's outputs.This means that WE decide which packages can be included in the final image that is generated.  We can also generate an SDK that is custom build using the yocto project, that is custom tailored to the image that we are making. 

Also, the FIRST image generation takes lot of time, but every time the image is rebuilt, the untouched parts of the image are NOT built again. 

Only the part that is tampered with is then rebuilt and this makes the rebuilding of a changed image very easy!



## Setting up yocto

1. run the following 
```
sudo apt update
sudo apt install gawk wget git diffstat unzip texinfo gcc build-essential \
     chrpath socat cpio python3 python3-pip python3-pexpect xz-utils \
     debianutils iputils-ping libsdl1.2-dev xterm zstd liblz4-tool
```

2. Create a working directory
```
mkdir -p ~/yocto
cd ~/yocto
```
3. Get the Poky reference distro
```
git clone git://git.yoctoproject.org/poky
cd poky
git checkout mickledore
```
4. Source the build
```
source oe-init-build-env
```
5. Configure Your Build
```

```




  /mnt/sda1/kshitij/yocto/meta-openembedded/meta-filesystems \
  /mnt/sda1/kshitij/yocto/meta-openembedded/meta-oe \
  /mnt/sda1/kshitij/yocto/meta-openembedded/meta-python \
  /mnt/sda1/kshitij/yocto/meta-openembedded/meta-networking \
  /mnt/sda1/kshitij/yocto/meta-openembedded/meta-multimedia \
  /mnt/sda1/kshitij/yocto/meta-qt5 \
  /mnt/sda1/kshitij/yocto/meta-clang \
  /mnt/sda1/kshitij/yocto/meta-rockchip \
  /mnt/sda1/kshitij/yocto/poky/meta \
  /mnt/sda1/kshitij/yocto/poky/meta-poky \
  /mnt/sda1/kshitij/yocto/poky/meta-yocto-bsp \

ERROR -> 
BitBake is trying to restrict network access using user namespaces, and Ubuntu 24.04's security policy blocks

FIX -> 
1. Comment the line `bb.utils.disable_network(uid, gid)` in bitbake-worker (/poky/bitbake/bin/bitbake-worker)
2. Add the following lines at the END of the local.conf file - 
      BB_NO_NETWORK_SANDBOX = "1"
	  BB_DISABLE_NETWORK_SANDBOX = "1"
      BB_DISABLE_SANDBOX = "1"
WHY->
On Ubuntu 24.04, /proc/self/uid_map is not writable by unprivileged users even if unprivileged_userns_clone = 1.

This leads to` PermissionError: [Errno 1] Operation not permitted.`
