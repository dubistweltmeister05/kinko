[[FreeRTOS]]
[[Blog Topics]]
[[Twitter Posting]]
# FreeRTOS Software Timer Weekend Lab
## STM32F407 + LEDs + User Button + UART

> **Goal:** Build a small "Timer Lab" application that deliberately exercises FreeRTOS software timers, the Timer Service/Daemon Task, timer callbacks, timer IDs, one-shot vs auto-reload timers, starting/stopping/resetting timers, changing periods, and interaction between ISRs/tasks and timers.

---

# 1. Project idea: "FreeRTOS Timer Lab"

Instead of building one large application, build a tiny interactive laboratory on the STM32F407.

The board behaves like a **Timer Console**:

- The **4 LEDs** visualize timer activity.
- The **user button** controls timers and generates events.
- **UART/printf** acts as the debug console.
- Several software timers run at different periods.
- The main task periodically prints a status snapshot.
- Button presses change timer state.
- Timer callbacks identify which timer fired using **timer IDs**.
- Eventually, deliberately stress the Timer Service Task and observe what happens.

The important idea is that **the timer callback is not its own task**.

All software timer callbacks execute in the context of the FreeRTOS **Timer Service Task (Timer Daemon Task)**. This makes the project particularly useful for understanding what that task is doing and why callback design matters.

---

# 2. What you should learn

By the end of the weekend, you should be comfortable with:

- Creating software timers.
- One-shot vs auto-reload timers.
- Starting and stopping timers.
- Resetting a timer.
- Changing a timer period.
- Querying timer state.
- Querying/changing the timer period.
- Using `pvTimerGetTimerID()` / `vTimerSetTimerID()`.
- Understanding callback execution context.
- Understanding the Timer Service Task.
- Understanding the timer command queue.
- Understanding timer expiry vs callback execution.
- Starting timers from a task.
- Starting/resetting timers from an ISR using the `FromISR` APIs.
- Understanding why blocking inside a timer callback is dangerous.
- Observing what happens when callbacks take too long.
- Using timer callbacks to notify/tasks or queues rather than doing substantial work directly.
- Reasoning about timer resolution and `configTICK_RATE_HZ`.

---

# 3. Hardware/software assumptions

You already have:

- STM32F407
- FreeRTOS running
- Four LEDs working
- User button working or accessible through your BSP/HAL
- `printf()` redirected to UART

You do **not** need additional hardware.

Suggested logical LED assignment:

| LED | Meaning |
|---|---|
| LED 1 | Heartbeat timer |
| LED 2 | Fast periodic timer |
| LED 3 | Button-triggered one-shot timer |
| LED 4 | Timer/daemon stress or diagnostic indicator |

The exact GPIO pins are irrelevant to the exercise. Hide them behind functions such as:

```c
LED1_On();
LED1_Off();
LED1_Toggle();

LED2_On();
LED2_Off();
LED2_Toggle();

...
```

---

# 4. Architecture

Start with this mental model:

```text
                    +----------------------+
                    |      Your Tasks      |
                    |                      |
                    |  Console / Monitor   |
                    |  Button processing   |
                    +----------+-----------+
                               |
                               | timer API calls
                               v
                    +----------------------+
                    | Timer Command Queue  |
                    +----------+-----------+
                               |
                               v
                 +----------------------------+
                 | Timer Service / Daemon Task|
                 |                            |
                 | - waits for timer expiry   |
                 | - processes timer commands |
                 | - calls callbacks          |
                 +-------------+--------------+
                               |
                               v
                    +----------------------+
                    |   Timer Callbacks    |
                    |                      |
                    | Timer A callback     |
                    | Timer B callback     |
                    | Timer C callback     |
                    +----------------------+

UART <------------------------------------+
LEDs  <-----------------------------------+
Button --> ISR/task --> timer APIs ------>+
```

A critical observation:

> **A software timer does not execute a callback at the moment the timer expires in some independent context. The Timer Service Task wakes up and executes the callback.**

That distinction should become obvious through this project.

---

# 5. The timers

Create the following timers.

## Timer A — Heartbeat

**Type:** Auto-reload  
**Period:** 1000 ms

Every second:

- Toggle LED 1.
- Print:

```text
[TIMER] heartbeat fired
```

Purpose:

- Basic auto-reload timer.
- Basic callback.
- Observe periodic behavior.

---

## Timer B — Fast timer

**Type:** Auto-reload  
**Period:** 250 ms

Every 250 ms:

- Toggle LED 2.
- Increment a counter.

Every 4th callback, print something like:

```text
[TIMER] fast timer: 4 callbacks
```

Don't print on every 250 ms callback forever. This gives you a chance to think about the cost of logging from a timer callback.

Purpose:

- Short-period timers.
- Callback frequency.
- Timer tick resolution.
- Callback workload.

---

## Timer C — Button timeout

**Type:** One-shot  
**Period:** 3000 ms

When the user presses the button:

1. Start/reset Timer C.
2. Turn LED 3 ON.
3. Print:

```text
[BUTTON] Timer C armed for 3 seconds
```

When Timer C expires:

1. Turn LED 3 OFF.
2. Print:

```text
[TIMER] Button timeout expired
```

Purpose:

- One-shot timers.
- `xTimerStart()`.
- `xTimerReset()`.
- Re-arming a timer.
- Timer IDs.

A nice experiment:

> Press the button repeatedly before the 3 seconds expire.

The LED should remain ON while the timer keeps being reset.

This is essentially a software implementation of a **debounce/timeout/inactivity timer**.

---

# 6. Timer IDs

Do NOT initially create separate callback functions like:

```c
heartbeat_callback()
fast_callback()
button_callback()
```

Instead, experiment with a common callback:

```c
static void timer_callback(TimerHandle_t xTimer)
{
    void *id = pvTimerGetTimerID(xTimer);

    ...
}
```

Assign a different ID to every timer.

For example:

```c
#define TIMER_ID_HEARTBEAT   ((void *)1)
#define TIMER_ID_FAST        ((void *)2)
#define TIMER_ID_BUTTON      ((void *)3)
```

Or, preferably, use an enum/structure rather than relying on magic pointer values.

For example:

```c
typedef enum
{
    TIMER_HEARTBEAT,
    TIMER_FAST,
    TIMER_BUTTON
} timer_id_t;
```

Then investigate how you can associate the enum with the timer ID.

Your callback can then do:

```text
callback()
    |
    +--> get timer ID
    |
    +--> identify timer
    |
    +--> perform timer-specific action
```

The important API to explore is:

```c
pvTimerGetTimerID()
```

Then investigate:

```c
vTimerSetTimerID()
```

---

# 7. Add a UART command console

Once the basic timers work, make the UART more interactive.

You don't need a sophisticated command parser.

Commands could be:

```text
h
```

Print help.

```text
s
```

Print timer status.

```text
1
```

Toggle heartbeat timer.

```text
2
```

Toggle fast timer.

```text
b
```

Arm/reset the button timeout timer.

```text
x
```

Stop all timers.

```text
r
```

Reset/restart the timer lab.

Example:

```text
> s

===== TIMER STATUS =====
Heartbeat : RUNNING
Fast      : RUNNING
Button    : STOPPED

Heartbeat period : 1000 ms
Fast period      : 250 ms
========================
```

This turns the project from a static demonstration into an actual lab.

---

# 8. The monitor task

Create a task such as:

```c
TimerMonitorTask()
```

Every 5 seconds it prints:

```text
===== TIMER MONITOR =====

Tick count       : 12345
Heartbeat count  : 12
Fast count       : 48
Button expiries  : 3

Heartbeat state  : RUNNING
Fast state       : RUNNING
Button state     : STOPPED

==========================
```

Useful APIs to investigate:

```c
xTimerIsTimerActive()
xTimerGetPeriod()
xTimerGetExpiryTime()
xTaskGetTickCount()
```

Depending on your FreeRTOS version/configuration, investigate the exact timer APIs available.

---

# 9. Button handling

There are two versions of this experiment.

## Version 1 — Button handled by a task

If your button is currently polled:

```text
ButtonTask
    |
    +--> detect press
    |
    +--> xTimerReset(ButtonTimer)
```

This is the easy version.

---

## Version 2 — Button interrupt

Later configure the user button GPIO interrupt.

The ISR should do as little work as possible.

Conceptually:

```text
Button ISR
    |
    +--> notify/button event
            |
            v
       Button Task
            |
            +--> xTimerReset()
```

Then experiment with the FreeRTOS timer `FromISR` API where appropriate.

Depending on your FreeRTOS version/API set, investigate:

```c
xTimerStartFromISR()
xTimerResetFromISR()
xTimerStopFromISR()
xTimerChangePeriodFromISR()
```

Pay attention to the `pxHigherPriorityTaskWoken` mechanism.

---

# 10. The really important experiment:
# Timer Service Task

Once everything works, stop thinking about the individual timers.

Think about this:

```text
Timer A expires
       |
       v
Timer Service Task wakes
       |
       v
callback(A)
       |
       v
Timer Service Task continues

Timer B expires
       |
       v
Timer Service Task wakes
       |
       v
callback(B)
```

There is **one Timer Service Task** servicing all software timers.

Look at your FreeRTOS configuration:

```c
configUSE_TIMERS
configTIMER_TASK_PRIORITY
configTIMER_QUEUE_LENGTH
configTIMER_TASK_STACK_DEPTH
```

Understand what each one means.

---

# 11. Experiment with Timer Service Task priority

Change:

```c
configTIMER_TASK_PRIORITY
```

Run the application with:

```text
Priority = low
```

Then:

```text
Priority = medium
```

Then:

```text
Priority = high
```

Create another CPU-heavy task if necessary.

Observe:

- Does LED timing change?
- Do callbacks become delayed?
- Does UART output change?
- Does the fast timer become less regular?

The key concept:

> A timer expiry is not the same thing as a callback executing at exactly that instant.

The Timer Service Task must actually get CPU time.

---

# 12. The dangerous experiment:
# Blocking inside a callback

This is one of the most educational parts of the project.

Temporarily do something terrible:

```c
static void timer_callback(TimerHandle_t xTimer)
{
    vTaskDelay(pdMS_TO_TICKS(500));
}
```

Then observe what happens.

Remember:

> The callback is running in the Timer Service Task.

So if the callback blocks:

```text
Timer Service Task
       |
       +--> callback A
              |
              +--> BLOCKS
                     |
                     X
              other timer callbacks wait
```

Try:

```c
vTaskDelay(pdMS_TO_TICKS(100));
```

Then:

```c
vTaskDelay(pdMS_TO_TICKS(500));
```

Then perhaps a deliberate busy loop.

Compare behavior.

**Restore the callback immediately afterward.**

The lesson is extremely important:

> Timer callbacks should be short and non-blocking.

---

# 13. Another dangerous experiment:
# Slow callback vs fast timer

Set:

```text
Fast timer period = 100 ms
Slow callback workload = ~200 ms
```

You now have:

```text
timer period < callback execution time
```

Observe what happens to the rest of the timer system.

This is a great way to understand why software timers are not equivalent to dedicated hardware timer interrupts.

---

# 14. Timer command queue experiment

This is where the project becomes more interesting.

FreeRTOS timer APIs don't necessarily manipulate the timer service's internal state directly from the calling task.

The timer API communicates with the Timer Service Task through its **timer command queue**.

Look at:

```c
configTIMER_QUEUE_LENGTH
```

Then generate lots of timer commands rapidly.

For example, repeatedly:

```text
xTimerStart()
xTimerStop()
xTimerReset()
xTimerChangePeriod()
```

from a task.

Investigate:

```c
xTimerStart()
```

and the meaning of its block time.

Ask yourself:

> What happens if the timer command queue is full?

This is much more valuable than simply memorizing the configuration macro.

---

# 15. Dynamic timer period

Add UART commands:

```text
p 1000
p 500
p 100
p 2000
```

Use them to change the fast/heartbeat timer period at runtime.

Explore:

```c
xTimerChangePeriod()
```

Observe the behavior carefully.

Ask:

- Does changing the period restart the timer?
- When does the new period take effect?
- What happens if the timer is stopped?
- What happens if it is active?

Document what you observe rather than assuming.

---

# 16. Add a "blinker mode"

Make the LEDs behave differently based on timer state.

For example:

### Mode 1 — Normal

```text
LED1: 1 Hz
LED2: 4 Hz
LED3: button timeout
LED4: OFF
```

### Mode 2 — Fast

```text
LED1: 4 Hz
LED2: 10 Hz
LED3: button timeout
LED4: ON
```

Use UART:

```text
m
```

to switch modes.

This gives you a practical reason to call:

```c
xTimerChangePeriod()
```

for multiple timers.

---

# 17. Add a "countdown timer"

Create another one-shot timer:

```text
Timer D
Period: 10 seconds
```

When started:

```text
10...
9...
8...
...
1...
GO!
```

However, **don't make the timer callback run every second just to print the countdown**.

Instead, experiment with a task that maintains the countdown while the software timer represents the final timeout.

This forces you to distinguish:

- timer as a periodic event source
- timer as a timeout mechanism
- task as the component doing longer-running work

---

# 18. Recommended final architecture

By the end, aim for something like:

```text
                           +----------------+
                           |     UART RX    |
                           +-------+--------+
                                   |
                                   v
                           +---------------+
                           | Console Task  |
                           +-------+-------+
                                   |
                +------------------+------------------+
                |                  |                  |
                v                  v                  v
          Timer APIs          Status APIs       Mode changes


Button ---> Button ISR ---> Button Task ---> Timer APIs


                  +-----------------------------+
                  | Timer Command Queue         |
                  +--------------+--------------+
                                 |
                                 v
                  +-----------------------------+
                  | Timer Service Task          |
                  |                             |
                  | Timer A callback            |
                  | Timer B callback            |
                  | Timer C callback            |
                  | Timer D callback            |
                  +--------------+--------------+
                                 |
                +----------------+----------------+
                |                |                |
                v                v                v
              LED 1            LED 2            LED 3
```

---

# 19. Weekend agenda

## Saturday morning — Foundations

### Milestone 1 — First software timer

Create:

```text
Heartbeat Timer
1000 ms
auto-reload
```

Acceptance criteria:

- LED1 toggles every second.
- UART reports the callback.
- No task is responsible for the periodic event.

---

### Milestone 2 — Multiple timers

Add:

```text
Heartbeat: 1000 ms
Fast:       250 ms
```

Acceptance criteria:

- Both callbacks execute.
- LED1 and LED2 behave independently.
- You can explain which task actually executes the callbacks.

---

### Milestone 3 — One-shot timer

Add Button Timer:

```text
3 seconds
one-shot
```

Acceptance criteria:

- Button arms the timer.
- LED3 turns on.
- LED3 turns off after 3 seconds.
- Repeated button presses restart the timeout.

---

## Saturday afternoon — Timer mechanics

### Milestone 4 — Timer IDs

Use one callback for multiple timers.

Acceptance criteria:

```text
callback()
    |
    +--> identify timer via ID
```

You should be able to explain why timer IDs exist.

---

### Milestone 5 — Timer state console

Add:

```text
h
s
1
2
b
x
```

Acceptance criteria:

- Timers can be started/stopped.
- UART reports state.
- You can inspect active/inactive timers.

---

### Milestone 6 — Dynamic periods

Add runtime period changes.

Acceptance criteria:

```text
p 100
p 500
p 1000
p 2000
```

changes timer behavior.

---

## Sunday morning — RTOS internals

### Milestone 7 — Timer Service Task investigation

Study:

```text
configTIMER_TASK_PRIORITY
configTIMER_QUEUE_LENGTH
configTIMER_TASK_STACK_DEPTH
configTICK_RATE_HZ
```

Write down what each does.

Then deliberately modify them and observe the system.

---

### Milestone 8 — Timer callback blocking experiment

Make a callback intentionally slow.

Record:

```text
Callback duration
Other timer behavior
LED behavior
UART behavior
```

Then restore the callback.

---

### Milestone 9 — Command queue experiment

Generate timer commands rapidly.

Investigate:

```text
xTimerStart()
xTimerStop()
xTimerReset()
xTimerChangePeriod()
```

and their return values/block times.

---

## Sunday afternoon — ISR integration + polish

### Milestone 10 — Button ISR

Move button handling to an interrupt.

Explore the ISR-safe timer APIs.

---

### Milestone 11 — Timer lab console

Make the UART output clean enough that the board feels like a little diagnostic instrument.

Example:

```text
========================================
        STM32 FREERTOS TIMER LAB
========================================

Heartbeat : RUNNING   1000 ms
Fast      : RUNNING    250 ms
Button    : STOPPED   3000 ms

Callbacks:
  Heartbeat : 42
  Fast      : 168
  Button    : 3

Timer task priority : 3
Tick rate            : 1000 Hz

Commands:
  h       help
  s       status
  1       toggle heartbeat
  2       toggle fast timer
  b       arm button timer
  x       stop timers
  p <ms>  change period
  m       change mode

========================================
```

---

# 20. Suggested source structure

Don't over-engineer it.

A reasonable structure:

```text
Core/
  Inc/
    timer_lab.h
    timer_lab_console.h
    timer_lab_led.h

  Src/
    timer_lab.c
    timer_lab_console.c
    timer_lab_led.c
```

Or keep everything in one file initially.

The important thing is the **experiments**, not architecture.

---

# 21. APIs worth exploring

Make yourself use these deliberately.

### Creation

```c
xTimerCreate()
```

### Start/stop

```c
xTimerStart()
xTimerStop()
```

### Reset

```c
xTimerReset()
```

### Change period

```c
xTimerChangePeriod()
```

### Delete

```c
xTimerDelete()
```

### State

```c
xTimerIsTimerActive()
```

### Period

```c
xTimerGetPeriod()
```

### Expiry

```c
xTimerGetExpiryTime()
```

### IDs

```c
pvTimerGetTimerID()
vTimerSetTimerID()
```

### ISR variants

```c
xTimerStartFromISR()
xTimerStopFromISR()
xTimerResetFromISR()
xTimerChangePeriodFromISR()
```

Don't just read these APIs.

**Use them, observe them, and deliberately make them fail.**

---

# 22. Questions you should be able to answer

At the end of the project, don't consider the lab complete until you can answer these without looking at the documentation.

### Basic

1. What is a FreeRTOS software timer?
2. What is the difference between a one-shot and auto-reload timer?
3. What actually causes the timer callback to execute?

### Timer Service Task

4. What is the Timer Service Task?
5. Why does FreeRTOS need one?
6. Does every software timer have its own task?
7. What happens if the Timer Service Task doesn't get CPU time?
8. What happens if a timer callback blocks?

### Timer command queue

9. Why does `configTIMER_QUEUE_LENGTH` exist?
10. What happens when you call `xTimerStart()`?
11. Does the calling task directly execute the timer callback?

### Timer IDs

12. Why would you use a timer ID?
13. How can one callback service several timers?

### Timing

14. What limits the timing precision of a software timer?
15. What role does `configTICK_RATE_HZ` play?
16. What happens when your requested timer period isn't an integer number of ticks?

### ISR

17. Why can't you blindly call normal FreeRTOS timer APIs from an ISR?
18. What are the `FromISR` APIs for?
19. What is `pxHigherPriorityTaskWoken` doing?

### Design

20. What should a timer callback ideally do?
21. When should a callback notify a task instead of doing work itself?
22. When would you use a hardware timer instead of a FreeRTOS software timer?

If you can answer these, you have learned considerably more than "how to blink an LED with a timer."

---

# 23. Stretch experiments

If you're having fun, keep going.

## A. Timer statistics

Track:

```text
callback count
last callback tick
maximum callback interval
minimum callback interval
```

Calculate approximate jitter.

---

## B. Callback execution time

Measure callback duration using a hardware timer/cycle counter if available.

Compare:

```text
normal callback
UART printf callback
busy-loop callback
```

This will make the "keep callbacks short" rule very concrete.

---

## C. Timer storm

Create many timers:

```text
T1 = 100 ms
T2 = 100 ms
T3 = 100 ms
T4 = 100 ms
...
```

Then make them all expire around the same time.

Observe the Timer Service Task processing them.

---

## D. Same expiry time

Create multiple one-shot timers with the same period.

Ask:

> If five timers expire at essentially the same time, which callback runs first?

Then inspect the FreeRTOS implementation/documentation and compare it with your observation.

---

## E. Software watchdog

Create an auto-reload timer that expects a "kick" periodically.

If the system doesn't kick it:

```text
[TIMER] WATCHDOG TIMEOUT
```

Turn LED4 on.

This introduces the idea of using timers for supervision rather than periodic work.

---

# 24. Important design rule

The most important rule for this project:

> **Don't turn timer callbacks into mini-tasks.**

Bad:

```c
timer_callback()
{
    printf(...);
    do_expensive_calculation();
    vTaskDelay(...);
    access_slow_peripheral();
    ...
}
```

Better:

```c
timer_callback()
{
    notify_task();
}
```

Then:

```text
Timer callback
      |
      v
Task notification / queue
      |
      v
Worker task
      |
      +--> expensive work
      +--> blocking operations
      +--> UART
      +--> peripheral access
```

The timer callback should generally be the **event trigger**, not the place where the entire job happens.

---

# 25. Final challenge

Once everything is working, delete your notes and implement this from scratch:

```text
FreeRTOS Timer Lab

3+ software timers
4 LEDs
user button
UART console
timer IDs
one-shot + auto-reload
start/stop/reset
dynamic period
Timer Service Task investigation
ISR interaction
callback timing experiment
```

Then explain the architecture to yourself as if you were reviewing someone else's embedded code.

If you can draw the path:

```text
API call
   ↓
timer command queue
   ↓
Timer Service Task
   ↓
timer expiry
   ↓
callback
   ↓
task notification / queue
   ↓
worker task
```

and explain exactly where each operation executes, you've achieved the real goal of the project.

---

# 26. Suggested git checkpoints

Since this is an exploration project, make the history part of the learning.

```text
01-basic-software-timer
02-multiple-timers
03-one-shot-button-timer
04-timer-ids
05-uart-console
06-dynamic-periods
07-timer-service-task-experiments
08-callback-blocking-experiment
09-button-isr
10-final-timer-lab
```

For the "bad" experiments, keep them in separate commits so you can reproduce them later without contaminating the final implementation.

---

# 27. Minimal starting point

Don't start by building the whole thing.

Your first target should literally be:

```c
TimerHandle_t heartbeat_timer;

static void heartbeat_callback(TimerHandle_t xTimer)
{
    LED1_Toggle();
    printf("[TIMER] heartbeat\n");
}

void TimerLab_Init(void)
{
    heartbeat_timer =
        xTimerCreate(
            "Heartbeat",
            pdMS_TO_TICKS(1000),
            pdTRUE,
            TIMER_ID_HEARTBEAT,
            heartbeat_callback
        );

    xTimerStart(heartbeat_timer, 0);
}
```

Get this running.

Then **add exactly one concept at a time**.

That will make the weekend project feel like an investigation rather than a large FreeRTOS application you have to debug all at once.
