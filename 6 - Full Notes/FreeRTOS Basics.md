[[RhyGen]]
[[FreeRTOS]]


## Concurrency and Parallelism
Concurrency is the illusion of parallelism, and it is imperative to understand the difference between them for any CS student. 

Concurrency means the ability to make progress on multiple tasks over a defined time period. Parallelism means the ability to SIMULTANEOUSLY execute multiple tasks. 

On single core systems, like an MCU, you typically observe concurrency, via schedulers that lie at the heart of an RTOS like FreeRTOS or Zephyr RTOS. Within multicore systems, like an Intel/AMD CPU, you have multiple physical cores within the CPU, each capable of executing tasks on their own, leading to true parallelism in it's operation. 


## Scheduling
The scheduler is the one responsible for the swapping out the active tasks with the passive ones. The Scheduling Policy is the algo that handles the active running task. The swapping happens only when the algo determines that there is a different task the needs to be executed on the CPU. The task being swapped out NEVER knows that it is being swapped, such as when the scheduling algorithm responds to an external event or timer expiration. It can also happen if the executing task explicitly calls an API function that results in it **yielding**, **sleeping** (also called **delaying**), or **blocking**. 

If a task yields, the scheduling algorithm could select the same task to execute again. If a task sleeps, it becomes unavailable for selection until the specified delay period elapses. Similarly, if a task blocks, it becomes unavailable for selection until either a specific event occurs (e.g., data arrives on a UART) or a timeout period expires.

## Real Time Scheduling
There is a single difference between this and the scheduling that we say earlier - determinism. Basically, we ensure that a task WILL BE SWAPPED OUT IF DEEMED NECESSARY FOR IT TO BE OUTTA HERE. No matter what the hell is it executing at any given point, it WILL be out of the processor if there is a condition in the real world that needs to be addressed ASAP by the system. Events occurring in the real world can have deadlines before which the real-time embedded system must respond and the RTOS scheduling policy must ensure these deadlines are met.

Now, the onus of ensuring that the correct response is reflected on the level of criticality that is carried by the real world event lies with the programmer. You see, you need to have SKILL in order to make and RTOS function as you intent it to. How? By assigning priorities to the tasks that you are writing, the ones that carry out the response that you intend. The priority is what tells the OS that it needs to ensure that your response is executed ASAP. When a task with a lower priority number (meaning a higher value of priority) enters the scheduling queue and is available to be executed, the scheduler simply switches context from the currently running process to the process with a higher value of priority, and starts executing it. 

Do note, that the process of switching tasks/processes is deterministic and  constant, meaning guaranteed switching within the specified time frame. 