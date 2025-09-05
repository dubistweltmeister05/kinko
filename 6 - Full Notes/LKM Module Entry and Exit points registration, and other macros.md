2025-01-09 00:24

Tags: [[Linux Kernel Module]]


### Module Entry and Exit Registration 

`module_init(my_kernel_module_entry);`
`module_exit(my_kernel_module_exit);`

• These are the macros used to register your module’s init function and clean up function with the kernel.

• Here `module_init/module_exit` is not a function, but a macro defined in `linux/module.h`

• For example, `module_init()` will add its parameter to the init entry point database of the kernel.

• `module_exit()` will add its parameter to exit entry point database of the kernel

### Module Descriptors
• MODULE_LICENSE is a macro used by the kernel module to announce its license type.

• If you a load module whose license parameter is non GPL(General Public License), then kernel triggers warning of being tainted. Its way of kernel letting the users and developers know its non free license based module.

• The developer community may ignore the bug reports you submit after loading the proprietary licensed module

• The declared module license is also used to decide whether a given module can have access to the small number of "GPL only" symbols in the kernel.

• Go to `linux/module.h` to find out what are the allowed parameters which can be used with this macro to load the module without tainting the kernel.

# Reference

