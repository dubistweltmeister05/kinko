A friend of mine who's studying CSE in college told me that her Digital Electronics teacher declared that Moore and Mealy FSMs are useless for them in the future.

Infuriating as it was for me to hear a statement as moronic, nitwitted, and fucking stupid as that, here's an example of using an FSM in actual production codebases.

This is a simplified version of an EC200 cellular modem driver I've been working on. For the uninitiated, ec200 is a module from a company called Quectel, that helps establish 4G connectivity. It's a wireless communication module, that is designed to help connect devices via M-2-M and IoT SIM cards.

The EC200 is not something you can simply throw an `AT` command at and expect everything to happen synchronously. There is a sequence of commands involved in the whole operation, and handling a-synchronicity  of the responses is of the most paramount importance.

First, you need to establish that the modem is responding. Then you turn echo off. Then you check whether the SIM is ready. Then you wait for network registration. Then you configure the PDP context. Then you activate it. Then you open an MQTT connection. Then you wait for the modem to asynchronously tell you that the socket has actually opened. Then you establish the MQTT connection. And _then_ you're finally in a state where the application can actually use the modem.

All that happens before a single byte of data is sent over the network. And as if that was not complex enough: **none of these operations need to block the processor while we wait for the modem.**

Ideally, the device shall issue a command, remember what was said, and wait for the response to arrive from the EC200, while it returns control of the MCU to the rest of the system for other processing applications. And lads - that's an FSM. We define the states of the machine as such - 

```c
typedef enum {

    EC200_AT_Sync = 0,

    EC200_Echo_Off,
    EC200_SIM_State,
    EC200_NW_Reg,
    EC200_DataAPN,
    EC200_Init_PDP,
    EC200_MQTT_Sock,
    EC200_MTOpen_Wait,
    EC200_MTConn_Wait,

    EC200_Idle_Online = 50,

    EC200_MTPublish = 60,
    EC200_Ack_Wait

} ec200_fsm_state;
```

And the actual state machine is essentially this:

```c
switch (dev->fsm_state) {

    case EC200_AT_Sync:
        // Send AT and wait for acknowledgment
        break;

    case EC200_Echo_Off:
        // Wait for AT response, then disable echo
        break;

    case EC200_SIM_State:
        // Check SIM
        break;

    case EC200_NW_Reg:
        // Wait for network registration
        break;

    case EC200_DataAPN:
        // Configure PDP context
        break;

    case EC200_Init_PDP:
        // Activate data context
        break;

    case EC200_MQTT_Sock:
        // Start MQTT socket
        break;

    case EC200_MTOpen_Wait:
        // Wait for asynchronous +QMTOPEN response
        break;

    case EC200_MTConn_Wait:
        // Wait for asynchronous +QMTCONN response
        break;

    case EC200_Idle_Online:
        // Modem is connected and available
        break;

    case EC200_MTPublish:
        // Send MQTT payload
        break;

    case EC200_Ack_Wait:
        // Wait for publish acknowledgment
        break;
}
```

The important thing here is that the FSM isn't some theoretical digital-electronics exercise that exists only because your professor needs something to put on an exam. It is literally a model of the software's control flow.

At any given point, the driver has a **state**. It also has a set of **events or conditions** that determine where it goes next.
For example:

```text
                    +----------------+
                    |   AT Sync      |
                    +-------+--------+
                            |
                         "OK"
                            v
                    +----------------+
                    |   Echo Off     |
                    +-------+--------+
                            |
                         "OK"
                            v
                    +----------------+
                    |   SIM State    |
                    +-------+--------+
                            |
                       SIM READY
                            v
                    +----------------+
                    |   Network Reg  |
                    +-------+--------+
                            |
                       REGISTERED
                            v
                    +----------------+
                    |   Configure    |
                    |      APN       |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    |   Activate PDP |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    |  Open MQTT     |
                    +-------+--------+
                            |
                     +QMTOPEN: 0,0
                            v
                    +----------------+
                    |  MQTT Connect  |
                    +-------+--------+
                            |
                     +QMTCONN: 0,0,0
                            v
                    +----------------+
                    |  IDLE / ONLINE |
                    +----------------+
```

And notice something else. The modem's responses don't necessarily arrive in the same function that sent the command. The UART receive callback is running independently:

```c
static void ec200_uart_rx_cb(
    pal_uart_t *self,
    uint16_t *bytes_in_buffer
)
{
    ...
}
```

It consumes bytes from the UART, reconstructs lines, and interprets things like:

```text
OK
ERROR
+CPIN: READY
+CEREG: ...
+QMTOPEN: 0,0
+QMTCONN: 0,0,0
+QMTSTAT: ...
```

Those responses modify the driver's internal state. For example:

```c
if (strstr(line, "+CPIN: READY") != NULL) {
    g_driver_inst->sim_ready = true;
}
```

Or:

```c
if (sscanf(line,
           "+QMTOPEN: %*d,%d",
           &g_driver_inst->qmtopen_res) == 1) {
}
```

The FSM then observes those conditions the next time `ec200_process()` executes. 

**The UART callback deals with events. The FSM decides what those events mean for the driver's control flow.**

This is one of the reasons FSMs are so useful in embedded systems. You could write this driver as one gigantic blocking function:

```c
send("AT");
wait_for_ok();

send("ATE0");
wait_for_ok();

send("AT+CPIN?");
wait_for_sim();

send("AT+CEREG?");
wait_for_network();

...
```

And now, you have a giant system that spends most of it's time waiting for a response that isn't guaranteed to come. And even if it is, there's no telling WHEN that response shall arrive. What if `wait_for_network()` takes ten seconds? Your CPU is sitting there waiting.

What if the application has other things to do?
What if you're running an RTOS?
What if you have sensor acquisition, logging, watchdog servicing, diagnostics, HMI updates, CAN communication and other peripherals operating concurrently?

You don't want your modem driver to monopolize execution while waiting for a response that may arrive whenever the cellular network feels like responding. Instead, the driver says: "I'm currently waiting for network registration."

And returns.

The rest of the system continues doing its job. A timer can also be associated with each state:

```c
if (now_ms - dev->state_timer > 3000) {
    dev->fsm_state = EC200_SIM_State;
}
```

Now the state machine isn't just describing _what we're doing_. It also describes **how long we're willing to remain there and what happens if the expected event never arrives.**

That becomes extremely valuable when dealing with real hardware, because that bastard does not always behave.

The UART packets may be corrupted, the SIM cards aren't inserted or get yanked out by someone or something, your cellular network disappears because of weather conditions going from bad to worse, the modem takes longer than expected to respond for no explainable reason, the MQTT broker becomes unreachable without any trace of the root cause, or a socket dies after you've successfully established it, and so much more that can happen "just because".
Those are not exceptional situations in the embedded world. They're normal operating conditions that your software has to account for.

And an FSM gives you a very clean way of representing them. For example:

```c
case EC200_MTOpen_Wait:

    if (dev->qmtopen_res == 0) {

        send_at(dev,
            "AT+QMTCONN=0,\"STM32_Logger_01\"\r\n");

        dev->state_timer = now_ms;
        dev->fsm_state = EC200_MTConn_Wait;

    }
    else if (now_ms - dev->state_timer > 15000) {

        dev->fsm_state = EC200_MQTT_Sock;

    }

    break;
```

There are two possible outcomes.  The modem responds successfully:

```text
WAITING
   |
   | +QMTOPEN: 0,0
   v
MQTT CONNECT
```

Or it doesn't respond within the timeout:

```text
WAITING
   |
   | 15 seconds elapsed
   v
RETRY
```

That's a state transition.

It's something that is bookish or theoretical, or something to learn "just to pass as exam".  It's control logic that software written by YOU is responsible for.

And the concept scales far beyond cellular modems. A motor controller can have:

```text
INIT
  ↓
IDLE
  ↓
ARMED
  ↓
RUNNING
  ↓
FAULT
  ↓
RECOVERY
```

A bootloader can have:

```text
RESET
  ↓
INITIALIZE
  ↓
CHECK IMAGE
  ↓
VERIFY
  ↓
PROGRAM
  ↓
BOOT
```

A USB device can have different enumeration and configuration states.

A network protocol has states.

A TCP connection has states.

A Bluetooth device has states.

A charging controller has states.

A CAN protocol handler can have states.

A washing machine has states.

A traffic light has states.

A fucking elevator has states.

The moment your system's behaviour can be described as:

> "Given that I am currently in **X**, if **Y** happens, transition to **Z** and perform some action."

you are already describing a finite-state machine whether you choose to call it one or not.

The takeaway I want you to have after reading all this - 
Computer science and electronics education sometimes separates concepts into neat little subjects: digital electronics teaches FSMs, operating systems teaches scheduling, networking teaches protocols, embedded systems teaches interrupts, and so on.

Actual engineering doesn't respect those boundaries.

A modem driver sits right at the intersection of all of them.

You have asynchronous I/O.

You have interrupts.

You have timers.

You have a communication protocol.

You have error recovery.

You have concurrency.

You have state.

And eventually, you need some sane way of making all of those things cooperate. For me, an FSM is one of the simplest and most powerful tools for doing exactly that.

So no, I don't think you should throw away everything you've learned about Moore and Mealy machines because someone said that they're "useless in the future." If anything, I'm finding more uses for the underlying idea the further I get into software!