[[Titan]]


## 1. Background

The firmware already implements a redundant, multi-level persistence strategy for:

- System configuration (`EM_CONFIG`)
- Energy data (`ptrTempEnergies`)
- Hour-meter / counter data

This strategy uses:

- **RTC Backup SRAM (BKPSRAM)** – two copies + checksum
- **External SPI flash (W25Q64)** – two copies + checksum
- **Fallback to defaults** if both storage levels are invalid

The PR review comment requests that **Alarm configuration** be stored using the *same* strategy as the above, instead of flash-only.

---

## 2. Existing Patterns

Key reference implementations are in:

- [TorTitan_withStm32CubeIde/Core/Src/nv_storage.c](TorTitan_withStm32CubeIde/Core/Src/nv_storage.c)
- [TorTitan_withStm32CubeIde/Core/Inc/nv_storage.h](TorTitan_withStm32CubeIde/Core/Inc/nv_storage.h)
- [TorTitan_withStm32CubeIde/Core/Src/EM.c](TorTitan_withStm32CubeIde/Core/Src/EM.c)
- [TorTitan_withStm32CubeIde/Core/Inc/EM.h](TorTitan_withStm32CubeIde/Core/Inc/EM.h)

### 2.1 Configuration (`EM_CONFIG`)

**Flash layout**

- Two flash copies with checksum:
  - `SPIF_WriteConfigData()`
  - `EM_ReadConfigData()`:
    - Try BKPSRAM
    - Then flash copy 1
    - Then flash copy 2
    - Then defaults

**BKPSRAM layout**

- Backup SRAM offsets: `CONFIG_OFFSET_1`, `CONFIG_OFFSET_2`
- Helpers:
  - `saveConfigDataToBKPSRAM()`
  - `getConfigDataFromBKPSRAM()`

**Combined save helper**

- `EM_SaveConfigFlash_RtcBackup()`:
  - Writes `EM_CONFIG` to flash (dual copy) if flash is ready
  - Always writes two copies to BKPSRAM
  - Used from various configuration / scheduler setters in [TorTitan_withStm32CubeIde/Core/Src/EM.c](TorTitan_withStm32CubeIde/Core/Src/EM.c)

### 2.2 Energy (`ptrTempEnergies`)

**Flash**

- `SPIF_WriteEnergyData()` (dual copy + checksum)
- `EM_ReadEnergyData()`:
  - BKPSRAM → flash copy 1 → flash copy 2 → defaults

**BKPSRAM**

- Offsets: `ENERGIES_OFFSET_1`, `ENERGIES_OFFSET_2`
- Helpers:
  - `saveEnergiesToBKPSRAM()`
  - `getEnergiesFromBKPSRAM()`

---

## 3. Current Alarm Configuration State

Alarm configuration currently uses:

- Global NV mirror: `EM_ALARM_CONFIG` of type `alarm_nv_config_t`
- Flash mapping and dual-copy write/read:
  - `EM_ALARM_CONFIG_SECTOR_1`, `EM_ALARM_CONFIG_SECTOR_2`, etc.
  - `SPIF_WriteAlarmConfigData()` – writes two flash copies with checksum
  - `EM_ReadAlarmConfigData()` – reads from flash with redundancy and checksum

Limitations vs. review request:

- **No BKPSRAM storage** for `EM_ALARM_CONFIG`
- `EM_ReadAlarmConfigData()` is commented out in the “read-all-config” path in [TorTitan_withStm32CubeIde/Core/Src/EM.c](TorTitan_withStm32CubeIde/Core/Src/EM.c)
- No unified “save both flash and BKPSRAM” helper for alarms

---

## 4. Requirements From PR Comment

1. **Mirroring existing configuration/energy strategy**  
   Alarm configuration must follow the same persistence approach as `EM_CONFIG` and energies:
   - Dual-copy in RTC Backup SRAM (BKPSRAM) with checksum
   - Dual-copy in external flash with checksum
   - Multi-level read strategy (BKPSRAM → flash copy 1 → flash copy 2 → defaults)

2. **Two BKPSRAM copies**  
   Store **two copies** of alarm configuration in BKPSRAM for redundancy.

3. **Two flash copies (already done)**  
   Dual-copy persistent storage in W25Q64 flash is already implemented and should be kept.

4. **Centralized read and write APIs**  
   - A dedicated *read* function (`EM_ReadAlarmConfigData`) that follows the multi-level strategy.
   - A *save* helper that writes both flash and BKPSRAM in one call (like `EM_SaveConfigFlash_RtcBackup`).

---

## 5. Proposed Alarm Configuration Persistence Design

### 5.1 BKPSRAM Layout

Add new offsets and size definitions in [TorTitan_withStm32CubeIde/Core/Src/nv_storage.c](TorTitan_withStm32CubeIde/Core/Src/nv_storage.c), next to the existing BKPSRAM layout:

- `ALARM_CONFIG_OFFSET_1` – first copy in BKPSRAM
- `ALARM_CONFIG_OFFSET_2` – second copy in BKPSRAM
- `ALARM_CONFIG_SIZE` (or reuse `EM_ALARM_CONFIG_SIZE`)

Constraints:

- Layout must remain within the 4 KB BKPSRAM region (`BKPSRAM_BASE_ADDR` to `BKPSRAM_BASE_ADDR + 0x1000`).
- Offsets should not overlap existing energy/config/HR-counter regions.

### 5.2 BKPSRAM Save/Load Helpers

Implement two new functions in [TorTitan_withStm32CubeIde/Core/Src/nv_storage.c](TorTitan_withStm32CubeIde/Core/Src/nv_storage.c) and declare them in [TorTitan_withStm32CubeIde/Core/Inc/nv_storage.h](TorTitan_withStm32CubeIde/Core/Inc/nv_storage.h):

1. `void saveAlarmConfigToBKPSRAM(void);`

   Behavior:

   - Build `buffer[EM_ALARM_CONFIG_SIZE + 1]`
   - `memcpy` from `&EM_ALARM_CONFIG` into `buffer`
   - Compute checksum using `calculate_checksum(buffer, EM_ALARM_CONFIG_SIZE)`
   - Append checksum at `buffer[EM_ALARM_CONFIG_SIZE]`
   - `bkSRAM_Init()`, then `bkSRAM_EnableWriteAccess()`
   - Write `buffer` to:
     - `BKPSRAM_BASE_ADDR + ALARM_CONFIG_OFFSET_1`
     - `BKPSRAM_BASE_ADDR + ALARM_CONFIG_OFFSET_2`
   - Use `SCB_CleanDCache_by_Addr()` for each write and then `__DSB()`, `__ISB()`
   - `bkSRAM_DisableWriteAccess()`

   This mirrors `saveConfigDataToBKPSRAM()` / `saveEnergiesToBKPSRAM()`.

2. `bool getAlarmConfigFromBKPSRAM(void);`

   Behavior:

   - `bkSRAM_Init()`
   - Read `CONFIG_SIZE + 1` bytes from:
     - `BKPSRAM_BASE_ADDR + ALARM_CONFIG_OFFSET_1`
     - If invalid, try `ALARM_CONFIG_OFFSET_2`
   - Copies data into `EM_ALARM_CONFIG` and validates checksum via `validate_checksum(...)`
   - Returns `true` if either copy is valid, `false` otherwise

   This mirrors `getConfigDataFromBKPSRAM()` / `getEnergiesFromBKPSRAM()`.

### 5.3 Alarm Read Path (`EM_ReadAlarmConfigData`)

Extend `EM_ReadAlarmConfigData()` to follow the same multi-level strategy as `EM_ReadConfigData()` / `EM_ReadEnergyData()`:

1. Try BKPSRAM first:

   ```c
   bool read_success = false;

   if (getAlarmConfigFromBKPSRAM()) {
       if (AlarmConfig_IsValid(&EM_ALARM_CONFIG)) {
           read_success = true;
       }
   }