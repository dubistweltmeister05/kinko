2025-01-08 17:26

Tags: [[Linux Kernel Module]]

- Basically, these are C macros that are defined in the /include/linux/init.h file.
- They expand to compiler directives, which basically tell the compiler to place the function code in a special section. 
- the `__init` macro places the code in the output section that is called as the .init section, and the `__exit` macro places it in a section called as the .exit section. 


### `__init` 
• `__init` is a macro which will be translated into compiler directive, which instructs the
compiler to put the code in .init section of the final ELF of linux kernel image.

• init section will be freed from the memory by the kernel during boot time once all the initialization functions get executed.

• Since the built in driver cannot be unloaded, its init function will not be called again until the next reboot, that’s why there is no need to keep references to its init function anymore.

• so using init macro is a technique, when used with  a function, the kernel will free the code memory of that function after its execution.

• Similarly, you can use initdata with variables that will be dropped after the initialization. initdata , which works similarly to init but for init variables rather than functions.

### `__exit`

• You know that for built in modules clean up function is not required
• So, when you use the `__exit` macro with a clean up function, the kernel build system will exclude those functions during the build process itself.

![[Pasted image 20250109002033.png]]

In this, the m1 and m2 init functions are called my the kernel during the boot, executed and then the memory that was reserved for these function is freed, while the memory for m3 is retained even after the execution of the function.
# Reference

