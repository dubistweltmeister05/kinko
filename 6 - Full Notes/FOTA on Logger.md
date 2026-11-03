![[fota_logger.excalidraw]]
# STM32F446RE FOTA Architecture Guide

For an STM32F446RE running on single-bank 512 KB Flash with an onboard SD card (SDIO/FatFs) and a Quectel EC200 modem over UART, the most reliable FOTA strategy is an **external-staging bootloader architecture** using AWS S3 pre-signed URLs. 

Because the STM32F446 lacks internal dual-bank Flash and has asymmetric sectors (16 KB, 64 KB, 128 KB), partitioning internal flash into active and staging slots would limit your application to under 224 KB. Staging incoming firmware to your onboard MicroSD card keeps ~448 KB available for your logging firmware while enabling reliable, brick-proof rollbacks.

---

## 1. System Architecture & Memory Partitioning

The system relies on three execution domains: a custom bootloader in internal flash, non-volatile metadata storage, and the staged firmware images stored on the SD card.

### Internal Flash Partitioning (STM32F446RE — 512 KB Total)

| Sector | Address Range | Size | Allocation / Role |
| :--- | :--- | :--- | :--- |
| **Sectors 0–1** | `0x08000000 – 0x08007FFF` | 32 KB | **Custom Bootloader** (Integrity checking, SD flashing, rollback) |
| **Sector 2** | `0x08008000 – 0x0800BFFF` | 16 KB | **Boot Metadata** (FOTA state, boot count, firmware version, hashes) |
| **Sector 3** | `0x0800C000 – 0x0800FFFF` | 16 KB | Reserved / Application persistent calibration data |
| **Sectors 4–7** | `0x08010000 – 0x0807FFFF` | 448 KB | **Main Application** (`logger_0.1` binary entry at `0x08010000`) |

### SD Card Staging (FatFs File Structure)

*   `/fota/firmware_update.bin`: Staged candidate binary downloaded by the active application.
*   `/fota/golden_backup.bin`: Known-good firmware image archived after the previous verified boot.
*   `/fota/manifest.json`: Metadata payload containing expected version, byte length, SHA-256 digest, and cryptographic signature.

---

## 2. Cloud Orchestration & Modem Download Pipeline

Communication splits between low-bandwidth control/telemetry over MQTT (AWS EC2) and high-throughput binary transfer over HTTPS (AWS S3).

```text
[AWS S3 Bucket]
       │
       │ (HTTPS GET via Pre-signed URL)
       ▼
[Quectel EC200] ──UART (DMA)──► [STM32 Application] ──SDIO──► [SD Card: /fota/update.bin]
       ▲
       │ (MQTT Pub/Sub: Notify, Progress, Status)
       ▼
[AWS EC2 (Elastic IP)]
```

### Release Packaging
*   CI/CD generates the raw binary from your CMake build (`arm-none-eabi-objcopy -O binary logger_0.1.elf logger_0.1.bin`).
*   Compute the SHA-256 hash and sign it with an ECDSA private key (secp256r1).
*   Upload `logger_0.1.bin` to a private S3 bucket.

### Update Notification via EC2
The EC2 backend publishes an update notification to MQTT topic `devices/{device_id}/fota/notify`:

```json
{
  "action": "firmware_update",
  "version": "1.2.0",
  "size_bytes": 184320,
  "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "url": "https://your-bucket.s3.amazonaws.com/fw/v1.2.0.bin?AWSAccessKeyId=...&Signature=...&Expires=..."
}
```

### Chunked Ingestion via Quectel EC200
*   The STM32 application sets up Quectel's HTTP engine using AT commands:
    *   `AT+QHTTPCFG="contextid",1`
    *   `AT+QHTTPCFG="sslctxid",1`
    *   `AT+QHTTPURL=<url_len>,60` (send S3 pre-signed URL)
    *   `AT+QHTTPGET=60`
*   Read the data in chunks (e.g., 2048 to 4096 bytes) using `AT+QHTTPREAD` or save directly to the Quectel UFS via `AT+QHTTPREADFILE` before streaming across UART to the STM32.
*   Write each incoming stream buffer directly into `/fota/firmware_update.bin` via FatFs (`f_write`), computing a running SHA-256 hash in parallel.

---

## 3. Bootloader Design & Execution Flow

Once the download finishes, control transitions through a strict state machine to prevent partial writes.

### State Machine Lifecycle
*   **`STATE_IDLE`**: Normal runtime operations.
*   **`STATE_DOWNLOAD_COMPLETE`**: Binary verified on SD card against manifest SHA-256. Metadata sector updated; soft reset triggered.
*   **`STATE_FLASHING`**: Bootloader boots, detects valid pending update flag, erases internal flash Sectors 4–7, and copies `/fota/firmware_update.bin` into `0x08010000`.
*   **`STATE_TESTING`**: Bootloader sets `boot_count = 1`, remaps vector table, and jumps to `0x08010000`.
*   **`STATE_CONFIRMED`**: Application initializes, connects to cellular, reports `FOTA_SUCCESS` to EC2 via MQTT, overwrites `/fota/golden_backup.bin` with the running image, and clears update flags.

### Application Handoff Code (Bootloader)

```c
void jump_to_application(uint32_t app_address) {
    uint32_t app_stack = *(__IO uint32_t*)app_address;
    pFunction app_entry = (pFunction)(*(__IO uint32_t*)(app_address + 4));

    // Disable all interrupts and clear pending flags
    __disable_irq();
    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL = 0;

    for (uint8_t i = 0; i < 8; i++) {
        NVIC->ICER[i] = 0xFFFFFFFF;
        NVIC->ICPR[i] = 0xFFFFFFFF;
    }

    // Set vector table offset to the application sector
    SCB->VTOR = app_address;

    // Set stack pointer and jump
    __set_MSP(app_stack);
    app_entry();
}
```

### Main Application Linker & Vector Adjustments

In your `STM32F446XX_FLASH.ld` for the application, adjust the FLASH start address and size:

```ld
MEMORY
{
  RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 128K
  FLASH (rx)      : ORIGIN = 0x08010000, LENGTH = 448K
}
```

In your application initialization (`SystemInit()` or top of `main()`):

```c
SCB->VTOR = 0x08010000;
__enable_irq();
```

---

## 4. Fail-Safe, Rollback & Security Considerations

*   **Watchdog Protection & Boot Counter:** Run the Independent Watchdog (IWDG) with an 8-to-16 second window during bootloader flashing. In the application, if the system hangs or encounters a `HardFault` prior to reaching `STATE_CONFIRMED`, the IWDG resets the board.
*   **Automatic Rollback:**
    *   Each boot under `STATE_TESTING` increments `boot_count` in Sector 2.
    *   If `boot_count > 2`, the bootloader marks the candidate binary invalid and flashes `/fota/golden_backup.bin` back into internal flash, notifying EC2 of the rollback upon reconnection.
*   **Power-Loss Tolerance:** Internal flash is erased only after the staged binary's SHA-256 and signature are validated on the SD card. If power cuts out mid-flash, the bootloader wakes up, sees `STATE_FLASHING`, and simply restarts the copy from `/fota/firmware_update.bin` (or falls back to `/fota/golden_backup.bin`).
*   **Signature Verification:** Compile the public key directly into the bootloader binary. Verify an ECDSA signature over the binary before the bootloader touches internal flash sectors to prevent tampering or unauthorized firmware injection.

---

## Q&A: The End-to-End Workflow

> **Question:** So basically, I should save the incoming app firmware images to the SD card, and then direct the bootloader to copy the newer firmware from the configured SD Card partition to the Main application location?

That is the exact model, commonly referred to as an **external staging (or off-chip staging) bootloader architecture**. Because your STM32F446RE has a single 512 KB internal flash bank, this approach avoids cutting your internal executable space in half, leaving ~448 KB available for your logging, CAN, and modem drivers.

### The Workflow Diagram

```text
1. Active App (0x08010000) ──Download via EC200──► Staged on SD Card (/fota/update.bin)
                                                            │
2. Active App verifies hash & sets metadata flag            │
3. NVIC_SystemReset()                                       │
                                                            ▼
4. Bootloader (0x08000000) ◄────── Reads file from SD Card ─┘
         │
         ├── Erases Application Flash (0x08010000 - 0x0807FFFF)
         ├── Programs Flash sector-by-sector
         └── Jumps to 0x08010000
```

### Step 1: Download & Stage (Main Application)
The active firmware keeps running its normal logger tasks while streaming the new binary chunks from the Quectel module over UART. Chunks are written to a file on the SD card (e.g., `/fota/firmware_new.bin`) using FatFs over SDIO. Once download completes, the application reads the file, computes its SHA-256 hash, and verifies it against the expected manifest hash.

### Step 2: Signal & Reset (Main Application)
If the file is valid, the application writes an update request to a non-volatile location (either the dedicated Flash metadata sector Sector 2, an RTC backup register, or a metadata text file on the SD card like `/fota/status.ini`). The application issues a software reset via `NVIC_SystemReset()`.

### Step 3: Flash Programming (Custom Bootloader)
Upon reset, the STM32 begins execution at `0x08000000` inside your custom bootloader. The bootloader checks the metadata flag:
*   **No update flag:** It validates the vector table at `0x08010000` and immediately jumps to the active app.
*   **Update flag present:** It mounts FatFs on the SD card, reads `/fota/firmware_new.bin`, erases internal Flash Sectors 4 through 7 (`0x08010000 – 0x0807FFFF`), and writes the new image into Flash in 32-bit words using `HAL_FLASH_Program()`.

The bootloader marks the update state as `TESTING` (or increments a boot attempt counter) and executes the jump to `0x08010000`.

### Step 4: Self-Test & Confirmation (New Application)
The updated application boots up from `0x08010000`. It initializes the Quectel module and tests its cloud connection. Once a successful connection to EC2 is established, the application marks itself `CONFIRMED` in the metadata sector. It renames `/fota/firmware_new.bin` to `/fota/golden_backup.bin` on the SD card for future rollbacks.

### Key Safeguards to Keep in Mind
*   **Network Interruption Safety:** If the modem loses signal or power drops mid-download, the internal Flash is never touched. The device simply discards the partial SD file and continues operating.
*   **Bad Build Recovery:** If the new firmware faults, freezes, or fails to connect, an Independent Watchdog (IWDG) reset occurs. On reboot, the bootloader notices that `boot_count` exceeded the threshold without confirmation, erases `0x08010000`, and flashes the saved `/fota/golden_backup.bin` back into internal Flash.

---

## Q&A: Dual-Partitioning vs. SD Card Staging

> **Question:** Why not make 2 flash partitions and configure the Quectel to dump incoming binaries into the non-active partition? We can simply set a flag once the download is done, and can switch the active partition as the location from which we boot the MCU.

While internal dual-partitioning (Slot A / Slot B) is common on microcontrollers designed for it (like dual-bank STM32s or ESP32s), implementing it on an STM32F446RE introduces several severe architectural traps.

### 1. The Relocation / Dual-Binary Trap
Cortex-M firmware is position-dependent. Machine instructions, vector tables, function pointers, and literal pool offsets are hardcoded to absolute memory addresses at compile time via the linker script (`.ld`). 

If you create two internal slots (`Slot A: 0x08010000` and `Slot B: 0x08040000`), a firmware image compiled to run at `0x08010000` will crash immediately into a `HardFault` if jumped to at `0x08040000`.

To make the "switch active partition" idea work, you would have to choose between three deeply flawed workarounds:
*   **Maintain two distinct build targets in CI/CD:** Generate a `v1.2_slotA.bin` (linked to `0x08010000`) and a `v1.2_slotB.bin` (linked to `0x08040000`). Your EC2 cloud backend would have to track which slot every single physical unit is running and push the opposite binary. If the cloud database gets out of sync, the device bricks.
*   **Compile with Position-Independent Code (`-fPIC`):** Compiling bare-metal embedded C/HAL with PIC is notoriously fragile. It reserves the `r9` register for the Global Offset Table (GOT), increases RAM consumption, bloats binary size, and frequently breaks with vendor HAL drivers.
*   **Hardware Bank Swapping:** Chips like the STM32F429, STM32G4, or STM32H7 have dual-bank flash with a bit (e.g., `BFB2` or `SWAP_BANK`) that physically remaps Bank 2 to address `0x08000000`. The STM32F446RE does not have this feature—it is strictly single-bank.

### 2. The Single-Bank Flash Stall (No Read-While-Write)
Because the STM32F446 has only one flash bank, the microcontroller cannot execute instructions from Flash while programming or erasing Flash.
*   Erasing a 128 KB internal sector (Sectors 5, 6, or 7) takes between 1.0 and 2.5 seconds (up to 4.0 seconds worst-case).
*   While a sector erases, the internal Flash bus stalls completely.
*   Unless your entire cellular UART driver, DMA ISRs, FreeRTOS scheduler, and circular ring buffers are running strictly out of SRAM (`__RAM_FUNC`), the MCU pauses dead in its tracks.
*   At 115200 or 921600 baud over UART, a 2-second stall will overflow your hardware FIFO, drop cellular packets, trigger watchdog timeouts, and cause connection drops with the Quectel modem.

With the SD card strategy, the modem streams packets to RAM, and the MCU writes them to the SD card via SDIO DMA. The internal flash memory is never touched while the cellular connection is active, so your CAN logging and modem tasks continue running uninterrupted.

### 3. Severe Usable Space Penalty (Asymmetric Sectors)
The STM32F446 flash layout is asymmetric:
*   **Sectors 0–3:** $4 \times 16\text{ KB} = 64\text{ KB}$
*   **Sector 4:** $1 \times 64\text{ KB} = 64\text{ KB}$
*   **Sectors 5–7:** $3 \times 128\text{ KB} = 384\text{ KB}$

You cannot cut 512 KB in half cleanly:
*   If Bootloader + Metadata take Sectors 0–2 (48 KB), Slot A can take Sectors 3, 4, and 5 $(16\text{ KB} + 64\text{ KB} + 128\text{ KB} = 208\text{ KB})$.
*   Slot B takes Sectors 6 and 7 $(128\text{ KB} + 128\text{ KB} = 256\text{ KB})$.

Because both slots must accommodate the same maximum application size, your maximum app size is capped at **208 KB**.

Your current build is already ~80 KB. As you add more CAN protocols, cloud payload formats, edge analytics, or network stacks, hitting a 208 KB ceiling is a very real possibility. With the SD card approach, your application has **448 KB** of headroom in internal flash.