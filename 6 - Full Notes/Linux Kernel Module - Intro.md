2025-01-08 09:49

Tags: [[Linux Kernel Module]]

 - Linux supports dynamic insertion and removal of code from the kernel while the system is up and running. The code what we add and remove at run time is called	a kernel module.
 - Once the module is mounted, we can use the new functionality and features that are exposed to the system via the kernel module with out having to restart the entire machine again. 
 - The LKM exposed new functionality to the system in a dynamic manner, introducing features to the kernel such as security, device drivers, file system drivers, system calls etc. 
 - An embedded system that is running Linux can enjoy the benefits of running the kernel on a minimal base image (leading to reduced runtime storage) and optional device drivers being supplied via dynamic insertion of modules. 
There are 2 types of LKM.
1. STATIC(y)
	These are 'built-in' to the kernel image, that is, the module becomes a part of the kernel image. Statically linked kernel modules do make the size of the kernel's final image a bit bigger, and we cannot 'unload' the module. It occupies a piece of the memory permanently during the run time of the system. 
2. DYNAMIC(m)
	These are linked dynamically, meaning that they are not a part of the final kernel image that is embedded onto the device. We compile and link these separately to produce a .ko file, and these can be dynamically loaded and unloaded from the kernel space via insmod, modprobe or rmmod. 

# Reference

