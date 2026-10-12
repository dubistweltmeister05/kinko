[[RhyGen]]
[[Tor]]

**Subject: Firmware modification request: Faster MODBUS update rate for Power parameters**

Hi Chetan,

I hope everything is going well! I’m currently using the Tor Titan 212 for a new application on my end, and I've run into a real-time polling requirement where I need the Active and Reactive Power measurements to update significantly faster over MODBUS.

While integrating the meter and profiling the communication using a Modbus scanner polling every 20ms, I noticed a discrepancy in the refresh rates across different registers. The THD values are updating incredibly fast (looks to be around every 80ms), but the core electrical parameters—specifically Active and Reactive Power, Voltage, and Current—are lagging, only changing their values every 700-800ms.

Given the difference, I am guessing the firmware applies an averaging filter to the V, I, and P readings for noise reduction before exposing them to the Modbus registers, while the THD values bypass this filter. For my current control application, I need the Active and Reactive Power values to update much faster—ideally matching that ~80ms base cycle, or at least in the 80-160ms range. My system can easily handle a slightly higher noise floor or measurement jitter in exchange for the speed.

Is it possible to request a custom firmware build with a reduced filtering window for these parameters? Please let me know what the process and lead time would be for this tweak. My organization is exploring the possibility of partnering with Tor.ai for acquiring upto 20 more MFMs in Q4, 2026 and even more so in the next quarter, but that remains contingent on the request detailed above. I am also happy to get on a call, to elaborate on my request and discuss a possible solution. 

Best regards,

Kshitij Neeraj Vaze,
Firmware Engineer at Tor.ai,
Contact Number - 9321652242