# Feeder Design

## H2 Feeder    

Dedicated board to the H2 Producer of the client system. 
### Hardware Components
There shall be 1 STM32F446 and 1 STM32F103 based dev boards in this system to take care of all the processing needs. The feeder shall interact with the Logger, via a CAN BUS, with a single can interface from both boards being on the same bus with that of the logger. Since the F103 has a single CAN peripheral, it's interaction shall be limited to receiving control signals that are sent from the HMI and routed through logger, while acquiring signals from the connected Digital and Analog Inputs and passing it on to the logger. 

The F446 has 2 CAN peripherals - one of which shall be allocated for interacting with the MPPT Array that shall be sending sensor inputs via their CAN channels. The second CAN Peripheral, shall be on the same bus with the F103, and the logger, with provisioned functionality for receiving control signals that are sent from the HMI and routed through logger, while acquiring signals from the connected Digital and Analog Inputs and passing it on to the logger.   


## Inputs and Outputs 
The Inputs that are to be received on this board are from specified sensors, and the outputs that emerge from either board are to drive relay driver. 
### Firmware Components

## Solar Feeder
This is a backup of sorts for us, since most of the sensing that's involved in this board shall be provided to us via the MPPT array. In case they are not able to source an MPPT that can publish data over CAN, then the associated sensors shall be procured by them, and this board shall be used in the system for data acquisition. In any case, the one constant feature that stays consistent is the 10 digital on-off switches that shall be present on the system. 

These are in place to drive relay contactors, so essentially - 10 MCU pins that connect to a relay driver IC, that connects to the customer's control circuitry. These can be turned on or off, based on the control signals that are sent from the HMI and routed through logger. 
### Hardware Components
2 Stm32F446 MCUs are set up in this system, 

### Firmware Components

# Logger Design

## 4G connectivity

## LoRa Module

## SD Card Interfacing

## Blackbox Application

# HMI design 

## Compute Selection

## Screen Selection

## Control Algo interfaces

  
**