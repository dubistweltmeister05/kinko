[[RhyGen]]
[[Twitter Posting]]

# The Problem
While using the EC200N module for the 4G uplink on the Logger via UART, I noticed that on power-up, both the  MCU and the UART Module were turning on at the same time. But, if left unattended, the 4G module never started publishing to the remote link that we have. I got a bit lucky with thiss,,, by using the old adage - "band karke first chaalu kar" - because it started working and publishing data for literal days without interruption when I reset the MCU once. After leaving the fix for a later date - today came the time to fix that. 1 damn day before I go to deploy the product at a factory 20KM away, because that's how engineering is done, of course. 

# The Fix
I thought that the issue was due to the bootup time of the 4G module itself. Basically, the MCU was sending a bunch of init commands to the module while it was still booting up, leading it it being locked into an idle state. Of course, something could be added to the FSM on the MCU to fix that, but I thought of a shorter fix - add a 20 second delay before we send ANYTHING to the 4G module. Let it boot up in peace, and then we harass it with out commands. 

I thought that it'd work for sure, but oh well, it didn't. And this is the point where I gotta thank GOD for LLMs, because I'd have taken hours to stumble upon the following info. The 4G module was streaming a bunch of Unsolicited Response Codes - URCs to the MCU, WHILE it was booting up along with the controller. And since we are using DMA for receiving data via UART, this led to the UART Rx Buffer being full before the MCU could so anything about it, effectively leaving the UART communication pipeline frozen. 

While using DMA, if the allocated buffer overflows - the MCU does not overwrite it, but drops the DMA pipeline all together! This is done to essentially protect teh memory from being corrupted. Sadly, this does end up leading the MCU to be DEAF over the UART line, meaning even when the firmware FSM tries to initialize the 4G module via the usual AT command, it never heard back the OK or ERROR that is sent by the module as response, deadlocking the system!

To mitigate this, the fix is quite simple. I had to implement an error handler, via the UART_ErrorCallback function. This gives me a way to implement custom code when an error flag in the processor is set as true. In this case, all I had to do was clear the internal DMA buffer by setting the read pointer back to zero, and re-call the DMA listen on the MCU - and BOOM! 

The deadlock was no more, and the code works exactly as it is supposed to now, with real-time log streaming on my dashboard, from technically, anywhere in India! 

