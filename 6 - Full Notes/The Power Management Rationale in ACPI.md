[[An Introduction to ACPI]]

The necessity to abstract ACPI and move power management to the OS is due to the fact that it enables the OS to evolve separately from the hardware and likewise, the hardware from the OS. 

Some issues with the old way of managing power are - 
- Minimal support, inhibiting the suppliers and vendors from touching and working with it.
	- Moving power management into the OS makes it available for every machine that installs the OS. The level of func may vary, but users will see similar things across devices. 
	- This will enable the vendors to invest into addition of power management utilities into their products. 
- Legacy power management algos are restricted to the into available to the platform that implemented them. This limits their implementation functionality.
	- Centralizing power management information and directives from user, hardware and the OS allows implementation of more powerful functionality. 
- 