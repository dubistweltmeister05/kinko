[[Twitter Posting]]

Why am I like this?

I want to read more about Linux kernels and operating system internals, and at the same time I want to understand how AC-DC converters actually work.

I want to write bare-metal device drivers for an FRAM chip connected to a microcontroller, but I also want to understand the design choices and methodology behind putting an LCL filter in front of an active front-end rectifier.

I want to implement TLS over MQTT connections and send binary log files to a remote server over a 4G connection using HTTPS, but I also want to understand the design of a grid-forming inverter.

I get to work on fine-tuning a PID control loop for a three-phase BLDC motor, but I also want to design and construct a PMSM.

I want to understand Linux scheduling, but also want to know how to design the gate-drive circuitry that makes a MOSFET switch.

I want to write a UART driver from the damn reference manual of an SoC, but also want to understand why transformer leakage inductance matters when choosing a power-converter topology.

I want to implement DMA-backed SPI transfers for a cool-ass HMI display, but then I find myself reading about magnetic materials because apparently I also need to understand how the inductor sitting in the power stage was designed to be at the exact inductance that it is at.

I want to understand TCP congestion control, but I also want to design the current loop underneath a three-phase inverter.

I want to write a J1939 interface script, while simultaneously wondering why the DC-link capacitor is sized the way it is.

I want to understand MMUs, page tables and virtual memory, and then immediately start calculating switching losses in an IGBT at 20 kHz.

I want to implement a flash translation layer, but then I get distracted by how GaN transistors change the practical limits of high-frequency power conversion.

I want to understand FreeRTOS scheduling, while also trying to figure out whether my current-control sampling frequency is actually sufficient for the dynamics of the plant.

I want to write a QSPI NOR driver with wear management, but then I end up reading about core losses in magnetic materials.

I want to understand Ethernet PHY initialization, but also want to know why the hell the common-mode choke is sitting exactly where it is.

I want to implement Modbus RTU, and then spend an evening learning about RS-485 termination and biasing because apparently "it communicates" isn't a sufficient explanation anymore.

I want to understand TLS handshakes, but also want to understand what actually happens to the electromagnetic field around a conductor when the switching edge gets faster.

I want to write a bootloader and OTA update mechanism, but also want to understand SVPWM mathematically instead of treating it as some magic library function.

I want to understand Linux's device model and sysfs, while simultaneously trying to understand the thermal path from a semiconductor junction to a heatsink.

I want to implement a ring buffer correctly in C, and then somehow end up worrying about whether the DC-link ripple current is going to kill my capacitor after 20,000 hours.

I want to understand interrupt latency, and then immediately start thinking about dead-time compensation in a three-phase inverter.

I want to understand CAN arbitration at the protocol level, but also want to understand the physical-layer behaviour that makes CAN arbitration possible in the first place.

I want to understand RTOS synchronization primitives, but also want to derive the small-signal model of the converter I'm trying to control.

I want to understand MQTT QoS semantics, while simultaneously wondering what happens when my grid-forming inverter encounters an islanding event.

I want to understand secure firmware verification, but also want to know exactly what happens to the power stage during a shoot-through fault.

I want to understand cache coherency and memory barriers, and then spend the next hour figuring out whether my ADC sampling instant is correctly synchronized with the PWM.

I want to write a Linux character driver, but somewhere in the back of my head there's also a question about why the magnetizing current looks the way it does in an isolated converter.

I want to understand PCIe, but I also want to build a little board with an MCU, FRAM, ADCs, current sensors and a power converter just to see whether I can make the entire thing work.

And the weirdest part is that none of these things feel unrelated to me.

I don't really seem to be collecting technologies.

I think I just want to understand the entire machine.

The electrons moving through the power stage.

The signals being sampled by the ADC.

The control algorithm deciding what the inverter should do.

The firmware turning that decision into PWM.

The MCU executing the firmware.

The RTOS scheduling the tasks.

The driver talking to the peripheral.

The kernel managing the system.

The network stack moving the data.

The TLS layer securing it.

The server receiving it.

And somehow, after understanding all of that, I'll still look at the system and think:

_"Okay, but what if I built the controller myself?"_

This is probably how I end up spending a perfectly reasonable Saturday building a system with an STM32, a three-phase inverter, an RTOS, a custom peripheral abstraction layer, an SD-card logger, MQTT over TLS, a 4G modem, and absolutely no sensible reason for all of those things to be connected.

And I'll probably have a great time doing it.