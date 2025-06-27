[[Linux Kernel Module]]

![[Pasted image 20250622210927.png]]

1. Check the requested `count` value against the `DEV_MEM_SIZE` of the device.
2. Copy `count` bytes from the user space to the device memory. 
3. Update the `current_file_position` 
4. Return number of bytes that are successfully written.