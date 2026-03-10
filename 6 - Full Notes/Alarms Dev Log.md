[[Twitter Posting]]
[[Titan]]

Right, I just had my PR merged into the main codebase, so here is a dev log of an embedded engineer. 

I work for an industrial IoT company, and one of our products is a 3-phase industrial energy meter. I was in charge of developing a Parametric Alarm system for the same. The idea here, is that there should be a module in the systems that can keep track of the measured quantities, that is the voltage, the current, and the frequency of the incoming signal, the order of the phases, as well as some derived quantities, such as the Max Demand. 

The module shall allow the user to configure set-points for these values, in an over/under/binary manner. If the configured setpoint has been breached, there shall be an alarm that gets triggered. Upon trigger, there shall be a custom screen that gets displayed on the On-Board LCD screen on the  device, and the instance, along with the time and date stamp shall be recorded into a history log that shall be available for on-demand access.  

Additionally, there are digital outputs that are available on the device, and the user can configure a particular DO (Digital Output) to perform an action once the alarm is triggered. For example, the user can connect a DO to a bell, and if an alarm is triggered, the DO can turn on the bell to physically alert the workers. The user can enable/disable these alarms, and configure it's behavior, like the setpoint, the DO actions, the sustain time, and a few other parameters via Bluetooth, and via MQTT. T

he work was quite fun, since I was in charge of making the whole thing from scratch. The arch, the flow or the logic, the development timelines, the deliverables, everything was under my control. I was responsible for developing the alarm manager FSM, the init and config functions, the threshold evaluations, the memory management and the persistence behavior, figuring out a way to record and store the history of alarms, designing the screens and integrating them with the firmware, and writing a whole parser for the BLE/MQTT too. 

Took me about 3 weeks to implement this end-to-end, and yes, I did use AI for this. A lot of the generated code was bug filled, causing hard-faults, so I pivoted to a half-and-half approach. I fed Opus a lot, and I mean, like 10 PAGE long markdown files that I wrote, which included the entire architecture, a list of the functions to be implemented, a list of the macros that were to be designed, and finally, EACH and every function with pesudo-code about what it was supposed to do. Literally, step by step instructions. And then, once I had GitHub Co-Pilot work the hard yards with Opus as the model, it was my job to debug the generated output. It fucked up quite a bit, but nothing that I couldn't handle. 

If you've read so far - well, thank you so much! 
