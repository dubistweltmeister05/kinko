[[04 Process Memory Structure]]

Whenever we run an `exe` file, the OS is responsible for creating a new process for it. Every process has a unique *Process ID (PID)* The spawning and loading of the new process is fully handled by the OS. 

The process will keep running till it is given a kill signal, like  `SIGTERM , SIGINT , or SIGKILL` The `SIGTERM` and `SIGINT` signals can be ignored, but `SIGKILL` will kill the process immediately and forcefully.

The first thing that happens when a process is created is the allocation of a bit of dedicated memory for the process by the OS, and then, the application of a predefined memory layout. 

The layout is divided into the following segments - 
- Uninitialized data segment or Block Started by Symbol (BSS) segment
- Data segment
- Text segment or Code segment
- Stack segment
- Heap segment