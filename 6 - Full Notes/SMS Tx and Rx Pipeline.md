[[RhyGen]]

## RX Pipeline
Everything that is to be received from the MODEM and made available for the MCU to process is handled in the `ec200_uart_rx_cb` function. EACH command is either Acknowledged with an OK or ERROR response from the Modem, so we store that in the `dev->cmd_ack` and `dev->cmd_ok` fields. Both these fields are set as false whenever a command OR a string is sent to the Modem via UART, so that the responses can properly acknowledge them in the callback. 

Now - for the actual code logic. 
1. validate Inputs. 
2. Call the UART read, and store the responses into a byte variable of size uint8_t.
3. Assign a char* pointer `line` to the address that is currently pointed to by the `rx_idx` - just to make sting operations a little simpler. 
4. On the basis of what has been read into the byte variable, we decide if we drop it or parse it, or do something else with the same. 
	1. A normal byte, that is not `\r` or  `\n` is stored into the rx_line array at the rx_idx position. This helps build up the modem's response. It may send us a lot of bytes, but they are decimated into individual bytes when popping from the configured Rx Buffer. Hence, we reconstruct that response when we need to take actions on it. Particularly useful when we might receive a long response on the SIM.
	2. **`\r`**: thrown away. The modem ends lines with `\r\n`, so `\r` carries no information.
	3. **`\n`**: the line is finished.
		- A `'\0'` goes in at `rx_line[rx_idx]`  turning the buffer into a proper C string.
		- If `rx_idx > 0`, the line is parsed. Empty lines (the blank `\r\n` the modem sends around responses) are skipped.
		- `rx_idx = 0`  This happens after every `\n`, empty line or not.
5. Once a line ends, we simply reset the index pointer, which leads to overwriting of the array upon the reception of a new byte.

```
/* 1. Trap SIM-saved messages (+CMTI)

        //+CMTI is the URC for Cellular Message Tracking Indication. This means
        that there is a received message that needs to be read. Also that the
        message is stored in the SIM! +CMTI: "<storage_type>",<memory_index>

        • <storage_type>: Tells you where the message is saved. Common tags
        include "SM" for the SIM card memory, or "ME" for the modem's internal
        equipment memory. • <memory_index>: The specific memory slot number
        where the SMS was saved
        */
```

```
/* 2. Flexible match for direct messages (+CMT)
        This means that there is indeed an SMS that was received, and has been
        routed to the configured interface. (Any serial interface - UART in our
        case)

        LINE 1 - +CMT: "+1234567890","", "26/10/14,17:30:00+22"
        LINE 2 - Hello, this is the message body text! (Payload)
        */
```

```
 /* Response Payloads for SIM registeration
        These are used to check if the SIM is initialized or not.

        CPIN - Checks for SIM status

        CEREG - Check if the SIM is registered on the Evolved Packet System

        CGREG - Checks for GPRS Network registeration system.
        */
```

```
/* Network time (reply to AT+QLTS=2)
        +QLTS: "2026/11/17,15:42:07+22,0"
        <date>,<time><tz in quarter-hours>,<dst>
        If the network never sent NITZ (Network Identity and Time Zone), the
        modem replies +QLTS: "" and this doesn't match.
        */
```

```
/* 6. Response Payloads for MQTT Connection and pipeline
        These are used to track the status of MQTT connections and message
        publishing operations.

        QMTOPEN - Checks the result of opening an MQTT network connection

        QMTCONN - Checks the result of establishing an MQTT client connection

        QMTPUBEX - Checks the result of publishing an MQTT message and tracks
                   the corresponding message ID

        QMTSTAT - Indicates that the MQTT connection status has changed,
                  typically due to a connection loss.
        */
```

## Tx Pipeline
The send_at command is what triggers all the Tx that happens to the 4G module. We have documented all the AT command interactions that are set in place in the note [[4G Uplink Codebase]] , so much time shall not be spent here. Instead, this is a proposal of how SMS TX could be integrated. 

No it's not. The whole thing is pretty simple - AT+CMGS and your destination info that's copied into a buffer. Modem handles the rest. 