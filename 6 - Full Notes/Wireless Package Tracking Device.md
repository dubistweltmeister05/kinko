[[Embedded System Design 02 - Wireless Joystick Controller]]
![[Pasted image 20260329151649.png]]
# The Goal

The product is supposed to track the real-time location of a shipping container that will be transported via road. The requirement here is to have real-time geo-location of the container, accessible via an app or a web interface. The location shall be accessible from anywhere, provided that the user has an internet connection.

The goal is to design a complete embedded system for this product — starting from the hardware building blocks required for a basic working unit, to the data packet that will be transmitted to the cloud. Additionally, we will explore the real-world shortcomings of such a system and propose practical solutions to address them.

---

# The Blocks

## 1. The Main Microcontroller

As the brains of the entire system, the choice of the MCU becomes one of the most critical decisions.

The primary requirement here is **ultra-low power operation**. The device is battery-powered, wireless, and expected to operate continuously for anywhere between 1–15 days depending on the shipping duration.

This makes the **ARM Cortex-M series** an ideal choice. These microcontrollers are specifically designed for low-power embedded applications while still providing sufficient computational capability.

To optimize power consumption further, several techniques can be employed:

- Deep sleep / standby modes
- Sleep-on-exit for interrupt-driven execution
- Clock gating for unused peripherals
- Proper GPIO configuration (e.g., pull-downs to avoid floating states)
- Event-driven firmware instead of continuous polling

In this system, the MCU acts as the **central orchestrator**, handling:

- Sensor data acquisition (GPS, IMU, BMS)
- Data logging to external memory
- Communication with the cellular module
- Power state management of the entire system

---

## 2. GPS Module – Positioning System

This forms the core functionality of the device.

The GPS module is responsible for determining the real-time geographical location of the container using satellite-based positioning. Under ideal conditions, it can provide accuracy in the range of **2–5 meters**.

However, GPS is both **power-intensive** and **environment-sensitive**:

- It consumes significant current during operation
- It struggles in enclosed environments (like shipping containers)

To mitigate this:

- The GPS module is activated **periodically**, not continuously
- It can be triggered based on:
    - Time intervals (via RTC)
    - Motion detection (via IMU)

This ensures that location tracking is achieved without unnecessary battery drain.

---

## 3. Cellular Connectivity (NB-IoT / LTE-M)

To transmit data to the cloud, the system uses a **cellular communication module**, specifically **NB-IoT or LTE-M**, instead of traditional GSM.

These technologies are designed for IoT applications and offer:

- Lower power consumption compared to GSM
- Better penetration (critical for containers and indoor scenarios)
- Optimized handling of small data packets

The cellular module is responsible for:

- Establishing a connection with the network
- Transmitting location and sensor data to the cloud server
- Retrying transmission in case of failures

To ensure robustness, the system can be designed with:

- **Store-and-forward capability** (using external flash)
- Multi-mode support (NB-IoT + LTE-M fallback where available)

---

## 4. Power System (Battery + BMS + Regulation)

The power subsystem is one of the most critical aspects of the design.

It consists of:

- **Battery** – primary energy source
- **Battery Management System (BMS)** – ensures safe operation
- **Power Regulation (Buck/Boost converters)** – stable voltage supply

The BMS is responsible for:

- Over-voltage / under-voltage protection
- Over-current protection
- Monitoring battery health and charge level

Additionally, due to the **high current bursts** required by cellular modules, the design must include:

- Bulk capacitors or supercapacitors
- Proper power regulation to prevent voltage dips and system resets

Efficient power design directly determines the **reliability and lifespan** of the device.

---

## 5. IMU – Motion and Orientation Detection

The IMU (Inertial Measurement Unit) adds an important layer of intelligence to the system.

While it can be used for orientation tracking in sensitive shipments, its more critical role in this system is **event detection**:

- Detecting motion → wake up the system
- Detecting shocks → log potential damage events
- Detecting tilt → identify mishandling

Instead of keeping the system active continuously, the IMU enables an **event-driven architecture**, significantly improving power efficiency.

---

## 6. External Non-Volatile Flash Memory

A crucial addition for real-world reliability.

Since cellular connectivity is not guaranteed at all times, the system must be capable of storing data locally.

The external flash memory is used to:

- Buffer GPS and sensor data
- Store unsent packets during network outages
- Enable **store-and-forward transmission**

This ensures that no critical tracking data is lost due to temporary connectivity issues.

---

## 7. Real-Time Clock (RTC)

The RTC enables precise timekeeping with extremely low power consumption.

It is used to:

- Wake the MCU at predefined intervals
- Schedule GPS acquisition
- Timestamp logged data

This allows the system to operate in a **duty-cycled manner**, which is essential for battery-powered devices.   
# The Data Packet

The device transmits a compact data packet to the cloud at regular intervals or on event triggers. Since the system is constrained by **power and bandwidth (NB-IoT / LTE-M)**, the packet must be **minimal, structured, and efficient**.

The packet contains the following:

---

1. Device ID

A unique identifier assigned to each tracking unit. This is used to:

- Associate incoming data with the correct container
- Enable fleet-level tracking and management

---

2. Timestamp (RTC)

A precise time and date value generated using the onboard RTC.

This ensures:

- Accurate tracking history
- Proper ordering of events (especially in store-and-forward scenarios)

---

3. Battery Status

Indicates the remaining battery level of the device. This can be:

- Percentage-based (0–100%)
- Or raw voltage (more precise, less user-friendly)

This allows:

- Predictive maintenance
- Alerts for low battery conditions

---

4. GPS Data

The core payload of the system.

Typically includes:

- Latitude
- Longitude
- (Optional) Speed and altitude

This provides real-time location tracking of the container.

---

5. IMU Data (Event-Based)

Instead of streaming raw IMU data (which is expensive), the system transmits **processed information**.

For example:
- Deviation from calibrated orientation
- Shock detection flag
- Motion status (moving / stationary)

This helps identify:

- Mishandling
- Drops or impacts
- Unauthorized movement

# The working of the system. 
Let’s discuss a few edge cases and the general management of the system.

Once the data packet is formed, the system operates in a duty-cycled manner to optimize for power. The device wakes up at fixed intervals (say, every 10 minutes) using the RTC within the MCU. Upon wake-up, the MCU enables the required peripherals, collects GPS and IMU data, forms the packet, and initiates transmission via the NB-IoT (or LTE-M) module. After sending the packet, the system waits for an acknowledgment from the cloud. If the transmission is successful, the system is put back into a deep sleep state until the next scheduled wake-up. In case of a failed transmission, the packet is written to external flash memory for retry at a later time.

It is important to clarify that the BMS is not responsible for putting the system to sleep. The MCU controls the power states of the system, selectively disabling peripherals and entering low-power modes, while the RTC remains active to trigger the next wake-up event. This distinction is crucial, as power management in embedded systems is largely a firmware responsibility.

Accident detection can be implemented using the IMU by observing abnormal orientation changes. For example, a sustained deviation in the roll (or pitch) axis, followed by a lack of significant acceleration, can indicate that the package has been tipped over or dropped and is now stationary in an unintended orientation. Such events can be flagged and either logged for later transmission or immediately pushed as high-priority alerts when connectivity is available.

During transport layovers, the system can infer a stationary condition by combining multiple inputs. If the IMU reports near-zero acceleration, the orientation remains stable, and the GPS coordinates do not change across multiple sampling intervals, the system can conclude that the container is stationary. The RTC timestamp provides additional context, allowing the backend to identify patterns such as overnight stops without falsely classifying them as anomalies.

In scenarios such as tunnels, dense forests, or other environments where cellular connectivity is temporarily unavailable, the system falls back to a store-and-forward mechanism. All observed readings are written to local non-volatile flash memory along with their timestamps. Once connectivity is restored, the device transmits the buffered data in sequence, ensuring that no tracking information is lost. This allows the system to trade off real-time visibility for reliability, which is often the more critical requirement in logistics applications. 
# Shortcomings

