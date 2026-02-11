[[LinkedIn Posting]]
[[PICT_EMBEDDED]]
[[Twitter Posting]]


So, in continuation of the course that I am teaching at PICT Pune, we has the second lecture of the10 part series. This session was all about the internals of an STM32F4xx processor, setting up a theoretical baseline for the students to reference from in the upcoming lectures. 

It also aimed to break down the mystique around the Cortex M-4 based MCU that we all have been told about while we pursue our degree and helped the students familiarise themselves with the in-chip peripherals of the board. 

We started our discussion with a brief about the memory of the microcontroller. We understood the meaning and operation of the Flash memory, looking into the sector division of the available memory, the ways to interface with the memory, what does it store and how is it used, the interfacing buses available - the I-Code and D-Code bus, and the OTP section of the flash. We then moved to the SRAM section of the memory, and understood the pparts of code that is held within the volatile section of the overall MCU memory. We also explored the peculiarities of the CCM - Core Coupled Memory, and understood the mechanism that enables 0 wait-state access to the stored 64KB runtime memory(it directly links to the D-Code bus). 

To round it all up, we spent some time understanding the DMA controllers, it's opertation, the division of peripheral and system DMA access, and the DMA arbitration algos.

Moving on to the GPIO section of the MCU, we took a look at the logic levels of the Pins, the characteristics of the GPIO pins and their controlling registers, explored the alternate functionality mechanism of the pins, and looked at the register mapping via the vector table. We spent quite a bit of time on understanding the NVIC and the EXTI registers, diving into the details of interrupt sources, interrupt channels, handling of the priorities of the incoming signals, and the need for 2 separate blocks for management of interrupts. 

Then, we spent time understanding the Timers of the MCU, diving into their operation, their modes, understanding the "counter" operations, the Input capture and Output compare modes, the PWM generation capabilities, while relating them to sensor interfacing, such as an Ultrasonic (HC-SR04) sensor's trigger pin being connected to a channel of the timer in PWM mode and the echo pin being interfaced with another channel in Input Capture mode.

Finally, to tie all the peripherals together, we took a look at teh Clock and Bus architecture, understanding the various sources for power inputs - HSE, HSI, LSE, LSI, and PLL. We took a dive into the electronic circuits of these sources, udnerstanding the Crystal Oscillators, the RC Oscillators, and the external square waves that can be input to the clock via the HSE Bypass modes. To understand the way data traverses between these peripherals and the MCU core, we took a look at the AHB and the APB bus systems and the BUs matrix that covers them all.

In the Lab part of the session, we tried to visualise the impact of clock sources on computation via a simple experiment, where we took a look at a simple setup of blinking an LED via a software interrupt. We ran the setup twice, with different System Clock speeds - the first being at 16 MHz and the second being at 168 MHz. The difference in the blinking pattern was quite evident, and a simple thought experiment where the variation that we saw at the MHz scale was extrapolated to the Ghz scale at which out phones and laptops operate at and then to the Data Center and Server level processors, which operate at even higher scales. 

Post the demo, we let the students have a bit of fun with blinking the 4 available LED in whatever pattern that they likes, and it was quite fun to see the formed groups work on different problems that they took up on their own. One of the groups wanted to do it all bare-metal, one of them wanted to use a button to control the pattern that the LEDs blinked in, another took it upon themselved to solve any hardware issues that the kits were facing - it was incredible to see the kids do it all on their own!