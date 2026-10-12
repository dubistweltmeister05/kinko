[[RhyGen]]

# The Gemini Prompt
Right, I believe that gives you all the context that we need for the following application. I have a PAL that I have written for abstracting the STM32 HAL and the CAN operation that I desire from my system. The transmission pipeline looks like the app simply posts a can message that I have defined into a ringbuffer, and the PAL handles the actual sending of the data. The reception works in such a way that the PAL pushed to a ring buffer and the app simply pops the ringbuffer to get the data. I have tested this via app files that I have written for test purposes, but they were all on bare-metal projects. The test apps simply put the CAN1 peripheral into loopback mode, and posted messages to a UART4 peripheral, that I have hooked to a serial terminal for monitoring the whole thing. 

I have a project need now, where I want to port this operation, and then the test app that I described, to a FreeRTOS project. I have setup the freertos code and have checked if it works via a blinky task on my stm32f446 that I am running. I need you to help me do that following - 

1. Check if there are changes that I need to make to the PAL itself, in order to support it's functionality in Freertos. If yes, I would like a thorough discussion about the proposed architecrtures that we can implement for the same. 
2. Once we clear and finalse the PAL, I want your help in writing the app, and then help me integrate it into the freertos project that I have. 


# Current PAL Vulnerabilities
- **Task-to-Task Race Conditions:** If two FreeRTOS tasks call `stm32_can_write()` simultaneously, they will both attempt to modify the `tx_queue` head pointer. This will corrupt the ringbuffer.
    
- **Task-to-ISR Race Conditions:** In `stm32_can_write()`, you check `if (!impl->is_tx_busy) { trigger_hardware_tx(impl); }`. If the Tx complete interrupt fires exactly after the `if` check but before `trigger_hardware_tx` is called, your software state desynchronizes from the hardware state.
    
- **CPU Starvation:** Your current app likely polls the `read()` function or relies on a bare-metal callback. In FreeRTOS, an application task should block (sleep) until data is ready, freeing up the CPU.

## Proposed FreeRTOS Architectures

We have two primary paths to make this PAL RTOS-ready.

### Architecture 1: The Native FreeRTOS Wrapper (Recommended)

This approach completely replaces your custom `ringbuffer_t` inside the STM32 implementation with native FreeRTOS `QueueHandle_t` objects. The generic PAL interface (`pal_can.h`) remains identical, but the STM32 backend changes.
- **Tx Path:** `stm32_can_write()` uses `xQueueSend()`. If the queue is full, you can specify a timeout (blocking the task instead of dropping the message).
- **Rx Path:** The ISR uses `xQueueSendFromISR()` to push the incoming CAN frame.
- **App Side:** The application task calls `stm32_can_read()`, which wraps `xQueueReceive()`. The task will automatically sleep until the ISR pushes a message into the queue.

### Architecture 2: The Critical Section & Task Notification Approach

This approach keeps your `pal_ringbuff.c` exactly as it is but adds FreeRTOS synchronization primitives around it.
- **Tx Path:** You wrap `stm32_can_write()` in a FreeRTOS Mutex (`xSemaphoreTake`) so only one task can write at a time. You use `taskENTER_CRITICAL()` briefly when interacting with `is_tx_busy` to prevent ISR preemptions.
- **Rx Path:** The ISR pushes to your custom ringbuffer and then calls `vTaskNotifyGiveFromISR()` to wake up the application task.
- **App Side:** The application task blocks using `ulTaskNotifyTake()`. Once awoken, it safely pops from your custom ringbuffer.

### DECISION TIME - 
I am going with the Queues. Reason? 

1 - I wish to have safety as my priority when I am designing the firmware, since this is a commercial project. Using semaphores would mean that I need to ensure that I manually wrap all reads and writes perfectly. Also, should I accidentally "take" a semaphore in an ISR - congratulations I have crashed the RTOS. 

2 - Adding semaphores also decouples the data's movements with the task states (putting inactive ones to sleep or blocking state when they are running), something that Queues handle inherently in FreeRTOS. If a task is preempted when it is writing, this leads to a race condition, when there is data that is available on reception, but the system has no way of knowing that it is, so it just stays there, and gets overwritten by another task that is running. 

As for managing and maintaining the PAL, we will be making a new folder with FreeRTOS suffixes for the PAL files for the FreeRTOS. 

---
## Changes that we implemented

### 1. Replacing Custom Ringbuffers with FreeRTOS Queues

In the bare-metal implementation, message buffering was handled by a custom `ringbuffer_t` struct, which relied on raw memory arrays and manual head/tail index manipulation. In the FreeRTOS port, these were completely replaced by native `QueueHandle_t` objects (`tx_queue` and `rx_queue`).

- **Thread Safety:** The FreeRTOS queue APIs inherently manage their own memory barriers and interrupt masking. This ensures that multiple tasks (or tasks and interrupts) can push and pop CAN messages concurrently without corrupting the buffer state.
- **Blocking Mechanisms:** While the current implementation uses a timeout of `0` to match the non-blocking bare-metal signature, queues natively support blocking. This allows tasks to yield CPU time completely until space frees up (on Tx) or data arrives (on Rx), preventing CPU starvation.
- **Dynamic Allocation:** The bare-metal initialization function required static memory arrays to be passed in. The FreeRTOS port ignores these static arrays and uses `xQueueCreate()` to dynamically allocate the queue memory from the FreeRTOS heap.

### 2. Strategic Use of Critical Sections

_Note: It is important to clarify that critical sections were not added to all write and read paths. The FreeRTOS queues handle their own internal data protection._

Instead, critical section macros (`taskENTER_CRITICAL()` and `taskEXIT_CRITICAL()`) were applied specifically in the `stm32_can_write` function to protect the hardware state flag.

- **The Race Condition Protected:** The `is_tx_busy` boolean tracks whether the STM32 hardware mailbox is currently transmitting. If a task checks this flag, but a hardware Tx interrupt fires immediately after the check (modifying the flag), the software state desynchronizes from the hardware reality.
- **The Implementation:** By wrapping the `if (!impl->is_tx_busy)` check and the subsequent hardware trigger function inside a critical section, we prevent the ISR from preempting the task during this highly sensitive state transition. The read paths do not require critical sections because they purely rely on the natively atomic `xQueueReceive` API.

### 3. Splitting Task and ISR Execution Contexts

FreeRTOS strictly dictates that standard APIs cannot be called from within an Interrupt Service Routine (ISR), as doing so will corrupt the scheduler. To handle this, the hardware interaction logic was split into two separate pipelines.

- **Task Context (`trigger_hardware_tx_from_task`):** Called when an application task posts a message and the hardware is idle. This function uses standard APIs like `xQueueReceive()`.
- **ISR Context (`trigger_hardware_tx_from_isr`):** Called strictly from the `HAL_CAN_TxMailbox...CompleteCallback` functions. This function uses `xQueueReceiveFromISR()`.
- **Deferred Context Switching:** The ISR context functions utilize a `BaseType_t xTaskWoken` flag. If pushing to or popping from a queue inside an ISR wakes up a higher-priority task, the ISR calls `portYIELD_FROM_ISR(xTaskWoken)` at the very end. This ensures the CPU immediately jumps to the most urgent task the moment the interrupt finishes, guaranteeing deterministic real-time behavior.

### 4. Migrating to Thread-Safe Heap Management

Standard C library memory management functions (`malloc` and `free`) are generally not thread-safe and can cause non-deterministic execution times or heap fragmentation in an RTOS environment.

- **`pvPortMalloc`:** In `pal_can_create_stm32`, the standard `malloc()` was replaced with `pvPortMalloc()` to instantiate the `pal_can_stm32_t` object. This utilizes the configured FreeRTOS memory manager (e.g., `heap_4.c`), which guarantees thread-safe memory allocation.
- **`vPortFree`:** Error handling during initialization also utilizes `vPortFree(impl)` to safely return memory to the RTOS heap if queue creation fails.

---
## Using Semaphores vs Mutexes
Semaphores vs Mutexes are a nice debate to have. A semaphore can be thought of as simply a flag, which can be used by tasks in an OS to indicate if something is available or not. A mutex, is something that ensures that the resource that it is locking is also functionally protected and cannot be modified till the mutex is released. 

How so? Functionally, a semaphore consists of a count (typically 0 or 1) along with a list of tasks waiting to take the semaphore. When a task calls take(), the count is decremented if the semaphore is available; otherwise, the task is blocked until another context calls give(). It is kinda dumb, as it has no concept of who the hell called it first. If Task A successfully takes a semaphore, there is nothing preventing Task B from giving it back later if the application is designed that way. The RTOS doesn't consider this an error because semaphores are intended for synchronization and event signalling rather than ownership. This is exactly why they're commonly used for things like waking a task from an ISR after a DMA transfer or a CAN frame reception.

You can see how this is an issue if we are using a semaphore to protect some resource, that is shared between tasks/threads like a piece of memory. In that case, a mutex is the way to go. A mutex, is implemented as a struct that includes the taskHandle of the task that is "taking" the mutex, and this changes the dynamics of locking and unlocking a mutex.. When Task A successfully locks the mutex, it becomes the owner, and only Task A is permitted to unlock it. If another task attempts to release the mutex, the RTOS treats it as an invalid operation.  And since we have identities of tasks now, we can control priorities now. If a high-priority task is blocking the CPU as it is waiting for a mutex held by a lower-priority task (and hence, it cannot run), the kernel can temporarily boost the priority of the mutex owner so it can finish its work and release the mutex sooner. This prevents priority inversion, something that simply isn't possible with a semaphore because there is no owner to boost.

So while semaphores and mutexes are implemented similarly under the hood, their semantics are very different:

Semaphore: Synchronization and event signalling. No ownership. Any context can give it.
Mutex: Mutual exclusion. Ownership is enforced. Only the owning task can unlock it, enabling priority inheritance and making it suitable for protecting shared resources.
That's how an RTOS helps protect critical stuff when dealing with multiple threads - which is always, since if it was a single thread, you wouldn't need an RTOS in the first place!

--- 

## Queue (by Copy) in FreeRTOS

### Queue via reference and copy
The implementation of a Queue in FreeRTOS is interesting to look at - at least architecture sure is an elegant piece of engineering. 

There are 2 ways of implementing a queue - by copy or by reference. While queuing by reference, you "pass" a pointer to the data that you wish to queue, while the data is actually held by a different part of the memory. This ends up making the queue quite lightweight, but a little janky when it comes to thread safety. If, your data is being held in a shared part of memory, and if a dumb programmer forgot to lock or protect it, then there is a very real chance of that data being manipulated, aka overwritten by some other thread. In that case, you can kiss your queue goodbye, because the pointer that it holds, references to corrupted data. 

Queuing by copy, however, offers an advantage in terms of data integrity. If you can afford to make the queue a bit heavy, then queuing by copy GUARANTEES that your data shall be safe. Also, this implementation can take the weight of "owning" the data away from the tasks that are sending and receiving the data. They can simply forget about it once written to a queue, and can receive at their discretion, without having to reserve memory for it in advance, and a lot more hassle that comes with it. 

The downside, is that data can indeed, be "too big" for the system to afford queuing it via a copy.  FreeRTOS implements a Queue by copy, and there's some nice ways that it handles this scenario - after all; in embedded systems, sensor data being "too much" to copy every time is a very real case. In such instances, FreeRTOS simply queues a pointer to the data instead of copying the data itself. The queue still performs a copy - but only of the pointer - making it a far more efficient solution for large chunks of data. The payload can be allocated dynamically at runtime, with the queue merely passing around a reference to it. This preserves the decoupled nature of task communication without paying the cost of repeatedly copying large buffers. The only catch is that the application is now responsible for managing the lifetime of that buffer until the receiver is done with it.

I know this sounds like "FreeRTOS queues via copy when data is small, and via reference when data is big", and I had the same train of thought while reading about it, and even when I am writing this very post, so here's what I leave you with - FreeRTOS queues always copy the queue item. For small payloads, the queue item is often the data itself. For larger payloads, the queue item is typically a pointer to the data, allowing only the pointer to be copied while the payload remains elsewhere in memory.

### Blocking in Queues
There can be multiple tasks that are in the blocked state as they are waiting for data to be available from the queue. In such a scenario, it is mandated that a single task shall be unblocked when data becomes available, and the task that has the highest priority is the one that is unblocked. On the flip side of things, there can be multiple writers to the queue. A queue being full would mean that all writer tasks are pushed to the blocked state. Hence, when the queue opens up again, a single task shall be unblocked, and that would be the one with the highest priority among the ones in the blocked state. 

---
## FreeRTOS APIs that we have implemented
#### `xQueueCreate()`
- **Functionality:** Allocates memory for a new queue structure and its storage area, then returns a handle to manage it.    
- **Inputs:**
    - `uxQueueLength`: The maximum number of items the queue can hold at one time.        
    - `uxItemSize`: The size, in bytes, of each item (e.g., `sizeof(uint32_t)` or `sizeof(MyStruct_t)`).        
- **Return Value:**    
    - `QueueHandle_t`: A handle to the created queue if successful.        
    - `NULL`: If there wasn't enough FreeRTOS heap memory available.        
#### `xQueueSend()`
- **Functionality:** Copies an item onto the back (tail) of the queue. If the queue is full, the task will block for up to `xTicksToWait`.    
- **Inputs:**    
    - `xQueue`: The handle of the target queue.        
    - `pvItemToQueue`: A pointer to the data you want to copy into the queue.        
    - `xTicksToWait`: The maximum time (in ticks) the task should wait in the Blocked state if the queue is full. Setting this to `portMAX_DELAY` blocks indefinitely.        
- **Return Value:**    
    - `pdPASS`: Data was successfully copied to the queue.        
    - `errQUEUE_FULL`: The queue remained full for the duration of `xTicksToWait`.        
#### `xQueueReceive()`
- **Functionality:** Copies an item out from the front (head) of the queue and removes it from the queue. If empty, the task will block for up to `xTicksToWait`.    
- **Inputs:**    
    - `xQueue`: The handle of the target queue.        
    - `pvBuffer`: A pointer to the memory buffer where the copied data will be placed.        
    - `xTicksToWait`: The maximum time (in ticks) the task should wait if the queue is empty.        
- **Return Value:**    
    - `pdPASS`: Data was successfully retrieved.        
    - `errQUEUE_EMPTY`: The queue remained empty for the duration of `xTicksToWait`.        
### 2. Interrupt Service Routine (ISR) Queue APIs

You cannot use standard queue functions inside an interrupt because they can cause the thread to block, which crashes an ISR. Instead, FreeRTOS provides dedicated, non-blocking ISR alternatives.
#### `xQueueSendFromISR()`
- **Functionality:** Copies an item to the back of a queue from inside an interrupt. It never blocks. If the queue is full, it fails instantly.    
- **Inputs:**    
    - `xQueue`: The handle of the target queue.        
    - `pvItemToQueue`: A pointer to the data to be copied.        
    - `pxHigherPriorityTaskWoken`: A pointer to a `BaseType_t` variable. FreeRTOS will set this to `pdTRUE` if sending the data unblocked a task that has a priority higher than the currently interrupted task.        
- **Return Value:**    
    - `pdPASS`: Data was successfully copied.        
    - `errQUEUE_FULL`: Data could not be sent because the queue was full.        
#### `xQueueReceiveFromISR()`
- **Functionality:** Reads and removes an item from the front of a queue from inside an interrupt. It never blocks.    
- **Inputs:**    
    - `xQueue`: The handle of the target queue.        
    - `pvBuffer`: A pointer to the memory buffer where the data will be copied.        
    - `pxHigherPriorityTaskWoken`: A pointer to a `BaseType_t` variable. FreeRTOS will set this to `pdTRUE` if removing the data unblocked a higher-priority task that was waiting to send data.        
- **Return Value:**    
    - `pdPASS`: Data was successfully retrieved.        
    - `pdFAIL`: The queue was empty.        

### 3. Context Switching

#### `portYIELD_FROM_ISR()`
- **Functionality:** Requests a context switch from inside an ISR. You pass the variable modified by `xQueueSendFromISR` or `xQueueReceiveFromISR` into this macro. If that variable is `pdTRUE`, the macro forces the CPU to switch directly to the newly unblocked, higher-priority task the moment the interrupt exits, rather than waiting for the next regular scheduler tick.    
- **Inputs:**    
    - `xHigherPriorityTaskWoken`: The standard `BaseType_t` variable that was evaluated by your ISR queue function.        
- **Return Value:** None.    

### 4. Critical Sections

Critical sections protect short, sensitive pieces of code by temporarily disabling interrupts. This prevents another task or an interrupt from preempting the current execution context midway through a hardware write or variable modification.
#### `taskENTER_CRITICAL()`
- **Functionality:** Disables interrupts up to the kernel's defined maximum syscall interrupt priority. It utilizes a nesting count, meaning you can call it multiple times safely, provided every "enter" call is paired with an "exit" call.    
- **Inputs / Return Value:** None.    
#### `taskEXIT_CRITICAL()`
- **Functionality:** Decrements the nesting count. If the nesting count reaches zero, it re-enables the interrupts that were disabled by `taskENTER_CRITICAL()`.    
- **Inputs / Return Value:** None.    

### 5. Queue Status

#### `uxQueueMessagesWaiting()`
- **Functionality:** Checks how many items are currently sitting in a queue. **Call this from regular tasks only.**    
- **Inputs:**    
    - `xQueue`: The handle of the queue being queried.        
    - **Return Value:** `UBaseType_t` representing the number of items currently in the queue.        

#### `uxQueueMessagesWaitingFromISR()`
- **Functionality:** The ISR-safe version of `uxQueueMessagesWaiting()`. It provides the exact same status check without risking internal scheduler corruption when called from an interrupt context.    
- **Inputs:**    
    - `xQueue`: The handle of the queue being queried.        
- **Return Value:** `UBaseType_t` representing the number of items currently in the queue.    

### 6. Memory Management

These are the FreeRTOS heap management wrappers. They replace standard C library dynamic memory functions (`malloc`/`free`) to ensure deterministic execution times and thread safety within the RTOS.
#### `pvPortMalloc()`
- **Functionality:** Allocates a block of memory from the FreeRTOS heap. The exact allocation strategy (avoiding fragmentation, speed, etc.) depends on which `heap_x.c` file you have compiled into your project.    
- **Inputs:**    
    - `xWantedSize`: The size of the memory block you want to allocate, in bytes.        
- **Return Value:**    
    - `void*`: A pointer to the allocated memory block.        
    - `NULL`: If there was insufficient heap space available.        

#### `vPortFree()`
- **Functionality:** Returns a previously allocated memory block back to the FreeRTOS heap so it can be reused.    
- **Inputs:**    
    - `pv`: A pointer to the memory block that needs to be freed (must have been allocated via `pvPortMalloc()`).        
- **Return Value:** None.
---
