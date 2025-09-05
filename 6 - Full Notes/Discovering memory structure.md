[[04 Process Memory Structure]]

We know that the executable and the process are two different things all together. Naturally, there have to be different tools that are used to work and inspect them. 

The executable file, contains the machine instructions that govern what is to be done by the processor. But a process, is a running program that is spawned BECAUSE of an executable, consumes some memory, and is constantly interacting with the CPU via fetch and execute instructions. 

A process is a living entity that is being executed inside the operating system
while the executable object file is just a file containing a pre-made initial layout
acting as a basis for spawning future processes. 

Now, there are 2 major memory layouts that are created. One - the *static memory layout* which comes from the executable file that was created. Two - the *dynamic memory layout* which comes from the program that was spawned. 

Static and dynamic memory layouts both have a predetermined set of segments. The
content of the static memory layout is prewritten into the executable object file by
the compiler, when compiling the source code. On the other hand, the content of the
dynamic memory layout is written by the process instructions allocating memory
for variables and arrays, and modifying them according to the program's logic.