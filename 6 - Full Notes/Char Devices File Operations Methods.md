[[LINUX KERNEL DRIVER]]
### Inode Object
- There is a clear distinction between the file contents and the file information. 
- An inode is a VFS Structure(struct inode) that holds the general information about a file.
- Whereas a WFS file Structure (struct file) tracks interaction on an opened file by a user process.
- Indoe has all the information that is needed by the filesystem to handle a file. 
- Each object is associated with an inode number, which uniquely identifies the file within a filesystem. 
- An inode object is created and stored in the memory as and when a new file is created. 
### File Object
- This is created in the kernel space whenever a file is opened.
- This stores the interaction between an open file and a process.
- This info exists in the kernel only till the time that the file is opened by a process. 

Whenever a file is created, the VFS calls `init_special_inode` and initializes the newly created inode with the `file_operations` and the `device number` of the Inode with the file that was created. 

![[Pasted image 20250620225834.png]]

# Summary

### When a Device File is created
1. Create a device file using udev
2. Inode object gets created in memory and an inode's i_rdev field is initialized with a device number
3. inod object's i_fop field is set to dummy default file operations (def_chr_fops)

### When a User Process executes and Open System Call
1.  User Process invokes an open system call on the device file 
2. file object gets created
3. inode's `i_fop` gets copied into the file object's `f_op` (dummy default file operations of a char device ).
4. open function of dummy default file operations gets called (`chrdev_open`). 
5. inode object's `i_cdev` field is initialized with `cdev`, that was added during the `cdev_add` functions's execution (lookup happens using the `inode->i_rdev`field).  
6. `inode->cdev->f_ops` gets copied to `file->f_op`. 
7. `file->f_op->open` method gets called.  
![[Pasted image 20250620231801.png]]

