[[Blog Topics]]
[[FreeRTOS]]
[[Twitter Posting]]

So, we are back with out notes on Operating systems. I make these as I read along the Mastering FreeRTOS manual, and I find them a nice way to document things that I learn. This is pretty simple in FreeRTOS, as the entire scheduling policy is defined by a couple of Macros that we have to define in the config header. Lemme explain those real quick. You have something called configUSE_PEEMPTION, and  configUSE_TIME_SLICING. For the geniuses among you who haven't already guessed, they are used to define  the use of preemption of tasks, and the use of time slices when it comes to scheduling tasks in a FreeRTOS application. 

Alright, I'll stop being elitist now and get to explaining. Before we dive into our discussion of scheduling policies, let's take a look at the terms being used.
- Fixed Priority
  The means that the algorithm that is being used does NOT tamper with the priorities of the tasks that it is scheduling. Now, an important caveat is that the task itself is allowed to do so within it's own code, but the SCHEDULER itself shall not tamper with it. 

- Preemption
  This means that if a task of higher criticality (priority) enters the ready state, the scheduler  shall be able to throw out any task of lower criticality (priority) from the processor and start executing the higher priority task on the processor. 

- Time Slicing
  This means that if there's 2 tasks of EQUAL priority that are at the ready queue, the right to be processed by the CPU shall be shared between them. The scheduler shall select a new task that has entered the ready state, provided that the task shares an equal priority with the task in the running state. 

So - what are the possible scheduling algorithms that can be defined in FreeRTOS? You can use either - 
1. PREEMPTION WITH TIME SLICING
2. PREEMPTION WITHOUT TIME SLICING
3. CO-OPERATIVE - Which basically means that the tasks are switched only when the running task leaves the processor and enters a blocked state. Even if a task of higher priority enters the ready state, it cannot be executed on the processor till the currently executing task VOLUNTARILY gives up the processor! How fun is that!
   
Right, that's all for this note. I'll maybe elaborate on these a bit more, and their practical uses in the next note. Thank you for reading so far!