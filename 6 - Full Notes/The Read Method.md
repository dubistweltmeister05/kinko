[[LINUX KERNEL DRIVER]]
### `ssize_t pcd_read(struct file *filp,char __user *buff, size_t count, loff *f_pos)`

The parameters are as follows - 
`filp` - the pointer to the file operation
`buff` - buffer provided by the user
`count` - read count given by the user
`f_pos` - poitner to the current file position from where the read has to begin. 

### What the fuck does it actually do?
- The read method reads `count` bytes from a device starting at position `f_pos`.
- Updates the `f_pos` by adding number of bytes successfully read
- Returns the number of bytes that it has successfully read, and 0 if there are no bytes left (EOF) or that nothing has been read. 