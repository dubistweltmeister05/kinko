[[Linux Kernel Module]]

- We can dynamically create a device file on demand in Linux. We do not have to go and create one under the `/dev` directory to access the hardware. 
- User-level programs can populate the /dev directory. (eg - udevd).
- `udevd` listens to the `uevents` that are generated by the hot plug events of the dynamic loading or un-loading of the kernel modules in the kernel. Once it receives such an event, it will scan the subdirectories under the `/sys/class` directory, looking for `/dev` files to create device files.
- For each `dev` file, which represents a combination of a major and a minor number for a device,  `udevd` creates a corresponding device file in the `/dev` directory.
- `uevents` are generated when the device driver takes help of a kernel API to trigger the dynamic creation of files. during the hot-plugging of a device into the system. 
- All that the driver needs to do is to ensure that the major or minor numbers assigned to a device that are controlled by the driver are exported to the user space through the sysfs.
- The export of all device info, is controlled by calling of the function `device_create`
- udev looks for a file called `dev` under `/sys/class` in the `sysfs` to determine what the major/minor number assigned to a specific device is.

### class_create() - 
Creates a directory for the class that you are defining under the `/sys/class/<your_class_name>` directory.

`struct class class_create(struct modele *owner, const char *name)`
### device_create() -
This creates a sub directory under `/sys/class/<your_class_name>` with the device's name.

	`struct device * device_create(struct class * class, struct device *parent, dev_t devt, void *drvdata, const char *fmt,... )`

![[Pasted image 20250622145245.png]]

