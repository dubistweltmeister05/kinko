[[Character Device And Driver]]

```C
int alloc_chrdev_region(dev_t *dev, unsigned int baseminor, unsigned int count, const char *name);
```

The parameters that are being passed, and their meaning are as follows - 
- `dev_t *dev`: Pointer to a `dev_t` variable where the allocated major number (and starting minor number) will be stored.
- `unsigned int baseminor`: The starting minor number (usually 0).
- `unsigned int count`: The number of consecutive minor numbers required.
- `const char *name`: The name associated with the device (used for `/proc/devices` and debugging).

The function returns 0 on success, and a negative value of failure. 

- The device number is a combination of major and minor numbers
- In Linux kernel, dev_t (typedef of u32) type is used to represent the device number.
- Out of 32 bits, 12 bits to store major number and remaining 20 bits to store minor number
- You can use the below macros to extract major and minor parts of dev_t type variable.
  ```C
dev_t device_number;
int minor_no = MINOR(device_number);
int major_no = MAJOR(device_number);
```
- You can find these macros in linux/kdev_t.h
- If you have, major and minor numbers , use the below macro to turn them into dev_t
type device number
MKDEV(int major, int minor);
