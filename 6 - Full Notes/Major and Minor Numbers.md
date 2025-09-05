2025-01-17 23:28

Tags: [[LINUX KERNEL DRIVER]]


### Major number

A major number is a unique identifier assigned to a device driver in the Linux kernel. Each device driver has a unique major number, which is used by the kernel to identify the device and determine which driver to use when a device is accessed. The major number is also known as the “device number” or “device major”.

For example, the major number for the first hard disk drive (HDD) is typically 8, while the major number for the first USB drive is typically 179. You can use the ls -l /dev command to see a list of devices and their corresponding major numbers.

### Minor number 

A minor number is a smaller identifier that is used in conjunction with the major number to uniquely identify a specific device within a class of devices. Minor numbers are used to identify specific devices within a major device class, such as different partitions on a hard disk drive or different USB drives.

For example, if you have two hard disk drives with major number 8, one with minor number 0 and another with minor number 1, they would be identified as /dev/sda (minor 0) and /dev/sdb (minor 1).

### Purpose
In the context of computer file systems, major and minor numbers are used to identify specific devices or types of operations within a device.

A major number identifies a particular driver or device type in the kernel of an operating system. For example, all block devices (such as hard drives and USB flash drives) might be assigned major number 8, while character devices (such as serial ports and keyboards) might be assigned major number 5.

Once a major number has been identified, a minor number is then used to distinguish between different instances of that device type or operation. For instance, if two hard drives are connected to a system, they would both have the same major number for the block device driver, but distinct minor numbers to indicate which physical drive each one corresponds to.

In summary, major and minor numbers provide a way to uniquely identify specific devices or operations in a file system hierarchy, allowing the operating system’s kernel to efficiently manage resources and communicate with hardware components.

# Reference

[Major and Minor numbers in Linux kernel | by Linux School Tech | Medium](https://medium.com/@linuxadminhacks/major-and-minor-numbers-in-linux-kernel-0c54af7a0ab8)