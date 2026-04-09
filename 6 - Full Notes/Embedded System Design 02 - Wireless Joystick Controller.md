[[Embedded System Design]]
![[Pasted image 20260409171008.png]]
Hey guys,

Continuing the series where I try to break down the *system design* behind everyday embedded products. Last time, we looked at a GPS tracker — something that operates quietly in the background. This time, let’s look at something far more interactive.

> A **wireless joystick controller**

---

## 🎯 The Problem Statement

At first glance, a controller seems simple:

- You press a button → something happens on screen  
- You move a joystick → character moves  

But under the hood, the system needs to:

- Capture multiple user inputs (buttons + analog joystick)
- Process them in real time
- Send them wirelessly
- Maintain **low latency**
- Run efficiently on battery

---

## 🧩 High-Level System Design

We can break the system into 4 major subsystems:

- **User Inputs**
- **Main MCU (processing unit)**
- **Communication (wireless)**
- **Power subsystem**

At a high level:

- Buttons → GPIO → MCU  
- Joystick → ADC → MCU  
- MCU ↔ Bluetooth (UART/SPI)  
- Battery → BMS → Regulators → System  

---

## 🎮 Input Subsystem

This is where the system meets the human.

### 🟡 Buttons (Digital Input)

- Simple push buttons connected to GPIOs  
- Require **debouncing**

Why debouncing matters:
- Mechanical switches bounce  
- One press can look like multiple presses  
- Needs software filtering  

---

### 🕹️ Joystick (Analog Input)

Two common approaches:

#### 1. Potentiometer-based
- Outputs analog voltage  
- Read using MCU ADC  
- Cheap and widely used  

#### 2. Hall-effect sensors
- Magnetic sensing (no physical contact)  
- No wear and tear  
- Higher precision (premium controllers)  

---

## 🧠 The Brain: MCU

The MCU is the central controller.

Responsibilities:

- Read GPIO (buttons)
- Read ADC (joystick)
- Process input state
- Package data for transmission

Example packet:

```c
struct packet {
    uint8_t buttons;
    uint16_t x_axis;
    uint16_t y_axis;
};
```

This packet is continuously updated and transmitted.

---

## 📡 Communication Subsystem

We now need to send this data to a host device.

### Options:
- Bluetooth
- WiFi

### Typical choice:

👉 **Bluetooth Low Energy (BLE)**

Why BLE?
- Low power consumption  
- Sufficient bandwidth  
- Standard for controllers  
- Supported by almost all devices  

---

## 🔋 Power Subsystem

One of the most important (and often ignored) parts.

### Components:

- Battery (Li-ion / Li-Po)
- Charging circuitry
- Battery Management System (BMS)
- Voltage regulators (e.g., 3.3V rail)

### Power Flow:

Battery → BMS → Regulators → MCU / Sensors / RF

This directly impacts:
- Battery life  
- Safety  
- Stability  

---

## ⚡ Real Engineering Challenges

### 1. Latency
- Needs to feel instant  
- Even small delays (~50ms) are noticeable  

---

### 2. Noisy Analog Signals
- Joystick readings fluctuate  
- Requires:
  - Filtering
  - Calibration
  - Dead zones  

---

### 3. Power Efficiency
- Wireless communication is expensive  
- Use:
  - Sleep modes  
  - Efficient transmission intervals  

---

### 4. Packet Reliability
- Packets can drop  
- Need to decide:
  - Retry?
  - Ignore?
  - Smooth/interpolate?

---

## 🧠 What We Didn’t Cover (Yet)

There’s more depth here:

- Firmware architecture (RTOS vs bare-metal)
- BLE stack internals
- Input scanning strategies
- EMI and PCB layout
- RF design considerations

---

## 🚀 Wrapping Up

A joystick controller might seem simple, but it combines:

- Analog sensing  
- Digital processing  
- Wireless communication  
- Power management  

> That’s peak embedded systems.

---

## 🔜 What’s Next?

Next, I want to explore something more autonomous:

- Smart thermostat  
- Fitness tracker  
- IoT device  

If you’ve got suggestions, let me know 👇
