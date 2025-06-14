2025-01-14 09:48

Tags: [[Linux Kernel Module]]

There are 2 ways to build the kernel module - the static method and the dynamically loadable kernel module. The dynamic module also has 2 methods of building - in_tree build and out_of_tree build.
Let's take a look at both.

### In-tree and Out of Tree modules
 - Out of tree means outside of the linux kernel source tree. 
 - The ones that are already looked at by the maintainers and kernel developers, approved by them, and integrated in the Linux Kernel source tree are called as the In-Tree modules.
 - When we write a module separately, build it, and link it against the kernel source tree,(it isn't approved yet because of potential bugs in the module) it is called as an out-of-tree module.
 - When we do that, we get a message that the kernel has been tainted, which we conveniently ignore [;-)] . 
### Building an Out Of Tree Module
 - modules are built using the kbuild build system. This must be used in order to stay compatible with the changes in the build infrastructure and to pick up the correct flags to GCC.
 - To build an external kernel module, we must have a pre-build kernel source that has the header files that we have included in the module that we wish to load, and the configuration that is used in the build. 
 - This ensures that as we change the kernel configuration, the custom driver is rebuilt using the new kernel configuration. 

The command that is used to build the kernel module is as follows - `make -C <path to Linux Kernel tree> M=<path to the module> [target] `

![[Pasted image 20250114100626.png]]

# Note

 - When we are building out of the kernel tree, better have a pre-compiled version of the kernel on the system. Remember, the module is linked against the pre-existing headers and configuration of the kernel source. 
 - We cannot just compile the kernel module on one version of the Linux kernel and run it on another version of the kernel. That won't work, and even if it does, it is a guaranteed recipe for DISASTER in the runtime. 

# Reference

[Course: Linux Device Driver Programming With Beaglebone Black (LDD1) | Udemy](https://www.udemy.com/course/linux-device-driver-programming-using-beaglebone-black/learn/lecture/21622430?start=405#reviews)
