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
## MCU Features and In-Chip Peripherals

### Flash
- Flash interface
	For any sort of compute system to function, a fundamental requirement would be a space to function within. A space to store and retrieve things as and when demanded by an operation. 
	The Flash memory, is that playground for our MCU. 
	
	If we talk specs, there is a flash memory of 512 Kbytes or 1 Mbytes available for storing programs and data, plus 512 bytes of OTP memory. But what does it mean? And how does a processor interact with it?
	
	Like I mentioned, the flash memory acts as the playground for any and all operations that the MCU wished to do. It serves as non-volatile storage for program code and constant data, allowing the MCU to retain its instructions and essential information even when power is removed. It is used to store the firmware or application code that the MCU executes upon startup, and it supports in-circuit programming, enabling updates without removing the chip from the circuit.
	
	In terms of our own board, there's provisions for Byte, half-word, word and double word write on the flash, along with sector and mass erase capabilities. The flash memory is organized as follows:
			– A main memory block divided into 4 sectors of 16 Kbytes, 1 sector of 64 Kbytes,
				7 sectors of 128 Kbytes
			– System memory from which the device boots in System memory boot mode
			– 512 OTP (one-time programmable) bytes for user data
				The OTP area contains 16 additional bytes used to lock the corresponding OTP
				data block.
			– Option bytes to configure read and write protection, BOR level, watchdog
				software/hardware and reset when the device is in Standby or Stop mode.
				
	Now, how the hell do we interact with this? Simple - via buses! A bus, is nothing but a collection of wires that transmit signals. Multiple wires are bundled together, which helps transmit a big chunk of data at the same time - a.k.a a parallel interface. When it comes to the Flash, there's 2 buses, the I-code and the D-code bus.
	 
- ART accelerator (prefetch, cache, wait states)

### SRAM
- Main SRAM
- CCM SRAM (Core-Coupled Memory)

### Ethernet MAC
- Ethernet MAC
- DMA coupling

### FSMC (Flexible Static Memory Controller)
- External SRAM
- PSRAM
- NOR / NAND Flash

### USB-OTG
- OTG FS
- OTG HS (ULPI)

### DMA
- DMA1
- DMA2



---

## Functional Peripherals and Communication Interfaces

### GPIOs
- GPIO ports
- Alternate Function (AF) muxing

### Timers and Counters
- Basic timers
- General-purpose timers
- Advanced-control timers
### Interrupt & Debug Infrastructure
- NVIC
- SysTick
- JTAG / SWD
- ITM / ETM
### SPI

### I2C

### UART / USART

### CAN

### ADC & DACs
- Analog clock domain

### Camera Interface
- DCMI (Digital Camera Interface)

### Miscellaneous Peripherals
- CRC
- RNG

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

