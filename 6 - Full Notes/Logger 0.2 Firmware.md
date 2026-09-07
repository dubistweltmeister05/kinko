[[RhyGen]]

Right. Time to think and draw and do a bunch of research about the way things are supposed to be done for the next version of the logger.  As of now, I have a working J1939 and RS485 interface, and now what remains is the blackbox and the 4G connectivity. I need to come up with a structure to interface and integrate these two with the logger - keeping the firmware ready as the hardware guys do their jobs. 

# 4G interface

The good news here is that we have a minimal interface setup between the MCU and the 4G module. I have a working UART module that helps me talk to the module via AT commands, that are sent over from the MCU as strings. I need to test out basic MQTT communication over this interface, and once that is up and running, I think I can plan out the entire wireless pipeline. I have plans for FOTA and OTA and much more, but I think remote telemetry of all the logs that have been collected needs to be the first priority for immediate development. 

My first thought for the 4G interface is to simple send over the binary files that are being stored to the SD card, over on the wireless interface that we'll now have access to. But I am not sure how feasible will that be. Sending binary files over what is essentially a UART connection from the MCU to the 4G logger might not be a good idea. 

The next thought is to send over customized strings of the logs that are being collected - albeit at a slower rate than the logging at the SD card, essentially sending snapshots of the state of the system that is being logged locally at a much faster rate locally.  This basically acts as a health-checkup on the system, while maintaining enough context about the system for these to be useful. Alternatively, there can be an SMS interface, where if we send an sms to the number that's on the 4G module, it should send over the binary file that contains the master logs over the MQTT or an HTTP/FTP interface The quectel module supports commands for their execution, so these should work out just fine for us. IDK how this'll really work, or even be feasible. But this would be a lovely addition. 

Apart form the binary file dump, I'll also love for the SMS functionality to be used to update the MQTT params. Maybe have it update the URL and topics to which it publishes via the SMS interface that we have. 

FOTA and OTA pipelines for the thing shall be worked upon later. I got 0 clue about it all, but that's what we are here for - research, learn, and implement what we can. GG. 

---
Post conversation 1 with GEMINI

The Tx of binary files, that will be triggered by an SMS, over MQTT is a bad idea. A better idea would be supporting HTTP or FTP and sending larger binary data over it. The SMS for trigger is a nice idea, and sending snapshots of the data that is being logged over MQTT is a nice way of going about things. 

per gemini - 
Since you are already using the EC200U, leverage its built-in TCP/IP stack for file transfers. When the STM32 receives the SMS trigger, have it switch from MQTT to an HTTP POST or FTP PUT routine. The Quectel module has dedicated AT commands specifically designed for bulk data uploads (e.g., the AT+QHTTPPOST or AT+QFTPPUT families). These commands handle the heavy lifting of TCP windowing and buffer management, making it far easier to stream chunks of your binary file directly from the SD card / QSPI through the UART to the server.

You essentially get the best of both worlds: MQTT for your live, lightweight telemetry dashboard, and HTTP/FTP invoked via SMS for heavy-duty log extraction.

*HTTP or FTP over 4G. I should be writing my master's thesis over optimizations for these techniques, lol.* 

---
## Formal the plan for 4G on logger 

- Objective 1 - MQTT Publication of System Data string.
	1.  This does seem to be a good starting point for using the 4G module to start things off. We can send strings that hold all the logged values of the system, say, once every 5 or 10  seconds or so? We can look for ways of compressing the data that is being sent over the air, so that the bandwidth is not affected too much, and we do not jam our servers with too much data, too soon.   
	2.  Also, WE NEED TO FIGURE OUT A WAY TO HAVE DEVICE IDs ON EACH AND EVERY LOGGER, SO THAT WE CAN MAKE SENSE OF THE THINGS THAT WE SEE ON THE SERVER!
- Objective 2 - SMS Integration for MQTT parameter configuration.
	1. This would be a good step 2. The goal would be that if we know the number of the SIM card that is deployed in any logger, a simple SMS with a defined format can be sent to the number in question, which gets parsed and updated for the 4G module's firmware. 
	2. This would include things like the MQTT server's IP, port, topics, and anything else that we can come up with that needs to be configured.  
- Objective 3 - HTTP/FTP Integration for Sending large File Chunks of the Log. 
	1. This is the big boy task IMO. Figuring out how to use HTTP or FTP to send a big file, the command for doing this being input via an SMS seems to be a big challenge from the outside. We will need to define how does the "Big File" that contains all the logs look like, where shall it be stored, shall it be the same as the SD card, and if so - that becomes a whole lot messier than usual. 
	2. If the file that we intend to send over HTTP is the same that is being stored in the SD card, this would mean that we need to stop logging to the card, figure out a way to get that file out of the SD card, chunk it up and send it over the 4G interface that we have configured, using HTTP or FTP protocol commands, and then resume logging to the SD card. We also need to figure out a way to NOT LOSE any logs while we are at it. 
	3. Another way to do this would be to keep storing the logs as we record them, at 10 Hz to a local JSON file, and then when the SMS command is received, we simple ship this file over 4G. The downside? Space. I am not sure how feasible this is, so I need to research about this with an LLM. 

# BlackBox Update 
This is what I'mma be working on. https://www.infineon.com/part/FM25V02A-GTR

NO fucking clue about how to interface with it, so I gotta take a look online. But the general mode of operation here is supposed to be that if there is a loss of primary power and the system shuts down, this memory should have a backup log of the last 1 minute of the operation. 

One was I think I can go about doing this is to maintain a ring-buffer that holds an array of objects of the logging structure, that gets pushed to and is simply let overflow once it fills up. The oldest object is discarded, and gets overwritten as the ring buffer is pushed to. It's size shall be pre-calculated to store exactly one minute worth of logs, so if the device shuts off, we have the last 1 min of data. 

The driver for this shall support writes to the memory, and a read from the memory that get re-written to either a debug UART or something else, the can be figured out later. So that if we take the PCB for the logger and try to peek at the device's FRAM via the UART of the MCU, I should be able to send the MCU a UART command of some sort, and I should get a dumb of the ring-buffer that was stored in the FRAM. 

