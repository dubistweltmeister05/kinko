2025-01-08 15:07

Tags: [[Linux Kernel Module]]

![[Pasted image 20250108150810.png]]

![[Pasted image 20250108163715.png]]

### Difference Between User Space and Kernel Space

## User Space:
- This is where **user applications** and **user-level programs** run.
- It has limited access to hardware and system resources for security and stability reasons.
- Processes in user space interact with the kernel using **system calls**.
- It’s isolated from the kernel to prevent accidental or malicious interference with system operations.
- Programs in this space cannot directly access hardware or modify critical system data.

## Kernel Space:
- This is where the **core of the operating system** runs, including the kernel itself, device drivers, and memory management.
- It has full access to hardware and all system resources.
- Code running in kernel space operates with higher privileges, so it can execute critical operations such as hardware control, process management, and system calls.
- The kernel is responsible for managing user-space applications' requests for system resources and services.

---

**Summary**:
- **User space** = User programs, limited access to system resources.
- **Kernel space** = Core system, full control over hardware and system operations.

# Reference

