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

The primary use case would be to send snapshots of the data that is being collected via the CAN bus that ties the feeder boards and the logger. These snapshot logs shall be periodic in their nature, ideally being sent once every 10 seconds or so. This shall act as a heartbeat of sorts for us, providing enough remote information about the system for us to ensure that things are working as expected. 

An SMS feature to update the MQTT parameters would be another function that gets integrated. A set format that shall be hardcoded into the firmware, shall parse an incoming SMS onto the module, and update the MQTT settings that have been set in place for the Connection, and re-establish the connection. There also needs to be a fall-back for this, should the update fail or should the connection to the broker fail to be re-established. 

Support for swapping SIM cards can be also considered. Say that the client wishes to swap out an Airtel SIM card for a JIO SIM instead. We can also enable provisions for updating things like the APN within the firmware via an SMS, BEFORE the new SIM card is inserted into the module. Another SMS feature would be remote trigger a log dump over the connection. This feature is contingent on a feasibility analysis, to be conducted as the 4G interface is worked upon.  

The SIM cards to be used for this are to be procured by the client, and any and all data plans are to be paid for by them as well. We shall recommend an IoT or M2M SIM card, from either JIO or Airtel to be used. In case the recommended carriers do not have network penetration at the deployment site, any suitable carrier should work just as well for us. 

## LoRa Module

## SD Card Interfacing

## Blackbox Application

# HMI design 

## Compute Selection

## Screen Selection

## Control Algo interfaces

  
**