[[PICT_EMBEDDED]]

# Embedded Scheduler Teaching Notes

## 1. Big Picture
•Preemptive Round Robin Scheduler  
This means 2 things.

•1. That the tasks are scheduled to run in a circular, continuous loop.

•2. That no matter the state of the a task, it WILL stop executing and the next-in-line task WILL start executing.

•SysTick → time base.  
This is a built-in 24-bit countdown timer included in the ARM Cortex-M processor core, primarily used to generate regular, periodic interrupts for operating systems (RTOS) to handle task switching and timing

•PendSV → context switching  
This is an exception that is of pretty low priority, that is used to change the current task that is running on the MCU. This is called from the SystickHandler.

•Goal: Run multiple tasks on single CPU

---

## 2. Core Building Blocks
•Task Control Block (TCB)  
The TCB is a master structure that holds critical info about each of the tasks that we intend to run on the MCU. It’s members are 
	•PSP (stack pointer) – this holds the value of the current context of each task.
	•State (READY / BLOCKED) – this holds the current state off the task, if it has been preempted(in that case -blocked) or is it ready to be executed.
	•Delay info – The time for which it has been blocked.
	•Task handler pointer – a pointer to the actual code that a task is supposed to run

•Global Variables
		•current_task – the task number which is currently being executed.
		•g_tick_count – the variable that is incremented once every time the systick handler is executed. In other words, it holds the number of times that the systick timer has been called upon.

---

## 3. Stack Architecture
•MSP (Main Stack Pointer) vs PSP (Process Stack Pointer) In this scheduler, we deliberately **split responsibilities** between two stacks.

•MSP → used by the scheduler and all exception handlers (SysTick, PendSV, faults)  
This ensures that all “OS-level” work happens in a **safe, isolated stack space**.

•PSP → used by individual tasks  
Each task gets its **own private stack**, and its execution context is preserved here.

In your code:

- `init_scheduler_stack()` → sets MSP
- `switch_sp_to_psp()` → switches CPU to PSP before starting task execution

Why this matters:

- Prevents tasks from corrupting scheduler state
- Makes context switching possible and clean

---

## 4. Task Initialization
Before any task runs, we **manually construct what its stack should look like** as if it was interrupted earlier.

In your scheduler:

- `init_tasks_stack()` prepares each task’s stack

What you push:

- xPSR → default state (`0x01000000`)
- PC → task handler address
- LR → `0xFFFFFFFD` (return using PSP)
- R0–R12 → dummy values

 Why this works:  
When the CPU “restores” this stack during a context switch, it believes:

> “Oh, I was already running this task”

So it directly jumps into the task function.

This is the **illusion that bootstraps multitasking**

---

## 5. SysTick (Heartbeat)
•SysTick → periodic interrupt generator

In your scheduler, SysTick acts as the **time engine**.

Every tick:

- `g_tick_count++` → global time progresses
- `unblock_tasks()` → checks if any blocked task should wake up
- Triggers PendSV → requests context switch

Important nuance:

- SysTick does **NOT switch tasks directly**
- It only says: _“Hey, scheduling is needed”_

That keeps SysTick:
- Fast
- Predictable
- Interrupt-friendly
---

## 6. Task States & Delay
•Two states in your design:  
•READY → eligible to run  
•BLOCKED → waiting for time to pass
In your scheduler:
- `task_delay(ticks)`:
    - Calculates wake-up time → `g_tick_count + ticks`
    - Moves task → BLOCKED
    - Calls `schedule()` (forces rescheduling)
Then:
- `unblock_tasks()`:
    - Runs every SysTick
    - Moves tasks from BLOCKED → READY when time matches
Insight:  
This is a **cooperative timing system built on top of a preemptive scheduler**

---
## 7. Scheduler Logic

•Function: `update_next_task()`

What it does:
- Cycles through tasks in circular fashion
- Picks the **next READY task**
- Skips blocked tasks
- Falls back to **idle task (task 0)** if nothing is ready
Important behavior:
- No priorities → purely rotational
- Fair but not optimized
This is the **decision-making part of your scheduler**

---

## 8. Context Switching (PendSV)
 Step 1: Save current task
- Read PSP
- Push R4–R11 onto stack
- Store updated PSP in TCB
Why only R4–R11?
- Because hardware already saved:
    - R0–R3, R12, LR, PC, xPSR
---

Step 2: Select next task
- Call `update_next_task()`
---
Step 3: Restore next task
- Load PSP from next task’s TCB
- Pop R4–R11
- Update PSP register
---
Step 4: Exit handler
- `BX LR`
- CPU restores remaining registers automatically
- Execution resumes in **new task**
---
## 9. Why PendSV?
- Lowest priority interrupt
- Safe for context switching

---

## 10. Idle Task
- Runs when no task is READY
- Prevents CPU idle issues

---

## 11. Critical Sections
- Disable interrupts during shared updates
- Prevent race conditions

---
## 12. Key Takeaways
- OS = Data + Timer + Context Switching
- Multitasking = fast switching
- Hardware enables efficient scheduling

---

## Quote
> “The real magic isn’t multitasking — it’s context preservation.”

Links
https://aticleworld.com/msp-vs-psp-arm-cortex-m/