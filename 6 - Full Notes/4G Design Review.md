[[RhyGen]]
# 4G Pipeline: High-Level Overview

This document outlines the state machine progression and AT command sequence used in the `ec200_driver.c` for initializing the 4G module, configuring SMS, establishing a data connection, and publishing MQTT telemetry.
## 1. Initialization & Network Registration

- **Module Sync:** Send standard `AT` and `ATE0` commands to establish UART synchronization and disable echo for cleaner parsing.
  - **Fallback:** If `ATE0` times out (>2s), the system drops back to resending the `AT` sync command.
- **Network Check:** Verify SIM readiness via `AT+CPIN?` and confirm Evolved Packet System (EPS) network registration using `AT+CEREG?`.
  - **Fallback:** If the SIM check times out (>3s) or fails, it retries `AT+CPIN?`. If network registration times out (>3s) or fails, it retries `AT+CEREG?`.

## 2. SMS Configuration

- **Format & Wipe:** Set SMS to text mode with `AT+CMGF=1`, then wipe existing messages from the SIM memory using `AT+CMGD=1,4` to prevent overflow.
- **Routing:** Direct all incoming SMS directly to the MCU over the UART interface using `AT+CNMI=2,2,0,0,0`.
  - **Fallback (Applies to all SMS steps):** If any configuration step times out (>5s), the driver resends the specific AT command for that current step.

## 3. Network Cleanup (Pre-connection)

- **Stale Session Teardown:** Before configuring the data network, the driver actively closes any lingering MQTT sockets (`AT+QMTCLOSE=0`) and deactivates old PDP contexts (`AT+QIDEACT=1`). This ensures a clean slate before attempting a new connection.
  - **Fallback:** Errors are ignored here (as the socket/context might already be closed), but if a step times out (>5s), the command is retried.

## 4. APN Configuration & PDP Activation

- **APN Setup:** Configure the Access Point Name (e.g., `airtelgprs.com`) via `AT+QICSGP` to act as the network entry point.
  - **Fallback:** If rejected (ERROR) or it times out (>3s), the system drops all the way back to the Network Check phase (`AT+CEREG?`).
- **Context Activation:** Activate the PDP context via `AT+QIACT=1` to establish the logical session with the Public Data Network.
  - **Fallback:** If rejected (ERROR) or it times out (>5s), the system assumes the context is stuck. It forces a deactivation (`AT+QIDEACT=1`) and routes back to the APN Setup phase.

## 5. MQTT Socket & Connection

- **Socket Opening:** Open the connection to the broker and port (e.g., test.mosquitto.org on 1883) using `AT+QMTOPEN`.
  - **Fallback:** If the initial activation fails/times out (>15s), it drops back to the Network Check phase. If it successfully sends but times out waiting for the socket response (>15s), it drops back to PDP Context Activation.
- **Client Connection:** Send the connection request via `AT+QMTCONN`. This command registers the **Client ID** (e.g., "STM32_Logger_01") with the broker.
  - **Fallback:** If this times out (>10s), the system drops back to the Socket Opening phase (`AT+QMTOPEN`).

## 6. Idle State & Telemetry Publishing

- **Idle Listening:** The module sits in the `EC200_Idle_Online` state, actively monitoring for incoming SMS routing or connection loss.
  - **Fallback:** If an MQTT Link Lost URC (`+QMTSTAT`) is detected, the system forces a PDP context deactivation (`AT+QIDEACT=1`) and jumps all the way back to APN Setup to restart the connection loop.
- **Publishing:** Triggered externally by `ec200_publish_telemetry`, the system transitions to sending data via `AT+QMTPUBEX`. This specific command declares the **Topic**, QoS level, packet ID, and payload length. The system then sends the payload and waits for the broker's acknowledgment before returning to idle.
  - **Fallback:** 
    - If the modem rejects the command or payload (ERROR), it aborts and returns to Idle.
    - If the prompt (`>`), payload transmission, or broker ACK times out (>3s for prompt/send, >30s for ACK), it aborts and returns to Idle.
    - If the broker explicitly NACKs the publish (result `2`), it aborts and returns to Idle.
    - If the MQTT link is lost during any publishing phase, the failure is flagged and the system returns to Idle (where the reconnect sequence will automatically trigger on the next loop).