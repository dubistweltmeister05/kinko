[[Character Device And Driver]]

To initialize a cdev object, we use the following - 
```C
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```

The parameters are as follows - 
- `struct cdev *cdev`: Pointer to a `struct cdev` object that you've allocated (usually statically or with `kmalloc`).
- `const struct file_operations *fops`: Pointer to a `file_operations` structure defining the callbacks for your character device (like `open`, `read`, `write`, etc.).
