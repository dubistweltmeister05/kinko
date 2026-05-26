# 4-Week Embedded C Programming Recovery Plan

## Goal

Rebuild strong C programming fundamentals specifically for embedded systems development.

Focus areas:
- Writing code without AI assistance
- Memory awareness
- Pointer confidence
- Embedded-oriented architecture
- Debugging skills
- API design
- State machine thinking
- Production-quality C

---

# Weekly Time Budget

## Weekdays
- 1 hour/day

### Breakdown
- 15 min → theory/review
- 35 min → implementation
- 10 min → debugging/reflection

---

## Weekends
- 3 hours/day

Use weekends for:
- larger projects
- refactoring
- debugging
- architecture work

---

# Rules For The Entire 4 Weeks

## AI Usage Rules (Critical)

### Allowed
- syntax lookup
- compiler warning explanations
- documentation/reference lookup
- reviewing code AFTER implementation

### Not Allowed
- generating implementations
- generating algorithms
- generating project architectures
- auto-completing functions

The primary skill being rebuilt is:

> The ability to stare at a blank editor and produce working code independently.

---

## Coding Habit Rules (Follow Every Day)

### Write The Contract Before The Code
Before writing any function, write a comment block describing:
- what inputs it takes
- what it returns
- what it does to memory (allocates? frees? borrows?)

Delete the comment afterward if you want — but you must have written it first.

### Write The Header File Before The Implementation
For any `.c` file, write the `.h` first. Define the API surface before thinking about internals. This forces design thinking and prevents you from writing code that has no clean interface.

### Plan Before Typing
Spend 3–5 minutes writing (on paper or in a comment block) what you are about to build before touching the keyboard. This is the habit AI destroyed. Rebuild it deliberately.

---

# Development Environment

Use:
- GCC or Clang
- GDB
- Valgrind
- Cppcheck
- clang-format

Compile with:

```bash
gcc -Wall -Wextra -Werror -pedantic -std=c11 main.c
```

Every program should:
- compile warning-free
- avoid undefined behavior
- handle edge cases
- avoid memory leaks
- avoid unnecessary globals

---

# WEEK 1 — Rebuild Core C Fundamentals

## Goal
Become comfortable writing raw C again without hesitation.

Focus:
- memory
- pointers
- arrays
- functions
- structs
- manual implementations

---

# Day 1 — Memory & Variables

## Topics
- stack vs heap
- memory layout (text, data, bss, stack, heap)
- scope and storage duration (`auto`, `static`, `extern`)
- integer types: `uint8_t`, `int16_t`, `size_t`, `ptrdiff_t`
- signed vs unsigned arithmetic pitfalls

## Implement
- custom `strlen`
- custom `strcpy`
- custom `memcpy`
- custom `memset`

## Understand — Spend Time On Each

```c
char *p;          // pointer to char
char p[];         // array of char (decays to pointer)
const char *p;    // pointer to const char (can't modify data)
char * const p;   // const pointer to char (can't move pointer)
const char * const p; // neither modifiable
```

## Self-Check
Without looking anything up, answer:
- Where does a local variable live after its function returns?
- What is the difference between `static int x` inside a function vs at file scope?
- Why does `sizeof(char *)` not equal `sizeof(char[])`?

---

# Day 2 — Pointers Deep Dive

## Topics
- pointer arithmetic and what `p + 1` actually means in bytes
- array decay (why arrays are not pointers but behave like them)
- double pointers: `int **pp` and when you need them
- pass-by-reference patterns in C
- function pointers as a preview

## Implement
- array reverse in-place
- swap two integers without a temp variable (two methods: XOR and arithmetic)
- 2D matrix traversal using pointer arithmetic only (no `[i][j]`)
- `find_max` that returns a pointer to the largest element

## Practice — Mentally Evaluate Each

```c
int arr[] = {10, 20, 30};
int *p = arr;
*(p + 2)        // what is this?
p[1]            // same as?
*(arr + 1)      // same as?
int **pp = &p;  // how do you get arr[0] through pp?
```

## Self-Check
- What does `p++` do vs `(*p)++`?
- Why can't you do `arr = p` if arr is an array?
- What happens when you pass an array to a function that takes `int *`?

---

# Day 3 — Functions & Modularity

## Topics
- header files and include guards (`#ifndef` vs `#pragma once`)
- translation units and the linker's role
- `static` functions (file-scoped) vs global functions
- `inline` functions and why they're not what you think
- function pointers: declaration, typedef, and use

## Implement

### Step 1 — Write the header file first (always)
Before writing `calculator.c`, write `calculator.h` with full API declared:
```c
typedef int (*op_fn)(int, int);
int calc_add(int a, int b);
int calc_sub(int a, int b);
int calc_mul(int a, int b);
int calc_div(int a, int b);
int calc_execute(op_fn op, int a, int b);
```
Then implement `calculator.c` against that header.

### Step 2 — Command dispatch table
Map string commands to function pointers:
```c
typedef struct {
    const char *name;
    op_fn       fn;
} command_t;
```
Loop over the table to find and execute a command by name.

## Self-Check
- What is the difference between declaring and defining a function?
- What happens if you define a function in a header file included by two `.c` files?
- What does `static` on a function do that leaving it out does not?

---

# Day 4 — Structs & Enums

## Topics
- struct padding and alignment rules
- `__attribute__((packed))` and when NOT to use it
- bitfields: what they are and their hardware register use case
- `enum` for state machines and bitmask flags
- `union` and type-punning

## Implement
- student record system with at least 3 different field types
- finite state machine using `enum` for states and a function per state
- a hardware register struct using bitfields:

```c
typedef union {
    uint32_t raw;
    struct {
        uint32_t enable   : 1;
        uint32_t mode     : 2;
        uint32_t reserved : 29;
    } bits;
} ctrl_reg_t;
```

## Inspect & Explain

```c
struct A { char a; int b; char c; };   // what is sizeof?
struct B { char a; char c; int b; };   // what is sizeof? why different?
```

Use `offsetof()` to verify your answers.

## Self-Check
- Why does the compiler insert padding?
- What is the risk of using `packed` structs for memory-mapped registers?
- Why is `enum` preferred over `#define` for state names?

---

# Day 5 — Dynamic Memory

## Topics
- `malloc`, `calloc`, `realloc`, `free` — and their differences
- internal fragmentation vs external fragmentation
- ownership rules: who allocates owns, who owns must free
- lifetime: stack vs heap vs static
- double-free and use-after-free as classes of bugs

## Implement
- dynamic integer vector with `push`, `get`, `resize`, `free`
- `safe_strdup` — duplicates a string, handles NULL input
- safe `realloc` wrapper that does not lose the pointer on failure:

```c
// WRONG — leaks on failure:
buf = realloc(buf, new_size);

// RIGHT:
void *tmp = realloc(buf, new_size);
if (tmp == NULL) { /* handle */ return; }
buf = tmp;
```

## Embedded Perspective
In most bare-metal embedded systems:
- heap is disabled or heavily restricted
- `malloc` is non-deterministic (fragmentation → unpredictable timing)
- stack overflows are silent and catastrophic
- all memory is typically allocated statically at startup

This is why you must understand dynamic memory deeply — so you know exactly what you are giving up when you eliminate it.

## Self-Check
- What does `calloc` do that `malloc` does not?
- Write the ownership rule for a function that returns a heap-allocated string.
- Why is a static global array often preferred over malloc in firmware?

---

# Weekend Project — Mini CLI Task Scheduler

## Features
- add task (with name, priority, deadline)
- remove task by ID
- list all tasks
- mark task complete
- save all tasks to a file
- load tasks from file on startup

## Requirements
- modular design with separate `.c` / `.h` files
- `task.h` written before `task.c`
- structs for task data
- dynamic memory with correct ownership
- clean APIs (caller does not need to know internals)
- no global state except the task list itself

## File Structure
```text
task.h / task.c       — task struct and operations
storage.h / storage.c — file save/load
main.c                — CLI loop
```

## Acceptance Criteria
Before considering this done:
- [ ] compiles with `gcc -Wall -Wextra -Werror -pedantic -std=c11`
- [ ] no errors from `valgrind --leak-check=full`
- [ ] no warnings from `cppcheck`
- [ ] handles: empty list, duplicate IDs, missing file on load
- [ ] you can explain every `malloc` and `free` call and who owns what

## Skills Reinforced
- memory ownership
- state management
- API design
- file I/O in C

---

# Week 1 — Self-Assessment Checkpoint

Before starting Week 2, close your notes and do the following:

## Re-Implement From Memory (no references)
- `strlen`
- `memcpy`
- an in-place array reverse using pointers
- a struct with a function pointer field, and call it

## Answer These Without Looking Up
- What is the difference between `const char *p` and `char * const p`?
- Why does `sizeof(arr) / sizeof(arr[0])` fail when `arr` is a function parameter?
- What is padding and why does the compiler add it?
- What does `static` mean on a local variable inside a function?

If you struggle on more than two of these, spend an extra day reviewing before Week 2.

---

# WEEK 2 — Data Structures + Embedded Thinking

## Goal
Write efficient low-level logic confidently.

---

# Day 1 — Linked Lists

## Implement
- singly linked list with: `insert_front`, `insert_back`, `delete_by_value`, `print_list`, `free_list`
- reverse in-place (iterative, then recursive)
- cycle detection using Floyd's two-pointer algorithm

## Important
After you finish, close the file. Re-implement from memory the next morning without looking at it.

## Self-Check
- Draw the pointer state before and after inserting a node at the front.
- What happens if you `free` a node but don't update the previous node's `next`?
- Why is a linked list cache-unfriendly compared to an array?

---

# Day 2 — Circular Buffers

## Implement
Ring Buffer with a proper struct:

```c
typedef struct {
    uint8_t  *buf;
    size_t    head;
    size_t    tail;
    size_t    capacity;
    size_t    count;
} ring_buffer_t;
```

Operations:
- `rb_init(ring_buffer_t *rb, uint8_t *storage, size_t capacity)`
- `rb_push(ring_buffer_t *rb, uint8_t byte)` — returns error on full
- `rb_pop(ring_buffer_t *rb, uint8_t *out)` — returns error on empty
- `rb_available(ring_buffer_t *rb)` — how many bytes are ready to read
- `rb_flush(ring_buffer_t *rb)`

## Two Design Approaches
Implement both and understand the tradeoff:
1. **count-based**: track number of items explicitly
2. **mask-based**: keep `head` and `tail` as raw integers, use `% capacity` or power-of-2 mask

## Embedded Relevance
Ring buffers are the #1 data structure in embedded C. Used in:
- UART RX/TX
- DMA double-buffering
- ISR-to-main-loop communication
- Audio sample buffers

## Self-Check
- How do you tell the difference between full and empty if you only track head and tail?
- Why must `capacity` be a power of 2 when using the mask approach?
- What does `volatile` do when a ring buffer is shared between an ISR and main code?

---

# Day 3 — Stacks & Queues

## Implement
- stack using a fixed-size array (not dynamic) with `push`, `pop`, `peek`, `is_full`, `is_empty`
- queue using your ring buffer from Day 2

## Solve
- balanced parentheses checker using your stack
- postfix (Reverse Polish Notation) expression evaluator: `"3 4 + 2 *"` → `14`

## Embedded Context
Fixed-size stacks are used in:
- expression parsers on microcontrollers
- undo/redo in small UIs
- call-stack simulation in schedulers

## Self-Check
- What happens in your stack if the caller pushes beyond capacity? How do you signal this?
- Why is a fixed-size array stack often preferred over a dynamically allocated one in firmware?

---

# Day 4 — Bit Manipulation

## Implement These Macros

```c
#define SET_BIT(reg, n)      ((reg) |=  (1U << (n)))
#define CLEAR_BIT(reg, n)    ((reg) &= ~(1U << (n)))
#define TOGGLE_BIT(reg, n)   ((reg) ^=  (1U << (n)))
#define CHECK_BIT(reg, n)    (((reg) >> (n)) & 1U)
#define GET_FIELD(reg, mask, shift)  (((reg) & (mask)) >> (shift))
#define SET_FIELD(reg, mask, shift, val) \
    ((reg) = ((reg) & ~(mask)) | (((val) << (shift)) & (mask)))
```

## Practice Exercises
- Given `uint8_t status = 0b10110010`, extract bits 4:2 (a 3-bit field)
- Pack two 4-bit values into a single `uint8_t`
- Check if a number is a power of 2 using a single bitwise expression
- Swap two nibbles in a byte: `0xAB` → `0xBA`

## Understand
- Little-endian vs big-endian: which byte is at the lowest address?
- Shifting pitfall: always use `1U` not `1` to avoid signed shift UB
- Why you must cast to `uint32_t` before shifting by 31

## `volatile` Preview
- Why `volatile` is required when accessing a hardware register via a pointer
- What the compiler is allowed to do without it

## Self-Check
- Write a function that returns the position of the highest set bit.
- Why is `x & (x-1)` useful? What does it do?

---

# Day 5 — State Machines & ISR Communication Patterns

## Part 1 — State Machines

Implement a Traffic Light FSM:

```c
typedef enum {
    STATE_RED,
    STATE_GREEN,
    STATE_YELLOW,
    STATE_COUNT
} traffic_state_t;
```

Then refactor from a `switch` statement to a **state transition table**:

```c
typedef struct {
    traffic_state_t next_state;
    uint32_t        duration_ms;
    void (*on_enter)(void);
} state_entry_t;

static const state_entry_t fsm_table[STATE_COUNT] = { ... };
```

Understand why the table approach scales better than nested switch-case.

---

## Part 2 — ISR Communication Patterns

This directly underpins the UART weekend project. Learn these before Saturday.

### The Core Pattern
```c
// ISR (interrupt context) — keep SHORT
void UART_IRQHandler(void) {
    volatile_flag = 1;           // set flag only
    rb_push(&rx_buf, UART->DR);  // write to ring buffer
    // NEVER: malloc, printf, complex logic
}

// Main loop (thread context)
while (1) {
    if (volatile_flag) {
        volatile_flag = 0;
        // handle the event here, safely
    }
}
```

### Key Rules
- ISR and main loop share data: the shared variable **must** be `volatile`
- `volatile` prevents the compiler from caching the value in a register
- ISR should do minimal work: set a flag, push to a ring buffer, nothing else
- Long operations in ISRs starve other interrupts

### Critical Sections
```c
// When you must read a multi-byte value that an ISR can partially update:
disable_interrupts();     // platform-specific
uint32_t snapshot = shared_value;
enable_interrupts();
```

### Simulate This
Write a program where:
1. A `simulate_isr()` function (called from a timer or manually) pushes bytes into a ring buffer
2. The main loop drains the ring buffer and processes complete messages
3. A `volatile uint8_t isr_flag` signals the main loop that data is ready

## Self-Check
- Why is `volatile` not enough for atomicity on multi-byte variables?
- What is the difference between an ISR writing a flag vs writing directly to a struct?
- What happens if you call `printf` inside an ISR?

---

# Weekend Project — UART Driver Simulation

## Goal
Simulate a complete UART driver in host C. No hardware needed.

## Simulate
- TX ring buffer (application writes here, "hardware" drains it)
- RX ring buffer ("hardware" writes here, application reads it)
- ISR behavior via a `simulate_rx_isr(uint8_t byte)` function
- blocking and non-blocking read APIs

## APIs To Implement

```c
// uart.h — write this first
void     uart_init(void);
int      uart_write(const uint8_t *data, size_t len);  // non-blocking
int      uart_read(uint8_t *out, size_t max_len);       // non-blocking
uint8_t  uart_available(void);                         // bytes in RX buffer
void     uart_flush_tx(void);                          // block until TX empty
void     simulate_rx_isr(uint8_t byte);               // test hook
```

## Use
- ring buffer from Day 2
- state machine for driver state (IDLE, RECEIVING, ERROR)
- bit flags for status
- `volatile` for ISR-shared variables
- critical section simulation for multi-byte reads

## File Structure
```text
uart.h / uart.c       — driver implementation
ring_buffer.h / .c    — reused from Week 2 Day 2
main.c                — test program sending/receiving data
```

## Acceptance Criteria
- [ ] compiles warning-free
- [ ] no Valgrind errors
- [ ] can send 1000 bytes in a loop and receive them all correctly
- [ ] gracefully handles RX overflow (buffer full scenario)
- [ ] all ISR-shared variables are `volatile`
- [ ] you can explain why each `volatile` is there

## Skills Reinforced
- ring buffer in practice
- ISR/main communication
- driver API design
- state machines

---

# Week 2 — Self-Assessment Checkpoint

Before starting Week 3, close your notes and do the following:

## Re-Implement From Memory
- Ring buffer: `push`, `pop`, `available` — from scratch, no notes
- Traffic light FSM using a transition table (not switch-case)

## Answer These Without Looking Up
- What are the three things that make a ring buffer implementation correct?
- Why must a variable shared between an ISR and main be `volatile`?
- Why is `volatile` alone not enough to make a 32-bit read from an ISR safe on an 8-bit MCU?
- Draw the pointer manipulation for inserting a node at the middle of a linked list.

If you struggle on more than two, spend an extra day before moving on.

---

# WEEK 3 — Embedded-Oriented C

## Goal
Think like a firmware/software infrastructure engineer.

---

# Day 1 — Volatile, Memory-Mapped IO & Linker Basics

## Part 1 — Volatile & Memory-Mapped IO

### Why `volatile` Exists
Without `volatile`, the compiler may cache a variable in a register and never re-read it from memory. Hardware registers change without the CPU writing to them. The compiler does not know this.

```c
// Without volatile — the loop may be optimized away entirely:
uint32_t *reg = (uint32_t *)0x40000000;
while (*reg == 0) {}    // compiler: "I just read it, it's still 0, infinite loop" — WRONG

// Correct:
volatile uint32_t *reg = (volatile uint32_t *)0x40000000;
while (*reg == 0) {}    // compiler: re-read every iteration
```

### What `volatile` Does NOT Solve
- It does not provide atomicity
- It does not prevent reordering on out-of-order CPUs (need memory barriers for that)
- It does not replace a mutex in a multi-threaded environment

### Simulate Hardware Registers
```c
// Simulate a peripheral's control/status registers
static volatile uint32_t fake_UART_SR  = 0;  // status register
static volatile uint32_t fake_UART_DR  = 0;  // data register

#define UART_SR_RXNE  (1U << 5)   // RX not empty flag
#define UART_SR_TXE   (1U << 7)   // TX empty flag
```

Write functions to "send" and "receive" through these fake registers.

---

## Part 2 — Linker Basics

This is what separates engineers who understand embedded C from those who just write it.

### Memory Sections
```text
.text    — your code (instructions). Lives in flash on embedded systems.
.rodata  — const data (string literals, const arrays). Also in flash.
.data    — initialized global/static variables. Stored in flash, copied to RAM at boot.
.bss     — zero-initialized globals. Not stored in flash; zeroed by startup code.
stack    — grows down. Local variables live here.
heap     — grows up. malloc/free lives here.
```

### What Startup Code Does (Before `main`)
1. Copy `.data` from flash to RAM
2. Zero-fill `.bss`
3. Call constructors (C++)
4. Call `main`

You never write this manually, but you must know it exists.

### Placing Variables In Specific Sections
```c
__attribute__((section(".noinit"))) uint32_t reboot_counter;
// survives reset because it's not zero-filled by startup code
```

### Exercise
Write a small C program. Compile it. Run:
```bash
size a.out
```
This shows `.text`, `.data`, `.bss` sizes. Then:
- add a large `const` array — which section grows?
- add a large uninitialized global — which section grows?
- add a large initialized global — which two sections change?

## Self-Check
- Where does a `const char *str = "hello"` string literal live at runtime?
- Why is `.bss` not stored in the flash image?
- What happens if your `.data` section is larger than your RAM?

---

# Day 2 — Macros & Preprocessor

## Topics
- include guards: `#ifndef` vs `#pragma once` (know both, prefer `#ifndef` for portability)
- token pasting: `##`
- stringification: `#`
- variadic macros: `__VA_ARGS__`
- conditional compilation: `#ifdef`, `#if defined()`

## Implement

### Logging Macro System
```c
#define LOG_LEVEL_DEBUG  0
#define LOG_LEVEL_INFO   1
#define LOG_LEVEL_WARN   2
#define LOG_LEVEL_ERROR  3

#ifndef CURRENT_LOG_LEVEL
#define CURRENT_LOG_LEVEL LOG_LEVEL_DEBUG
#endif

#define LOG(level, fmt, ...) \
    do { \
        if ((level) >= CURRENT_LOG_LEVEL) { \
            printf("[%s] " fmt "\n", #level, ##__VA_ARGS__); \
        } \
    } while (0)
```

Understand why the `do { } while (0)` wrapper is necessary.

### ASSERT Macro
```c
#define ASSERT(expr) \
    do { \
        if (!(expr)) { \
            printf("ASSERT failed: %s, file %s, line %d\n", \
                   #expr, __FILE__, __LINE__); \
            /* in embedded: trigger breakpoint or reset */ \
        } \
    } while (0)
```

### Register Access Macros
```c
#define MMIO32(addr)        (*(volatile uint32_t *)(addr))
#define PERIPH_BASE         0x40000000U
#define UART1_BASE          (PERIPH_BASE + 0x4400U)
#define UART1_SR            MMIO32(UART1_BASE + 0x00)
#define UART1_DR            MMIO32(UART1_BASE + 0x04)
```

## Self-Check
- Why is `#define MAX(a,b) a > b ? a : b` dangerous? What is the correct form?
- What does `do { } while (0)` buy you that a plain block `{ }` does not?
- What is the difference between `#ifdef FOO` and `#if defined(FOO)`?

---

# Day 3 — GDB, Static Analysis & Debugging

## Part 1 — GDB With Intentional Bugs

Do not just "learn GDB commands." Learn GDB by debugging real bugs.

Create three separate programs, each with a specific bug:

### Bug 1 — Use-After-Free
```c
char *p = malloc(10);
free(p);
strcpy(p, "hello");  // undefined behavior
```
Task: run under Valgrind, identify the exact line, understand the error message.

### Bug 2 — Off-By-One Buffer Overflow
```c
char buf[8];
for (int i = 0; i <= 8; i++) buf[i] = 'A';  // writes buf[8]
```
Task: compile with `-fsanitize=address`, run, read the output. Identify the exact overwrite.

### Bug 3 — Double Free
```c
char *p = malloc(20);
char *q = p;
free(p);
free(q);  // double free
```
Task: reproduce the crash, use GDB to see the backtrace with `bt`.

---

## Part 2 — Core GDB Commands To Master

```bash
gcc -g -O0 program.c -o program  # compile with debug info
gdb ./program

(gdb) break main          # set breakpoint
(gdb) run                 # start execution
(gdb) next                # step over
(gdb) step                # step into
(gdb) print var           # print variable
(gdb) print *ptr          # dereference pointer
(gdb) x/4xb addr          # examine 4 bytes at address as hex
(gdb) watch var           # break when var changes
(gdb) bt                  # backtrace (call stack)
(gdb) info locals         # show all local variables
(gdb) set var = 5         # modify a variable at runtime
```

## Part 3 — Tools

```bash
# Memory error detection:
valgrind --leak-check=full --track-origins=yes ./program

# Address sanitizer (faster than Valgrind):
gcc -fsanitize=address,undefined -g program.c -o program

# Static analysis:
cppcheck --enable=all --std=c11 .
```

## Self-Check
- What is the difference between a segfault and a Valgrind "invalid read" error?
- What does `--track-origins=yes` add to Valgrind output?
- When would you choose `-fsanitize=address` over Valgrind?

---

# Day 4 — Memory Efficiency & Fixed-Point Arithmetic

## Part 1 — Memory Efficiency

### Topics
- cache locality: sequential access vs pointer chasing
- stack frame size and how to reduce it
- avoiding heap fragmentation
- fixed-size allocators

### Implement: Simple Memory Pool Allocator

```c
// pool.h
typedef struct pool pool_t;
pool_t  *pool_create(size_t block_size, size_t block_count, uint8_t *storage);
void    *pool_alloc(pool_t *pool);
void     pool_free(pool_t *pool, void *ptr);
```

Use a free-list (array of pointers to free blocks). No `malloc` inside the pool.

Why this matters: in firmware, you pre-allocate all memory at startup. The pool gives you `alloc`/`free` semantics with deterministic timing.

---

## Part 2 — Fixed-Point Arithmetic

On microcontrollers without an FPU (e.g., Cortex-M0), floating-point is slow or unavailable. Fixed-point is the solution.

### What Is Q16.16 Format
- 32-bit integer where the upper 16 bits are the integer part and lower 16 bits are the fractional part
- Value = raw_int / 65536
- Example: `3.14` → `(int32_t)(3.14 * 65536)` = `205887`

```c
typedef int32_t fixed_t;  // Q16.16

#define FIXED_FROM_INT(x)    ((fixed_t)((x) << 16))
#define FIXED_FROM_FLOAT(x)  ((fixed_t)((x) * 65536.0f))
#define FIXED_TO_FLOAT(x)    ((float)(x) / 65536.0f)

// Multiply: result is Q16.16 = (a * b) >> 16
#define FIXED_MUL(a, b)      ((fixed_t)(((int64_t)(a) * (b)) >> 16))

// Divide: (a << 16) / b
#define FIXED_DIV(a, b)      ((fixed_t)(((int64_t)(a) << 16) / (b)))
```

### Implement: PID Controller In Fixed-Point
```c
typedef struct {
    fixed_t kp, ki, kd;
    fixed_t integral;
    fixed_t prev_error;
} pid_t;

fixed_t pid_update(pid_t *pid, fixed_t setpoint, fixed_t measurement);
```

Test it with known inputs and verify the output matches a float-based reference.

### Why This Matters
PID controllers, sensor filtering, and signal processing are all common embedded tasks that require math. If you can't use floats, you need fixed-point.

## Self-Check
- In Q16.16, what is the maximum integer value representable?
- Why do you need a 64-bit intermediate for multiplication?
- What happens if `ki` accumulates a large integral term? How do you prevent overflow (saturation)?

---

# Day 5 — Error Handling Patterns

## Implement
- a complete `error_t` enum covering at least 8 error codes relevant to your domain
- defensive input validation at API boundaries (never trust caller input)
- cleanup pattern using `goto`
- result-wrapping pattern

## The `goto cleanup` Pattern

This is idiomatic C for resource cleanup. It is not bad style — it is the correct tool:

```c
int process_file(const char *path) {
    int      ret  = -1;
    FILE    *fp   = NULL;
    char    *buf  = NULL;

    fp = fopen(path, "r");
    if (!fp) { ret = ERR_FILE_NOT_FOUND; goto cleanup; }

    buf = malloc(1024);
    if (!buf) { ret = ERR_OUT_OF_MEMORY; goto cleanup; }

    // ... do work ...
    ret = 0;

cleanup:
    if (buf) free(buf);
    if (fp)  fclose(fp);
    return ret;
}
```

The key insight: every resource acquired gets a corresponding release at `cleanup`, and the flow is always linear.

## Result-Wrapping Pattern

```c
typedef struct {
    int      ok;
    uint32_t value;
    int      error;
} result_t;

result_t parse_uint(const char *str);
```

Callers check `result.ok` before using `result.value`.

## Self-Check
- Why is `goto` acceptable for cleanup but not for general flow control?
- What is the bug in this code: `free(ptr); ptr = NULL;` inside a function when `ptr` was passed by value?
- How does your error enum handle the case where 0 means "no error"?

---

# Weekend Project — Cooperative Scheduler

## Build
A tick-based cooperative scheduler that runs multiple tasks on a single thread.

## Design

```c
// scheduler.h — write this first
typedef void (*task_fn)(void);

typedef struct {
    task_fn   fn;
    uint32_t  period_ticks;   // run every N ticks
    uint32_t  last_run_tick;  // when it last ran
    uint8_t   enabled;
} task_entry_t;

void     sched_init(void);
int      sched_register(task_fn fn, uint32_t period_ticks);
void     sched_tick(void);    // call this every 1ms (simulated)
void     sched_run(void);     // the main dispatch loop
```

## Features
- register up to 8 tasks
- each task has a period (e.g., run every 10 ticks, every 100 ticks)
- `sched_tick()` increments a global tick counter
- `sched_run()` loops, calling tasks whose period has elapsed
- tasks cannot block — they must return quickly

## Simulate With These Tasks
```c
void task_led_blink(void);   // toggles every 500ms
void task_uart_poll(void);   // checks UART buffer every 10ms
void task_sensor_read(void); // reads mock sensor every 100ms
```

## Acceptance Criteria
- [ ] compiles warning-free
- [ ] all three tasks run at correct intervals (verify with print output)
- [ ] adding a 9th task returns an error, does not crash
- [ ] no dynamic memory used anywhere in the scheduler
- [ ] scheduler state is encapsulated — main.c does not touch internals

## Skills Reinforced
- embedded architecture
- timing and tick logic
- modularity and encapsulation
- static memory design

---

# Week 3 — Self-Assessment Checkpoint

Before starting Week 4, close your notes and do the following:

## Re-Implement From Memory
- Memory pool allocator: `pool_alloc`, `pool_free`
- The `goto cleanup` pattern in a function with 3 acquired resources
- A macro that safely sets a register field using mask and shift

## Answer These Without Looking Up
- What three sections of a binary are affected when you add a global `const uint8_t table[1024]`?
- Why does `volatile` not solve atomicity on a 32-bit variable on an 8-bit MCU?
- What does the startup code do before calling `main` in a bare-metal system?
- In fixed-point Q16.16, why do you need a 64-bit intermediate for multiplication?

If you struggle, spend an extra day reviewing before Week 4.

---

# WEEK 4 — Production-Level Embedded C

## Goal
Move from “I can code” to “I can engineer.”

---

# Day 1 — API Design & MISRA-C Awareness

## Part 1 — API Refactoring

Pick your ring buffer and scheduler from Weeks 2–3. Audit them against these API quality criteria:

### Naming
- Is the module prefix consistent? (`rb_push` not `push_to_rb`)
- Do function names say what they do, not how? (`uart_read` not `uart_drain_rx_buffer_into_ptr`)

### Encapsulation
- Is the struct definition in the `.c` file with only a forward declaration in `.h`?
- Can the caller reach inside the struct directly, or only via functions?

### Ownership
- Is it clear from the function signature who owns heap-allocated memory?
- Are IN/OUT parameters distinguished? (use `const` for inputs)

### Consistency
- Do all fallible functions return the same error type?
- Is `init` always called before `use`? Is this enforced or just documented?

### The Test
> Cover your implementation files. Can another engineer use your `.h` files without reading your `.c` files?

---

## Part 2 — MISRA-C Awareness

You don’t need to follow MISRA fully, but you need to know why the rules exist.

### High-Impact Rules To Know

| Rule | Why It Matters |
|------|----------------|
| No dynamic allocation after init | Non-deterministic timing, fragmentation |
| No recursion | Stack depth is unbounded |
| All `switch` statements need a `default` | Catches unhandled enum values |
| No implicit integer conversions | `int` + `uint8_t` can silently overflow |
| Avoid `goto` except for cleanup | Unpredictable flow in large codebases |
| Use `uint8_t` etc. instead of `int` for registers | Size must be explicit in embedded |
| Never compare signed and unsigned | Silent wraparound bugs |

### Exercise
Run a MISRA-style audit over your Week 2 UART code. For each violation you find, note:
- what the violation is
- why it could cause a bug in firmware
- how to fix it

## Self-Check
- What is the bug in `uint8_t x = 200; uint8_t y = x + 100;`?
- Why is recursion banned in safety-critical embedded code?

---

# Day 2 — Unit Testing

## Framework Options
- **Unity** — minimal, single-file, widely used in embedded C
- **CMocka** — supports mocking, better for driver testing
- **assert-based** — zero dependency, write your own `TEST_ASSERT` macro

## Start With Assert-Based Testing
Before pulling in a framework, write this yourself:
```c
#define TEST_ASSERT(cond) \
    do { \
        if (!(cond)) { \
            printf("FAIL: %s (%s:%d)\n", #cond, __FILE__, __LINE__); \
            test_failures++; \
        } else { \
            test_passes++; \
        } \
    } while (0)
```

## Write Tests For

### Ring Buffer
```c
void test_rb_push_pop(void);       // basic round-trip
void test_rb_full_detection(void); // can't push when full
void test_rb_empty_detection(void);// can't pop when empty
void test_rb_wrap_around(void);    // indices wrap correctly
void test_rb_overflow_handling(void);
```

### Scheduler
```c
void test_sched_task_runs_at_correct_period(void);
void test_sched_max_tasks_enforced(void);
void test_sched_disabled_task_does_not_run(void);
```

### Memory Pool
```c
void test_pool_alloc_free(void);
void test_pool_alloc_when_full_returns_null(void);
void test_pool_free_and_realloc(void);
```

## Then Switch To Unity
Install Unity (single `.c/.h` file). Refactor your assert-based tests to Unity format. Notice what the framework gives you that your manual approach did not.

## Self-Check
- What is the difference between a unit test and an integration test?
- How do you test a function that reads from hardware? (answer: mock the hardware dependency)

---

# Day 3 — Build Systems

## Part 1 — Makefiles

Learn the real syntax, not just copy-paste:

```makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -Werror -pedantic -std=c11 -g
SRCS    = main.c uart.c ring_buffer.c scheduler.c
OBJS    = $(SRCS:.c=.o)
TARGET  = firmware_sim

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c -o $@ $<

clean:
	rm -f $(OBJS) $(TARGET)
```

Understand: `$@`, `$<`, `$^`, pattern rules, `.PHONY`.

Then add automatic dependency generation:
```makefile
DEPS = $(OBJS:.o=.d)
-include $(DEPS)

%.o: %.c
	$(CC) $(CFLAGS) -MMD -MP -c -o $@ $<
```

This means changing a header file triggers a rebuild of all `.c` files that include it.

---

## Part 2 — CMake (Minimal)

Most professional embedded projects use CMake (STM32CubeIDE, Zephyr, ESP-IDF).

```cmake
cmake_minimum_required(VERSION 3.16)
project(firmware_sim C)

set(CMAKE_C_STANDARD 11)
add_compile_options(-Wall -Wextra -Werror -pedantic)

add_executable(firmware_sim
    main.c
    uart.c
    ring_buffer.c
    scheduler.c
)
```

Build it:
```bash
mkdir build && cd build
cmake ..
cmake --build .
```

Then add a test target:
```cmake
enable_testing()
add_executable(test_ring_buffer test/test_ring_buffer.c ring_buffer.c)
add_test(NAME ring_buffer_tests COMMAND test_ring_buffer)
```

## Self-Check
- What does `make clean` do in your Makefile? Trace the commands.
- What is the difference between `add_executable` and `add_library` in CMake?
- Why does changing a `.h` file not trigger a rebuild without dependency files?

---

# Day 4 — Reading & Re-Implementing Real Embedded Code

## The Exercise

Passive reading does not build skill. This day is active.

### Step 1 — Read One Function Deeply
Choose one of these:
- `xQueueSend` from FreeRTOS (`queue.c`)
- `ring_buf_put` from Zephyr (`lib/os/ring_buffer.c`)
- `list_add` from the Linux kernel (`include/linux/list.h`)

Read it until you can answer:
- What data structures does it use?
- How does it handle the edge cases (empty, full, concurrent access)?
- What does it return and under what conditions?
- What would break if you removed any one line?

### Step 2 — Close The Source
Close the file. Do not look at it again.

### Step 3 — Re-Implement From Memory
Implement the same function with the same interface from scratch.

### Step 4 — Compare
Open the original. Compare yours:
- Did you handle all the edge cases?
- Is your naming consistent with the original style?
- What did you miss? Why?

Write a short note in a comment at the top of your file: what you learned.

## Focus Points When Reading Professional Code
- How do they name things (modules, types, functions)?
- How do they handle errors?
- How do they avoid global state?
- How do they separate policy from mechanism?

## Resources
- FreeRTOS source: https://github.com/FreeRTOS/FreeRTOS-Kernel
- Zephyr source: https://github.com/zephyrproject-rtos/zephyr
- Linux kernel list: `include/linux/list.h`

---

# Day 5 — Timed Coding Session

## Rules
- no AI
- no internet (exception: man pages and compiler docs only)
- 45-minute hard timer
- code must compile before time is up

## Prompt Rotation

Pick one. Do not pick the easiest one.

### Prompt A — Generic Ring Buffer
Implement a ring buffer that works with any element type and size:
```c
int  rb_init(ring_buf_t *rb, void *storage, size_t capacity, size_t elem_size);
int  rb_push(ring_buf_t *rb, const void *elem);
int  rb_pop(ring_buf_t *rb, void *out);
```
The buffer must not know anything about what it stores.

### Prompt B — Cooperative Scheduler (From Scratch)
Implement a scheduler with up to 8 tasks, tick-based periods, and an enable/disable API. No dynamic memory.

### Prompt C — Config String Parser
Parse a null-terminated string of format `"key=value;key2=value2"` into a struct array:
```c
typedef struct { char key[32]; char value[32]; } kv_pair_t;
int parse_config(const char *input, kv_pair_t *pairs, size_t max_pairs);
```
Handle: empty string, missing value, truncated key/value.

### Prompt D — Memory Pool Allocator
Fixed-size block pool allocator. Storage is a caller-provided array. No `malloc` anywhere.

### Prompt E — Debounce Filter FSM
A GPIO debounce filter using a state machine:
- states: IDLE, PRESSED, DEBOUNCING, RELEASED
- input: `gpio_read()` called every 1ms
- output: `on_press` and `on_release` callbacks
- debounce window: 20ms of stable signal required

---

After the timer, run your code through Valgrind and `cppcheck`. Fix whatever they find — but no timer for that part.

---

# Final Weekend Project — Embedded Sensor Framework

## Goal
Build a complete, production-structured embedded C application that integrates everything from the 4 weeks.

## What You Are Building
A simulated embedded system with:
- multiple mock sensor drivers
- a cooperative scheduler dispatching tasks
- an event/callback system for sensor data
- a logging system with levels
- a simple CLI for runtime control

## Project Structure

```text
app/
    main.c            — startup, CLI loop
    cli.c / cli.h     — command line interface
core/
    scheduler.c / .h  — cooperative scheduler (your Week 3 version, refined)
    event.c / .h      — event dispatch (register callbacks by event type)
    log.c / .h        — leveled logging (DEBUG/INFO/WARN/ERROR)
drivers/
    temp_sensor.c / .h    — mock temperature sensor
    accel_sensor.c / .h   — mock accelerometer
utils/
    ring_buffer.c / .h    — your Week 2 ring buffer
    pool.c / .h           — your Week 3 memory pool
```

## Required Interfaces

### Event System
```c
typedef void (*event_cb)(uint8_t event_id, const void *data);
int  event_subscribe(uint8_t event_id, event_cb cb);
void event_publish(uint8_t event_id, const void *data);
```

### Driver Interface (uniform across all sensors)
```c
typedef struct {
    int  (*init)(void);
    int  (*read)(int32_t *out);
    void (*shutdown)(void);
    const char *name;
} sensor_driver_t;
```

### CLI Commands
At minimum:
- `status` — print all task states and last run tick
- `log <level>` — change log level at runtime
- `sensor <name>` — read a specific sensor immediately
- `help` — list commands

## Constraints
- No dynamic allocation after `app_init()` completes
- All scheduler tasks must return in < 1ms (simulated)
- No circular header dependencies
- Every module has its own `.h` that is self-contained

## Acceptance Criteria
- [ ] compiles with `gcc -Wall -Wextra -Werror -pedantic -std=c11`
- [ ] no Valgrind errors
- [ ] no cppcheck warnings
- [ ] CLI responds correctly to all listed commands
- [ ] sensors publish events; at least one callback prints data
- [ ] logging compiles out cleanly when `LOG_LEVEL` is set to ERROR
- [ ] you can add a third sensor in under 10 minutes without touching `core/` or `app/main.c`

The last criterion is the real test of your architecture.

## Skills Reinforced
- full system architecture
- module separation and dependency management
- API design for extensibility
- static memory design
- event-driven thinking
- debuggability and logging

---

# Daily Non-Negotiables

Every single day, without exception:
- write code manually (no AI generation)
- write the header/contract before the implementation
- compile after every logical change — not at the end
- fix every warning immediately, never suppress
- run at least one Valgrind check per session
- spend 5 minutes at the end asking: "what would break this code?"

---

# Critical Embedded Concepts To Internalize

## Memory Ownership
- Who allocates?
- Who frees?
- How long does it live?
- What happens if the caller forgets to free?

Ownership must be explicit in every function signature and documented at every allocation site.

---

## State Machines
Most embedded systems are state machines at every level:
- button handler: idle → debouncing → pressed → released
- UART driver: idle → receiving → frame complete → error
- application: init → running → low power → fault

Always prefer a transition table over a `switch` in `switch` in `switch`.

---

## Bitwise Thinking
Registers are structured bits. Every peripheral is controlled by:
- setting a bit to enable something
- clearing a bit to disable it
- reading a bit to check status
- writing a field (multi-bit value) to configure it

You must be able to read a register map and write the macro that accesses it without thinking.

---

## Deterministic Logic
Embedded systems run in real time. Avoid anything with variable or unpredictable timing:
- no heap allocation after startup
- no unbounded loops
- no blocking operations in ISRs
- no floating-point on MCUs without FPU

---

## API Boundaries
Large embedded systems become unmanageable without clean interfaces.
Each module must be independently testable on a host machine.
If a module can’t be tested without hardware, the abstraction is wrong.

---

## Volatile Is Not Magic
`volatile` tells the compiler to always read/write memory. It does NOT:
- prevent race conditions
- provide atomicity
- act as a memory barrier for the CPU

For shared data between ISR and main, `volatile` is necessary but not always sufficient.

---

# Recommended Practice Implementations

Build from scratch:
- linked list
- vector
- hashmap
- ring buffer
- scheduler
- parser
- allocator

Do NOT copy implementations.

---

# Recommended Books

## Must-Read For This Plan
- "Test Driven Development for Embedded C" — James W. Grenning
- "Making Embedded Systems" — Elecia White

## General C Mastery
- "The C Programming Language" (K&R) — Kernighan & Ritchie
- "Expert C Programming: Deep C Secrets" — Peter van der Linden

## Software Engineering
- "Clean Code" — Robert C. Martin (read critically; not all advice applies to embedded)
- "A Philosophy of Software Design" — John Ousterhout

## Reference
- "C: A Reference Manual" — Harbison & Steele (when you need to know exactly what the standard says)

---

# Measuring Progress

At the end of 4 weeks, you should be able to:
- sit down at a blank editor and implement any of the core data structures from memory
- debug a segfault or memory error using GDB + Valgrind without frustration
- design a module API (header file) before writing a single line of implementation
- explain every `malloc`, `free`, `volatile`, and cast in your code
- build a multi-file project with a working Makefile and CMakeLists.txt
- read real embedded C (FreeRTOS, Zephyr) and understand what it is doing
- add a new component to your sensor framework without touching existing modules

---

# Most Important Advice

Do not optimize for:
- speed
- number of topics
- tutorial consumption

Optimize for:

> Hours spent struggling through implementation yourself.

That discomfort is where skill rebuilding happens.

