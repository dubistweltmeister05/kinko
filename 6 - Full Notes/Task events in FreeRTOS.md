[[FreeRTOS]]
[[Twitter Posting]]
[[Blog Topics]]

Y'all gon learn some today. 

In an OS, there are task states that are governed by an FSM, with each task transitioning to and from these states. The FSM usually maintains the states of a task as running, blocked, suspended, and ready. 

Let's take a look at their meaning, and the way these transitions occur. I am well aware that this is quite rudimentary, trivial even for some - but not a lot are into OS to begin with, so lemme make it a bit easier to make sense of!

Le Running State - Exactly what you'd think of it to be. A task that is actively being executed on the processor is said to be in the running state. 

Le Blocked State - This is where it gets interesting. A blocked state is said to be invoked for a task when it is not being executed on the processor. But, not because it chose not to be executed, but because it's waiting for an EVENT to happen. The event can be temporal, or synchronization related in nature, and I believe a brief about these is in order. 

A temporal event is something that's related to time. In the context of an OS, it occurs when a delay period expires and leads to a timeout, or something like that.  

A synchronization event is something that originates from another task or an interrupt. It can be a semaphore, queue, mutex, or a message buffer that's yet to basically do it's job, because of which the task we are concerned about is not in a position to start executing. 

Le Suspended State - This is kind like when a task is put in timeout by a function called vTaskSuspend() (in the context of FreeRTOS). It means that the scheduler will not be able to schedule it even if it became sentient, and like, really wants to (lol). This is rarely used in most applications.

Le Ready State - Here, everything that is needed fro a task to run, is ready for it to do so. All it needs now, is some real-estate on the processor and it can start executing. There might be a bunch of tasks that are in the ready state in an application, and the OS shall schedule them according to their priority, and the scheduling algorithm that's at it's heart.

Do take a look at the FreeRTOS tasks FSM down below - 