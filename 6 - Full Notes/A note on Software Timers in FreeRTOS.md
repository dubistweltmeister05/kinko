[[FreeRTOS]]
[[Twitter Posting]]
[[Blog Topics]]

So, welcome back to the notes that I publish as I read through Mastering FreeRTOS - the official book of the kernel that I grow increasingly in awe of as I read more about it. 

# What the hell is a Software Timer?
A software timer is something that helps schedule the execution of a function, either at a set time in the future, or periodically with a fixed frequency. The function in question here, is called a timer callback.  The peculiar thing about the callback is that it always HAS to return void. As this function is asynchronously executed, there is no guarantee that there will be something waiting to catch or handle any returned value. 

Thus, they can be imagined as little sandboxes that can execute code, and can be scheduled to do that either at a set time in the future, or regularly with a fixed time interval. Usually, it either sets or clears a global variable that is used to control a process or two, or pushes/consumes data to/from a queue or a pipe, and things of that nature. Asynchronous data operations, basically. A good practice is to keep them as short as you can. 

Also, we have to take special care for NOT calling any function within the callback that can put the calling task into a blocked state. Why so? Read on to find out!

---
# Attributes of a timer
Easy enough to begin with here! Let's take a look at some attributes of a software timer - 
1. Period - This pertains to the time taken by the system, from the moment a timer starts, till the moment that it's related callback function ends. 
2. Types - One shot and Auto-reload are the types. One shot is pretty self explanatory, it executes once and goes away. The Auto-reload timer, and it's callback, are the ones that recur throughout the life of a program. They fire their callback, and start loading up again for the callback to be executed again soon enough!
3. State - Dormant and Running. In the dormant state, a timer and it's callback simply exist in the system memory - doing absolutely nothing, not consuming any CPU cycles, minding their own goddamn business. In the running state, they have been triggered and start consuming CPU cycles, firing their callback and getting that bastard to execute on the processor!
---

# Context of a Timer
Now, if you know your OS-101, you'd be aware of the fact that each task has it's own context. Naturally, I state so to segway into the next part - the context of a timer callback! That'll be something called the RTOS Daemon task. This is the bastard that is responsible for actually executing all the timer callbacks that we might have configured in a FreeRTOS app. 

From what I can understand, the Daemon is what bookkeeps all the timers and their callbacks. It is what keeps track of the time elapsed, and eventually is responsible for executing the callback function that was defined when the task was first declared in the application. Also, this is the reason why we should never call a function that can put it's calling task into the blocked state. Because this happening from a single timer callback leads to the entire daemon being blocked, and that is not good news for anyone involved here!

The daemon task is scheduled like any other FreeRTOS task; it will only process commands, or execute timer callback functions, when it is the highest priority task that is able to run.

---
# Timer Command Queue
This is interesting. Imagine a scenario where a task of higher priority than the daemon is executing. It calls a bunch of timer functions, like `xTimerStart()`, `xTimerStop()`, `xTimerReset()`, `xTimerChangePeriod()`, and so on.

Now, these functions don't actually go and manipulate the timer directly. Instead, they package up what they have been asked to do into a command, and chuck that command into something called the **Timer Command Queue**.

The daemon task is sitting on the other end of this queue, waiting for commands to arrive. So the whole thing is basically a glorified mailbox:
**Application Task → Timer Command Queue → Timer Daemon Task → Timer**

The queue itself is private to FreeRTOS. We don't get to directly poke around inside it or pull commands out of it ourselves. We interact with it indirectly through the timer API. `xTimerStart()`, for example, sends a "start this timer" command to the queue. Eventually, when the daemon gets scheduled, it reads that command and actually starts the timer. This distinction is rather important.

Suppose our application task is higher priority than the daemon. It calls `xTimerStart()`. The API call can return, but that does NOT necessarily mean that the daemon has already processed the command and started the timer. All that has necessarily happened is that the command has been placed into the timer command queue. The daemon will get around to processing it when it gets a chance to run.

This is also where `configTIMER_TASK_PRIORITY` starts becoming rather important. The daemon is just another task as far as the scheduler is concerned. Give it a high priority, and it gets to the queue and processes commands quickly. Give it a lower priority, and a higher-priority task can happily continue doing its thing while timer commands sit there waiting for the daemon to get some CPU time.

And there is another little bastard lurking here: the queue can fill up. The size of the timer command queue is controlled by `configTIMER_QUEUE_LENGTH`. If enough timer commands are generated before the daemon gets around to processing them, there simply won't be any more room in the queue. This is particularly relevant when a high-priority task is hammering the timer API, or when an ISR is repeatedly using the interrupt-safe timer APIs. The daemon can't empty the queue if it isn't getting scheduled, and eventually the poor bastard running the timer APIs has nowhere left to put his commands.

There is a slightly subtle point here as well: the amount of time a timer is supposed to wait is associated with when the command is sent, rather than when the daemon eventually gets around to processing that command. So a busy daemon can introduce lateness in when a callback actually executes, even though the timer's expiry time itself was calculated from the point at which the command was issued.

Which brings us neatly to another little feature that initially seems rather pointless, but is actually pretty damn useful.

---

# Timer IDs

Every software timer has something called a **Timer ID**.

The easiest way to think about this is as a little tag attached to the timer. FreeRTOS itself doesn't really care what the tag means. It simply stores it for us. The ID is a `void *`, which makes it deliberately generic. We can use it to store a pointer to some application-specific object, or use it as an integer value with the appropriate casting, or basically use it as whatever little piece of information we need to associate with that particular timer. And here's where it becomes particularly useful.

Imagine I have ten timers, but I don't particularly fancy writing ten different callback functions.

No problem.

I can give all ten timers the exact same callback function, and use the Timer ID to figure out **which bastard just expired**.

Inside the callback, I can retrieve the timer's ID using `pvTimerGetTimerID()`, and suddenly the generic callback knows exactly which timer it is dealing with. Lovely bit of software design if you'd ask me! The callback doesn't need to be duplicated just because ten different timers use it. The timer itself carries some context along with it.  The ID can also be changed using `vTimerSetTimerID()`, and read back using `pvTimerGetTimerID()`. Interestingly, these two functions are a little different from most of the timer API: they access the timer directly rather than sending a command through the timer command queue.

So, conceptually, a timer is not just:

"Hey FreeRTOS, call this function after 500 ms."

It's more like:

"Hey FreeRTOS, keep track of this timer, associate this period and callback with it, and also remember this little piece of application-specific information that I have attached to it."

And when the timer expires, the daemon task gets hold of the timer, invokes its callback, and the callback can use that ID to figure out what it is supposed to be doing.

---

# So, what the hell is a Software Timer, then?

After all this, I think the easiest way to think about a FreeRTOS software timer is this:

It is **not a task**.

It is **not an interrupt**.

And it is not some magical independent entity that wakes up and executes its callback by itself.

It is essentially a piece of timer state maintained by FreeRTOS, with a callback attached to it. The actual work of managing that timer and executing its callback is performed by the **Timer Daemon Task**. The application communicates with that daemon through the **Timer Command Queue**. When a timer expires, the daemon eventually gets around to processing that expiry and executes the callback **in the context of the daemon task**.

This gives us a pretty neat architecture:

**Application**  
↓  
**Timer API**  
↓  
**Timer Command Queue**  
↓  
**Timer Daemon Task**  
↓  
**Timer expires**  
↓  
**Timer Callback**

And that architecture explains almost all of the rules that initially seem arbitrary.

Why should callbacks be short? 
Because they're running on the daemon's stack and the daemon can't go off and do other timer-related work while your callback is running.

Why shouldn't a callback block? 
Because that would block the daemon task itself, potentially preventing every other software timer from being processed.

Why does timer priority matter? 
Because the daemon is an ordinary FreeRTOS task from the scheduler's perspective.

Why can a timer callback execute later than its nominal expiry time? 
Because the daemon still has to actually get CPU time to execute it.

Why does the timer command queue exist? 
Because application tasks and ISRs need a safe way of telling the daemon, **"Oi, do this thing with this timer."**

And finally, why does the timer have an ID? 
Because sometimes the same callback needs to service multiple timers, and it needs some way of knowing which particular timer just called it.

So, underneath all the fancy API names, software timers are really just another example of the RTOS doing what it does best: - taking asynchronous events, turning them into manageable pieces of work, and handing that work to a task that can deal with it in a controlled environment. 