[[RhyGen]]

IO EXPANDER INTERFACING https://www.youtube.com/watch?v=GyHiSyoyk_0&t=86s

# 1. System Overview

The customer currently operates a hydrogen generation plant using a PLC-based architecture.

The proposed solution will replace the PLC with a modular embedded system consisting of:

- Hydrogen Plant Feeder Board
- Solar Plant Feeder Board    
- Central Logger Unit    
- Raspberry Pi HMI Gateway    
- IP Cameras    
- LTE Connectivity  
The architecture is intentionally modular to allow future expansion without redesigning the complete system.

---

# 2. High-Level System Architecture

```text
+------------------------------------------------+
|          HYDROGEN GENERATION PLANT             |
+------------------------------------------------+

                ┌─────────────────┐
                │   Solar Plant   │
                │  Feeder Board   │
                └────────┬────────┘
                         │
                         │ CAN
                         │
                         ▼

                ┌─────────────────┐
                │                 │
                │     LOGGER      │
                │                 │
                └────────┬────────┘
                         │
                         │ CAN
                         │
                         ▼

                ┌─────────────────┐
                │ Hydrogen Plant  │
                │  Feeder Board   │
                └─────────────────┘


                         │
                         │ Ethernet
                         ▼

                ┌─────────────────┐
                │ Raspberry Pi    │
                │ HMI Gateway     │
                └──────┬─────┬────┘
                       │     │
                       │     │
                Ethernet Ethernet

                       │     │

                 IP Cameras
                   (4x)

LOGGER LTE
    │
    ▼
Remote Monitoring / Alerts

Optional Future:
Remote Camera Access
```

---

# 3. System Partitioning

The system is divided into distinct functional blocks.

## Hydrogen Plant Feeder Board
Responsible for:
- Sensor acquisition    
- Output control    
- Local safety monitoring    
- CAN communication    

Not responsible for:
- Data logging    
- Cloud communication    
- User interface    
- Camera handling    
---

## Solar Plant Feeder Board

Responsible for:
- MPPT monitoring    
- Solar array monitoring    
- Irradiance measurement    
- CAN communication    

Not responsible for:
- Data logging    
- User interface    
- Camera handling    
---

## Logger

Responsible for:

- Data aggregation    
- CAN master functionality    
- LTE communication    
- Alarm handling    
- Historical logging    
- Local storage    

Acts as the central controller within the system.

---

## Raspberry Pi HMI Gateway

Responsible for:
- Local operator interface    
- Dashboard rendering    
- Historical visualization    
- Camera integration   
- Configuration interface    

Provides a monitor, mouse, and keyboard-based interface.  (Touchscreen support may be added in future revisions.)

---

# 4. Hydrogen Plant Feeder Board Requirements

## Inputs

### Digital Inputs

|Parameter|Count|
|---|---|
|Multipurpose Digital Inputs|18|

---

### Analog Inputs

|Parameter|Count|Range|
|---|---|---|
|Multipurpose Analog Inputs|16|4–20 mA|
|Voltage Inputs|2|0–2 V / 12–60 V|
|Current Inputs|4|0–100 A / 0–1000 A|

---

### Sensors

Expected sensors include:

- Pressure sensors    
- Temperature sensors    
- Voltage sensors    
- Current sensors    
- Liquid level sensors    
- Electrolyte concentration sensors (TBD)    

Most sensors are expected to provide:
- 4–20 mA outputs    

Additional digital sensors may also be present.

---

## Outputs

### Digital Outputs

|Parameter|Count|
|---|---|
|Relay Outputs|15|

Supported output voltages:

- 12 V    
- 24 V    
- 48 V    
---

### Analog Outputs

|Parameter|Count|
|---|---|
|4–20 mA Outputs|5|

---

## Future Expansion

Board should provide spare resources:
- 2–3 additional digital inputs    
- 2–3 additional analog inputs    
to accommodate future requirements.

---

# 5. Solar Plant Feeder Board Requirements

## Architecture
Each board supports one solar module consisting of:
- 7 MPPT channels    

The architecture must support expansion to multiple modules.
Examples:

- 7 MPPT    
- 14 MPPT    
- 21 MPPT    
- 49 MPPT    
- 
Multiple feeder boards shall communicate over CAN.

---

## Inputs

### MPPT Monitoring

For a single 7-MPPT module:

|Parameter|Count|
|---|---|
|MPPT Input Voltage|7|
|MPPT Input Current|7|
|MPPT Output Current|7|

---

### Additional Measurements

|Parameter|Count|
|---|---|
|DC Bus Voltage|1|
|Solar Irradiance|1|

---

### Master Board Only

The master board additionally measures:
- Solar irradiance    
- MPPT output voltage
- DC bus voltage    
because all MPPT outputs are connected to the same DC bus.

---

# 6. Communication Architecture

## Primary Field Bus

CAN shall be used as the primary communication bus.

Benefits:
- Industrial robustness    
- Noise immunity    
- Long cable lengths    
- Multi-node support    
- Simple expansion    

---

## Proposed CAN Network

```text
Solar Feeder #1
Solar Feeder #2
Solar Feeder #3
Hydrogen Feeder
Kitchen Board (Future)

        │
        │
        ▼

      Logger
```

Logger acts as:

- CAN Master    
- Network Coordinator    
- Data Concentrator    

---

# 7. Data Logging Strategy

## Local Storage

Customer preference is local storage. No mandatory cloud storage requirement currently exists.

---

## Logger Storage

Suggested storage:

- 64 GB eMMC  
    or
- 64 GB Industrial SD Card
Stores:

- Telemetry    
- Alarms    
- Events    
- Diagnostic information    

---

## Logging Frequency

Configurable by user:

|Mode|Update Rate|
|---|---|
|Fast Monitoring|1 second|
|Normal Logging|1 minute|
|Custom|1 s – 5 min|

Configuration should be possible through the local dashboard.

---

## Alarm Logging

Upon alarm occurrence:

- Increased logging frequency    
- Full parameter capture    
- Event timestamping    
- Historical retrieval    

---

# 8. LTE Connectivity

The logger shall contain LTE connectivity.

Primary functions:

- Alarm notifications    
- Fault reporting    
- Remote monitoring    
- Remote actuation    
---

## Remote Control

Customer requirement:

> Remote user should be able to issue commands and actuate plant functions.

Therefore:

- Secure command channel required    
- Command acknowledgement required    
- Event logging required    

---

# 9. HMI Architecture

Customer requires:

- Monitor support    
- Mouse support    
- Keyboard support    
Touchscreen is not mandatory for the initial release.

---

## Raspberry Pi Functions

### Dashboard

Display:

- Plant status  
- Solar status    
- Alarm    
- ---

### Historical Data

Display:

- Historical telemetry    
- Alarm history    
- Diagnostic logs    

---

### Configuration

Allow adjustment of:

- Sampling rates    
- Alarm thresholds    
- Network settings    

---

# 10. Camera Integration
Need clarification on the need and nature of this 
## Proposed Architecture

```text
IP Camera
      │
      ├── Ethernet Switch
      │
      ▼

Raspberry Pi

      ▼

Local Dashboard
```

---

## Recommended Protocols

- RTSP
- ONVIF
---

## Initial Scope

Version 1:

- Local viewing only

Future:

- Remote viewing
- Recording
- Event-triggered snapshots

---

# 11. Preliminary Hardware Recommendations

## Analog Front End

4–20 mA inputs:

```text
Sensor
  │
250 Ω Precision Shunt
  │
1–5 V Signal
  │
ADC
```

Recommended ADC resolution:

- 16-bit minimum

---

## Current Measurement

Recommended technologies:

- Hall-effect sensors
- LEM current transducers
- Closed-loop current sensors

Avoid direct shunt measurement for high-current channels.

---
