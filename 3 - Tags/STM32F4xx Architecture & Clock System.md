[[PICT_EMBEDDED]]


# Lecture 2 – STM32F40xx Architecture and Clock / Bus Systems

![[Pasted image 20260108171448.png]]


---
Right. We have now understood the basics of a Cortex M-4 MCU. Now, we shall take a look at the actual implementation of the Cortex M-4 architecture onto an MCU, which you now hold in your hands. Here is what the product specification from ST itself says about the F407 MCU - 

The STM32F407/417 lines are designed for medical, industrial and consumer applications where the high level of integration and performance, embedded memories and rich peripheral set inside packages as small as 10 x 10 mm are required.
The STM32F407/417 offers the performance of the Cortex™-M4 core (with floating point unit) running at 168 MHz.

***Performance***: At 168 MHz, the STM32F407/417 deliver 210 DMIPS/566 CoreMark performance executing from Flash memory, with 0-wait states using ST’s ART (Adaptive Real-Time) Accelerator. The DSP instructions and the floating point unit enlarge the range of addressable applications.

***Power efficiency***: ST’s 90 nm process, ART Accelerator and the dynamic power scaling enables the current consumption in run mode and executing from Flash memory to be as low as 238 µA/MHz at 168 MHz.

***Rich connectivity***: Superior and innovative peripherals - Compared to the STM32F4x5 series, the STM32F407/417 product lines feature Ethernet MAC10/100 with IEEE 1588 v2 support and a 8- to 14-bit parallel camera interface to connect a CMOS camera sensor.

2x USB OTG (one with High Speed support).

Audio: dedicated audio PLL and 2 full duplex I²S

Up to 15 communication interfaces (including 6x USARTs running at up to 11.25 Mbit/s, 3x SPI running at up to 45 Mbit/s, 3x I²C, 2x CAN, SDIO)
Analog: two 12-bit DACs, three 12-bit ADCs reaching 2.4 MSPS or 7.2 MSPS in interleaved mode

Up to 17 timers: 16- and 32-bit running at up to 168 MHz

Easily extendable memory range using the flexible static memory controller supporting Compact Flash, SRAM, PSRAM, NOR and NAND memories
Analog true random number generator

The STM32F417 also integrates a crypto/hash processor providing hardware acceleration for AES 128, 192, 256, Triple DES, and hash (MD5, SHA-1).

Integration: The STM32F417x portfolio provides from 512 Kbytes (on WLCSP90 package only) to 1 Mbyte of Flash, 192 Kbytes of SRAM and from 64 to 144 pins in packages as small as 4 x 4.2 mm.

The STM32F407/417 product lines provide from 512 Kbytes to 1 MByte of Flash, 192 Kbytes of SRAM, and from 100 to
176 pins in packages as small as 10 x 10 mm.

Well, the info dump is cool and all, sounds like it's a superlative processor after all. But we are here to find out about what does it all mean, aren't we? What are the actual uses of these impressive sounding peripherals and how do they exactly work? Well, I sure am glad that you asked! 

This lecture shall go deep into a few of these peripherals, and to tie it all together, we shall take a look at the Clock and Bus interfaces of this processor, which helps us understand how do these peripherals get their juice (power) and how do they communicate their data to and from the central CPU. 

--- 
## MCU Memory and Access

### Flash
- For any sort of compute system to function, a fundamental requirement would be a space to function within. A space to store and retrieve things as and when demanded by an operation. The Flash memory, is that playground for our MCU.
    
    If we talk specs, there is a flash memory of 512 Kbytes or 1 Mbytes available for storing programs and data, plus 512 bytes of OTP memory. But what does it mean? And how does a processor interact with it?
    
    Like I mentioned, the flash memory acts as the playground for any and all operations that the MCU wished to do. It serves as non-volatile storage for program code and constant data, allowing the MCU to retain its instructions and essential information even when power is removed. It is used to store the firmware or application code that the MCU executes upon startup, and it supports in-circuit programming, enabling updates without removing the chip from the circuit.
    
    In terms of our own board, there's provisions for Byte, half-word, word and double word write on the flash, along with sector and mass erase capabilities. The flash memory is organized as follows: – A main memory block divided into 4 sectors of 16 Kbytes, 1 sector of 64 Kbytes, 7 sectors of 128 Kbytes – System memory from which the device boots in System memory boot mode – 512 OTP (one-time programmable) bytes for user data The OTP area contains 16 additional bytes used to lock the corresponding OTP data block. – Option bytes to configure read and write protection, BOR level, watchdog software/hardware and reset when the device is in Standby or Stop mode.
    
    Now, how the hell do we interact with this? Simple - via buses! A bus, is nothing but a collection of wires that transmit signals. Multiple wires are bundled together, which helps transmit a big chunk of data at the same time - a.k.a a parallel interface. When it comes to the Flash, there's 2 buses, the I-code and the D-code bus.
    
    The I-Code bus stands for the Instruction-Code Bus, while the D-Code bus means Data Bus. The Cortex M-4 arch is Harvard in nature, which means it had 2 separate buses for an Instruction and a piece of Data. Since the instructions are to be fetched in a linear manner and data fetches are irregular in nature, it makes sense to separate these two in order to not interrupt the flow of instructions with data stalls!
    
- ART accelerator (prefetch, cache, wait states) Basically, the ART accelerator helps speed up the instruction fetch access of the Cotred-M4 processor's internal memory. Basically, a part of code is stored into a segment of memory that helps provide near-instant access to the CPU, thus avoiding latency due to memory wait states.
    
    The cache memory is organized in 64 lines of 256 bits each. Each line contains eight 32-bit segments. Each segment can contain one 32-bit or two 16-bit instruction codes. The cache buffer is an extra line of 256 bits keeping a copy of one of the cache lines. For a cacheable access, it allows the instruction fetch from cache in case of a cache hit and it manages the cache memory line refill (instruction code load) in case of a cache miss. For non-cacheable memory accesses, the instruction is directly fetched from memory.
    

### SRAM

- Main SRAM This, is the volatile memory of the MCU. This is the place where all the run-time compute is performed at. All the local variables, function call return addresses, CPU registers and system context during switching is stored at. The MCU has 192KB of System SRAM and 4KB of backup SRAM. The embedded SRAM can be accessed as bytes, half-words (16 bits) or full words (32 bits). Read and write operations are performed at CPU speed with 0 wait state. It is quite fast in it's speed, with the obvious downside of it being volatile in nature.
- CCM SRAM (Core-Coupled Memory) This is a special memory that guaranteed 0 wait-state access during the runtime, due to it having a dedicated bus to the MCU's core. We can explicitly store data into the CCM by writing to specific addresses. It is often used to store compute data for real time control loops and look-up tables for processing critical data.

### FSMC (Flexible Static Memory Controller)

- Quite often, it just so happens that the on-chip memory does not fulfill the requirements of the system that is being developed. Afterall, we are talking about KILOBYTES of RAM here, y'all! Interfacing an external chip of memory with the MCU is quite common, and dare I say the norm these days in the industry. ST recognizes this and decided to develop a dedicated interface for interacting with external memory chips, called the FSMC.
    
    This block is used to provide an interface with (a)synchronous memories and 16-bit PC memory cards. It translates the AHB instructions to the appropriate external memory protocol, while keeping the timing requirements of the device. All external memories share the addresses, data and control signals with the controller. Each external device is accessed by means of a unique chip select. The FSMC performs only one access at a time to an external device. ![[Pasted image 20260120101632.png]]
    

### DMA
Direct Memory Access (DMA) is used to provide high-speed data transfer between peripherals and memory, as well as between memory locations. DMA allows data movement without continuous CPU intervention, freeing CPU resources for other tasks and reducing overall system latency.

The STM32F407 includes two DMA controllers, DMA1 and DMA2, providing a total of 16 streams, with 8 streams per controller. Each stream can be mapped to one of up to eight channels, where each channel corresponds to a specific peripheral request. Arbitration logic within the DMA controller handles priority when multiple DMA requests occur simultaneously.

The DMA controllers connect to the system through AHB master interfaces, allowing access to both memory and peripherals. Data transfers can be configured as 8-bit, 16-bit, or 32-bit accesses, depending on peripheral requirements and memory alignment. Each DMA stream includes an optional FIFO buffer that improves bus efficiency and supports burst transfers. The FIFO threshold level can be configured in software to 1/4, 1/2, or 3/4 of its total capacity.

When operating in direct mode (FIFO disabled), the DMA uses a single internal data buffer to minimize latency. In memory-to-peripheral transfers, this allows one data item to be preloaded so that the transfer can occur immediately when the peripheral issues a DMA request. Each DMA stream can be configured for peripheral-to-memory, memory-to-peripheral, or memory-to-memory transfers, with software-programmable priorities.

#### DMA Transaction
Each DMA transaction has 3 operations: 
	• A loading from the peripheral data register or a location in memory, addressed through the DMA_SxPAR or DMA_SxM0AR register 
	• A storage of the data loaded to the peripheral data register or a location in memory addressed through the DMA_SxPAR or DMA_SxM0AR register 
	• A post-decrement of the DMA_SxNDTR register, which contains the number of transactions that still have to be performed

After an event, the peripheral sends a request signal to the DMA controller. The DMA controller serves the request depending on the channel priorities. As soon as the DMA controller accesses the peripheral, an Acknowledge signal is sent to the peripheral by the DMA controller. The peripheral releases its request as soon as it gets the Acknowledge signal from the DMA controller. Once the request has been deasserted by the peripheral, the DMA controller releases the Acknowledge signal. If there are more requests, the peripheral can initiate the next transaction

---

## GPIOs and Interrupts

### GPIOs
In the world of Microelectronics, GPIOs act as the intermediary of all interactions between the raw, untamed power of the processor and the outer world. Pick up any microcontroller, and you’ll find these arranged in a neat array, usually along the edge of the development board (usually referred to as board GPIO ); while they are situated at the underbelly of any microprocessor, which is the side that you’d slot it along into a motherboard (Also called as the Integrated Circuit GPIOs) .

Now, on its own, the processor is of absolutely no use whatsoever in the absence of its intermediary, as it cannot either receive or send data to its environment. The GPIOs, are of no pre-defined function. It is entirely up to the user to set their function and control its behavior. As the name suggests, there are 2 primary functions that can be assigned to these pins.

When in Input mode, it is capable of receiving data from the external world. This means that the pin is programmed to be capable of receiving voltage from external Input devices, which can be anything such as your mouse, your touchscreen, or any kind of sensor such as ultrasonic, IR, and many more.

Meanwhile, when configured in its output mode, it is capable of delivering voltage signals to external devices, It may be a control signal or a feedback signal, depending on the application loop, as specified in the program written by the programmer.

Adding another level of control to the functionality of these pins is the ability to select the type of I/O (Input / Output) signal that we wish to either receive or send over any particular pin. As of today, most of the signals involved in electronics are either digital or analogue signals. A digital signal is discreet in nature, usually switching between 2 set voltage levels to represent a logical 0 and a logical 1. The analogue signals, however, are continuous in nature, with their values oscillating between 2 limiting voltage levels, usually in a periodic manner.

These pins have an upper limit, which is the maximum voltage that they can either receive or send. These are called logic levels and are usually set at either 5 volts or 3.3 volts. It is advised to adhere to these limits, as overvoltage at these pins can cause significant damage to them, rendering the pin useless.

Now, there is a special feature about these, and that is the ability of some GPIO pins to act as multi-functional pins. The thing is, not all pins of an MCU are assigned/mapped to a particular GPIO pin. The reason being there's simply not enough space for them all to live together on a single board. This leads to a select few pins having the ability to have Alternate Functions. These functions are configurable by the user, as in, the user can decide which functionality should the GPIO Pin be configured into. 
### Timers and Counters
A timer (sometimes referred to as a counter) is a special piece of hardware inside many microcontrollers. Their function is simple: they count (up or down, depending on the configuration--we'll assume up for now). For example, an 8-bit timer will count from 0 to 255. Most timers will “roll over” once they reach their max value. So, our 8-bit timer would start over again from 0 once it reaches 255.

You can apply a variety of settings to most timers to change the way they function. These settings are usually applied via other special function registers inside the microcontroller. For example, instead of counting to a maximum of 255, you might tell the timer that you want it to roll over at 100 instead. Additionally, you can often connect other hardware or peripherals inside the microcontroller to the timer, like toggling a specific pin automatically when the timer rolls over.

Here are some of the common hardware functions you’ll see with timers:

- Output compare (OC): toggle a pin when a timer reaches a certain value
    
- Input capture (IC): measure the number of counts of a timer between events on a pin
    
- Pulse width modulation (PWM): toggle a pin when a timer reaches a certain value and on rollover. By adjusting the on versus off time (duty cycle), you can effectively control the amount of electrical power going to another device. 
### Interrupt & Debug Infrastructure
These are a few special signals that are given priority of being processed regardless of what is already being processed by the MCU when they are received. 

Interrupts are quite curious, since they allow us to have a say over which signal gets processed on a priority. This is quite powerful when it comes to designing systems, where signals of high importance to the system's stability - for example, the break signal in a bike or a car is supposed to be given a higher priority for being processed, over, say, the indicator being blinked?

Now, the cool thing about ST MCUs is that it has a dedicated hardware block called the Nested Vector Interrupt Controller, which takes care of managing the actual interrupt delivery of the signals that it receives. This block is responsible for managing how interrupts are accepted, prioritized, and serviced by the CPU.

Depending on the device family, the NVIC can handle 82 to 91 maskable interrupt channels, basically, reasons to interrupt the processor (apart from the 16 core Cortex-M system exceptions). Each interrupt can be assigned one of 16 programmable priority levels (4-bit priority field), allowing fine-grained control over which events preempt others. The hardware is designed for very low-latency exception handling, so critical signals are serviced almost immediately. It also integrates power-management control hooks (like sleep/wake behavior) and exposes system control registers that let firmware configure, enable, disable, or nest interrupts safely and efficiently.

In short, instead of software constantly polling for events, the NVIC lets the hardware intelligently decide who deserves the CPU first - making real-time behavior both predictable and fast.

Now, who does end up providing the interrupt signals to the NVIC? Is it the external signal itself that ends up so close to the processor? Nope. It is another hardware block, called the EXTI. The EXTI block, receives the external signal and detects the edges or the signal levels. Depending on how has the interrupting signal been configured in the EXTI registers, it will turn them into interrupt requests or events.

Think about it this way - the EXTI determines if any particular signal should have the ability to generate an interrupt. The NVIC decides which interrupt gets services in case there are multiple present. The EXTI has lines, while the NVIC has channels. Depending on the board and the manufacturer, different lines may map to different channels. 

You may think why the hell do we need all this hardware in the first place? Well, the answer is that interrupts need to be filtered; there's a need to determine if the interrupt is supposed to be masked or not, which edge should be considered as the trigger for the interrupt, and a lot more. We cannot waste CPU cycles to do all this. The name of the game is to reduce the workload on the CPU, and if hardware is needed to do that, then so be it!  
![[Pasted image 20260204224604.png]]


---

## How Do We Tie ’Em All Together?
### Clocks for Power and Buses for Memory

---

## Clocks

### HSI

### HSE

### LSI

### LSE

### PLL
- PLL source selection
- SYSCLK generation
- Peripheral clock generation

### Clock Tree & Prescalers
- AHB prescalers
- APB prescalers

---

## Buses

### AHB Buses
- AHB1 (GPIO, DMA, SRAM)
- AHB2 (USB, RNG, DCMI)
- AHB3 (FSMC)

### APB Buses
- APB1 (low-speed peripherals)
- APB2 (high-speed peripherals)

### Timer Clock Behavior
- APB prescaler impact
- Timer clock doubling

### Bus Matrix & Arbitration
- CPU access paths
- DMA arbitration
- Peripheral contention

Links

https://www.allaboutcircuits.com/technical-articles/introduction-to-microcontroller-timers-periodic-timers/

https://medium.com/@kshitijvaze/general-purpose-input-output-a-generic-guide-ff1838968c13

https://www.digikey.com/en/maker/projects/getting-started-with-stm32-timers-and-timer-interrupts/d08e6493cefa486fb1e79c43c0b08cc6

https://www.motioncontroltips.com/what-is-nested-vector-interrupt-control-nvic/