[[Twitter Posting]]
[[FreeRTOS]]

### heap_1.c

`heap_1` is the simplest and most deterministic memory allocator. It supports **memory allocation only** and does **not** implement `vPortFree()`, meaning once memory is allocated it can never be returned to the heap. Internally, it simply advances a pointer through a statically allocated heap buffer (`ucHeap`) until all memory is exhausted. Since no bookkeeping for deallocation or fragmentation is required, it has extremely low overhead and constant execution time. This implementation is ideal for deeply embedded systems where all dynamic objects (tasks, queues, semaphores, timers, etc.) are created once during initialization and remain allocated for the lifetime of the application.

---

### heap_2.c

`heap_2` extends `heap_1` by supporting both allocation and deallocation. It maintains a linked list of free memory blocks and searches for a suitably sized block when `pvPortMalloc()` is called. However, it **does not merge adjacent free blocks** after memory is freed, making it susceptible to external fragmentation over time. Although adequate for applications that allocate and free similarly sized objects infrequently, it is generally discouraged for systems with varying allocation sizes because fragmentation can eventually prevent large allocations despite sufficient total free memory.

---

### heap_3.c

`heap_3` does not implement its own allocator at all. Instead, it wraps the standard C library's `malloc()` and `free()` functions, allowing FreeRTOS to use the heap management already provided by the compiler's runtime library. This makes integration simple and enables coexistence with middleware that also uses standard dynamic allocation. However, execution time, fragmentation behavior, memory usage, and thread safety depend entirely on the underlying C library implementation. Since many embedded `malloc()` implementations are not deterministic, `heap_3` is generally unsuitable for hard real-time systems unless the runtime allocator is specifically designed for real-time use.

---

### heap_4.c

`heap_4` is the most widely used FreeRTOS heap implementation because it balances performance, flexibility, and memory efficiency. Like `heap_2`, it maintains a linked list of free blocks, but it additionally **coalesces adjacent free blocks** whenever memory is released. This significantly reduces external fragmentation and allows long-running applications with frequent allocations and deallocations to continue operating efficiently. Allocation uses a first-fit search strategy, and metadata overhead remains relatively small. For most embedded applications requiring dynamic memory throughout runtime, `heap_4` is considered the recommended default implementation.

---

### heap_5.c

`heap_5` builds upon the functionality of `heap_4` by allowing the heap to be divided across **multiple non-contiguous memory regions**. During initialization, the application supplies an array of memory regions using `vPortDefineHeapRegions()`, after which the allocator treats them as a single logical heap while still performing block coalescing within each region. This makes `heap_5` particularly valuable on microcontrollers with fragmented RAM architectures, such as separate SRAM banks, tightly coupled memory (TCM), external SRAM, or SDRAM. It provides the same fragmentation resistance as `heap_4` while offering the flexibility to utilize all available memory, even when it is not physically contiguous.