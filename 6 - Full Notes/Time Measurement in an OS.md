[[FreeRTOS]]
[[Twitter Posting]]
[[Blog Topics]]

The concept of a time slice is imperative to an Operating system, and even more so when it comes to a real-time OS. For the uninitiated, a time slice to an OS, is what seconds are to humans. They are the smallest unit of measurement for the task state machine to base it's operation off. In a more technical sense, , a time slice, or scheduling quantum, is the amount of processor time a task may be allowed to consume before the scheduler gets an opportunity to schedule another task. In a real-time operating system, however, this should not be confused with the OS's fundamental unit of timekeeping.

You see, while we set up an operating system, we usually initialize a timer of the processor as the heartbeat of the OS. We configure the timer to raise an interrupt as soon as it is done counting (up or down, doesn't really matter), and each time the interrupt is generated, we consider that to be a tick event. We are very assured about the interrupt being raised at a precisely defined interval. Setting the frequency of the timer's interrupt, enables us to control the length of a time slice! For example, setting up the timer to raise an interrupt at 100Hz, shall help us run the system at a time slice of 10 ms.

To me, this is an inherently beautiful concept. That the very fact of "Software" basing it's operation on the inherent determinism that is promised by "Hardware" is indeed quite poetic. This is also why the timer can be thought of as the heartbeat of the operating system, as every beat advances the kernel's notion of time and gives the scheduler an opportunity to make a decision.

---
gippity

The concept of a time slice is fundamental to an operating system, particularly when it comes to preemptive multitasking. For the uninitiated, you can think of the OS's notion of time somewhat like how humans perceive seconds: it provides a common unit against which the scheduler can make decisions. More technically, a time slice, or scheduling quantum, is the amount of processor time a task may be allowed to consume before the scheduler gets an opportunity to schedule another task. In a real-time operating system, however, this should not be confused with the OS's fundamental unit of timekeeping.

To keep track of time in a deterministic and predictable manner, operating systems commonly use the concept of a tick interrupt.

When initializing an operating system, we can configure one of the processor's hardware timers to act as the heartbeat of the OS. The timer is configured to periodically generate an interrupt. It doesn't really matter whether the hardware timer counts upward or downward; what matters is that it generates an interrupt at a precisely defined interval. Every time that interrupt occurs, the OS considers it a tick event. For example, if we configure the timer to generate an interrupt at 100 Hz, we get:

1 tick = 1 / 100 Hz = 10 ms

This gives the RTOS a 10 ms resolution for its kernel timekeeping. The scheduler can use these tick events to manage task delays, timeouts, periodic tasks, and — depending on the scheduling policy — preempt or rotate between tasks. This is why the timer can be thought of as the heartbeat of the operating system: every beat advances the kernel's notion of time and gives the scheduler an opportunity to make a decision.
