[[PICT_EMBEDDED]]


![[Pasted image 20260108171448.png]]
# Lecture 2 – STM32F40xx Architecture and Clock / Bus Systems

---

## MCU Features and In-Chip Peripherals

### Flash
- Flash interface
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

### Interrupt & Debug Infrastructure
- NVIC
- SysTick
- JTAG / SWD
- ITM / ETM

---

## Functional Peripherals and Communication Interfaces

### GPIOs
- GPIO ports
- Alternate Function (AF) muxing

### Timers and Counters
- Basic timers
- General-purpose timers
- Advanced-control timers

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

