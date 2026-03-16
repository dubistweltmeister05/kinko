[[Titan]]

# Alarm Configuration NV Storage — Detailed Explanation
## The Problem

Previously, `EM_ALARM_CONFIG` was only stored in **external SPI flash** (two copies). Other critical data like `EM_CONFIG`, `Energies`, and `HR Counter` used a more robust strategy: they were stored in both **RTC Backup SRAM** (battery-backed, instant access) and **SPI flash** (non-volatile, slower). The review feedback was to bring alarm config in line with these.

## The Storage Architecture

### Three tiers of persistence, four copies total:

1. **RTC Backup SRAM** (BKPSRAM at `0x38800000`) — 4 KB of SRAM powered by a coin cell battery (VBAT). Survives main power loss. Two copies at offsets **1280** and **1480**.

2. **SPI Flash** (W25Q64FV) — External NOR flash on the SPI bus. Two copies in sectors **6** and **7** (pages 96 and 112).

Every copy has a **1-byte checksum** appended: the buffer is `sizeof(alarm_nv_config_t) + 1` bytes. The checksum is the sum of all data bytes plus 1 (to avoid a valid all-zero checksum).

---

## How Saving Works

### `AlarmManager_SaveToNV()` — alarm_manager.c

This is the central save entry point. Called whenever alarm configuration changes (via BLE, UART, or test mode).

**What it does:**
1. Iterates through the runtime `alarms[]` array and builds `EM_ALARM_CONFIG`:
   - Constructs `enabled_mask` bitmask from each alarm's `.enabled` flag
   - Copies each alarm's `.setpoint` into the config array
   - Takes global parameters (sustain time, auto-clear, DO mapping) from `alarms[0]`
2. Calls `SPIF_WriteAlarmConfigData()` — writes to SPI flash (both copies)
3. Calls `saveAlarmConfigToBKPSRAM()` — writes to BKPSRAM (both copies)

### `SPIF_WriteAlarmConfigData()` — nv_storage.c

1. Checks `W25Q64_IsReady()` — if flash is busy, returns immediately
2. Copies `EM_ALARM_CONFIG` into a local buffer
3. Calculates checksum and appends it as the last byte
4. **Copy 1**: Erases sector 6, writes buffer to page 96
5. **Copy 2**: Erases sector 7, writes buffer to page 112
6. If any erase/write fails, sets `ReadResultDummy = false` as an error flag

### `saveAlarmConfigToBKPSRAM()` — nv_storage.c

1. Copies `EM_ALARM_CONFIG` into a local buffer with checksum
2. Calls `bkSRAM_Init()` — one-time initialization of VBAT regulator, backup domain unlock, and AHB1 clock enable
3. Calls `bkSRAM_EnableWriteAccess()` — enables write to backup domain
4. **Copy 1**: `memcpy` to address `0x38800000 + 1280`, then `SCB_CleanDCache_by_Addr` to flush the D-cache so the write actually reaches physical SRAM
5. **Copy 2**: Same at offset 1480
6. `__DSB(); __ISB()` — memory barrier to guarantee write completion
7. `bkSRAM_DisableWriteAccess()` — re-locks the backup domain

The D-cache flush (`SCB_CleanDCache_by_Addr`) is critical on STM32H7 — without it, data could sit in the CPU's cache and never reach BKPSRAM before a power loss.

---

## How Loading Works

### `EM_ReadAlarmConfigData()` — nv_storage.c

Called during boot via `AlarmManager_Init()` → `AlarmManager_LoadFromNV()` → `AlarmManager_ApplyConfigFromEM()`.

**Priority chain** (first valid source wins):

```
Step 1: BKPSRAM Copy 1 → checksum valid? → done
        BKPSRAM Copy 2 → checksum valid? → done
Step 2: Flash Copy 1   → 3 retries, checksum valid? → done
Step 3: Flash Copy 2   → 3 retries, checksum valid? → done
Step 4: Safe defaults   → all alarms disabled, zero setpoints
```

**BKPSRAM path** (`getAlarmConfigFromBKPSRAM()`):
1. `bkSRAM_Init()` — ensure backup SRAM is accessible
2. Reads `EM_ALARM_CONFIG_SIZE + 1` bytes from offset 1280
3. Extracts data into `EM_ALARM_CONFIG`, extracts stored checksum
4. Calls `validate_checksum()` — recalculates and compares
5. If invalid, tries offset 1480 (second copy)
6. Returns `true` if either copy is valid

**Flash path** (inline in `EM_ReadAlarmConfigData()`):
1. Does a dummy 17-byte read first (flash stability workaround)
2. Reads the full buffer from page 96 (or 112 for copy 2)
3. Validates checksum
4. Retries up to 3 times with 10ms delays between attempts

**Defaults** (if everything fails):
```c
EM_ALARM_CONFIG.enabled_mask = 0;          // All alarms disabled
EM_ALARM_CONFIG.setpoint[0..10] = 0.0f;    // Zero thresholds
EM_ALARM_CONFIG.sustain_time_sec = 0;
EM_ALARM_CONFIG.autoclear = false;
EM_ALARM_CONFIG.autoclear_delay_sec = 0;
EM_ALARM_CONFIG.do_map = 0;                // No DO mapping
```

### `AlarmManager_ApplyConfigFromEM()` — alarm_manager.c

After `EM_ReadAlarmConfigData()` populates `EM_ALARM_CONFIG`, this function distributes it into the runtime `alarms[]` array:
- Each alarm gets its `.enabled` flag from the bitmask
- Each alarm gets its `.setpoint` from the array
- Global settings (sustain time, auto-clear, DO map) are applied to all alarms

---

## Brownout Protection

### `NV_Storage_Brownout_Callback()` — nv_storage.c

Called from the **PVD interrupt handler** when the programmable voltage detector senses a power droop. This is essentially "power is dying, save everything NOW."

**Sequence:**
1. Startup guard: ignores triggers in the first 5 seconds (avoids false triggers during boot)
2. Re-entrancy guard: `u32WritingInProgress` flag prevents overlapping saves
3. Saves HR counter → Energy → Config → **Alarm Config** to flash
4. Saves **Alarm Config** to BKPSRAM

This ensures alarm configuration survives even unexpected power loss.

---

## Factory Reset

### `perform_factory_reset()` — config_ui.c

Resets `EM_CONFIG` via `ResetToFactorySettings()`, then also resets `EM_ALARM_CONFIG` to safe defaults (all alarms disabled, zero setpoints, no DO mapping). Persists the reset config to both flash and BKPSRAM so the old alarm settings don't survive a factory reset.

---

## Data Flow Summary

```
┌─────────────────────────────────────────────────────┐
│  BLE / UART SET command                             │
│  or test mode config                                │
└──────────────┬──────────────────────────────────────┘
               ▼
     AlarmManager_SetConfig()
        │  updates alarms[] runtime array
        ▼
     AlarmManager_SaveToNV()
        │  builds EM_ALARM_CONFIG from alarms[]
        ├──► SPIF_WriteAlarmConfigData()  → Flash sectors 6 & 7
        └──► saveAlarmConfigToBKPSRAM()   → BKPSRAM offsets 1280 & 1480

┌─────────────────────────────────────────────────────┐
│  Boot / Power-on                                    │
└──────────────┬──────────────────────────────────────┘
               ▼
     AlarmManager_Init()
        ▼
     AlarmManager_LoadFromNV()
        ▼
     AlarmManager_ApplyConfigFromEM()
        │  calls EM_ReadAlarmConfigData()
        │     tries: BKPSRAM → Flash Copy 1 → Flash Copy 2 → Defaults
        │  populates EM_ALARM_CONFIG
        ▼
     distributes config into alarms[] runtime array

┌─────────────────────────────────────────────────────┐
│  Power loss (PVD interrupt)                         │
└──────────────┬──────────────────────────────────────┘
               ▼
     NV_Storage_Brownout_Callback()
        ├──► SPIF_WriteAlarmConfigData()  → Flash
        └──► saveAlarmConfigToBKPSRAM()   → BKPSRAM
```

## Why Dual Copies at Each Tier?

If power is lost mid-write (e.g., during a flash sector erase), one copy gets corrupted. The second copy remains valid from the previous successful write. On the next boot, the checksum validation detects the corrupt copy and falls through to the good one. This is a standard industrial pattern for safety-critical embedded storage.
