[[LINUX KERNEL DRIVER]]
## 1. These are IOCTL macro commands

`#define POP_DATA _IOR('a', 'c', struct data *). `
This is an ioctl read command that expands into `_IOC_READ,(type),(nr),sizeof(size)`

`#define SET_SIZE_OF_QUEUE _IOW('a', 'a', int *)`
This is an ioctl write command that expands into `(_IOC_WRITE,(type),(nr),sizeof(size))`

#define PUSH_DATA _IOW('a', 'b', struct data *)
This is an ioctl read command that expands into `_IOC_READ,(type),(nr),sizeof(size)`

Here, a,b and c are group identifier that are used to group all commands that are similar to each other.

- `a`: This is the type identifier. It's used to group related ioctl commands together.
  In this case, all commands are grouped under the type `a`.

- `b` and `c`: These are the command numbers. They uniquely identify each ioctl command within the type
  `a`. For example:
    - `a`, `a` is used for the SET_SIZE_OF_QUEUE command.
    - `a`, `b` is used for the PUSH_DATA command.
    - `a`, `c` is used for the POP_DATA command.

## 2. DECLARE_WAIT_QUEUE_HEAD()

This function macro expands to `struct wait_queue_head name = __WAIT_QUEUE_HEAD_INITIALIZER(name)`, which is in
turn expands to ```#define __WAIT_QUEUE_HEAD_INITIALIZER(name) {					\
	.lock		= __SPIN_LOCK_UNLOCKED(name.lock),			\
	.head		= LIST_HEAD_INIT(name.head) }```
Yeah, they got a MF STRUCTURE MACRO in here. Because of course they do, it's Linux baby ;-):

This macro in Linux is used to declare and initialize a wait queue head in a single step. This macro is part of
the Linux kernel's wait queue mechanism, which is used to manage processes that need to wait for certain conditions
to be met before they can proceed.

## 3. File Operations Structure.

``` 
static struct file_operations fops = {
.open = dev_open,
.release = dev_release,
.unlocked_ioctl = dev_ioctl,
};
```

This is one of the more important things that you are going to come across here.
`file_operations` is a Linux structure that has all the possible operations at the kernel level that are possible
for a driver to perform.

The members of this structure are defined in the `file_operations` structure.

`long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);`

`int (*open) (struct inode *, struct file *);`

`int (*release) (struct inode *, struct file *);`

The implementation of these functions should match the signature of the members of the structures. 

### a.`static int dev_open(struct inode *inodep, struct file *filep)`
This function is implemented whenever a device is opened. The driver shall increment the count of the 
number of open devices in the program by incrementing the `open_count` variable. 

We use the `atomic_inc()` function to do this, as it is considered to be thread safe when compared 
to the standard `<variable>++` way of incrementing a variable. Lemme document what do I really mean to say here. 

Take a look at the code snippet below. 
  
```
int counter = 0;

// Thread 1
counter++;

// Thread 2
counter++;
```

This would result in the counter's value being incremented to 1. The sequence of events would be as 
follows - 
1. Thread 1 reads the value of the counter as 0.
2. Thread 2 reads the value of the counter as 0. 
3. Thread 1 will increment the value to 1.
4. Thread 2 will increment the value to 1.

That's bad, isn't it. The solution to this is -  
```
atomic_t counter = ATOMIC_INIT(0);

// Thread 1
atomic_inc(&counter);

// Thread 2
atomic_inc(&counter);
```
This ensures that the value is correctly incremented to 2. While the implementation of the `atomic_inc()`
function is highly architecture specific, here is an example - 
```markdown
void atomic_inc(atomic_t *v) {
    __asm__ __volatile__(
        "lock; incl %0"
        : "+m" (v->counter)
    );
}
```
### b. `static int dev_release(struct inode *inodep, struct file *filep)`

This is called whenever a device is closed by the driver. The `open_count` variable is decremented in an atomic manner.

### c. `static long dev_ioctl(struct file *file, unsigned int func, unsigned long datap)`
This is the function where the actual logic of the circular queue that interacts with the syscalls is implemented. 

There are 3 syscalls, `SET_SIZE_OF_QUEUE`, `POP_DATA`, and `PUSH_DATA`. An `if-elseif` ladder is implemented to deal
with all three.

**Parameters**
 - struct file *file: Pointer to the file structure representing the open file.
 - unsigned int func: The ioctl command to be executed.
 - unsigned long datap: Pointer to the data passed from user space.

**Return Value**
 - Returns 0 on success.
 - Returns a negative error code on failure.

#### PUSH DATA
1. First, we check if the queue is full or not. If it's full ,we return the out-of-memory error code called `ENOMEM`
2. If the check passes, then we copy data _**FROM**_ the user space via the `datap` pointer _**TO**_ the kernel space
   via the `u.data` pointer. Should the copy fail, we return the `-EFAULT` error code.
3. Once that is done, we validate the length of the copied data, as an added safety check to prevent buffer overflow 
   If the entered data length exceeds the MAX_INPUT length (255), we return the error code `-EINVAL`.
4. Should that be cleared up, we then proceed to allocate memory for the received data in the queue. This is done 
   via the `kmalloc()` function. Again, if the memory allocation fails, we return `ENOMEM`. 
5. After that, we copy the data **_FROM_** the kernel driver structure `u_data.data` to the rear of the queue. And if 
   that fails, then we return `-EFAULT`. 
6. finally, we update the queue indices and the count. The rear is updated and the count is incremented. 
7. Then, print a victory statement and rest :-)

#### SET_SIZE_OF_QUEUE
1. Again, we copy the size of the queue to be set as _**FROM**_ the user space via the `size` variable, and _**TO**_ 
   the kernel space via the `datap` pointer. As always, we return the macro `-EFAULT` if the copy operation fails .
2. Upon successful completion of the copy operation, simply reset the front and rear indices and set the size of the 
   queue.
3. Empty the queue in it's entirety, like, every single address needs to be freed, using the `kfree()` function.
4. Once that's done, assign new memory to all the queue nodes that need to be created in accordance to the size,
   via the `kmalloc()` function. Should that fail, then simply return the error code `-ENOMEM`. 
5. If all is well, then orint out a victory message and say gg!

#### POP_DATA
1. First of all, check if the count is 0. If it is, well, then there's nothing to be popped from the queue. So, we wait
   for a little while. 
2. Once that is dealt with, we then copy _**FORM**_ the kernel space via the `queue[front]`**_TO_** the user space.
   Again, if that fails, we return `-EFAULT`. 
3. And, it's over. 

## 4. INIT and EXIT Functions
So, these are the functions that are called whenever a device driver is registered or unregistered from the linux 
kernel tree. Pretty straightforward really. 

### a. `static int __init Vaaman_init(void)`
The very first function that you'll call is called `register_chardev()`. Here's a brief overview - 
 * **___register_chrdev()_** - create and register a cdev occupying a range of minors
 * @major: major device number or 0 for dynamic allocation
 * @baseminor: first of the requested range of minor numbers
 * @count: the number of minor numbers required
 * @name: name of this range of devices
 * @fops: file operations associated with this devices
 *
 * If @major == 0 this functions will dynamically allocate a major and return
 * its number.
 *
 * If @major > 0 this function will attempt to reserve a device with the given
 * major number and will return zero on success.
 *
 * Returns a -ve errno on failure.
 *
 * The name of this device has nothing to do with the name of the device in
 * /dev. It only helps to keep track of the different owners of devices. If
 * your module name has only one type of devices it's ok to use e.g. the name
 * of the module here.

Then, we call the `class_create()` function - 
 * _**class_create()**_ - create a struct class structure
 * @name: pointer to a string for the name of this class.
 *
 * This is used to create a struct class pointer that can then be used
 * in calls to device_create().
 *
 * Returns &struct class pointer on success, or ERR_PTR() on error.
 *
 * Note, the pointer created here is to be destroyed when finished by
 * making a call to class_destroy().

Lastly, we call the _**device_create()**_ function - 
* device_create - creates a device and registers it with sysfs
 * @class: pointer to the struct class that this device should be registered to
 * @parent: pointer to the parent struct device of this new device, if any
 * @devt: the dev_t for the char device to be added
 * @drvdata: the data to be added to the device for callbacks
 * @fmt: string for the device's name
 *
 * This function can be used by char device classes.  A struct device
 * will be created in sysfs, registered to the specified class.
 *
 * A "dev" file will be created, showing the dev_t for the device, if
 * the dev_t is not 0,0.
 * If a pointer to a parent struct device is passed in, the newly created
 * struct device will be a child of that device in sysfs.
 * The pointer to the struct device will be returned from the call.
 * Any further sysfs files that might be required can be created using this
 * pointer.
 * Returns &struct device pointer on success, or ERR_PTR() on error.
