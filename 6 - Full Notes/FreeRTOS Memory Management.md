[[FreeRTOS]]

# Static Vs Dynamic Memory Allocation

Dynamic alloc is the simplest and most hassle free way of doing this. The API is simpler, there is less design and planning and lesser footprint of the final executable that is loaded onto the RAM. Static on the other hand, is a lot more determinism about it, and ensures that there is no fragmentation in the heap. For those unaware, fragmentation of memory means that there is memory available, but not in a contiguous manner, making things available when diagnostics ask about it, but not when the system actually needs it. 

There is a glaring lack of uniformity when it comes to the standard C functions of allocating and freeing memory, such as malloc(), calloc(), and free(). What, you really thought that they are same regardless of the architecture and toolchain that you are implementing it on? LoL. Also, they have no way of knowing about fragmentation of the heap, so that's another major issue about these. So, OS Like FreeRTOS said - "Fine. I'll do it by myself", and started treating memory management as a feature, instead of leaving it to the programmer. They have implemented 5 ways of managing the heap, creatively naming them heap_1.c,heap_2.c,heap_3.c,heap_4.c, and heap_5.c!

There are also special alloc and free functions in FreeRTOS, called pvPortMalloc(), vPortFree() and others, that primarily exist and are used for offering thread safe implementations of these memory functions. 

---
We'll take a look at the very creatively named 5 implementations of heap memory management in this now - 
## heap_1.c
This is tailored to those who do not need a lot of compute to be honest. Lemme explain. Applications that can declare their tasks and primitives before the Kernel starts are invited to use this implementation. heap_1 implements a version of pvPortMalloc(), which divides a simple uint8_t array called the FreeRTOS heap into the size of how many bytes have been requested every time that it's been called. Each call invokes 2 pvPortMalloc() calls, one for the TCB of the task, and the second for the task stack of the caller. 

This makes the FreeRTOS code seem to consume a lot of RAM, since the arrays are statically allocated at compile time, but this ensures determinism of memory and non-fragmented memory at runtime. 

## heap_2.c
This implements vPortFree(), and does a simillar division of the massive FreeRTOS Heap array as heap_1 does. This one uses a best-fit algorithm, and this ensures that the freed memory is always used again when called upon. For example. Suppose there's 3 blocks of memory that'as freed, each of 5, 25, and 100 bytes. 

Say that there's been a call for 20 bytes of the memory. The algo shall split the 25 byte block into 20 and 5, before returning a pointer to the 20 byte block, and marking the 5 byte block as available for future pvPortMalloc() calls (kinda inaccurate, but shall do for now).

## heap_3.c 
LoL, this literally uses malloc() and free() from the stdlib that you and I are all too familiar with (you know how these things work, right????). I guess the makers of this were lazy bastards, who literally suspend the damn scheduler when these functions are called for memory allocation and freeing. Proper madlads, I tell you!

## heap_4.c
This is heap_2.c but better. Same principles so far, divide a large uint8_t array into smaller parts when pvPortMalloc() is called, and collect the freed memory back. But here, we make it available via the first-fit algorithm. This ensures that pvPortMalloc() uses the very first block of memory that is big enough to hold the requested memory.  This implementation combines adjacent blocks of memory that are freed into a single block of memory that is available to be allocated, minimizing the risk of fragmentation. 

## heap_5.c
This improves upon the heap_4's implementation (a recurring theme, isn't it) , by being able to combine multiple memory spaces into a single heap. I'll be dead honest with you - even I am lost at this point. How the hell is this even possible? I'll get around to that in a later post I suppose. 

---
So, which creative `heap_X.c` file should you actually pick for your project?

If your system is hyper-critical, runs fixed tasks, and needs 100% determinism without the risk of memory leaks, **`heap_1.c`** (or raw static allocation) is your best friend. If you’re building a standard IoT device or embedded app that constantly creates and deletes queues, tasks, or buffers on a single RAM bank, **`heap_4.c`** is the gold standard. And if your hardware designer gave you a complex microcontroller with RAM scattered across multiple memory banks, **`heap_5.c`** saves you from going insane.

FreeRTOS taking memory management into its own hands isn't just control-freak behavior - it’s the difference between a system that runs forever and one that silently crashes 3 weeks into deployment because of a fragmented heap.