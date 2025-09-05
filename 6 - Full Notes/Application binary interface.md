[[03 Object Files]]

Let's be clear, when designing a library, nothing but an interface to the functions of the library should be exposed to the populous. The rest, should be a black box. 

The problem here, is that should the API undergo some further development, it is a compulsion for the rest of the software to update itself as well, to stay in accordance with the new API. Of course, we can always ignore the new API version, but come on now. For how long?!?!

In comes APPLICATION BINARY INTERFACE. The API ensures that the software shall be compliant with the dependency, in order to serve each other. The ABI, guarantees this compatibility at the MACHINE CODE Level. The limitations, of course, are that some vital and obvious functionalities like dynamic linking, loading an exe, function calling conventions, all should be done using the agreed upon ABI.

An ABI covers the following - 
- Target architecture instruction set, which includes the endianness, memory layout, registers, memory, vector table, etc. 
- Existing data types, their sizes an alignment.
- The function calling convention.
- Defining the `system calls` and their calling in a UNIX-Like system.
- The object-file format that is to be used, for having relocatable, executable, and shared object files.



  