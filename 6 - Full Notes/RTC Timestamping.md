[[RhyGen]]
![[RTC_Timestamp_Flow.excalidraw]]

The plan is really simple. We are using the 4G module to get the current time in Asia/Kolkata. Then, we call the RTC Set time function from the HAL ONCE, during the Initialization of the module. Then, in the logging task, we simply call the RTC get time function, and keep on timestamp-ing the log database. 

I am keeping the timestamp from the RTC at the elec_db timestamp, and the HAL_TICK timestamp at the mech_db timestamp. This is done, because there is an issue with the RTC. It is drawing it's clock source from the LSI oscillator, which is not the most stable clock source. The LLM suggests that there could be a few seconds' worth of drift per hour. Also, we are batching the logs into packets of 20, and that logic is based on timestamps. I need a stable and deterministic timestamp for that batching logic to work perfectly, and the HAL_GetTICK value works out perfectly for me. 

The AT Command that tells the 4G module to give me the local time here is `AT+QLTS`. The command should return something in the format as follows `<date>,<time><tz in quarter-hours>,<dst>`. We simply need to extract the time from this, that's in hh:mm:ss format. And then, add it to the timestamp of the elec_paams, and call it a day. 

If only it were that simple. 
See, we are logging at 10 hertz, which means there is a log once every 100ms. MILLISECONDS. And we aren't getting that from the AT Command response, so guess what - math time it is!

Thankfully, we have people much smarter than me who have figured this shit out. Here is how it goes - 
```
  // SSR counts down from PREDIV_S (SecondFraction) to 0 once per second.
  // ms = (PREDIV_S - SSR) / (PREDIV_S + 1) * 1000  (see HAL_RTC_GetTime docs).
  // SSR can briefly exceed PREDIV_S after a shift operation, so clamp to 0.
  uint32_t ss = sTime.SubSeconds;
  uint32_t frac = sTime.SecondFraction;
  time->milliseconds =
      (ss > frac) ? 0 : (uint16_t)(((frac - ss) * 1000U) / (frac + 1U));

```

SSR is the sub-second register that is present in the RTC that we have on the chip. A down counter loads a value called a PREDIV_S (synch predivider) into itself, and starts counting down. This value is 255 by default, and can be as high as 32,767.  We need 1ms resolution on the RTC, so we need to mess with the prescalers such the Down-Counter shall overflow exactly every 1 second. 

1Hz = Freq_RTC_CLK/((PREDIV_A+1)* (PREDIV_S+1))


### SetRTCTime 
The AT Command response starts with +QLTS and  lands in the rx_callback,  setup to catch all modem responses. It triggers a call to `ec200_parse_net_time`, where we first convert the hh:mm:ss time into Seconds of Day, and then assign the respective members of dev->net_time their appropriate values. The dev->net time struct is passed to the SetTime function of the PAL to set the reference for out RTC. 

### GetRTCTime
This is a simple call to the function when we are logging the elec params timestamp on the system. 

