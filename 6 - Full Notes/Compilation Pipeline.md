2025-05-26

Tags: [[02 From Source To Binary]]

There are 4 stages of compiling a program - 
- Preprocessor
- Compiler
- Assembler
- Linker

For single files, the first 3 stages are enough to get an executable that is essentially an output. 

### Platform
This means the combination of a processor, an OS , and it's instruction set.

There is a difference in *cross_platform* and *portable* software. A cross platform software had different installers and binaries on different platforms, while portable software has the same set of executable and binaries everywhere.

# Building a C Project

We know that there are 2 types of Files in a C project - a header and a source.

The header contains the enumerations, macros, and typedefs, as well as the declarations of functions, global variables, and structures. In C, some programming elements such as functions, variables, and structures can have their declaration separated from their definition placed in different files.

A rule of thumb - declarations in the header and definitions in the source. 

Headers can include other headers, but NEVER a source file. Source files can only have header files in their includes. 

**declaration** - A function declaration consists of a return type and a function signature. A function signature is simply the name of the function together with the list of its input parameters
```
double average(int *, int);
```

**definition** - A function definition contains the actual implementation of the function logic and the code to execute it. 
```
double average(int* array, int length) {
if (length <= 0) {
return 0;
}
double sum = 0.0;
for (int i = 0; i < length; i++) {
sum += array[i];
}
return sum / length;
}
```


There are only two rules when it comes to building the project - 
 - We build only the source files
 - We build every file separately
























































# Reference

