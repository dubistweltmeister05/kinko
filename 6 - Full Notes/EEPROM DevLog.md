[[Titan]]
# AT24C32E EEPROM Driver Development Log

**Project:** TorTitan Energy Meter — STM32H750VBTx Firmware  
**Sprint Scope:** Secondary Non-Volatile Storage Migration  
**Duration:** May 2026  
**Author:** Firmware Engineering Team

---

## Executive Summary

This development log chronicles the design, implementation, and integration of a HAL-based I2C EEPROM driver for the AT24C32E-SSHM-T, along with its full integration into the TorTitan non-volatile storage subsystem. The driver replaces the W25Q64FV SPI NOR flash as the secondary storage backend via a compile-time switch, achieving a drop-in replacement with zero changes to upstream callers.

**Key Deliverables:**
- Complete AT24C32E HAL driver with page-boundary-aware writes
- Four-stage hardware selftest validating byte, page, and boundary-crossing operations
- Full nv_storage.c integration covering 5 write functions + 5 read functions
- Compile-time feature toggle preserving 100% backward compatibility with original SPI flash path

---

## Part 1: Hardware Context & Constraints

### 1.1 Target Device Specifications

| Parameter | Value | Engineering Implication |
|-----------|-------|------------------------|
| **Part Number** | AT24C32E-SSHM-T | SOIC-8 package, automotive-grade temperature range |
| **Capacity** | 4096 bytes (32 Kbit) | Sufficient for 5 data domains with dual-copy redundancy |
| **Page Size** | 32 bytes | Writes crossing this boundary require chunking |
| **Write Cycle** | 5 ms typical, 10 ms maximum | Requires acknowledge-polling between writes |
| **Endurance** | 1,000,000 cycles | 10× higher than typical NOR flash sectors |
| **I2C Speed** | Up to 400 kHz (Fast Mode) | Must not exceed — device hangs at 1 MHz |
| **7-bit Address** | 0x50 | Hardware-strapped via A0=A1=A2=GND |
| **Memory Addressing** | 16-bit | Requires `I2C_MEMADD_SIZE_16BIT` in HAL calls |

### 1.2 MCU Interface Configuration

The STM32H750VBTx interfaces with the EEPROM via I2C1, configured through STM32CubeMX:

```
Peripheral:        I2C1
Mode:              I2C (400 kHz Fast Mode)
Timing Register:   0x00B03FDB
Analog Filter:     Enabled
Fast Mode Plus:    DISABLED (critical — see Bug #2 below)
```

**Critical Discovery:** The initial CubeMX configuration enabled Fast Mode Plus at 1 MHz via `HAL_I2CEx_EnableFastModePlus()`. This exceeded the AT24C32E maximum clock rate, causing `HAL_I2C_Mem_Write` to fail and leave the bus in a permanent hung state. The firmware would lock in the HAL timeout loop, never returning. The fix required reconfiguring CubeMX to 400 kHz and regenerating `i2c.c`.

---

## Part 2: Driver Architecture & Implementation

### 2.1 File Structure

```
Core/
├── Inc/
│   └── at24c32.h      (39 lines — constants + API prototypes)
└── Src/
    └── at24c32.c      (175 lines — Init, Read/Write, SelfTest)
```

### 2.2 API Design Philosophy

The driver exposes six public functions, each following a deliberate design pattern:

| Function | Purpose | Return | Blocking? |
|----------|---------|--------|-----------|
| `AT24C32_Init()` | Store handle, probe device | `bool` | Yes (100 ms max) |
| `AT24C32_ReadByte()` | Single-byte random read | `void` | Yes (~1 ms) |
| `AT24C32_WriteByte()` | Single-byte write + ack-poll | `void` | Yes (~10 ms) |
| `AT24C32_ReadPage()` | Multi-byte sequential read | `void` | Yes (~5 ms for 256B) |
| `AT24C32_WritePage()` | Multi-byte write with page chunking | `void` | Yes (~10 ms/page) |
| `AT24C32_SelfTest()` | Four-stage validation suite | `bool` | Yes (~200 ms) |

**Design Decision: Void Returns on Read/Write**

The read and write functions return `void` rather than status codes. This mirrors the existing W25Q64 driver interface and simplifies the migration. Data integrity is enforced at a higher level via checksum validation in `nv_storage.c` — a failed write will produce an incorrect checksum on read, triggering the fallback chain.

### 2.3 Page-Boundary Write Chunking

The AT24C32E has a critical hardware constraint: writes that cross a 32-byte page boundary cause address wraparound within the same page, corrupting data. The `AT24C32_WritePage()` function implements automatic chunking:

```c
void AT24C32_WritePage(uint16_t address, uint8_t *data, uint16_t len)
{
    if (_hi2c == NULL || data == NULL || len == 0) return;

    while (len > 0)
    {
        /* Bytes left before the next 32-byte page boundary */
        uint16_t page_remaining = AT24C32_PAGE_SIZE - (address % AT24C32_PAGE_SIZE);
        uint16_t chunk = (len < page_remaining) ? len : page_remaining;

        HAL_I2C_Mem_Write(_hi2c,
                          (AT24C32_Address << 1),
                          address,
                          I2C_MEMADD_SIZE_16BIT,
                          data,
                          chunk,
                          100);

        /* Acknowledge polling — wait for internal write cycle */
        uint8_t retries = 0;
        while (HAL_I2C_IsDeviceReady(_hi2c, (AT24C32_Address << 1), 1, 10) != HAL_OK)
        {
            HAL_Delay(1);
            if (++retries >= 20) break;  /* Safety cap: 20 ms */
        }

        address += chunk;
        data    += chunk;
        len     -= chunk;
    }
}
```

**Algorithm Walkthrough:**

1. Calculate bytes remaining until the next 32-byte boundary: `page_remaining = 32 - (address % 32)`
2. Write the smaller of `len` or `page_remaining` bytes in a single HAL transaction
3. After each HAL write, poll `HAL_I2C_IsDeviceReady()` in 1 ms increments until the EEPROM ACKs (indicating the internal write cycle completed)
4. Advance pointers and repeat until all bytes written

**Example:** Writing 48 bytes starting at address `0x0F90`:
- First chunk: 16 bytes (addresses `0x0F90–0x0F9F`, fills remainder of page)
- Second chunk: 32 bytes (addresses `0x0FA0–0x0FBF`, full page)
- No third chunk needed (48 - 16 - 32 = 0)

### 2.4 Timing Considerations: HAL_Delay vs osDelay

The driver exclusively uses `HAL_Delay()` for acknowledge-polling delays, not `osDelay()`. This is a deliberate choice:

| Function | Underlying Timer | Scheduler Requirement |
|----------|-----------------|----------------------|
| `HAL_Delay()` | SysTick | Works before and after RTOS starts |
| `osDelay()` | FreeRTOS kernel | **Crashes if called before `osKernelStart()`** |

This design allows the driver to be called from `main()` during early bring-up (before the scheduler runs) as well as from RTOS tasks. In production, the selftest runs inside `Em_task` after the scheduler starts, but the driver itself remains portable.

---

## Part 3: Self-Test Implementation

### 3.1 Test Strategy

The selftest validates all driver code paths using the last 128 bytes of EEPROM (`0x0F80–0x0FFF`) as a scratch area. This region is reserved and never used by production data storage, ensuring the test is non-destructive.

### 3.2 Four-Stage Validation Suite

```c
bool AT24C32_SelfTest(void)
{
    /* TEST 1: WriteByte / ReadByte — single byte at 0x0F80 */
    AT24C32_WriteByte(0x0F80, 0xA5);
    AT24C32_ReadByte(0x0F80, &rx_byte);
    if (rx_byte != 0xA5) return false;

    /* TEST 2: Overwrite validation — write 0x00, confirm cleared */
    AT24C32_WriteByte(0x0F80, 0x00);
    AT24C32_ReadByte(0x0F80, &rx_byte);
    if (rx_byte != 0x00) return false;

    /* TEST 3: Single-page write — 28 bytes, no boundary cross */
    for (uint8_t i = 0; i < 28; i++) tx_buf[i] = i + 1;
    AT24C32_WritePage(0x0F80, tx_buf, 28);
    AT24C32_ReadPage(0x0F80, rx_buf, 28);
    for (uint8_t i = 0; i < 28; i++)
        if (rx_buf[i] != tx_buf[i]) return false;

    /* TEST 4: Cross-boundary write — 48 bytes starting at 0x0F90 */
    for (uint8_t i = 0; i < 48; i++) tx_cross[i] = 0x80 + i;
    AT24C32_WritePage(0x0F90, tx_cross, 48);
    AT24C32_ReadPage(0x0F90, rx_cross, 48);
    for (uint8_t i = 0; i < 48; i++)
        if (rx_cross[i] != tx_cross[i]) return false;

    return true;  /* All tests passed */
}
```

### 3.3 Test Coverage Matrix

| Test | Operation | Address | Size | Validates |
|------|-----------|---------|------|-----------|
| 1 | WriteByte + ReadByte | `0x0F80` | 1 B | Basic HAL communication, 16-bit addressing |
| 2 | Overwrite + verify | `0x0F80` | 1 B | In-place overwrite (no erase required) |
| 3 | WritePage + ReadPage | `0x0F80` | 28 B | Multi-byte transfer within single page |
| 4 | WritePage across boundary | `0x0F90` | 48 B | Page-chunking algorithm correctness |

**Test 4 Detail:** Address `0x0F90` is 16 bytes into a 32-byte page, so the first chunk writes 16 bytes (`0x0F90–0x0F9F`), and the second chunk writes 32 bytes (`0x0FA0–0x0FBF`). This explicitly validates the boundary-splitting logic.

### 3.4 Integration Point: freertos.c

The selftest is invoked at the top of `Em_task`, immediately after the FreeRTOS scheduler starts:

```c
void Em_task(void const * argument)
{
    /* EEPROM bring-up: Init + Self-Test */
    if (AT24C32_Init(&hi2c1))
    {
        volatile bool eeprom_ok = AT24C32_SelfTest();
        /* Debug breakpoint: inspect eeprom_ok
         *   true  = all 4 tests passed
         *   false = check pull-ups, WP pin, address strapping */
        (void)eeprom_ok;
    }

    EM_Init();  /* Proceeds with normal system initialization */
    // ...
}
```

The `volatile` qualifier prevents compiler optimization from eliding the result, making it inspectable in the debugger. The `(void)` cast suppresses unused-variable warnings in release builds.

---

## Part 4: NV_Storage Integration

### 4.1 Compile-Time Feature Toggle

A single macro in `Device_SKU.h` controls the entire storage backend:

```c
/* Line 509 in Device_SKU.h */
#define EEPROM_SECONDRY_MEMORY (1)   // Use AT24C32E I2C EEPROM
// #define EEPROM_SECONDRY_MEMORY (0)   // Use W25Q64FV SPI Flash (original)
```

**Spelling Note:** The macro uses `SECONDRY` (not `SECONDARY`). This intentionally matches the existing production macro naming convention.

### 4.2 Conditional Inclusion Pattern

At the top of `nv_storage.c`, the EEPROM driver is conditionally included:

```c
#include "w25q64.h"                  /* SPI Flash driver */
#if (EEPROM_SECONDRY_MEMORY)
#include "at24c32.h"                 /* I2C EEPROM driver */
#endif
```

### 4.3 EEPROM Memory Map

The 4096-byte EEPROM is partitioned into five data domains, each with dual-copy redundancy:

```
┌────────────────────────────────────────────────────────────────┐
│ Address   │ Content                │ Size     │ End Address   │
├───────────┼────────────────────────┼──────────┼───────────────┤
│ 0x0000    │ Energy Data (Copy 1)   │ 256 B    │ 0x00FF        │
│ 0x0100    │ Energy Data (Copy 2)   │ 256 B    │ 0x01FF        │
├───────────┼────────────────────────┼──────────┼───────────────┤
│ 0x0200    │ Config Data (Copy 1)   │ 256 B    │ 0x02FF        │
│ 0x0300    │ Config Data (Copy 2)   │ 256 B    │ 0x03FF        │
├───────────┼────────────────────────┼──────────┼───────────────┤
│ 0x0400    │ HR Counter (Copy 1)    │  32 B    │ 0x041F        │
│ 0x0420    │ HR Counter (Copy 2)    │  32 B    │ 0x043F        │
├───────────┼────────────────────────┼──────────┼───────────────┤
│ 0x0440    │ Alarm Config (Copy 1)  │  96 B    │ 0x049F        │
│ 0x04A0    │ Alarm Config (Copy 2)  │  96 B    │ 0x04FF        │
├───────────┼────────────────────────┼──────────┼───────────────┤
│ 0x0500    │ Alarm History (Copy 1) │ 805 B    │ 0x0824        │
│ 0x0840    │ Alarm History (Copy 2) │ 805 B    │ 0x0B64        │
├───────────┼────────────────────────┼──────────┼───────────────┤
│ 0x0B65    │ Reserved / Unused      │ 1179 B   │ 0x0FFF        │
└────────────────────────────────────────────────────────────────┘

Total Used: 2917 bytes (71%)
Available:  1179 bytes (29% headroom for future growth)
```

The address constants are defined inside `nv_storage.c`:

```c
#if (EEPROM_SECONDRY_MEMORY)
#define EE_ENERGY_ADDR_1        0x0000U
#define EE_ENERGY_ADDR_2        0x0100U
#define EE_CONFIG_ADDR_1        0x0200U
#define EE_CONFIG_ADDR_2        0x0300U
#define EE_HR_ADDR_1            0x0400U
#define EE_HR_ADDR_2            0x0420U
#define EE_ALARMCFG_ADDR_1      0x0440U
#define EE_ALARMCFG_ADDR_2      0x04A0U
#define EE_ALARMHIST_ADDR_1     0x0500U
#define EE_ALARMHIST_ADDR_2     0x0840U
#endif
```

### 4.4 Write Function Integration Pattern

All five write functions follow an identical integration pattern. The buffer construction and checksum calculation remain device-agnostic; only the physical write operation differs:

```c
void SPIF_WriteXxxData(void)
{
    /* ═══════════════════════════════════════════════════════════
     * DEVICE-AGNOSTIC: Build payload buffer + calculate checksum
     * ═══════════════════════════════════════════════════════════ */
    uint8_t buffer[DATA_SIZE + 1];
    memcpy(buffer, &source_data, DATA_SIZE);
    buffer[DATA_SIZE] = calculate_checksum(buffer, DATA_SIZE);

#if (!EEPROM_SECONDRY_MEMORY)
    /* ═══════════════════════════════════════════════════════════
     * SPI FLASH PATH: Erase sector, then write page
     * ═══════════════════════════════════════════════════════════ */
    if (!W25Q64_IsReady()) return;
    W25Q64_EraseSector(SECTOR_COPY_1);
    W25Q64_WritePage(PAGE_COPY_1, buffer, DATA_SIZE + 1, 0);
    W25Q64_EraseSector(SECTOR_COPY_2);
    W25Q64_WritePage(PAGE_COPY_2, buffer, DATA_SIZE + 1, 0);
#else
    /* ═══════════════════════════════════════════════════════════
     * I2C EEPROM PATH: Direct write (no erase required)
     * ═══════════════════════════════════════════════════════════ */
    AT24C32_WritePage(EE_XXX_ADDR_1, buffer, DATA_SIZE + 1);
    AT24C32_WritePage(EE_XXX_ADDR_2, buffer, DATA_SIZE + 1);
#endif
}
```

**Key Difference:** SPI NOR flash requires sector erase before write; EEPROM supports direct in-place overwrite. This simplifies the EEPROM path significantly.

### 4.5 Functions Modified

| Function | Data Domain | Buffer Size | Flash Sectors | EEPROM Addresses |
|----------|-------------|-------------|---------------|------------------|
| `SPIF_WriteEnergyData` | Energy totals | 233 B | 0, 1 | `0x0000`, `0x0100` |
| `SPIF_WriteConfigData` | Calibration | 201 B | 2, 3 | `0x0200`, `0x0300` |
| `SPIF_WriteAlarmConfigData` | Alarm setpoints | 65 B | 6, 7 | `0x0440`, `0x04A0` |
| `SPIF_WriteAlarmHistoryData` | Event log | 805 B | 8, 9 | `0x0500`, `0x0840` |
| `SPIF_WriteHrCounterData` | Hour meters | 17 B | 4, 5 | `0x0400`, `0x0420` |

### 4.6 Read Function Integration Pattern

Read functions preserve the full three-level fallback hierarchy:

```
Priority 1: BKPSRAM Copy 1  →  BKPSRAM Copy 2
Priority 2: Flash/EEPROM Copy 1  →  Flash/EEPROM Copy 2
Priority 3: Safe Defaults
```

The integration pattern wraps only the flash/EEPROM access:

```c
bool EM_ReadXxxData(void)
{
    /* Level 1: Try BKPSRAM (unchanged) */
    if (getXxxFromBKPSRAM() && validate_checksum()) {
        return true;
    }

    /* Level 2: Try Copy 1 from secondary storage */
#if (!EEPROM_SECONDRY_MEMORY)
    if (W25Q64_IsReady()) {
        W25Q64_ReadPage(PAGE_COPY_1, buffer, DATA_SIZE + 1, 0);
#else
    {
        AT24C32_ReadPage(EE_XXX_ADDR_1, buffer, DATA_SIZE + 1);
#endif
        if (validate_checksum(buffer)) return true;
    }

    /* Level 3: Try Copy 2 (same pattern) */
    // ...

    /* Level 4: Apply safe defaults (unchanged) */
    ApplyDefaults();
    return false;
}
```

**EEPROM Path Simplifications:**
- Removed `W25Q64_IsReady()` guard — EEPROM is always accessible
- Removed dummy reads (`ReadDummyArrr`) — SPI bus stabilization artifact, not needed for I2C
- Checksum validation remains unchanged and handles any read corruption

---

## Part 5: Bugs Discovered & Resolved

### Bug #1: HardFault on Selftest (osDelay Before RTOS)

**Symptom:** Device entered HardFault handler immediately after reset.

**Root Cause:** Initial implementation placed `AT24C32_SelfTest()` in `main()` at `USER CODE BEGIN 2`, before `osKernelStart()`. The ack-polling loop used `osDelay(1)`, which requires the FreeRTOS scheduler.

**Stack Trace Analysis:**
```
HardFault_Handler
  → osDelay
    → vTaskDelay
      → NULL pointer dereference (scheduler not running)
```

**Resolution:** 
1. Replaced all `osDelay(1)` calls with `HAL_Delay(1)` in `at24c32.c`
2. Moved selftest invocation to `Em_task()` in `freertos.c`

### Bug #2: I2C Bus Lockup at 1 MHz

**Symptom:** `HAL_I2C_Mem_Write` returned `HAL_BUSY`, then subsequent operations hung indefinitely in HAL timeout loops.

**Root Cause:** CubeMX had I2C1 configured at 1 MHz (Fast Mode Plus). The AT24C32E maximum is 400 kHz. Exceeding this caused the EEPROM to NACK every transaction, leaving the I2C state machine in an unrecoverable error state.

**Resolution:**
1. Changed CubeMX I2C1 speed to 400 kHz Fast Mode
2. Removed `HAL_I2CEx_EnableFastModePlus()` call
3. Updated timing register to `0x00B03FDB`
4. Regenerated `i2c.c`

### Bug #3: Missing Include in freertos.c

**Symptom:** Compilation error: `'hi2c1' undeclared`

**Root Cause:** `freertos.c` called `AT24C32_Init(&hi2c1)` but did not include `i2c.h` where `hi2c1` is declared extern.

**Resolution:** Added `#include "i2c.h"` to `USER CODE BEGIN Includes` section.

### Bug #4: Config Save Skipped in EEPROM Path

**Symptom:** Calibration data not persisting to EEPROM after calibration routine completed.

**Root Cause:** `EM_SaveConfigFlash_RtcBackup()` guarded `SPIF_WriteConfigData()` with `if (W25Q64_IsReady())`. With no SPI flash present, this always returned false, silently skipping the save.

**Resolution:**
```c
#if (!EEPROM_SECONDRY_MEMORY)
    if (W25Q64_IsReady()) {
        SPIF_WriteConfigData();
    }
#else
    SPIF_WriteConfigData();  /* EEPROM always available */
#endif
```

---

## Part 6: Performance & Timing Analysis

### 6.1 Write Timing Comparison

| Data Domain | Payload | Pages | SPI Flash Time | EEPROM Time |
|-------------|---------|-------|----------------|-------------|
| Energy | 233 B | 8 | ~7 ms | ~160 ms |
| Config | 201 B | 7 | ~6 ms | ~140 ms |
| HR Counter | 17 B | 1 | ~1 ms | ~20 ms |
| Alarm Config | 65 B | 3 | ~3 ms | ~60 ms |
| Alarm History | 805 B | 26 | ~22 ms | ~520 ms |
| **Total (5 domains)** | | **45** | **~39 ms** | **~900 ms** |

EEPROM writes are ~23× slower due to the 10 ms page write cycle. This is acceptable for normal operation but impacts brownout protection.

### 6.2 Brownout Mitigation Strategy

The PVD (Programmable Voltage Detector) interrupt triggers `NV_Storage_Brownout_Callback()` when input voltage droops. The available window is typically 50–200 ms before VBAT takeover fails.

**Priority Order in Brownout:**
1. **Energy + HR Counter** — 180 ms — Most critical, change frequently
2. **Config + Alarm Config** — 200 ms — Important but rarely change
3. **Alarm History** — 520 ms — **SKIP** — Already cached in BKPSRAM

The callback automatically prioritizes based on available time remaining.

---

## Part 7: What Remains Unchanged

The following subsystems require **zero modification** regardless of `EEPROM_SECONDRY_MEMORY`:

| Subsystem | Reason |
|-----------|--------|
| **BKPSRAM Logic** | On-chip memory, no external dependency |
| **Checksum Algorithm** | Operates on byte buffers, device-agnostic |
| **Retry Logic** | Same 3-attempt, 10 ms delay pattern |
| **Default Fallback** | `InitializeEnergyDefaults()`, etc. |
| **Brownout Callback** | Calls same `SPIF_Write*` functions |
| **All Callers** | `hour_meter.c`, `di_counter.c`, `alarm_manager.c`, `config_handler.c`, `config_ui.c`, `EM.c`, `freertos.c` |
| **W25Q64 Driver** | Remains available when macro = 0 |

---

## Part 8: File Change Summary

| File | Status | Lines Changed | Description |
|------|--------|---------------|-------------|
| `Core/Inc/at24c32.h` | **Created** | 39 | Constants, API prototypes |
| `Core/Src/at24c32.c` | **Created** | 175 | Full driver implementation |
| `Core/Inc/Device_SKU.h` | Modified | 1 | Added `EEPROM_SECONDRY_MEMORY` macro |
| `Core/Src/i2c.c` | Modified | ~5 | CubeMX regen: 400 kHz timing |
| `Core/Src/freertos.c` | Modified | 12 | Init + SelfTest in `Em_task` |
| `Core/Src/nv_storage.c` | Modified | ~200 | Conditional paths in 10 functions |

---

## Part 9: Testing Verification Checklist

### Compilation Tests
- [x] `EEPROM_SECONDRY_MEMORY = 1`: Zero errors, zero warnings
- [x] `EEPROM_SECONDRY_MEMORY = 0`: Zero errors, zero warnings (original path preserved)

### Selftest Validation
- [x] `eeprom_ok == true` at debugger breakpoint
- [x] Test 1: Single-byte write/read verified
- [x] Test 2: Overwrite verification passed
- [x] Test 3: 28-byte page write/read verified
- [x] Test 4: 48-byte boundary-crossing write verified

### Power Cycle Persistence
- [x] Energy data survives power cycle
- [x] Config data survives power cycle
- [x] HR counter survives power cycle
- [x] Alarm config survives power cycle
- [x] Alarm history survives power cycle

### Fallback Chain Validation
- [x] Corrupt Copy 1 → falls back to Copy 2
- [x] Both copies corrupt → falls back to defaults
- [x] BKPSRAM-first priority maintained

### Integration Tests
- [x] Calibration save via `EM_SaveConfigFlash_RtcBackup()` writes to EEPROM
- [x] Brownout callback saves critical data to EEPROM
- [x] Full system end-to-end run with all modules active

---

## Appendix A: Quick Reference

### Switching Storage Backend

Edit `Core/Inc/Device_SKU.h` line 509:

```c
#define EEPROM_SECONDRY_MEMORY (1)  // AT24C32E EEPROM
#define EEPROM_SECONDRY_MEMORY (0)  // W25Q64FV SPI Flash
```

Rebuild project. No other changes required.

### Selftest Debug Procedure

1. Set breakpoint at `(void)eeprom_ok;` in `freertos.c` line 247
2. Run to breakpoint
3. Inspect `eeprom_ok` in debugger:
   - `true` = All 4 tests passed
   - `false` = Check: I2C pull-ups, WP pin state, A0/A1/A2 strapping

### EEPROM I2C Address Calculation

```
7-bit address: 0x50 (with A0=A1=A2=GND)
8-bit write:   0xA0 (0x50 << 1 | 0)
8-bit read:    0xA1 (0x50 << 1 | 1)
```

---

## Appendix B: Lessons Learned

1. **Always verify I2C clock speed against device datasheet** — Exceeding maximum frequency causes non-obvious failures that look like hardware connection issues.

2. **Use HAL_Delay() in drivers that may run before RTOS** — FreeRTOS delay functions require the scheduler; HAL delay works universally.

3. **Page boundary handling is non-negotiable for EEPROM** — Unlike flash which has sector granularity, EEPROM page wrap corrupts data silently.

4. **Compile-time switches enable safe A/B testing** — The dual-path approach allowed validation without breaking the production SPI flash path.

5. **Reserve a test region in EEPROM** — The last 128 bytes (`0x0F80–0x0FFF`) provide a safe sandbox for selftest without touching production data.

---

*End of Development Log*
