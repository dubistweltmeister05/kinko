[[LINUX KERNEL DRIVER]]

## `int pcd_release(struct inode *inode, struct file *filp)`

### What the fuck does it actually do?
- The release method is the exact inverse of what the open method does.
- It leaves the device in it's default state, and frees any data structure that were allocated by the open method. 
- Returns 0 on success. If it does not - there is an error. 