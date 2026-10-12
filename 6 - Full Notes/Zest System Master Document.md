[[RhyGen]]
# Feeder Design

## H2 Feeder    

Dedicated board to the H2 Producer of the client system. 
### Hardware Components
There shall be 1 STM32F446 and 1 STM32F103 based dev boards in this system to take care of all the processing needs. The feeder shall interact with the Logger, via a CAN BUS, with a single can interface from both boards being on the same bus with that of the logger. Since the F103 has a single CAN peripheral, it's interaction shall be limited to receiving control signals that are sent from the HMI and routed through logger, while acquiring signals from the connected Digital and Analog Inputs and passing it on to the logger. 

The F446 has 2 CAN peripherals - one of which shall be allocated for interacting with the MPPT Array that shall be sending sensor inputs via their CAN channels. The second CAN Peripheral, shall be on the same bus with the F103, and the logger, with provisioned functionality for receiving control signals that are sent from the HMI and routed through logger, while acquiring signals from the connected Digital and Analog Inputs and passing it on to the logger.   
### Inputs and Outputs 
The Inputs that are to be received on this board are from specified sensors, and the outputs that emerge from either board are to drive relay driver. 
### Firmware Components

## Solar Feeder
This is a backup of sorts for us, since most of the sensing that's involved in this board shall be provided to us via the MPPT array. In case they are not able to source an MPPT that can publish data over CAN, then the associated sensors shall be procured by them, and this board shall be used in the system for data acquisition. In any case, the one constant feature that stays consistent is the 10 digital on-off switches that shall be present on the system. 

These are in place to drive relay contactors, so essentially - 10 MCU pins that connect to a relay driver IC, that connects to the customer's control circuitry. These can be turned on or off, based on the control signals that are sent from the HMI and routed through logger. 
### Hardware Components
2 Stm32F446 MCUs are set up in this system, primarily because we need all 16 channels of the MCU to be used. A total of 32 channels of the ADCs are needed to be utilized here, along with 10 GPIO pins that shall be used to drive a relay module, as requested by  the client. There shall also be 2 CAN transceivers, one for each MCU, that shall connect them both to the logger's CAN bus.  That's all that there is in this board. 
### Inputs and Outputs 
The Inputs that are to be received on this board are from the sensors that shall gather data from the solar panels, and the outputs that emerge from either board are to drive relay driver. 
### Firmware Components

# Logger Design

## 4G connectivity
We have chosen Quectel's EC-200N to be integrated with the Logger for providing 4G connectivity. Easy availability of the module was among the prime reasons for this to be our module of choice, along with a ready-to-use module based around this chip being plenty in stock on ROBU. 

The primary use case would be to send snapshots of the data that is being collected via the CAN bus that ties the feeder boards and the logger. These snapshot logs shall be periodic in their nature, ideally being sent once every 10 seconds or so. 
ASSUMPTION - Sending logs every 1 second hall be too heavy on the bw
This shall act as a heartbeat of sorts for us, providing enough remote information about the system for us to ensure that things are working as expected. 

An SMS feature to update the MQTT parameters would be another function that gets integrated. A set format that shall be hardcoded into the firmware, shall parse an incoming SMS onto the module, and update the MQTT settings that have been set in place for the Connection, and re-establish the connection. There also needs to be a fall-back for this, should the update fail or should the connection to the broker fail to be re-established. 

Support for swapping SIM cards can be also considered. Say that the client wishes to swap out an Airtel SIM card for a JIO SIM instead. We can also enable provisions for updating things like the APN within the firmware via an SMS, BEFORE the new SIM card is inserted into the module. Another SMS feature would be remote trigger a log dump over the connection. This feature is contingent on a feasibility analysis, to be conducted as the 4G interface is worked upon.  

The SIM cards to be used for this are to be procured by the client, and any and all data plans are to be paid for by them as well. We shall recommend an IoT or M2M SIM card, from either JIO or Airtel to be used. In case the recommended carriers do not have network penetration at the deployment site, any suitable carrier should work just as well for us. 

## LoRa Module
Nothing as of now is planned for this, it's all left for future integrations - particularly, when they'll need systems for the end user's cooking setup.
## SD Card Interfacing
The SD Card serves as the primary log storage for our system. It shall hold binary files, with decoding scripts on python that shall be held on our end. Any decoding for these shall be possible only with someone who has the scripts. The cards shall be 16 or 32 GB in size, depending on what the customer needs. It shall be formatted to FAT16/32 format, and the filesystem that is employed shall be the FATfs.

There shall be a defined size boundary for the logs that are being stored, with some space being left vacant for future extensions. There shall also be a cap on the size of each file, with a new file gets created once the current file overflows.

We shall have a roll-over logic for the number of log files that are stored in the card. There shall be a cap on the number of files that shall be held onto the card. Say, 100 - with the last written log file being deleted and re-written once that cap is breached.
## Blackbox Application
An FRAM Chip has been chosen for this, partially because of the page-less storage that it offers. This means that we (or the memory's internal controller) don't have to accumulate data to fit the page or the sector boundary, and we can simply stream the data that we have with us to the chip. 

This works perfectly for us in out application, since the purpose of a blackbox is to have historical data on it when a catastrophic event occurs. The plan is to maintain a ringbuffer on the FRAM, that is constantly written/streamed to. The size of the ringbuffer shall a little biger that what shall be needed to store as many logging struct's objects as can be produced within 1 minute. 

The ringbuffer shall be let overflow or overwrite and lose the oldest written data to it - which shall ensure that the last 1 minute's system data is held onto the memory chip, in the case of a catastrophic event occuring in the system, causing it to cease operations.   
# HMI design 

## Compute Selection
Get what's available. All it needs to have, is support for a screen. The rest shall be figured out as we go on. 
## OS Selection
his is the most interesting problem that we have to face in this project. The debate is between linux + DE (desktop enviromnent) and android + kiosk mode customization. The decision boils down to 2 things - reproducibility of the image and all custom scripts that are being written for it, and ease of integration of said scripts that shall be written for it. The advantage that I see at the moment is with Linux and a DE, and that boils down to the YOCTO build that we can have for it. 

YOCTO, is a build system that helps us create custom OS distros, all tailor made by us for the application that we have intended. Via bitbake recipes, we can integrate as many scripts as we want, at any depth  and degree of closeness to the linux kernel as we want, and ensure that the output binary that is flashed to the processor is consistent, across one or 100 such SBCs. 

Android, in kiosk mode, is what I have seen being used MULTIPLE times in such SBC based HMI systems. The integration of python scripts on this is questionable for me - simply because I have never done it or seen it being done. However, the beauty and smoothness of the UI that I have seen on kiosk-android deployments is unmatched - albeit the compute-heavy nature of Android. 

And to make this more complicated - we may very well see an entirely different compute unit being used in this system in the future. This means that the OS image should also be compatible on the newer SBC or compute mechanism that we see in the system. 

After talking to Gemini - YOCTO build for linux + a DE is the best option, and it boils down to the fact that YOCTO is build for shit like this. The meta-layers is all that needs to be swapped out for a change of SBC, and the bitbake scripts need to be setup ONCE for it to be reproducible as many times as we want it to be. Integrating custom python logic and the client's algorithm on this is much easier than Android, that apparently does NOT support native python on it. 

So yeah - as much as I hate to say it, all roads do indeed lead back to YOCTO.
## Screen Selection
A 7-inch touch screen display seems to be ideal for our application. One with a USB-C interface shall be even better. We need to have a Desktop environment for this on the SBC, and a bunch of python scripts that shall help visualize the system's parameters. 

The major challenge here is the re-producibility of the OS image, along with these python scripts that we shall have. 

As much as I fucking hate to say this - we might need a YOCTO build for this. 
## Control Algo interfaces
These shall be simple python function interfaces that shall be exposed to the client's control algorithms. The function shall trigger a message to the logger, with information about which Output to actuate, and the logger shall pass on control signals for the same to the respective feeder boards. For the Input interfaces, the python function shall simple request the specified parameter form the logger, and the logger shall return the last known value of the said input sensor to the SBC. This shall be used by the client's control algorithm to do, well, whatever. 


