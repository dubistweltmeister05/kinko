[[FreeRTOS]]
[[Twitter Posting]]
[[Blog Topics]]

SO - let's talk about TLS, not Transport Layer Security, but the other one -> Thread Local Storage.

This is a goddamn oxymoron if I have ever seen one, because as per Wikipedia - it is defined as a memory management technique in operating systems that used static/global memory, that is LOCAL to a thread. How the fuck does that even work? Well, let's find out.

The concept allows storage of data that appears globally in a system, within separate threads of a system. The implementation of this concept itself is specific to the OS that you are using. We, shall take a look at the Pthreads implementation, which is defined as follows - 

The functions `pthread_key_create` and `pthread_key_delete` are used respectively to create and delete a key for thread-specific data. The type of the key is explicitly left opaque and is referred to as `pthread_key_t`. This key can be seen by all threads. In each thread, the key can be associated with thread-specific data via `pthread_setspecific`. The data can later be retrieved using `pthread_getspecific`.

In addition `pthread_key_create` can optionally accept a destructor function that will automatically be called at thread exit, if the thread-specific data is not _NULL_. The destructor receives the value associated with the key as parameter so it can perform cleanup actions (close connections, free memory, etc.). Even when a destructor is specified, the program must still call `pthread_key_delete` to free the thread-specific data at process level (the destructor only frees the data local to the thread).

I came across this term while learning about tasks in FreeRTOS. According to the manual, we can store arbitrary data from the task into the Task Control Block itself! This is used to store data that would be otherwise stored in a global variable, by a non-reentrant function. What the hell is a re-entrant function? 

It is a function that can be called by multiple threads without any side effects . Now usually, critical sections of the memory is used for global values to be stored and accessed by such a func, but repeated use of the critical section shall lead to system performance degradation, so TLS is preferred over here. 

Enough theory, let's see a practical example of this. There is a global called `errno` that is used in C to for providing extended error codes for common std lib functions.

Now here's where things get interesting.

Imagine two threads executing the same function simultaneously. Thread A calls a library function, which fails and sets `errno` to `EACCES`. At almost exactly the same time, Thread B calls another library function, which fails and sets `errno` to `ENOENT`.

If `errno` were truly a single global variable shared by the entire process, Thread B could overwrite the value before Thread A gets a chance to read it. Thread A would then see the wrong error code. That would make the entire concept of multithreaded programming rather painful.

This is where TLS comes in.

Although `errno` looks like a global variable, modern implementations typically make it thread-local. Each thread gets its own instance of `errno`, so when Thread A writes `EACCES`, it is writing to Thread A's copy of `errno`. When Thread B writes `ENOENT`, it is writing to Thread B's copy.

So we have something that is globally accessible in the source code, but physically/logically separate for every thread.

And THAT is the entire fucking trick behind Thread Local Storage.

The variable is global from the perspective of the program's interface, but its storage is local to the thread accessing it.

In fact, this is exactly why TLS is so useful for implementing otherwise non-reentrant library functions. Instead of having one shared piece of state that every thread must synchronize access to with a mutex or critical section, each thread gets its own copy of the state.

Conceptually, you can think of it like this:

```
                 errno
                   │
          ┌────────┴────────┐
          │                 │
       Thread A          Thread B
          │                 │
      errno = EACCES    errno = ENOENT
```

There isn't one `errno` being fought over by every thread. There is effectively an `errno` associated with each thread.

And now the oxymoron starts making sense - Thread Local Storage is global in scope, but local in storage. TLS doesn't make a global variable magically thread-safe. It changes the storage model so that each thread gets its own instance.

That same fundamental idea exists in FreeRTOS when we store task-specific data in the Task Control Block. The exact mechanism is different from POSIX TLS, but the underlying philosophy is the same: give each execution context its own instance of state that would otherwise have to be shared globally.