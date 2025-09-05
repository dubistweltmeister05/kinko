[[An Introduction to ACPI]]

ACPI is the key part of implementing the Operating System configured Power Management (OSPM). 

From a power management perspective, OSPM/ACPI promotes the concept that systems should conserve energy by transitioning unused devices into lower power states including placing the entire system in a low-power state (sleeping state) when possible.

The principal goals of ACPI and OSPM are to:
- Enable all computer systems/motherboards to implement motherboard config and power management functions, using appropriate cost/function trade-offs:- 
	- Computer systems include (but are not limited to) desktop, mobile, workstation, and server machines.
    
	- Machine implementers have the freedom to implement a wide range of solutions, from the very simple to the very aggressive, while still maintaining full OS support.
    
	- Wide implementation of power management will make it practical and compelling for applications to support and exploit it. It will make new uses of PCs practical and existing uses of PCs more economical.
- Enhance power management functionality and robustness:
	- Power management policies too complicated to implement in platform firmware can be implemented and supported in the OS, allowing inexpensive power managed hardware to support very elaborate power management policies.
    
	- Gathering power management information from users, applications, and the hardware together into the OS will enable better power management decisions and execution.
    
	- Unification of power management algorithms in the OS will reduce conflicts between the firmware and OS and will enhance reliability.
- Facilitate and accelerate industry-wide implementation of power management:
	- OSPM and ACPI reduces the amount of redundant investment in power management throughout the industry, as this investment and function will be gathered into the OS. This will allow industry participants to focus their efforts and investments on innovation rather than simple parity.
    
	- The OS can evolve independently of the hardware, allowing all ACPI-compatible machines to gain the benefits of OS improvements and innovations.
- Create a robust interface for configuring motherboard devices:
	- Enable new advanced designs not possible with existing interfaces.
