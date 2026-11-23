[[RhyGen]]

Welcome to hell. This is about to be fun. 


## Goal
We have an I2C interface with the BNO055, and I need to figure out which commands to send, and in what format to do that for me to get the data from all 3 of it's sensors.

I better do it in the dummy repo or a separate branch of the logger 0,2 code, so that the board config does not beat my ass when I get to it. 

Also, thank GOD for bosch and it's datasheets. 

First thoughts are that I'll run it in IMU (fusion) mode mode, and get the accel and gyro data. But for just that, I don't think I will be using the sensor to it's full capacity. Also, this is primarily going to be used in vehicles, so I need to find out how is an IMU actually used there.  