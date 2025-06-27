[[LINUX KERNEL DRIVER]]

## `int pcd_open(struct inode *inode, struct file *filp)`

### What the fuck does it actually do?
-  Initializes the device and makes it respond for read and write access
- Detects device initialization errors
- Checks open permissions (O_RDONLY, O_WRONLY, O_RDWR)
- Identify the device being opened using the minor number
- Prepares the device private data structure if needed.
- Updates the f_pos if needed.
- THE OPEN METHOD IS OPTIONAL. If not provided, the open operation will forever succeed, and the driver will never be notified. 
- 