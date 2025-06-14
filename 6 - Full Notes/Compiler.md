2025-05-26

Tags: [[02 From Source To Binary]]


In this section, we are going to have a deeper look at compilers. We will explain how compilers, driving the compilation step, produce intermediate representations from source code and then translate them into assembly language.

Once we have the translation unit, we move to this step. The output here is the ASSEMBLY code for the translation unit. Basically, all the stuff that we wrote is not converted to assembly. Go figure.

Use the "-S" flag to see what comes out of this. 

The assembly is PLATFORM SPECIFIC. It is compiles in the assembly that is going to interact with the target hardware.

Keep in mind, that the BUILD ARCHITECTURE (the machine where you are writing the code, essentially) and the TARGET ARCHITECTURE (the machine that will run the fucking code) can be different. Say hello, to CROSS_COMPILATION.


# Reference

