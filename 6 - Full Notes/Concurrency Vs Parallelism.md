[[Twitter Posting]]

Concurrency is the illusion of parallelism, and it is imperative to understand the difference between them for any CS student. 

Concurrency means the ability to make progress on multiple tasks over a defined time period. Parallelism means the ability to SIMULTANEOUSLY execute multiple tasks. 

On single core systems, like an MCU, you typically observe concurrency, via schedulers that lie at the heart of an RTOS like FreeRTOS or Zephyr RTOS. Within multicore systems, like an Intel/AMD CPU, you have multiple physical cores within the CPU, each capable of executing tasks on their own, leading to true parallelism in it's operation. 