[[Character Device And Driver]]

```C
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```

The parameters and their meanings are - 
- `struct cdev *p`: Pointer to the initialized `cdev` structure (typically set up with `cdev_init()`).
- `dev_t dev`: Device number (combination of major and minor) to associate with the `cdev`
- `unsigned count`: Number of minor numbers to register (usually `1` for a single device).