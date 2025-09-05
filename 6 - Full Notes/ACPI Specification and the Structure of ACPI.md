[[An Introduction to ACPI]]

![[Pasted image 20250627153736.png]]

Enough said really.

There are 3 components of ACPI - 
- ACPI System Descriptor Tables
	Describes the hardware interfaces. Some descriptions limit what can be built, but most allow the the hardware to be built in arbitrary ways and can describe arbitrary operation sequences needed to make the hardware function. 
	 
	 ACPI Tables containing “Definition Blocks” can make use of a pseudo-code type of language, the interpretation of which is performed by the OS. That is, OSPM contains and uses an interpreter that executes procedures encoded in the pseudo-code language and stored in the ACPI tables containing “Definition Blocks.” 
	 
	 The pseudo-code language, known as ACPI Machine Language (AML), is a compact, tokenized, abstract type of machine language.
	 
- ACPI Registers
	The constrained part of the hardware interface, described (at least in location) by the ACPI System Description Tables.
- ACPI Platform Firmware
	Refers to the portion of the firmware that is compatible with the ACPI specifications. Typically, this is the code that boots the machine (as legacy BIOSs have done) and implements interfaces for sleep, wake, and some restart operations. 
	
	It is called rarely, compared to a legacy BIOS. The ACPI Description Tables are also provided by the ACPI Platform Firmware.

