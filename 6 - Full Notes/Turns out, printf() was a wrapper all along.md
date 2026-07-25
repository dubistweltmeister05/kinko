
[[Twitter Posting]]
[[Blog Topics]]
# What if I told you, `printf()` is **NOT** what prints the string that you want to see?

One of the biggest misconceptions among beginner C programmers is believing that `printf()` is the function responsible for putting text onto your terminal. It certainly looks that way-after all, you write `printf("Hello, World!\n");`, compile your program, and the words magically appear on your screen. But under the hood, that's not what actually happens.

In reality, `printf()` never talks to your monitor, UART peripheral, USB interface, or terminal emulator. It doesn't know whether your output is going to a serial console, a file, a network socket, or even nowhere at all. Its job is far more modest: **it takes your format string, processes all the format specifiers, constructs the final stream of characters, and then hands those characters to a much lower-level output routine.** The actual act of writing those bytes somewhere else is someone else's responsibility.

Think of `printf()` as a translator rather than a courier. Suppose you write:

```c
printf("Temperature: %d°C\n", temperature);
```

The function first examines the format string, notices the `%d`, converts the integer stored in `temperature` into its ASCII representation, appends the remaining characters, and finally produces something like:

```
Temperature: 28°C
```

At this point, `printf()` has completed the difficult part of its job. The formatted string now exists in memory as a sequence of bytes. But those bytes are still sitting in RAM-they haven't gone anywhere yet. Something else has to physically move them to their destination.

On a desktop operating system such as Linux, that responsibility eventually falls to the `write()` system call. The standard C library passes the formatted buffer to `write()`, along with the file descriptor representing the destination. For standard output, that file descriptor is `1`, commonly known as `stdout`.

Conceptually, the flow looks something like this:

```text
printf()-> Formatting engine -> vfprintf() -> write(1, buffer, length) -> Linux Kernel -> Terminal
```

Notice that `printf()` itself never communicates directly with the operating system. Instead, it relies on a lower-level API that the operating system understands. The `write()` system call tells the kernel, "Please copy these bytes to this output stream." From that point onward, the kernel takes over, routing the data to the terminal, a redirected file, a pipe, or wherever `stdout` currently points.

This design is one of the reasons why simple shell commands like these work without any modification to your C program:

```bash
./app > output.txt
./app | grep Error
```

Your application never knows whether it's printing to a terminal or a file. As far as `printf()` is concerned, it simply formats characters and passes them to `write()`. The operating system decides where those bytes ultimately end up.  Now things become even more interesting when you move into the world of embedded systems. A microcontroller like an STM32 doesn't have an operating system. There is no Linux kernel waiting to service a `write()` system call. There isn't even a concept of a terminal. The processor has absolutely no idea what "printing" means.
Yet, developers still happily write:
```c
printf("System Initialized!\r\n");
```

So where does the output go?  The answer is that **it goes wherever you tell it to go.**

Embedded C libraries such as **newlib** define a low-level hook called `_write()`. Unlike Linux, where the runtime already provides the implementation, on bare-metal systems **you are expected to implement this function yourself**. The standard library calls `_write()` whenever `printf()` finishes formatting its output, effectively asking, "I've prepared the bytes-what should I do with them?"
A typical implementation might look like this:

```c
int _write(int fd, char *buf, int len)
{
    HAL_UART_Transmit(&huart2, (uint8_t *)buf, len, HAL_MAX_DELAY);
    return len;
}
```

Here, every call to `printf()` eventually results in a UART transmission. The C library has no knowledge of UARTs or STM32 peripherals-it simply calls `_write()`. Your implementation bridges the gap between the portable C library and the hardware-specific driver.

The execution flow on an STM32 therefore becomes:

```text
printf() -> Formatting engine -> _write() -> HAL_UART_Transmit() -> USART Peripheral -> Serial Terminal
```

Many embedded projects take this abstraction one step further by routing `_write()` into their own hardware abstraction layer rather than directly invoking the vendor's HAL. In such systems, `_write()` may simply call a PAL UART interface, which in turn invokes the underlying STM32 HAL. This extra level of indirection keeps application code completely independent of the underlying microcontroller vendor.

The flow then becomes:

```text
printf() -> _write() -> Platform Abstraction Layer (PAL) -> STM32 HAL -> USART Peripheral
```

The beauty of this architecture becomes apparent when the project is ported to another platform. If the firmware is migrated from an STM32 to an NXP, Nordic, ESP32, or even Linux, the application code remains completely unchanged. The only component that needs to be rewritten is the PAL implementation. `printf()` continues to function exactly as before because it was never responsible for interacting with the hardware in the first place.

This separation of responsibilities is a recurring theme throughout systems programming. High-level libraries focus on _what_ needs to be done, while lower layers determine _how_ it is actually accomplished. `printf()` is responsible for formatting data into human-readable text. The operating system-or, in bare-metal systems, your `_write()` implementation-is responsible for physically transporting those bytes to their final destination.

So the next time you write:

```c
printf("Hello, World!\n");
```

remember that `printf()` never actually "prints" anything. It simply prepares the data. The real work of getting those bytes onto your screen is performed much further down the software stack-either by the operating system's `write()` system call or by your own implementation of `_write()` in an embedded system. Once you understand this distinction, you've taken an important step toward understanding how software layers interact, and why abstraction is such a powerful concept in systems programming.