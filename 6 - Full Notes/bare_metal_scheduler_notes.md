[[PICT_EMBEDDED]]
# 🧵 Bare-Metal Scheduler (STM32) — Lecture Notes

---

## 📘 1. Big Picture: What Are We Building?

- Preemptive scheduler using **SysTick**
- Multiple tasks (`task1 → task4`) running “concurrently”
- Uses **context switching + PSP (Process Stack Pointer)**
- Scheduling policy → **Round Robin**

**Core Idea:**
> Interrupt fires → save current task → switch → restore next task

---

## ⚙️ 2. Tasks & Infinite Loops

- Each task is a function with `while(1)`
- No OS → tasks never return

```c
void task1_handler(void) {
    while (1) {
        printf("Task 1\n");
    }
}
```

**Key Concept:**
> Scheduler forces switching, not tasks themselves

---

## 🧠 3. Stack Management (MSP vs PSP)

- **MSP (Main Stack Pointer)** → used by scheduler / interrupts
- **PSP (Process Stack Pointer)** → used by individual tasks

**Why?**
- Isolation between tasks
- Prevent stack corruption

```c
switch_to_psp();
```

**Insight:**
> Each task thinks it owns the CPU

---

## 📦 4. Task Stack Initialization

- Done in `init_task_stack()`
- Manually builds a **fake stack frame**

Includes:
- xPSR
- PC (task handler)
- LR
- R0–R12

**Key Idea:**
> We mimic what hardware expects after an exception return

---

## ⏱️ 5. SysTick Timer (Scheduler Heartbeat)

```c
init_systick_timer(timer_freq);
```

- Generates periodic interrupts
- Drives the scheduler

**Key Concept:**
> Preemption happens only on SysTick interrupt

---

## 🔄 6. Context Switching (Core of Scheduler)

### In `SysTick_Handler`

### 1. Save Current Task
- Read PSP
- Store R4–R11
- Save PSP to array

### 2. Select Next Task
```c
current_task = (current_task + 1) % MAX_TASKS;
```

### 3. Restore Next Task
- Load PSP
- Restore registers
- Resume execution

**Important:**
> This is the heart of the scheduler

---

## 🔁 7. Round Robin Scheduling

- Task0 → Task1 → Task2 → Task3 → repeat
- Equal CPU time for all tasks
- No priority handling

**Insight:**
> Fair but not smart scheduling

---

## 🚨 8. Fault Handling (Debug Safety)

Enabled faults:
- MemManage
- BusFault
- UsageFault

Handlers:
- Print error
- Halt execution

**Why important?**
- Helps catch invalid memory access
- Critical for debugging bare-metal systems

---

## 🎯 9. Execution Flow

1. System boots → `main()`
2. Scheduler stack initialized
3. Task stacks prepared
4. SysTick configured
5. Switch to PSP
6. First task starts
7. SysTick interrupt fires
8. Context switch happens
9. Repeat forever

---

## 💬 Closing Note

> This is essentially a tiny RTOS built from scratch — no libraries, just hardware and stack manipulation.
