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

And I know what follows is a bit long, but i believe it is much needed to understand the FSM Code, it's the driver structure - 

```C 

typedef struct {
  pal_uart_t *uart; // Modem UART interface
  ec200_status_t status;

  // Line buffer for modem AT responses
  uint8_t rx_line[256];
  uint16_t rx_idx;

  // Command request tracking
  ec200_fsm_state fsm_state;
  uint32_t state_timer;
  volatile bool cmd_ack;
  volatile bool cmd_ok;
  volatile bool prompt_received; // Keeps track of '>' prompt for SMS & QMTPUBEX

  // Network & Network Registration Status
  volatile bool sim_ready; // Set true on +CPIN: READY
  volatile bool net_registered; // Set true on +CEREG / +CGREG home or roaming status

  // Asynchronous MQTT URC Result Codes
  volatile int8_t qmtopen_res; // Stores result code from +QMTOPEN URC
  volatile int8_t qmtconn_res; // Stores result code from +QMTCONN URC
  volatile bool mqtt_link_lost; // Set true if +QMTSTAT URC reports a dropped socket

  // Telemetry / MQTT Action Parameters
  char mqtt_topic[64]; // Target topic for telemetry packet
  uint8_t tx_buf[512]; // Binary payload buffer for telemetry logs
  uint16_t tx_len;     // Length of data payload to send

  // sms vars
  char sms_target[20];
  char sms_payload[160];
  volatile bool incoming_sms_flag;

  // Add these for receiving:
  char rx_sms_sender[64];
  char rx_sms_payload[160];
  volatile bool unread_sms;

  uint8_t sms_setup_step;
} ec200_driver_t;
```

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

This is the initial state to start things off with the FSM, where we send a simple `AT\r\n`, and then waits for the  module to respond with an OK or an ERROR.  You set both cmd_ack and cmd_ok as true or false - after receiving an OK or an ERROR respectively from the 4G module. 

## Echo_Off
```C 
  case EC200_Echo_Off:
    if (dev->cmd_ack && dev->cmd_ok) {
      printf("\r\n[FSM] Echo_Off -> Sending 'ATE0'\r\n");
      send_at(dev, "ATE0\r\n");
      dev->state_timer = now_ms;
      dev->fsm_state = EC200_SIM_State;
    } else if (now_ms - dev->state_timer > 2000) {
      printf("\r\n[FSM] Timeout in Echo_Off. Retrying AT_Sync.\r\n");
      dev->fsm_state = EC200_AT_Sync;
    }
    break
```

This just stops the 4G module from sending back  what you have sent to it. Just helps keep the UART cleaner with, a lot less chatter.  
## SIM_State 
```C 
  case EC200_SIM_State:
    if (dev->cmd_ack && dev->sim_ready) {
      printf("\r\n[FSM] SIM Ready -> Checking Network (CEREG?)\r\n");
      dev->status = EC200_STATUS_REGISTERING;
      send_at(dev, "AT+CEREG?\r\n");
      dev->state_timer = now_ms;
      dev->fsm_state = EC200_NW_Reg;
    } else if (now_ms - dev->state_timer > 3000) {
      printf("\r\n[FSM] SIM not ready. Retrying...\r\n");
      send_at(dev, "AT+CPIN?\r\n");
      dev->state_timer = now_ms;
    }
    break;
```

This is where we register the SIM tht we are using on the network, and officially - the fun stuff begins now. We will be dealing with the carrier infra and a bunch of things that are related to this, so buckle up. We use the command AT+CEREG, which queries through the SIM about it's registration with the carrier network. 
```
Note - For SMS, should we not also use AT+CREG? 
Answer - Not really, since modern Modems are used to responding to all comms via the LTE Network.
```

### Handling in the Rx_Callback
```C 
else if (strstr(line, "+CPIN: READY") != NULL) {
          g_driver_inst->sim_ready = true;
} 

//NOTE - Do we even need to check for CGREG here? We are not sending an AT command with that anywhere, lol. 
else if (strstr(line, "+CEREG:") != NULL || strstr(line, "+CGREG:") != NULL) {
          if (strstr(line, ",1") != NULL || strstr(line, ",5") != NULL) { 
			//if the response contains 1 - home network. If it contains 5 - roaming network.
            g_driver_inst->net_registered = true;
          }
}
```

If the ACK or OK is set as false, and if 3 seconds pass - we check for the SIM status, via the  command  `AT+CPIN? -- need a pin to be active? `, `AT+CGREG - are you registered at a GPR switched network`, or `AT+CEREG - are you registered on a packet switched network`.  And that's how we determine if the SIM in the modem is connected or not.  

## NW_Reg
```C 
  case EC200_NW_Reg:
    if (dev->cmd_ack && dev->net_registered) {
      printf(
          "\r\n[FSM] Network Registered -> Config SMS Text Mode (CMGF=1)\r\n");
      dev->sms_setup_step = 0;
      send_at(dev, "AT+CMGF=1\r\n");
      dev->state_timer = now_ms;
      dev->fsm_state = EC200_SMS_Setup;
    } else if (now_ms - dev->state_timer > 3000) {
      printf("\r\n[FSM] Network not registered. Retrying CEREG.\r\n");
      send_at(dev, "AT+CEREG?\r\n");
      dev->state_timer = now_ms;
    }
    break;
```

Now that we have confirmation that the modem is registered with the network, we can start sending and receiving things over said network, and resume the general operations that we are supposed to perform. Starting with - sending a damn SMS. To do that, we are setting the text mode of the SMS - via the command `AT+CMGF=1`. 
And if we have a timeout of over 3seconds, we are checking if the modem is still registered on the network, via `AT+CEREG?`. Simple enough, innit? 
## SMS_Setup

```C
  case EC200_SMS_Setup:
    if (dev->cmd_ack) {
      if (dev->sms_setup_step == 0 && dev->cmd_ok) {
        printf("\r\n[FSM] Config_CMGF -> Wiping SIM Memory (CMGD=1,4)\r\n");
        send_at(dev, "AT+CMGD=1,4\r\n");  // Deletes all internal SIM card messages
        dev->sms_setup_step = 1;
        dev->state_timer = now_ms;
      } else if (dev->sms_setup_step == 1) {
        printf("\r\n[FSM] Memory Cleared -> Routing SMS to UART "
               "(CNMI=2,2,0,0,0)\r\n");
        send_at(dev, "AT+CNMI=2,2,0,0,0\r\n"); // Network manager interface, which routes the incoming SMS to the MCU's UART Console
        dev->sms_setup_step = 2;
        dev->state_timer = now_ms;
      } else if (dev->sms_setup_step == 2 && dev->cmd_ok) {
        printf("\r\n[FSM] SMS Ready! Cleaning up stale network state.\r\n");
        enter_net_cleanup(dev, now_ms);
      }
    } else if (now_ms - dev->state_timer > 5000) {
      if (dev->sms_setup_step == 0)
        send_at(dev, "AT+CMGF=1\r\n"); //NW_Reg
      else if (dev->sms_setup_step == 1)
        send_at(dev, "AT+CMGD=1,4\r\n"); //Delete all SMS
      else if (dev->sms_setup_step == 2)
        send_at(dev, "AT+CNMI=2,2,0,0,0\r\n"); //Re-set the SMS routing to the MCU UART.
      dev->state_timer = now_ms;
    }
    break;
```

This is to setup the parameters of the module that helps take care of the SMS interface that we have planned for our logger. Basically, we first delete any old SMS messages that have been left unhandled on the 4G SIM card, and then tell the module which way is the SMS supposed to be routed via "AT+CNMI".  Once all that is done, we call the network cleanup helper, which basically closes any old MQTT connections that are now essentially dead, via the "AT+QMTCLOSE" command, and forcing the fsm into the Net_Cleanup State. 
## Net_Cleanup 

```C 
case EC200_Net_Cleanup:
    // Entered via enter_net_cleanup(), which has already sent QMTCLOSE.
    // Note: We ignore dev->cmd_ok here. If the socket/PDP context is already
    // closed, the modem returns ERROR. We don't care, we just want to ensure
    // it's closed. A timeout is treated the same way - move on regardless.
    if (dev->cleanup_step == 0) {
      // QMTCLOSE: OK/ERROR is quick, 5s is plenty
      if (dev->cmd_ack || now_ms - dev->state_timer > 5000) {
        printf("\r\n[FSM] Net_Cleanup -> Deactivating old PDP context "
               "(QIDEACT=1)\r\n");
        send_at(dev, "AT+QIDEACT=1\r\n");
        dev->cleanup_step = 1;
        dev->state_timer = now_ms;
      }
    } else {
      // QIDEACT: modem may take up to 40s to respond
      if (dev->cmd_ack || now_ms - dev->state_timer > 40000) {
        printf("\r\n[FSM] Cleanup Complete! Transitioning to Data APN.\r\n");
        // Closing the socket ourselves must not look like a link drop
        dev->mqtt_link_lost = false;
        dev->fsm_state = EC200_DataAPN;
        dev->state_timer = now_ms;
        // Seed the ACK so DataAPN immediately evaluates
        dev->cmd_ack = true;
        dev->cmd_ok = true;
      }
    }
    break;
```

Here, we take care of closing any and every connection that may or may not have been left idle before we move on to the MQTT side of things.  Now that the cleanup and closing of old connections is now done, we can go for initiating the PDP and eventually sending data.

## DataAPN
```C
  case EC200_DataAPN:
    if (dev->cmd_ack && dev->cmd_ok) {
      printf("\r\n[FSM] Configuring APN Context "
             "(QICSGP=1,1,\"airtelgprs.com\",\"\",\"\",0)\r\n");
      dev->status = EC200_STATUS_INITIALIZING;
      send_at(dev, "AT+QICSGP=1,1,\"airtelgprs.com\",\"\",\"\",0\r\n"); 
      //Command to send over the connection details that are needed for communicating with the wireless interwebs.
      dev->state_timer = now_ms;
      dev->fsm_state = EC200_Init_PDP;
    } else if (dev->cmd_ack && !dev->cmd_ok) {
      // Catch QIDEACT errors from the Init_PDP failure fallback
      printf(
          "\r\n[FSM ERROR] Setup Rejected. Restarting Network Checks...\r\n");
      dev->fsm_state = EC200_NW_Reg;
    } else if (now_ms - dev->state_timer > 3000) {
      printf("\r\n[FSM ERROR] Data APN Setup Timeout.\r\n");
      dev->fsm_state = EC200_NW_Reg;
    }
    break;
```

Basically, if the previous command has been acked and okayed, we send over details with whatever is needed for connecting to the network / data. The string that is being sent, has the following data - `AT+QICSGP=<contextID>,<context_type>,"<APN>","<username>","<password>",<authentication>` .  And of course, if this doesn't break or times out, we gotta go back to the [Network registration part of the FSM](#nw_reg) state. Why? It was indeed unnecessary. 

