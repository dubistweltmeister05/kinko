[[Character Device And Driver]]

We have support for these user level system calls - 
- Open
- Close
- Read
- Write
- Lseek

But first - 
### The Inode Object

- UNIX makes a clear distinction between the contents of the file and the information about the bastard.
- An inode is a VFS data structure (`struct inode`) that holds information about the file. 
- The VFS file structure (`struct file`)  tracks teh interaction that occurs between the file that is opened by the user space program. 
- The Inode, on the other hand, has all the info that a user space application will need about the file to operate on it. 
- Each Inode has an Inode number, that is used to identify the bitch in the filesystem. 
### The File Object
- This is created every time a file is opened in the user space. 
- Does nothing but stores the interaction beteween the user application and the file.
- Only exists in teh kernel memory, and is not written back to the disk, like an Inode is.
  
Whenever a new device file is created, the function `init_special_inode()` is called. Here, the inode object's member element `i_rdev` is initialized, and the member `i_fop` is initialized with default file operations. 