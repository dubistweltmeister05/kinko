[[Basic Concepts]]


There are 4 main parts of doing graphics on a platform that is supposed to be untouched for years on years. They are as follows - 
![[Pasted image 20251106164554.png]]

pretty straightforward, but there's a lot to unpack here. At the heart of it all, is the MCU.

### 1 MCU - 
The central processor that is in charge of it all. In the context of graphics, it takes care of the "read from" and "write to" processes. Reads component info from the flash, and writes to the RAM that is reserved for the display to render from. 

### 2 RAM - 
The playground for the MCU to do it's work in, and for the display to render from. Enough said tbh.

### 3 FLASH - 
Holds the static data. Components, text, fonts, images - basically a storehouse for the MCU to put the data that is required for the rendering. More often than not, this is an external flash memory that is interfaced with the MCU, since the actual MCUs internal flash memory is not large enough for any meaningful graphics rendering. 

### 4 DISPLAY - 
Come on now. Nothing to say here, is it. 