[[Titan]]
# Alarm Configuration NV Storage

## Overview

The alarm configuration (`EM_ALARM_CONFIG`) is now persisted using the same dual-copy redundancy architecture as `EM_CONFIG`, `Energies`, and `HR Counter` data. Two copies are stored in **RTC Backup SRAM** (VBAT-backed) and two copies in **external SPI flash** (W25Q64FV).

## Storage Layout

### RTC Backup SRAM (0x38800000, 4 KB)

| Data            | Copy 1 Offset | Copy 2 Offset |
|-----------------|---------------|---------------|
| Energies        | 0             | 240           |
| EM_CONFIG       | 480           | 680           |
| HR Counter      | 880           | 1080          |
| **Alarm Config**| **1280**      | **1480**      |

### SPI Flash (W25Q64FV)

| Data            | Sector 1 | Sector 2 | Page 1 | Page 2 |
|-----------------|----------|----------|--------|--------|
| Energies        | 0        | 1        | 0      | 16     |
| EM_CONFIG       | 2        | 3        | 32     | 48     |
| HR Counter      | 4        | 5        | 64     | 80     |
| **Alarm Config**| **6**    | **7**    | **96** | **112**|

## Load Priority (Boot)

```
BKPSRAM Copy 1 → BKPSRAM Copy 2 → Flash Copy 1 → Flash Copy 2 → Safe Defaults
```

This matches the hierarchy used by all other configurations.

## Data Structure

```c
typedef struct {
    uint16_t enabled_mask;              // Bit N → alarm ID N enabled
    float    setpoint[EM_ALARM_COUNT];  // Per-alarm threshold values
    uint32_t sustain_time_sec;          // Global sustain time (seconds)
    bool     autoclear;                 // Global auto-clear mode
    uint32_t autoclear_delay_sec;       // Global auto-clear delay (seconds)
    uint8_t  do_map;                    // Global DO mapping (0=NONE, 1=DO1, 2=DO2)
} alarm_nv_config_t;
```

## Save Paths

| Trigger                     | Flash | BKPSRAM |
|-----------------------------|-------|---------|
| `AlarmManager_SaveToNV()`   | ✓     | ✓       |
| Brownout callback (PVD IRQ) | ✓     | ✓       |
| Factory reset               | ✓     | ✓       |

`AlarmManager_SaveToNV()` is called from:
- `AlarmManager_SetConfig()` — BLE/UART configuration changes
- `config_ParsAlarmConfig()` — SET ALARM_CONFIG command via config handler

## Functions Added

| Function                       | File          | Purpose                              |
|--------------------------------|---------------|--------------------------------------|
| `saveAlarmConfigToBKPSRAM()`   | nv_storage.c  | Write dual copies to BKPSRAM         |
| `getAlarmConfigFromBKPSRAM()`  | nv_storage.c  | Read from BKPSRAM with checksum      |

## Functions Modified

| Function                       | File            | Change                                         |
|--------------------------------|-----------------|-------------------------------------------------|
| `EM_ReadAlarmConfigData()`     | nv_storage.c    | Added BKPSRAM as first-priority source           |
| `AlarmManager_SaveToNV()`      | alarm_manager.c | Added `saveAlarmConfigToBKPSRAM()` call          |
| `NV_Storage_Brownout_Callback()`| nv_storage.c   | Added `saveAlarmConfigToBKPSRAM()` call          |
| `perform_factory_reset()`      | config_ui.c     | Added alarm config reset to defaults + persist   |
