[[Blog Topics]]

# Embedded Systems in Drones

## What the Hell is an Embedded System?

When people hear the term _embedded systems_, they often imagine a tiny microcontroller sitting on a PCB somewhere. While that is technically correct, it misses the bigger picture.

An embedded system is a computer designed to perform a specific function as part of a larger machine. Unlike your laptop or smartphone, an embedded system is not intended to be a general-purpose computing device. Instead, it is optimized for reliability, efficiency, and real-time operation.

Examples of embedded systems are everywhere:

- Engine Control Units (ECUs) in cars
- Washing machines
- Smart thermostats    
- Industrial automation equipment
- Medical devices
- Drones
    

A modern drone is one of the best examples of an embedded system because it combines sensing, computation, control theory, communication, and increasingly, artificial intelligence, into a single flying machine.

At its core, a drone continuously performs the following loop:

1. Sense the environment
    
2. Estimate its current state
    
3. Decide what to do
    
4. Act on that decision
    
5. Repeat hundreds or thousands of times per second
    

Everything else is built on top of this cycle.

---

# Anatomy of a Drone

A drone may look simple from the outside, but internally it consists of several interconnected subsystems.

The major components are:

- Flight Controller
- Electronic Speed Controllers (ESCs)
- Brushless Motors
- Propellers
- Battery
- Sensors
- Radio Receiver
- Telemetry System
- Cameras

The flight controller serves as the central nervous system of the drone. It receives data from various sensors, processes this information, calculates corrective actions, and sends commands to the motors.

The motors provide thrust, but without intelligent control they would simply cause the drone to flip over immediately. Maintaining stable flight requires constant feedback and adjustment.

This is where embedded systems become essential.

---

# The Brain of the Drone

## Microcontrollers (MCUs)

Most flight controllers are powered by microcontrollers.

Popular examples include:

- STM32F4    
- STM32F7    
- STM32H7    
- RP2040 (smaller platforms)
    
Microcontrollers are ideal because they offer:

- Deterministic execution    
- Low power consumption    
- Fast interrupt handling    
- Real-time operation    

Unlike a desktop CPU, an MCU can guarantee that critical control code executes exactly when required.

---

## Microprocessors (MPUs)

Some drones also contain microprocessors.

Examples include:

- Raspberry Pi    
- NVIDIA Jetson    
- Qualcomm Flight Platforms    

These systems run Linux and are typically used for:

- Computer Vision    
- AI Inference    
- Path Planning    
- Autonomous Navigation    
The MPU usually works alongside the MCU rather than replacing it.

---

## Flight Controllers

The flight controller is responsible for:

- Reading sensors    
- Running estimation algorithms    
- Executing control loops    
- Generating motor commands   

Popular open-source flight stacks include:

- PX4    
- ArduPilot    
- Betaflight   

These systems contain decades of accumulated engineering and are capable of controlling everything from racing drones to autonomous delivery vehicles.

---

## Real-Time Systems

A drone is a real-time system.

Sensor updates may arrive at:

- 100 Hz
    
- 500 Hz
    
- 1 kHz
    

Control loops often run at similarly high frequencies.

Missing deadlines can cause:

- Oscillations
    
- Loss of stability
    
- Complete crashes
    

This requirement for deterministic execution is one reason why embedded systems dominate drone flight control.

---

# How a Drone Knows Where It Is

For a drone to fly safely, it must continuously estimate its position, orientation, and motion.

No single sensor can provide all this information accurately.

Instead, drones rely on multiple sensing systems.

## Inertial Measurement Unit (IMU)

The IMU is arguably the most important sensor on the drone.

It typically contains:

- Accelerometer
    
- Gyroscope
    

These sensors allow the drone to estimate motion and orientation at very high rates.

Think of the IMU as the drone's inner ear.

---

## Accelerometer

Measures acceleration along multiple axes.

Uses include:

- Motion detection
    
- Tilt estimation
    
- Dynamic response measurement
    

However, accelerometers are noisy and cannot be trusted alone.

---

## Gyroscope

Measures angular velocity.

It tells the drone how quickly it is rotating around:

- Roll axis
    
- Pitch axis
    
- Yaw axis
    

Gyroscopes provide smooth short-term measurements but drift over time.

---

## Magnetometer

Functions as a digital compass.

Provides heading information relative to Earth's magnetic field.

Useful for long-term orientation estimation.

---

## Barometer

Measures atmospheric pressure.

Pressure changes can be used to estimate altitude.

Barometers are commonly used for:

- Altitude hold
    
- Hover stabilization
    

---

## Time-of-Flight (ToF) Sensors

Measure distance by calculating the travel time of light pulses.

Useful for:

- Landing
    
- Obstacle detection
    
- Indoor positioning
    

---

## Optical Flow Sensors

Optical Flow Sensors estimate motion by tracking visual features on the ground.

They allow drones to:

- Hover indoors
    
- Maintain position without GPS
    
- Estimate velocity
    

This technology is commonly found in consumer drones.

---

## GNSS / GPS

GPS provides global position information.

Advantages:

- Global coverage
    
- Absolute positioning
    

Limitations:

- Poor indoor performance
    
- Multipath errors
    
- Limited update rates
    

GPS alone is not sufficient for stable flight.

---

## LiDAR

LiDAR measures distance using laser pulses.

Applications include:

- Terrain mapping
    
- Obstacle avoidance
    
- Autonomous navigation
    

LiDAR can generate highly accurate 3D representations of the environment.

---

## Cameras and Vision Sensors

Cameras provide the richest information source available to a drone.

Applications include:

- Mapping
    
- Inspection
    
- Navigation
    
- Object detection
    
- Tracking
    

Modern autonomous systems increasingly rely on vision-based navigation.

---

# The Secret Sauce: Sensor Fusion

Each sensor has strengths and weaknesses.

For example:

- GPS is accurate but slow.
    
- IMUs are fast but drift.
    
- Barometers estimate altitude but are noisy.
    

Sensor fusion combines multiple sensors to produce a more accurate estimate than any individual sensor can provide.

---

## Why Sensor Fusion is Necessary

Imagine trying to navigate using only GPS.

The drone would respond too slowly.

Using only an IMU would cause drift.

By combining both, the drone gains the advantages of each while minimizing their weaknesses.

---

## Kalman Filters

Kalman Filters are among the most important algorithms in modern robotics.

The filter performs two operations:

1. Prediction
    
2. Correction
    

The system predicts its next state and then corrects that prediction using sensor measurements.

---

## Extended Kalman Filters (EKF)

Most drone systems are nonlinear.

The Extended Kalman Filter extends the standard Kalman Filter to nonlinear systems.

PX4 and ArduPilot both rely heavily on EKF-based estimation.

---

## State Estimation

The drone continuously estimates:

- Position
    
- Velocity
    
- Orientation
    
- Sensor biases
    

This estimate is often more valuable than the raw sensor data itself.

---

# Control Systems

Knowing where you are is only half the battle.

The drone must also control itself.

## Open Loop vs Closed Loop Control

Open-loop systems do not use feedback.

Closed-loop systems continuously measure performance and apply corrections.

Drones are almost entirely closed-loop systems.

---

## PID Controllers

PID controllers are the workhorses of drone control.

They contain:

- Proportional Term
    
- Integral Term
    
- Derivative Term
    

Together they allow the drone to:

- Reach a target
    
- Remove steady-state errors
    
- Reduce oscillations
    

---

## Cascaded Control Loops

Most drones use multiple control loops.

Position Loop  
→ Attitude Loop  
→ Rate Loop  
→ Motor Commands

Each loop operates at a different frequency.

---

## Attitude Control

Controls the drone's orientation.

Maintains:

- Roll
    
- Pitch
    
- Yaw
    

---

## Rate Control

Controls angular velocity.

This is often the fastest-running control loop.

---

## Position Control

Controls where the drone should be in space.

Used for:

- Hovering
    
- Waypoint following
    
- Autonomous navigation
    

---

## Motor Mixing

The flight controller ultimately converts desired motion into motor commands.

Motor mixing determines how motor thrust should be adjusted to achieve:

- Roll
    
- Pitch
    
- Yaw
    
- Altitude changes
    

---

# Communication Systems

A drone rarely operates alone.

Communication is critical.

## RC Communication

The primary control link between pilot and drone.

Carries:

- Stick positions
    
- Mode selections
    
- Auxiliary commands
    

---

## Telemetry

Allows the drone to transmit information back to the ground station.

Examples:

- Battery status
    
- GPS position
    
- Sensor health
    

---

## MAVLink

The most widely used drone communication protocol.

Common message types include:

- HEARTBEAT
    
- ATTITUDE
    
- GPS
    
- MISSION
    

MAVLink serves as the language spoken by modern drones.

---

## CAN Bus in Drones

Traditional drones relied heavily on UART.

Modern systems increasingly use CAN.

Advantages include:

- Robustness
    
- Error detection
    
- Multi-device networking
    
- Reduced wiring complexity
    

DroneCAN has become particularly popular in advanced UAV platforms.

---

## UART

Simple and widely supported.

Often used for:

- GPS
    
- Telemetry radios
    
- Companion computers
    

---

## I2C

Commonly used for lower-speed sensors.

---

## SPI

Used for high-speed sensors such as IMUs and flash memory.

---

# Where AI Fits in a Drone

Artificial Intelligence is becoming increasingly important in UAV systems.

However, AI does not replace the flight controller.

It augments it.

## Computer Vision

Allows drones to interpret visual information.

---

## Object Detection and Tracking

Used in:

- Inspection drones
    
- Surveillance systems
    
- Follow-me drones
    

---

## Obstacle Avoidance

AI can identify and classify obstacles before planning avoidance maneuvers.

---

## Autonomous Navigation

AI helps determine safe routes through complex environments.

---

## Visual Navigation

Navigation without GPS.

Particularly useful indoors and in urban environments.

---

## Landing Zone Detection

AI can identify safe landing areas automatically.

---

## TinyML and Edge AI

Machine learning models can now run directly on microcontrollers.

This enables onboard intelligence without cloud connectivity.

---

## AI-Assisted Control Systems

AI can assist with:

- Fault detection
    
- Adaptive tuning
    
- Mission planning
    

While traditional controllers still handle stabilization.

---

# AI vs Classical Control

AI and classical control solve different problems.

## What AI Can Do

- Perception
    
- Classification
    
- Decision support
    
- Route planning
    

## What PID Controllers Still Do Better

- Stability
    
- Determinism
    
- Real-time response
    
- Certification
    

The drone's stabilization loops are still overwhelmingly handled by classical control systems.

---

# Future Trends

## Visual-Inertial Odometry (VIO)

Combines cameras and IMUs for GPS-independent navigation.

---

## Event Cameras

Capture changes rather than frames.

Offer extremely low latency.

---

## Swarm Drones

Multiple drones cooperating as a distributed system.

---

## AI Accelerators

Dedicated hardware for neural network inference.

Examples include:

- Jetson
    
- Hailo
    
- Edge TPU
    

---

## TinyML

Bringing machine learning directly onto MCUs.

---

## Autonomous Systems

The long-term goal of many UAV platforms is full autonomy with minimal human intervention.

---

# Conclusion

A drone is far more than a flying machine.

It is a complex embedded system that combines:

- Sensors
    
- Estimation
    
- Control Theory
    
- Communication
    
- Artificial Intelligence
    

Every stable flight relies on hundreds of interconnected software and hardware components working together in real time.

Understanding drones therefore provides an excellent introduction to the broader field of embedded systems and robotics.

---

# Questions and Discussion