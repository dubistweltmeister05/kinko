[[Linux Kernel Module]]

![[Pasted image 20250622203619.png]]

1. Check if the user has requested more than the `DEV_MEM_SIZE` of the device.
2. Copy `count` number of bytes from device memory to the user space. 
3. Update the `f_pos`.
4. return the number of bytes that have been read successfully. 
5. If f_pos is at EoF, return 0.

But wait - HOW TO COPY TO OR FROM USER? Kernel Functions, of course!

- `copy_to_user()`
- `copy_from_user()`

