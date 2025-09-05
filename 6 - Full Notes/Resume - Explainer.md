  

# EXPERIENCE

## 1- 2025-Present VICHARAK COMPUTERS Surat, India

### Linux Kernel Developer –

**• Led and successfully completed migration efforts on support for Linux Kernel v5.10 to Linux Kernel v6.1.75 on the YOCTO build for the Product “Vaaman”. Added application support for USB 2.0/3.0, MIPI-CSI, Audio Codec (Rockchip ES-8316), RTC, and Wi-Fi/Bluetooth Chip (RTL8822CS) in the YOCTO build Image.**

YOCTO being a build system for custom images is extremely useful when ti comes to building support for embedded targets. It significantly cuts down the time to add features to the final product, due the caching system that it implements. The first build takes 6-8 hours, (more or less, depending on the features being requested), but adding new ones is relatively easy, since we don’t have to re-compile the whole kernel.

This is done via Poky, the reference system of YOCTO, and bitbake, the task execution engine of YOCTO.
Poky is the Yocto Project reference system and is composed of collection of tools and metadata. Poky is platform-independent and performs cross-compiling, using Bitbake Tool, OpenEmbedded Core, and a default set of metadata. The main objective of Poky is to provide all the features an embedded developer needs. 

![](file:///tmp/lu2689727xna.tmp/lu2689727xx0_tmp_3162b430.png)  

Bitbake is a task scheduler that parses Python and Shell script mixed code, which we called Recipes. The code parsed generates and runs tasks. They are a set of steps orders according to the code's dependencies.
Metadata is where all the Recipes are located. Metadata is composed of a mix of Python and Shell Script text files. Poky uses this to extend OpenEmbeddded Core, meta-yocto, and meta-yocto-bsp. 

OUR WORK - "https://medium.com/p/c97c5417b93c"

**• Worked for adding Kernel Driver support for Audio Codec within the Private Linux Kernel Source.** 
During my work with the YOCTO project, I found out that the audio the final image was lacking any meaningful support for the Audio Codec. The board used an ES-8316 Audio Codec, but the image that was generated w=via the YOCTO project, lacked any drivers that could be used to interface with the chip. Hence, I had to dive deeper into the issue, and it turns out, the kernel source that was being used by the YOCTO build was misconfigured. I verified this by compiling the private Linux-6.1 kernel and flashing it on the same embedded target. This revealed that there were relevant drivers being included, but crucially - the device tree node for the Audio codec was incorrectly configured.  Turns out, there were a couple crucial properties missing - there were no routing configs for the audio output, and there was no node for the SARADC, which controls the channel configuration of the audio output. Adding these two to the device tree was the fix to my problem. 

Then, I quickly compiled the linux kernel and flashed it on another board, switching out the kernel and the dtb from the pre-installed image on the board, and verified the playback of audio. After raising a PR for the fix, I wasted no time in waiting for the PR to be merged, and generated a patch for my changes, and applied the same in the YOCTO build, resolving the issue. 

**• Implemented complete board bring-up procedure for a custom All-Winner A20 based development board. Automated the process via scripting, leading to a reduction of time taken for bring-up by 50%.**

## 2 - KNORR-BREMSE TECHNOLOGY CENTER, INDIA – KIEPE ELECTRIC Pune, India

### Embedded Software Intern – R&D (12 Months) – Rail division.
**• Spearheaded integration testing for the Base Software of CSM and CSE controllers, ensuring seamless functionality and enhancing overall system reliability by identifying critical performance gaps.**

**• Researched the CI/CT pipeline for Embedded Testing at the base software level for the complete product development process. Working with build-automation tools like Jenkins and Docker.**

**• Conducted architectural analysis of tri-core DSP processors from Texas Instruments for performance enhancements and applicability, presenting findings to Dortmund headquarters to guide the design of a Traction Control Unit for railways.**

**• Led a code review and requirement verification activity for the base software of a product called Modular System Controller, which handles diagnostics and sensor communication for a wagon in railways.**

**• Worked on Unit testing for various software components across many projects at the company. Have gained hands-on experience with software tools like Tessy.**


## COLLEGIATE CLUB CONTRIBUTION

### 2022-2024 TEAM AUTOMATONS Pune, India

#### Embedded Software Lead

**• Responsible for reviving the usage of 32-bit microcontrollers at the team. Delivered a wireless navigation control system for manual control of a 4 and 3-wheeled robot, written using self-written bare metal libraries. This led to a 25% improvement in control system performance.**

**• Conceived a custom Programmer’s Model and libraries for GPIOs, RCC, NVIC&EXTI, and communication protocols like I2C, SPI, UART, and USART Cortex M-4-based microcontrollers from ST Microelectronics.**

**• Was responsible for developing backup navigation and application codes for a double-flywheel mechanism for the ROBOCON challenge's ball-passing task.**

**• Led a sub-team for the documentation round of ROBOCON 2024, securing a score of 97 out of 100.**

**• Wrote custom libraries for Arduino Interfacing of commonly used sensors at the team, such as TF Mini (ToF Sensor), TCS230 (Color Detection Sensor), and Motor Drivers like BTS7960 and Cytron MD10C/ MD30C.**

**• Created and maintained the Github organization for Team Automatons, helping create a centralized repository for the legacy code-base of the team.**

**• Participated in DD ROBOCON 2024, stood AIR 2.**

#### Junior Engineer
**• Handled BLDC Motors and motor drivers, and was tasked with designing, developing, and debugging modular scripts for integration of motor control with Arduino and a wireless control device.**

**• Was responsible for prototyping various application mechanisms such as a double flywheel mechanism, arm throwing mechanism, pinch-gripper mechanism, and claw-picking mechanism.**

**• Worked with electronic sensors and actuators for custom robotic application systems.**

**•Participated in DD ROBOCON 2023, stood at AIR 14.**

## EDUCATION

### 2021-2025 PIMPRI CHINCHWAD COLLEGE OF ENGINEERING Pune, India

Bachelor of Technology – Electronics and Telecommunication – 8.12 CGPA  
#### Projects–

**• Ronin Virtual Machine** –
A stack-based virtual machine (VM) coded in C, capable of interpreting and executing a set of basic instructions including arithmetic operations, conditional jumps, and stack manipulations. Designed to simulate a minimal computing environment with features like program reading and writing to files.

**• Semi-Autonomous BLDC Mobile Robot** –
Responsible for operation and maintenance of BLDC Motors and motor drivers that were used for mobile navigation along with the development and operation of a double-flywheel ring throwing mechanism. Prototyped mechanisms and electronic PCBs for power delivery and voltage regulation applications.

**• Fully Autonomous Pick-And-Place Robot** –
Led a team of 3 engineers, and oversaw Microcontroller application development for the backup navigation system of a 3-wheeled Mobile robot.


#### Volunteering

Conducted workshops on “Electronics in robotics” at various schools through the IEEE SPS PCCOE Chapter under the IEEE Try Engineering Initiative.