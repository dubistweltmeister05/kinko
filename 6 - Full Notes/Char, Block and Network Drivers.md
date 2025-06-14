2025-01-18 17:51

Tags: [[LINUX KERNEL DRIVER]]

There are mostly 3 major types of a kernel level driver in Linux, as it predominantly distinguishes between the 3 fundamental device types. The CHAR, the BLOCK and the NETWORK device. 

This division of modules into different types, or classes, is not a rigid one; the programmer can choose to build huge modules implementing different drivers in a single chunk of code. 
![[Pasted image 20250118175640.png]]

## Character Devices
A character device is the one which can be accessed as a stream of bytes; withe the char driver being responsible for implementing this behavior. Such a driver at least has the `open, close, read and write` system calls. 

The text console (/dev/console) and the serial ports (/dev/ttyS0 and friends) are examples of char devices, as they are well represented by the stream abstraction. Char devices are accessed by means of filesystem nodes, such as /dev/tty1 and /dev/lp0.

The one difference between a char device and a file is that in a file, you can move back and forth to access the data that is on there; but in a char device, you can only access the data sequentially as the device is nothing but a stream of data, which cannot be accessed once it flows away from your access point. (There exist, nonetheless, char devices that look like data areas, and you can move back and forth in them; for instance, this usually applies to frame grabbers, where the applications can access the whole acquired image using `mmap` or `lseek`.)
## Block Devices
These are accessed by filesystem nodes in the /dev directory. A block device is a device that can host a filesystem. 

In most UNIX systems, a block device can only handle I/O operations that transfer one or more whole blocks (each of 512 bytes in length). Linux, instead, allows the application to read and write a block device like a char device—it permits the transfer of any number of bytes at a time.

As a result, block and char devices differ only in the way data is managed internally by the kernel, and thus in the kernel/driver software interface. Like a char device, each block device is accessed through a filesystem node, and the dif ference between them is transparent to the user. Block drivers have a completely different interface to the kernel than char drivers.

## Network Devices
Any network transaction is made through an interface, that is, a device that is able to exchange data with other hosts. Usually, an interface is a hardware device, but it might also be a pure software device, like the loopback interface.

A network interface is in charge of sending and receiving data packets, driven by the network subsystem of the kernel, without knowing how individual transactions map to the actual packets being transmitted.

Many network connections (especially those using TCP) are stream-oriented, but network devices are, usu ally, designed around the transmission and receipt of packets. **A network driver knows nothing about individual connections; it only handles packets.**

# Reference
[,ch01.2168](https://static.lwn.net/images/pdf/LDD3/ch01.pdf)
