[[RhyGen]]

https://www.robustel.store/blogs/industrial-iot-blog/the-sd-card-vs-emmc-debate-a-reliability-guide-for-edge-products#:~:text=And%20it's%20what%20must%20be,is%20100%25%20vibration%2Dproof.
# Architecture Notes
A record of arguments to support or oppose the decisions that are made for the next version of the logger

## IoT Connectivity
ELITE Reference - https://onomondo.com/blog/lte-standards-for-iot-comparison/#:~:text=It's%20a%20low%2Dpower%2C%20wide,cycle%20durations%20and%20other%20parameters.
## Automotive Data Logger

### 1. Core Processing & System Controls
- **Component Example:** `STM32F446ZET6` (144-pin LQFP package)
- **Why this variant:** While the 100-pin version is compact, the 144-pin package is highly recommended for this layout. Pin multiplexing conflicts can quickly stack up when simultaneously utilizing QSPI (6 pins), SDIO (6 pins), Dual CAN (4 pins), and multiple UART channels with hardware flow control.
- **Implementation Note:** Run an RTOS (like **FreeRTOS** or **ThreadX/Azure RTOS**) to isolate the low-latency blackbox writing thread from the heavier network handling stacks.

### 2. The Dual-Memory Storage Topology
Because the F446 will handle both memories over separate hardware peripherals, they will not compete for bus time.

- **Blackbox Memory (QSPI Flash):** eg -`Winbond W25Q256JVEIQ` (32MB / 256Mb)
	- **Specifications:** WSON8 package (no pins to snap or vibrate loose), Industrial/Automotive temp range ($-40^\circ\text{C}$ to $+105^\circ\text{C}$).
	- **Role:** Raw binary ring-buffer for the "last-gasp" 1-minute logs, system configuration profiles, and fail-safe golden boot images.

- **Mass Storage (eMMC):** eg - `Kingston EMMC32G-TA28-A52B` (32GB)

    - **Specifications:** JEDEC eMMC 5.1 standard, 153-ball BGA package, optimized for high write-endurance and native bad-block management.    
    - **Role:** Long-term historic logging repository running a fault-tolerant filesystem like **LittleFS**.
    
- **Hardware Integration Trick:** The STM32F446’s SDIO peripheral does not support the blazing $200\text{ MHz}$ HS400 modes of modern eMMC. However, eMMC chips are entirely backward compatible. You can wire it in **4-bit legacy MMC mode** at $48\text{ MHz}$. This yields a throughput of roughly $24\text{ MB/s}$—vastly outpacing any logging data stream you throw at it.
### 3. IoT Telematics & Cellular Connectivity
- **Component Example:** `Quectel BG95-M3` / `ECU200`

    - **Specifications:** Multi-mode LTE Cat M1 / Cat NB2 / EGPRS module with integrated GNSS (GPS/GLONASS) for asset tracking. LGA form factor.    
    - **Role:** Manages the primary data pipeline to the cloud via LTE-M, shifting down to 2G/EGPRS fallback if the machine drops into an ultra-remote region lacking LTE infrastructure.    
    - **Interfacing:** Communicates with the STM32 via `USART1` or `USART2` using hardware flow control (TX/RX/RTS/CTS) mapped up to $921600\text{ bps}$. 
    
- **Sim Connection:** `STMicroelectronics ST4SIM-200M` (MFF2 Form Factor) - this is the sim SLOT that we can use, should we go down the path of not having a module and just the quectel chipset for us 
    - **Role:** This is an industrial-grade **eSIM** chip soldered directly onto your PCB. It eliminates the physical SIM card tray completely, preventing contact failures caused by engine vibrations.    