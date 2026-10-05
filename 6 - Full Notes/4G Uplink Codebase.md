[[RhyGen]]

![[AT Command Workflow Pipeline-2026-10-05-125908.png]]


Man, that diagram is a work of art, ISTG. Either way, we have to explain a bunch of stuff that's going on here, so let's do that with some good old code snippets. 

This is an FSM that we have got going on for us to manage the 4G uplink that is setup between the logger and an MQTT Broker along with a bunch of other things that we have been planning to do, such as send and receive control commands via SMS to this thing. 

```
Note - the cmd_ack and cmd_ok fields are set primarily by the rx_cb, with g_driver_inst being assigned the pointer to dev, in the init function of the driver. This is important, because it sure messed my brain for a good 10 mins. Till I put the pieces together, LoL. 
```
The way a 4G module typically works, is that we send it something called AT (attention) commands as strings via a UART interface. And then we wait for something called an Unsolicitated Response Code, or a URC. Once we have the URC that we expect, then we are good to go on and process the next state. This isn't true for ALL AT commands, but is good enough for most of them to hold true.  
``` C
typedef enum {

EC200_AT_Sync = 0,
// Basic connection setup

EC200_Echo_Off,
EC200_SIM_State,
EC200_NW_Reg,
EC200_DataAPN,
EC200_Init_PDP,
EC200_MQTT_Sock,
EC200_MTOpen_Wait,
EC200_MTConn_Wait,
EC200_Net_Cleanup,
// FSM's states that relate to SMS
EC200_SMS_Setup,
EC200_SMS_SendCmd,
// FSM's states that relate to Publishing data
EC200_Idle_Online = 50,
EC200_MTPublish = 60,
EC200_Ack_Wait
} ec200_fsm_state;
```

This is the master state enum that controls the whole FSM. We'll be looking at all the states that are here, along with the piece of code that they execute when called upon. 

## AT_Sync
```C
case EC200_AT_Sync:
    if (now_ms - dev->state_timer >= 1000) {
      printf("\r\n[FSM] AT_Sync -> Sending 'AT'\r\n");
      dev->status = EC200_STATUS_INITIALIZING;
      dev->sim_ready = false;
      dev->net_registered = false;
      dev->mqtt_link_lost = false;
      send_at(dev, "AT\r\n");
      dev->state_timer = now_ms;
      dev->fsm_state = EC200_Echo_Off;
    }
    break;
```

This is the most standard state, where we send a simple `AT\r\n`, and then waits for the  