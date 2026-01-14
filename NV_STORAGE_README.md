# Non-Volatile Storage Module (nv_storage.c)

## Overview

The Non-Volatile Storage module provides comprehensive data persistence functionality for the STM32H7-based energy meter system. It implements a multi-level storage hierarchy with redundancy, error recovery, and data integrity validation to ensure critical system data survives power loss and corruption events.

## Hardware Components

- **SPI Flash (W25Q64FV)**: 64Mbit external flash memory for persistent storage
- **Backup SRAM (BKPSRAM)**: 4KB VBAT-backed SRAM for critical data retention during power loss
- **RTC Integration**: Real-time clock integration for timestamping
- **Power Management**: PVD (Programmable Voltage Detector) for brownout detection

## Storage Architecture

### Multi-Level Storage Hierarchy

The module implements a sophisticated storage hierarchy for maximum reliability:

1. **RTC RAM (BKPSRAM)** - Highest priority, VBAT-backed, fastest access
2. **Flash Copy 1** - First flash sector copy with checksum validation
3. **Flash Copy 2** - Second flash sector copy with checksum validation
4. **Default Values** - Fallback when all storage sources fail

### Dual-Copy Redundancy

All data types are stored in **two separate flash sectors** to protect against:
- Sector corruption
- Partial write failures
- Flash wear-out
- Power interruption during writes

### Memory Layout

#### Backup RAM (BKPSRAM) Layout
```
Base Address: 0x38800000

Energy Data:
  - Copy 1: Offset 0     (Size: 232 bytes) - Energies_MD_double_t
  - Copy 2: Offset 240   (Size: 232 bytes)

Configuration Data:
  - Copy 1: Offset 480   (Size: ~200 bytes) - config_para_t
  - Copy 2: Offset 680   (Size: ~200 bytes)

Hour Counter Data:
  - Copy 1: Offset 880   (Size: variable)   - HrCounterData_t
  - Copy 2: Offset 1080  (Size: variable)
```

#### SPI Flash Layout
```
Sector Size: 4KB (0x1000)

Energy Data:
  - Copy 1: Sector 0, Page 0
  - Copy 2: Sector 1, Page 16

Configuration Data:
  - Copy 1: Sector 2, Page 32
  - Copy 2: Sector 3, Page 48

Hour Counter Data:
  - Copy 1: Sector 4, Page 64
  - Copy 2: Sector 5, Page 80
```

## Data Integrity

### Checksum Algorithm

The module uses a simple but effective 8-bit checksum algorithm:
```
checksum = (sum of all bytes) + 1
```

The "+1" offset prevents confusion with all-zero checksums. Each stored data block includes its checksum appended at the end.

### Validation Process

1. Read data from storage
2. Calculate checksum of received data
3. Compare with stored checksum
4. Accept data only if checksums match
5. Retry or fallback if validation fails

## Function Reference

### Data Integrity Functions

#### `calculate_checksum()`
```c
static uint8_t calculate_checksum(uint8_t *data, uint32_t size)
```

**Purpose**: Calculate 8-bit checksum for data integrity validation

**Working**:
1. Initialize checksum accumulator to 0
2. Sum all bytes in the data buffer
3. Add 1 to avoid all-zero confusion
4. Return 8-bit checksum value

**Parameters**:
- `data`: Pointer to data buffer
- `size`: Size of data in bytes

**Returns**: 8-bit checksum value

**Usage Example**:
```c
uint8_t buffer[100];
uint8_t checksum = calculate_checksum(buffer, 100);
```

---

#### `validate_checksum()`
```c
static bool validate_checksum(uint8_t *data, uint32_t size, uint8_t stored_checksum)
```

**Purpose**: Validate data integrity by comparing calculated and stored checksums

**Working**:
1. Calculate checksum of current data
2. Compare with stored checksum
3. Return true if match (data valid), false if mismatch (data corrupted)

**Parameters**:
- `data`: Pointer to data buffer to validate
- `size`: Size of data in bytes
- `stored_checksum`: Previously stored checksum value

**Returns**: 
- `true` - Data is valid (checksums match)
- `false` - Data is corrupted (checksums don't match)

**Usage Example**:
```c
if (validate_checksum(buffer, 100, stored_checksum)) {
    // Data is valid, proceed
} else {
    // Data corrupted, use backup
}
```

---

### Flash Write Functions

#### `SPIF_WriteEnergyData()`
```c
void SPIF_WriteEnergyData(void)
```

**Purpose**: Write energy measurement data to SPI flash with dual-copy redundancy

**Working**:
1. Check if flash is ready for operation
2. Create buffer: energy data (232 bytes) + checksum (1 byte)
3. Copy energy data from `ptrTempEnergies` to buffer
4. Calculate checksum and append to buffer
5. **Write Copy 1**: Erase sector 0, write to page 0
6. Verify Copy 1 write success
7. **Write Copy 2**: Erase sector 1, write to page 16
8. Verify Copy 2 write success
9. Set error flags if operations fail

**Data Stored**: `Energies_MD_double_t` structure (double-precision energy values)

**Flash Locations**:
- Copy 1: Sector 0 (Address: 0x0000)
- Copy 2: Sector 1 (Address: 0x1000)

**Error Handling**: Sets `EraseResult` and `WriteResult` flags for error tracking

---

#### `SPIF_WriteConfigData()`
```c
void SPIF_WriteConfigData(void)
```

**Purpose**: Write system configuration data to SPI flash with dual-copy redundancy

**Working**:
1. Check flash readiness
2. Create buffer: config data + checksum
3. Copy configuration from `EM_CONFIG` structure
4. Calculate and append checksum
5. **Write Copy 1**: Erase sector 2, write to page 32
6. Verify Copy 1 write
7. **Write Copy 2**: Erase sector 3, write to page 48
8. Verify Copy 2 write

**Data Stored**: `config_para_t` structure (system calibration and configuration parameters)

**Flash Locations**:
- Copy 1: Sector 2 (Address: 0x2000)
- Copy 2: Sector 3 (Address: 0x3000)

---

#### `SPIF_WriteHrCounterData()`
```c
void SPIF_WriteHrCounterData(void)
```

**Purpose**: Write hour meter counter data to SPI flash with dual-copy redundancy

**Working**:
1. Check flash readiness
2. Create buffer: hour counter data + checksum
3. Copy hour counter data
4. Calculate and append checksum
5. **Write Copy 1**: Erase sector 4, write to page 64
6. **Write Copy 2**: Erase sector 5, write to page 80
7. Verify both writes

**Data Stored**: `HrCounterData_t` structure (runtime counters and hour meter data)

**Flash Locations**:
- Copy 1: Sector 4 (Address: 0x4000)
- Copy 2: Sector 5 (Address: 0x5000)

---

### Flash Read Functions

#### `EM_ReadEnergyData()`
```c
void EM_ReadEnergyData(void)
```

**Purpose**: Read energy data with multi-level storage hierarchy and automatic error recovery

**Working** (Multi-level fallback strategy):

**Level 1 - RTC RAM (Highest Priority)**:
1. Attempt to read from backup SRAM
2. If successful and valid, use this data
3. If failed, proceed to Level 2

**Level 2 - Flash Copy 1**:
1. Check flash readiness
2. Retry loop (up to 3 attempts):
   - Read data from Flash Copy 1 (Sector 0)
   - Read stored checksum
   - Validate data integrity
   - Check for valid timestamp (not 0 or 0xFFFFFFFF)
   - If valid, accept data and exit
   - If invalid, delay 10ms and retry
3. If all retries fail, proceed to Level 3

**Level 3 - Flash Copy 2**:
1. Reset retry counter
2. Retry loop (up to 3 attempts):
   - Read data from Flash Copy 2 (Sector 1)
   - Validate with checksum
   - Check timestamp validity
   - If valid, accept and exit
   - If invalid, retry with delay
3. If all retries fail, proceed to Level 4

**Level 4 - Default Values**:
1. Call `InitializeEnergyDefaults()`
2. Load factory default energy values
3. Set initialization flag

**Features**:
- Automatic retry with 10ms delays for transient errors
- Checksum validation at each level
- Data sanity checks (timestamp validation)
- Graceful degradation through storage hierarchy

**Result**: Energy data loaded into `EM_PARAMS.EM_Energies` and `ptrTempEnergies`

---

#### `EM_ReadConfigData()`
```c
void EM_ReadConfigData(void)
```

**Purpose**: Read configuration data with multi-level storage hierarchy and error recovery

**Working** (Similar to energy data but for configuration):

**Level 1 - RTC RAM**:
1. Attempt `getConfigDataFromBKPSRAM()`
2. If successful, use configuration data
3. Set `bkpsram_failed` flag if RTC RAM read fails

**Level 2 - Flash Copy 1**:
1. Check flash readiness
2. Retry loop (up to 3 attempts):
   - Read from Flash Copy 1 (Sector 2)
   - Validate checksum
   - Perform data sanity checks (non-zero calibration values, no NaN)
   - If valid, load into `EM_CONFIG`
3. If all retries fail, proceed to Copy 2

**Level 3 - Flash Copy 2**:
1. Reset retry counter
2. Retry loop (up to 3 attempts):
   - Read from Flash Copy 2 (Sector 3)
   - Validate and check sanity
   - If valid, load configuration
3. If failed, proceed to defaults

**Level 4 - Default Configuration**:
1. Call `ApplyDefaultConfig()`
2. Load factory default configuration
3. System starts in uncalibrated state

**Features**:
- Configuration validation (checks for NaN values)
- Calibration status verification
- Automatic fallback to safe defaults

---

#### `EM_ReadHrCounterData()`
```c
void EM_ReadHrCounterData(void)
```

**Purpose**: Read hour meter counter data with multi-level fallback

**Working**: Same multi-level hierarchy as energy and config data:
1. Try RTC RAM first
2. Try Flash Copy 1 with retries
3. Try Flash Copy 2 with retries
4. Fall back to default values

**Data Loaded**: Hour meter counters and runtime statistics

---

### Backup RAM Functions

#### `bkSRAM_Init()`
```c
void bkSRAM_Init(void)
```

**Purpose**: Initialize Backup SRAM system for read/write operations

**Working**:
1. Check if already initialized (one-time initialization guard)
2. If already initialized, return early
3. **Enable BKPSRAM Clock**: `__HAL_RCC_BKPRAM_CLK_ENABLE()`
4. **Enable VBAT Regulator**: `HAL_PWREx_EnableBkUpReg()`
5. **Unlock Backup Domain**: `HAL_PWR_EnableBkUpAccess()`
6. **Wait for Regulator Ready**: Poll `PWR->CR2 & PWR_CR2_BRRDY`
7. Set initialization flag to prevent re-initialization

**Important Notes**:
- Must be called before any backup SRAM read/write operations
- Uses static flag to ensure one-time initialization after reset
- Enables VBAT-backed power supply for data retention during power loss

---

#### `bkSRAM_EnableWriteAccess()`
```c
void bkSRAM_EnableWriteAccess(void)
```

**Purpose**: Enable write access to backup domain

**Working**:
1. Unlocks the backup domain for write operations
2. Allows modification of BKPSRAM contents
3. Called before write operations, disabled after

**Note**: Currently commented out as access control is managed at init level

---

#### `bkSRAM_DisableWriteAccess()`
```c
void bkSRAM_DisableWriteAccess(void)
```

**Purpose**: Disable write access to backup domain for data protection

**Working**:
1. Re-locks the backup domain
2. Prevents accidental writes to BKPSRAM
3. Enhances data integrity

**Note**: Currently commented out as access control is managed at init level

---

#### `saveConfigDataToBKPSRAM()`
```c
void saveConfigDataToBKPSRAM(void)
```

**Purpose**: Save configuration data to backup RAM with dual-copy redundancy

**Working**:
1. Create buffer: config data (CONFIG_SIZE) + checksum (1 byte)
2. Copy configuration from `EM_CONFIG` to buffer
3. Calculate checksum and append to buffer
4. Initialize RTC SRAM (if not already initialized)
5. Enable write access to backup domain
6. **Write Copy 1** (Atomic operation):
   - Calculate destination address: `BKPSRAM_BASE_ADDR + CONFIG_OFFSET_1`
   - Copy data: `memcpy(dst1, buffer, CONFIG_SIZE + 1)`
   - Clean D-Cache to ensure data reaches SRAM3: `SCB_CleanDCache_by_Addr()`
7. **Write Copy 2** (Atomic operation):
   - Calculate destination: `BKPSRAM_BASE_ADDR + CONFIG_OFFSET_2`
   - Copy data
   - Clean D-Cache
8. Memory barriers: `__DSB()` and `__ISB()` to ensure completion
9. Disable write access

**Key Features**:
- **Atomic writes**: Each copy written separately to ensure at least one valid copy during reset
- **Cache coherency**: D-Cache cleaning ensures data reaches physical SRAM
- **Memory barriers**: Ensure write completion before power loss

---

#### `getConfigDataFromBKPSRAM()`
```c
bool getConfigDataFromBKPSRAM(void)
```

**Purpose**: Read configuration data from backup RAM with validation

**Working**:
1. Initialize backup SRAM
2. **Try Copy 1**:
   - Read data + checksum from offset 480
   - Validate checksum
   - Perform sanity checks (non-zero values, no NaN)
   - If valid, copy to `EM_CONFIG` and return true
3. **Try Copy 2** (if Copy 1 failed):
   - Read from offset 680
   - Validate checksum
   - Perform sanity checks
   - If valid, copy to `EM_CONFIG` and return true
4. If both copies invalid, return false

**Returns**:
- `true` - Configuration successfully loaded from backup RAM
- `false` - Both copies invalid, need to try flash or defaults

---

#### `saveEnergiesToBKPSRAM()`
```c
void saveEnergiesToBKPSRAM(void)
```

**Purpose**: Save double-precision energy data to backup RAM

**Working**:
1. Create buffer with energy data + checksum
2. Copy from `ptrTempEnergies` (double-precision structure)
3. Calculate and append checksum
4. Initialize RTC SRAM
5. Enable write access
6. **Write Copy 1** to offset 0 (atomic)
   - Clean D-Cache
7. **Write Copy 2** to offset 240 (atomic)
   - Clean D-Cache
8. Memory barriers
9. Disable write access

**Data Size**: 232 bytes + 1 byte checksum = 233 bytes per copy

---

#### `getEnergiesFromBKPSRAM()`
```c
bool getEnergiesFromBKPSRAM(void)
```

**Purpose**: Read energy data from backup RAM with dual-copy validation

**Working**:
1. Initialize backup SRAM
2. **Try Copy 1** (offset 0):
   - Read 233 bytes (data + checksum)
   - Validate checksum
   - Check for valid timestamp
   - If valid, copy to `ptrTempEnergies` and return true
3. **Try Copy 2** (offset 240):
   - Same validation process
   - If valid, copy and return true
4. Return false if both copies invalid

**Returns**:
- `true` - Energy data successfully loaded
- `false` - Both copies invalid

---

#### `saveHrCounterDataToBKPSRAM()`
```c
void saveHrCounterDataToBKPSRAM(void)
```

**Purpose**: Save hour meter counter data to backup RAM

**Working**:
1. Create buffer with hour counter data + checksum
2. Calculate checksum
3. Initialize RTC SRAM
4. Enable write access
5. Write Copy 1 to offset 880 (atomic, with cache clean)
6. Write Copy 2 to offset 1080 (atomic, with cache clean)
7. Memory barriers
8. Disable write access

---

#### `getHrCounterDataFromBKPSRAM()`
```c
bool getHrCounterDataFromBKPSRAM(void)
```

**Purpose**: Read hour meter data from backup RAM with validation

**Working**:
1. Try Copy 1 (offset 880) with checksum validation
2. If failed, try Copy 2 (offset 1080)
3. Return success/failure status

---

### Combined Save Functions

#### `EM_SaveConfigFlash_RtcBackup()`
```c
void EM_SaveConfigFlash_RtcBackup(void)
```

**Purpose**: Save configuration to both flash and backup RAM for maximum redundancy

**Working**:
1. Set configuration mode to OO (0x4F4F = "Calibrated")
2. Update Modbus holding registers with configuration data
3. **If flash available**:
   - Call `SPIF_WriteConfigData()` to save to flash
4. **Always**:
   - Call `saveConfigDataToBKPSRAM()` to save to backup RAM

**Use Case**: Called after calibration or configuration changes to ensure data is immediately persisted to both storage systems

---

### Brownout Protection

#### `NV_Storage_Brownout_Callback()`
```c
void NV_Storage_Brownout_Callback(void)
```

**Purpose**: Emergency data save during power loss (brownout) event

**Working**:
1. Triggered by PVD (Programmable Voltage Detector) interrupt
2. Detects imminent power loss
3. **Emergency save sequence**:
   - Save energy data to backup RAM
   - Save configuration to backup RAM
   - Save hour counter data to backup RAM
4. Backup RAM retains data via VBAT during power loss

**Critical Feature**: Ensures critical data survives unexpected power loss events

---

## Usage Guide

### Initialization Sequence

```c
// 1. Initialize backup SRAM
bkSRAM_Init();

// 2. Read configuration (tries RTC RAM -> Flash Copy 1 -> Flash Copy 2 -> Defaults)
EM_ReadConfigData();

// 3. Read energy data
EM_ReadEnergyData();

// 4. Read hour counter data
EM_ReadHrCounterData();
```

### Saving Data

```c
// Save configuration after changes
EM_SaveConfigFlash_RtcBackup();  // Saves to both flash and backup RAM

// Save energy data periodically
saveEnergiesToBKPSRAM();         // Quick save to backup RAM
SPIF_WriteEnergyData();          // Persistent save to flash

// Save hour counter data
saveHrCounterDataToBKPSRAM();
SPIF_WriteHrCounterData();
```

### Error Recovery

The module automatically handles errors through:
- **Retry mechanisms**: Up to 3 retries with 10ms delays
- **Dual-copy redundancy**: Always maintains 2 copies of critical data
- **Multi-level hierarchy**: Falls through RTC RAM → Flash 1 → Flash 2 → Defaults
- **Checksum validation**: Ensures data integrity at every level

---

## Data Flow Diagrams

### Write Flow
```
Application Data
      ↓
Calculate Checksum
      ↓
Create Buffer (Data + Checksum)
      ↓
   ┌──────┴──────┐
   ↓             ↓
Flash Copy 1  Flash Copy 2
   ↓             ↓
Backup RAM    Backup RAM
Copy 1        Copy 2
```

### Read Flow
```
Start Read
    ↓
Try RTC RAM Copy 1
    ↓ [Failed]
Try RTC RAM Copy 2
    ↓ [Failed]
Try Flash Copy 1 (with retries)
    ↓ [Failed]
Try Flash Copy 2 (with retries)
    ↓ [Failed]
Load Default Values
    ↓
Data Ready
```

---

## Best Practices

1. **Always initialize backup SRAM first** using `bkSRAM_Init()` before any RTC RAM operations
2. **Use backup RAM for frequent updates** to reduce flash wear
3. **Periodically save to flash** for long-term persistence
4. **Never disable brownout protection** - it's your last line of defense
5. **Monitor error flags** (`EraseResult`, `WriteResult`) to detect storage issues
6. **Test power loss scenarios** during development to verify data persistence

---

## Error Codes and Debugging

- `EraseResult`: Indicates flash sector erase success/failure
- `WriteResult`: Indicates flash write operation success/failure
- `ReadResult`: Indicates flash read operation success/failure
- `ReadResultDummy`: Indicates dummy read operation success/failure

When debugging storage issues:
1. Check flash initialization status
2. Verify backup SRAM regulator is ready
3. Validate checksum calculations
4. Check power supply stability (brownout events)
5. Monitor flash wear leveling

---

## Performance Characteristics

- **RTC RAM Access**: ~1-10 μs (fastest)
- **Flash Read**: ~100 μs per sector
- **Flash Write**: ~1-5 ms per sector (includes erase)
- **Brownout Save**: <10 ms (must complete before power loss)

---

## Safety Features

1. **Write Protection**: Backup domain write protection when not in use
2. **Dual Redundancy**: Every data type stored in 2 independent copies
3. **Checksum Validation**: All reads verified before use
4. **Atomic Writes**: Copy operations designed to survive mid-write resets
5. **Cache Coherency**: D-Cache cleaning ensures data reaches physical SRAM
6. **Memory Barriers**: DSB/ISB instructions ensure write completion
7. **Brownout Detection**: PVD interrupt triggers emergency save

---

## Constants Reference

| Constant | Value | Description |
|----------|-------|-------------|
| `BKPSRAM_BASE_ADDR` | 0x38800000 | Base address of backup SRAM |
| `SPIF_SECTOR_SIZE` | 0x1000 (4KB) | Flash sector size |
| `OO` | 0x4F4F | "Calibrated" status indicator |
| `DD` | 0x4444 | "Calibration Done" message from user |
| `MAX_RETRIES` | 3 | Maximum retry attempts for flash operations |

---

## Version History

| Date | Version | Author | Description |
|------|---------|--------|-------------|
| 24-Jul-2025 | 1.0 | Anil More | Initial implementation |
| 29-Jul-2025 | 1.1 | Anil More | Updated to TOR documentation format |
| 05-Aug-2025 | 1.2 | Anil More | Updated to TOR Doxygen documentation standard |

---

## Dependencies

- `w25q64.h`: SPI Flash driver
- `calc.h`: Energy calculation structures
- `Device_SKU.h`: Device configuration
- `do_control.h`: Digital output control
- `user_rtos_delay.h`: Custom timer functions
- STM32 HAL libraries (RCC, PWR, etc.)

---

## License

Copyright (c) by tor.ai. All rights reserved.

This software is proprietary and confidential. Unauthorized use, duplication, transmission, distribution, or disclosure is expressly forbidden.
