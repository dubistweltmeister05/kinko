2025-01-08 16:37

Tags: [[Linux Kernel Module]]

Now that we know what a LKM is, let's see what does a LKM look like. In all programming languages, there is a start and an ending point. It will start from the start of the main function and run through the functions which are calling from the main function. Finally, it exits at the main function closing point. 
In the case of an LKM, however, there are 2 separate functions - 
1. Init function
2. Exit Function

### Init Function
This is the function that will execute first when the Linux device driver is loaded into the kernel. For example, when we load the driver using **`insmod`**, this function will execute. Please see below to know the syntax of this function.
`static int __init hello_world_init(void) /* Constructor */`
`{`
    `return 0;`
`}`
`module_init(hello_world_init);`

This function should register itself by using **`module_init()`** macro.
• Prototype: int fun_name (
• Must return a value ; 0 for success, nonzero means module initialization failed. So the module will not get loaded in the kernel.
• This is an entry point to your module ( like main). This function will get called during boot time in the case of static modules 
• In the case of dynamic modules, this function will get called during module insertion
• There should be one module initialization entry point in the module

THIS IS A MODULE SPECIFIC FUNCTION AND SHOULD NEVER BE CALLED FROM ANY OTHER MODULE. IT SHOULD NEVER PROVIDE ANY FUNCTIONALITY THAT CAN BE REQUESTED BY ANY OTHER MODULE. HENCE, WE MAKE IT PRIATE USING A `static` TYPE QUALIFIER. 

### Exit Function
This is the function that will execute last when the Linux device driver is unloaded from the kernel. For example, when we unload the driver using **`rmmod`**, this function will execute. Please see below to know the syntax of this function.
`void __exit hello_world_exit(void)`
`{`

`}`
`module_exit(hello_world_exit);`
This function should register itself by using **`module_exit()`** macro.

• Prototype void fun_name()
• This is an entry point when the module is removed
• Since you can not remove static modules, clean up function will get called only in the case of dynamic modules when it is removed using user space command such as rmmod
• If you write a module and you are sure that it will always be statically linked with the kernel, then there is no need to implement this function.
• Even if your static module has a clean up function, the kernel build system will remove it during the build process if there is an `__exit` marker.
• Typically, you must do exact reverse operation what you had done in the module init function. undoing init function.
• Free memory which are requested in init function
• De init the devices or leave the device in the proper state


### The **`PrintK()`** function 
Like the printf(), there is a printk() function. 
- **`Printk()`** is a kernel-level function, which has the ability to print out to different log levels as defined in. We can see the prints using **`dmesg`** command.
- `printf()` will always print to a file descriptor – STD_OUT. We can see the prints in the STD_OUT console.

Note: In the newer Linux kernels, you can use the APIs below instead of this **`printk`**.

- **`pr_info`** – Print an info-level message. (ex. **`pr_info("test info message\n")`**).
- **`pr_cont`** – Continues a previous log message in the same line.
- **`pr_debug`** – Print a debug-level message conditionally.
- **`pr_err`** – Print an error-level message. (ex. **`pr_err(“test error message\n”)`**).
- **`pr_warn`** – Print a warning-level message.


## Header 

There exist different headers for Kernel Space and User Space modules. 
1. kernel space header - `#include <Linux/module.h>`
2. user space header - Couldn't find a standard one, there's a LOT of them to be fair. All the common ones in C programming are applicable here.  

• Since you write a kernel module that is going to be executed in kernel
space, you should be using kernel headers, never include any user
space library headers like C std library header files.
• No user space library is linked to the kernel module during the build procedure. You don't just link the kernel module to a C library. 
• Most of the relevant kernel headers live in
`linux_source_ base / linux`.



# Reference

[Course: Linux Device Driver Programming With Beaglebone Black (LDD1) | Udemy](https://www.udemy.com/course/linux-device-driver-programming-using-beaglebone-black/learn/lecture/21622416#reviews)

[First Linux Device Driver - Linux Device Driver Tutorial Part 2](https://embetronicx.com/tutorials/linux/device-drivers/linux-device-driver-tutorial-part-2-first-device-driver/)
