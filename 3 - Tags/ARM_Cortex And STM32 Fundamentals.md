[[PICT_EMBEDDED]]

# Content

- What is ARM and why is it split into **A / R / M** instead of “one CPU for everything”
    ARM is a British company that designs and licenses Architecture for a wide range of CPUs. But before we take a closer look at them, let's understand the NEED for what it is based on - the RISC arch.    
    
    RISC architecture of computing is inherently designed to simplify the individual instructions given to the computer to accomplish tasks. The goal is to offset the need to process more instructions by increasing the speed of each instruction, in particular by implementing an instruction pipeline, which may be simpler to achieve given simpler instructions.
    
    Compared to the Complex Instruction Set Computer (CISC), a RISC computer might require more machine code in order to accomplish a task because the individual instructions perform simpler operations. 
       
    The key operational concept of the RISC computer is that each instruction performs only one function (e.g., copy a value from memory to a register). The RISC computer usually has many (16 or 32) high-speed, general-purpose registers with a load–store architecture in which the instructions that perform arithmetic and tests operate only on the registers, and the instructions that access data in the main memory of the computer only load data from memory into registers or store data from registers into memory. 
    
    The design of the CPU allows RISC computers few simple addressing modes and predictable instruction times that simplify design of the system as a whole.
    
	ARM (Acorn RISC Machines / Advanced RICS Machines), is an implementation of this philosophy.
	
	Unlike Intel or AMD, ARM does not manufacture chips. Instead, it creates **CPU blueprints** that other companies license and integrate into their own silicon. This model allows ARM architectures to appear in a vast range of products, from tiny microcontrollers to high-performance application processors.
	
	Primarily, the ARM solutions are dedicated towards -
	- system-on-chips architecture-built products. Here, the core systems are integrated, and any customization is left to the end user.  
	- system-on-modules architecture-built products. Here, the customization is shifted onto the chip, converting it to a MODULE. This is done primarily to save time and is essentially - "pay, plug, and play".
	
	However, ARM’s customers needed processors with **very different guarantees**. Some systems needed virtual memory and rich operating systems. Others needed strict real-time behavior with zero tolerance for missed deadlines. Others needed ultra-low power and extreme simplicity. Trying to solve all of these with a single architecture would force unacceptable compromises. As a result, ARM split its Cortex product line into three distinct architectural profiles, each optimized for a different class of systems. And if anyone needed any more proof about the immense creativity of us engineers, the lines are named Cortex-A, Cortex-R, and Cortex-M.  
   ![[2018-04-06-12-43-40-1024x844.jpg.webp]]
    - ### Cortex-A → MMU, Linux, user/kernel space
	    **Cortex-A processors represent ARM’s application-class computing line.** These are the “big-boy” CPUs from ARM, designed to run full operating systems like Linux and Android, with clear separation between user space and kernel space. Architecturally, Cortex-A cores include everything required for application-level compute: a Memory Management Unit (MMU) for virtual memory, high-bandwidth memory subsystems with multi-level caches, out-of-order execution to maximize instruction throughput, and SIMD engines such as NEON for media and signal processing workloads. 
	    
	    This makes them suitable for compute-heavy systems ranging from smartphones and tablets to networking equipment, automotive infotainment, robotics platforms, and servers. Naturally, this level of capability comes at a higher silicon cost and power budget compared to microcontroller-class cores. 
	    
	    Within the Cortex-A family, different SKUs exist to target different power-performance trade-offs: Cortex-A5 and A7 are power-efficient, in-order cores; Cortex-A9 and A15 prioritize raw performance through out-of-order execution; Cortex-A17 improves efficiency while retaining out-of-order pipelines; and the A50-series cores such as A53, A57, and A72 introduce the 64-bit ARMv8-A architecture, enabling modern operating systems and server-class workloads.


		- #### A5
			The Cortex A5 is the smallest processor of the A-range, but it can still demonstrate multicore performance. It is the right choice for designers who have experience with ARM926EJ-S or similar, and additionally, it is compatible with A9 and A15 processors.

		- #### A7   
			In terms of power consumption, the A5 and A7 processors are nearly the same. However, the performance provided by the A7 is 20% higher. The Cortex-A7 finds its application in smartphones and tablets and is famous for its power optimization technology. This technology can potentially save up to 75% of CPU energy and in this way can greatly increase battery life. It is compatible with the A15 and A17.

		- #### A15
			The Cortex-A15 has the highest performance among the members of this set. The performance demonstrated by the A15 is 200% higher than the A9. The A15 processor finds its application in high-end devices, low-power servers, and wireless technologies. This is the first processor in the range that fits data management and virtual environment solutions.

		- #### A17
		    The effectiveness of the Cortex-A17 is impressive. This is why its main aim is satisfying the needs of premium-class devices. With considerably improved efficiency in comparison to other members of the A-team, the A17 demonstrates 60% higher performance than that of the A9. The processor can be configured with 1–4 cores with out-of-order pipelines. An A17 in combination with an A7 creates the big.little configuration mentioned above.

		- #### A50
		    The Cortex-A50 processor extends the areas of application to low-power servers. It demonstrates useful power-optimized features and creates great conditions for the migration of 64-bit operating systems to mobile solutions.
		    
    - ### Cortex-R → hard real-time, safety
	    **Cortex-R processors are designed for systems where timing matters more than raw throughput.** Unlike Cortex-A cores, which optimize average performance, Cortex-R cores are built to guarantee bounded and predictable execution. They target hard real-time applications such as automotive control units, storage controllers, industrial automation, and networking data-paths, where missing a deadline is a system failure.
	    
	    Architecturally, Cortex-R processors avoid features that introduce unpredictability, such as aggressive speculation or complex virtual memory systems. Instead, they rely on tightly coupled memories (TCM) for deterministic access, carefully controlled caches, fast and predictable interrupt handling, and extensive reliability features like ECC, lock-step execution, and fault detection. 
	    
	    While they resemble high-end microcontrollers in terms of real-time behavior, Cortex-R cores deliver significantly higher performance and scalability, allowing them to sit in the gap between Cortex-M microcontrollers and Cortex-A application processors. In practice, Cortex-R processors run bare-metal firmware or real-time operating systems, prioritizing worst-case execution time over peak performance.

		- #### R4
		  The Cortex-R4 microcontroller can be clocked to 600 MHz. Its main characteristics are a dual-issue 8-stage pipeline and prefetch prediction. R4 controllers can quickly handle any incoming interruptions.

		- #### R5
		    This type of processor takes care of networking and data storage. It extends the set of R4 features by adding improved efficiency and reliability. It deals with error management and enables fast peripheral reading/writing. The dual-core implementation of the Cortex-R5 allows the construction of strong systems with lightning-fast response.
		- #### R7
		    The processors from this category demonstrate very high performance. They feature an 11-stage pipeline and enable both out-of-order execution and high-level branch prediction. Tools can be implemented for lock-step, symmetric, and asymmetric multiprocessing. The generic interrupt controller is another significant feature that should be mentioned.
        
    - ### Cortex-M → MCU, bare-metal / RTOS
	    **Cortex-M processors are designed to make software feel as close to hardware as possible.** This family targets the microcontroller market, where simplicity, low power consumption, fast startup, and deterministic behavior are more important than high compute throughput. Cortex-M cores eliminate complex features like MMUs and deep pipelines, allowing programs to start executing immediately after reset and respond to interrupts with extremely low latency. 
	    
	    A defining feature of Cortex-M is the tight integration of the NVIC interrupt controller into the core itself, enabling fast and predictable interrupt handling that is critical for control-oriented systems. Different Cortex-M variants exist to address different segments: M0 and M0+ emphasize minimal silicon area and ultra-low power, M3 provides a general-purpose 32-bit MCU platform, M4 adds DSP extensions for signal-processing workloads, and higher-end variants trade some simplicity for performance while remaining deterministic. 
	    
	    Cortex-M processors commonly run bare-metal code or lightweight RTOSes and dominate applications such as sensors, motor control, wearables, power systems, medical devices, and embedded control logic. In essence, Cortex-M cores are CPUs designed to disappear into the product, doing their job quietly, efficiently, and predictably.
		
		- #### M0+
		    This kind of processor shows rather low performance. It uses the Thumb-2 instruction set and has a 2-stage pipeline. Significant features are the bus for single-cycle GPIO and the micro trace buffer. The Cortex-M0+ is a level up from an 8-bit MCU. With this processor, bug fixing gets easier.
		- #### M3 & M4
		    They both provide 1.25 DMIPS/MHz performance, 32-bit bus, increased clock speed, and efficient debugging options. The architectures are the same, as well as the instruction sets. What is the difference between them? Well, the difference is that the core of M4 is capable of DSP.
### Links
https://sirinsoftware.com/blog/the-arm-processor-a-r-and-m-categories-and-their-specifics
https://en.wikipedia.org/wiki/Arm_Holdings
https://en.wikipedia.org/wiki/Reduced_instruction_set_computer

