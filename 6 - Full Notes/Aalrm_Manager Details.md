[[Titan]]
# Alarm Manager System — Handover Document

## Document Information

| Item | Value |
|------|-------|
| **Module Name** | Alarm Manager |
| **Version** | 1.0 |
| **Author** | Kshitij Vaze |
| **Target Platform** | STM32H750VBTX |
| **IDE** | STM32CubeIDE |
| **Last Updated** | May 2026 |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Supported Alarms](#3-supported-alarms)
4. [Data Structures](#4-data-structures)
5. [State Machine](#5-state-machine)
6. [Configuration System](#6-configuration-system)
7. [Non-Volatile Storage](#7-non-volatile-storage)
8. [Digital Output Integration](#8-digital-output-integration)
9. [API Reference](#9-api-reference)
10. [Integration Points](#10-integration-points)
11. [Test Mode](#11-test-mode)
12. [Maintenance & Troubleshooting](#12-maintenance--troubleshooting)
13. [File Reference](#13-file-reference)
14. [Revision History](#14-revision-history)

---

## 1. Executive Summary

The **Alarm Manager** is a central alarm evaluation engine for the TorTitan Energy Meter firmware. It provides:

- **11 configurable alarm types** covering electrical faults (voltage, current, frequency, demand, phase issues)
- **Per-alarm state machine** with NORMAL → PENDING → ACTIVE lifecycle
- **Configurable delay timers** to prevent nuisance trips
- **Global acknowledgement lockout** preventing multiple simultaneous active alarms
- **Per-alarm digital output (DO) mapping** to control external relays
- **LIFO alarm history buffer** (100 entries) with persistent storage
- **Dual-copy redundant storage** in both BKPSRAM and SPI Flash
- **Brownout protection** for critical data preservation

### Key Design Principles

1. **Single Active Alarm**: Only one alarm can be ACTIVE at a time via `global_ack` mechanism
2. **Fault Confirmation**: Sustain timer prevents false triggers from transient faults
3. **Power-Loss Safe**: Four copies of configuration data survive unexpected power loss
4. **Modular Integration**: Clean interfaces with DO Control, UI, BLE/UART handlers

---

## 2. Architecture Overview

### 2.1 System Context

```
┌─────────────────────────────────────────────────────────────────┐
│                        Energy Meter System                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │ ADC Module  │───►│   EM_PARAMS     │───►│  Alarm Manager  │  │
│  │ (calc.c)    │    │ (measurements)  │    │(alarm_manager.c)│  │
│  └─────────────┘    └─────────────────┘    └────────┬────────┘  │
│                                                      │           │
│         ┌────────────────────────────────────────────┼───────┐   │
│         │                    │                       │       │   │
│         ▼                    ▼                       ▼       │   │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────┐  │   │
│  │  DO Control │    │    TouchGFX     │    │ NV Storage  │  │   │
│  │(do_control.c)    │ (Alarm Screen)  │    │(nv_storage.c)   │   │
│  └─────────────┘    └─────────────────┘    └─────────────┘  │   │
│         │                    │                       │       │   │
│         ▼                    ▼                       ▼       │   │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────┐  │   │
│  │ DO1 / DO2   │    │    LCD Panel    │    │ SPI Flash + │  │   │
│  │   (GPIO)    │    │                 │    │   BKPSRAM   │  │   │
│  └─────────────┘    └─────────────────┘    └─────────────┘  │   │
│                                                               │   │
└───────────────────────────────────────────────────────────────┘   │
                                                                     │
                    ┌──────────────────────────────────────────────┘
                    │ External Interfaces
                    ▼
            ┌─────────────┐    ┌─────────────────┐
            │ BLE/WiFi    │    │ UART/Modbus     │
            │ (ESP32)     │    │ (config_handler)│
            └─────────────┘    └─────────────────┘
```

### 2.2 Module Dependencies

| Module | Header | Purpose |
|--------|--------|---------|
| alarm_manager | `alarm_manager.h` | Core alarm logic |
| calc | `calc.h` | Energy measurements (EM_PARAMS) |
| do_control | `do_control.h` | Digital output control |
| rtc | `rtc.h` | Timestamp capture |
| nv_storage | `nv_storage.h` | Persistent storage |
| user_rtos_delay | `user_rtos_delay.h` | Tick timer for elapsed time |
| config_ui | `config_ui.h` | UI alarm screen control |

### 2.3 Task Integration

The Alarm Manager runs as part of the **DO Control Task** in FreeRTOS:

```c
// In do_control.c task loop (100ms cycle):
void DO_Control_Task(void *argument)
{
    while (1)
    {
        // ... other DO control logic ...
        
        AlarmManager_Update();  // Evaluate all alarms
        
        osDelay(100);  // 100ms cycle time
    }
}
```

**Code Explanation:**
- The `DO_Control_Task` is a FreeRTOS task that runs in an infinite loop
- `AlarmManager_Update()` is called once per iteration to evaluate all alarm conditions
- `osDelay(100)` yields CPU for 100ms, creating a ~100ms evaluation cycle
- This timing means alarms are checked approximately 10 times per second
- The alarm manager internally tracks elapsed time between calls using `UserDelay_GetTick()`

---

## 3. Supported Alarms

### 3.1 Alarm Types

| ID | Alarm Name | Type | Trigger Condition | Measurement Source |
|----|------------|------|-------------------|-------------------|
| 0 | Phase Reversal | Binary | Phase sequence = RBY | `detect_phase_sequence()` |
| 1 | Phase Failure | Binary | Any phase voltage lost | `detect_phase_failure_voltage()` |
| 2 | Over Voltage | Over | max(VR, VY, VB) > setpoint | `EM_PARAMS.VR/VY/VB` |
| 3 | Under Voltage | Under | min(VR, VY, VB) < setpoint | `EM_PARAMS.VR/VY/VB` |
| 4 | Over Current | Over | max(IR, IY, IB) > setpoint | `EM_PARAMS.IR/IY/IB` |
| 5 | Under Current | Under | min(IR, IY, IB) < setpoint | `EM_PARAMS.IR/IY/IB` |
| 6 | Over Frequency | Over | Freq > setpoint | `EM_PARAMS.Freq` |
| 7 | Under Frequency | Under | Freq < setpoint | `EM_PARAMS.Freq` |
| 8 | Max Demand | Over | MDTotal > setpoint | `EM_PARAMS.MDTotal` |
| 9 | Phase Voltage Imbalance | Over | ((Vmax - Vmin) / Vavg) × 100 > setpoint | Calculated |
| 10 | Phase Current Imbalance | Over | ((Imax - Imin) / Iavg) × 100 > setpoint | Calculated |

### 3.2 Threshold Types

```c
typedef enum {
    ALARM_TYPE_OVER   = 0,  // Fault if measured > setpoint
    ALARM_TYPE_UNDER  = 1,  // Fault if measured < setpoint
    ALARM_TYPE_BINARY = 2   // Fault if measured != 0
} AlarmThresholdType_t;
```

**Code Explanation:**
- `ALARM_TYPE_OVER`: Used for alarms like Over Voltage, Over Current where the fault occurs when the measured value **exceeds** the configured setpoint
- `ALARM_TYPE_UNDER`: Used for alarms like Under Voltage, Under Current where the fault occurs when the measured value **falls below** the configured setpoint
- `ALARM_TYPE_BINARY`: Used for boolean conditions like Phase Reversal and Phase Failure where there's no setpoint comparison—the fault is either present (1.0) or not (0.0)

### 3.3 Single-Phase Mode Behavior

When `EM_CONFIG.MeterWiringType == METER_WIRING_SINGLE_PHASE_2_WIRE`, the following alarms are automatically disabled:

- Phase Reversal (ID 0)
- Phase Failure (ID 1)
- Phase Voltage Imbalance (ID 9)
- Phase Current Imbalance (ID 10)

---

## 4. Data Structures

### 4.1 Master Alarm Structure

```c
typedef struct {
    /* Configuration (from BLE/NV) */
    bool enabled;                       // Alarm enabled/disabled
    AlarmThresholdType_t threshold_type;// OVER/UNDER/BINARY
    float setpoint;                     // Threshold value
    uint32_t delay_msec;                // Sustain time before activation
    bool auto_clear;                    // Auto-clear when fault resolves
    AlarmDOConfig_t do_config;          // DO output assignment (NONE/DO1/DO2)

    /* Runtime State */
    AlarmState_t state;                 // Current state (NORMAL/PENDING/ACTIVE)
    uint32_t timer_elapsed_ms;          // Elapsed time in PENDING state
    uint32_t clear_timer_elapsed_ms;    // Elapsed time for auto-clear
    float last_value;                   // Measured value at activation
    bool owns_ack;                      // This alarm owns global_ack flag

    /* Timestamp at Activation */
    RTC_TimeTypeDef trigger_time;       // RTC time when activated
    RTC_DateTypeDef trigger_date;       // RTC date when activated
} MasterAlarm_t;
```

**Code Explanation:**

This structure is the core data container for each alarm, divided into three logical sections:

**Configuration Fields** (persisted to NV storage):
- `enabled`: Master on/off switch for this alarm type
- `threshold_type`: Determines the comparison logic (>, <, or boolean)
- `setpoint`: The threshold value (e.g., 250V for Over Voltage)
- `delay_msec`: "Sustain time" — fault must persist this long before triggering
- `auto_clear`: If true, alarm auto-clears after `autoclear_delay_sec`
- `do_config`: Which digital output (if any) to assert when this alarm activates

**Runtime State Fields** (volatile, reset on boot):
- `state`: Current position in the state machine (NORMAL/PENDING/ACTIVE)
- `timer_elapsed_ms`: Accumulates time while in PENDING state for delay logic
- `clear_timer_elapsed_ms`: Accumulates time while ACTIVE for auto-clear logic
- `last_value`: Captures the measured value when the alarm activated (for history/UI)
- `owns_ack`: True if this alarm currently holds the global acknowledgement lock

**Timestamp Fields** (captured at activation):
- `trigger_time` / `trigger_date`: RTC timestamp when alarm became ACTIVE, used for history logging and UI display

### 4.2 Alarm History Entry

```c
typedef struct {
    uint8_t alarm_id;     // Alarm ID (0-10)
    rtc_time_t time;      // Hour, minutes, seconds
    RTC_DateTypeDef date; // Year, month, day, weekday
} alarm_history_entry_t;
```

**Code Explanation:**
- This is a **compact** representation of a historical alarm event
- `alarm_id`: Maps to the alarm type (0=Phase Reversal, 2=Over Voltage, etc.)
- `time`: Custom struct holding hour/minute/second (smaller than HAL's `RTC_TimeTypeDef`)
- `date`: Standard HAL date structure with year, month, day, weekday
- The compact format was chosen to fit more entries within the limited BKPSRAM space
- Additional data like setpoint and measured value is **not stored** in history to save space

### 4.3 Alarm History Ring Buffer

```c
typedef struct {
    uint16_t head;                                   // Next write index
    uint16_t count;                                  // Valid entry count (0-100)
    alarm_history_entry_t entries[100];              // LIFO buffer
} alarm_history_params_t;
```

**Code Explanation:**
- Implements a **circular LIFO buffer** that stores the last 100 alarm events
- `head`: Points to the **next write position**; after writing, head is incremented modulo 100
- `count`: Tracks how many valid entries exist (0-100); once full, stays at 100
- `entries[]`: Fixed-size array of 100 history entries
- **LIFO behavior**: Index 0 always returns the newest entry, index (count-1) the oldest
- When reading, the `AlarmHistory_GetEntry(index)` function calculates the actual array position using `head` and `count`
- When full, oldest entries are **overwritten** as new alarms occur

### 4.4 NV Configuration Structure

```c
typedef struct {
    uint16_t enabled_mask;              // Bit N → alarm ID N enabled
    float    setpoint[11];              // Per-alarm thresholds
    uint32_t sustain_time_msec;         // Global sustain time (milliseconds)
    bool     autoclear;                 // Global auto-clear mode
    uint32_t autoclear_delay_sec;       // Auto-clear delay (seconds)
    uint8_t  do_map;                    // Global DO mapping (0=NONE, 1=DO1, 2=DO2)
} alarm_nv_config_t;
```

**Code Explanation:**

This structure is the **persistent mirror** of alarm configuration, stored in both BKPSRAM and SPI Flash:

- `enabled_mask`: A 16-bit bitmask where each bit controls one alarm:
  - Bit 0 = Phase Reversal enabled
  - Bit 2 = Over Voltage enabled
  - Example: `enabled_mask = 0x000C` means alarms 2 and 3 are enabled (Over Voltage + Under Voltage)

- `setpoint[11]`: Array of 11 float values, one threshold per alarm type. E.g., `setpoint[2] = 250.0f` sets Over Voltage threshold to 250V

- `sustain_time_msec`: **Global** delay applied to ALL alarms — the fault must persist this many milliseconds before the alarm activates. Prevents nuisance trips from transient spikes.

- `autoclear`: **Global** flag — when true, active alarms will automatically clear after `autoclear_delay_sec`

- `autoclear_delay_sec`: How long (in seconds) an ACTIVE alarm waits before auto-clearing

- `do_map`: **Global** DO assignment for all alarms (0=None, 1=DO1, 2=DO2). When an alarm activates, this DO is asserted.

**Design Note:** Global parameters (sustain time, auto-clear, DO mapping) apply to ALL alarms uniformly. Per-alarm setpoints allow individual threshold tuning.

---

## 5. State Machine

### 5.1 State Definitions

```c
typedef enum {
    ALARM_STATE_NORMAL  = 0,  // No fault condition present
    ALARM_STATE_PENDING = 1,  // Fault detected, awaiting delay expiry
    ALARM_STATE_ACTIVE  = 2,  // Alarm active, DO asserted, UI notified
    ALARM_STATE_ACKED   = 3,  // (Reserved for future use)
    ALARM_STATE_CLEAR   = 4   // Alarm cleared, returning to normal
} AlarmState_t;
```

**Code Explanation:**
- `ALARM_STATE_NORMAL (0)`: Default state — no fault condition detected, alarm is idle
- `ALARM_STATE_PENDING (1)`: Fault detected but sustain timer hasn't expired yet. If fault clears during this period, returns to NORMAL without triggering.
- `ALARM_STATE_ACTIVE (2)`: Alarm has fully triggered — DO output is asserted, UI shows alarm screen, history entry is logged, global_ack is held
- `ALARM_STATE_ACKED (3)`: Reserved for future "acknowledged but not cleared" feature (operator saw alarm but hasn't resolved it)
- `ALARM_STATE_CLEAR (4)`: Transitional state used during configuration changes to force re-evaluation

### 5.2 State Transition Diagram

```
                  ┌────────────────────────────────────────┐
                  │                                        │
                  ▼                                        │
           ┌──────────┐                                    │
           │  NORMAL  │◄──────────────────────────────┐    │
           └────┬─────┘                               │    │
                │                                     │    │
      Fault detected                        Fault cleared │
      (threshold crossed)                   before delay  │
                │                                     │    │
                ▼                                     │    │
           ┌──────────┐                               │    │
           │ PENDING  │───────────────────────────────┘    │
           └────┬─────┘                                    │
                │                                          │
      Delay elapsed AND                                    │
      global_ack == false                                  │
                │                                          │
                ▼                                          │
           ┌──────────┐                                    │
           │  ACTIVE  │────────────────────────────────────┘
           └──────────┘
                │
      Manual clear (joystick)
      OR auto-clear timer expires
```

### 5.3 Global ACK Lockout

**Critical Behavior**: When one alarm becomes ACTIVE, it sets `global_ack = true` and `owns_ack = true`. While `global_ack` is set:

- Other alarms in PENDING state are immediately reset to NORMAL
- No other alarm can transition from PENDING → ACTIVE
- Only manual clear via `AlarmManager_ClearActiveAlarm()` or auto-clear timer can release the lock

```c
// In AlarmManager_Update() PENDING state handler:
if (global_ack) {
    // Another alarm owns ACK - cancel this pending instance
    alarm->state = ALARM_STATE_NORMAL;
    alarm->timer_elapsed_ms = 0U;
    break;
}
```

**Code Explanation:**
- This check happens at the **start** of PENDING state processing
- `global_ack` is a module-level boolean that indicates "an alarm is already ACTIVE"
- If true, this PENDING alarm is **immediately cancelled** — it cannot compete with the existing active alarm
- The timer is reset to 0 so if the first alarm clears and this fault is still present, evaluation starts fresh
- This ensures **exactly one alarm** can be ACTIVE at any time, simplifying UI and operator response
- The `break` exits the switch statement for this alarm, skipping further PENDING logic

### 5.4 Auto-Clear Logic

When `auto_clear` is enabled for an alarm:

1. Once the alarm becomes ACTIVE, `clear_timer_elapsed_ms` starts incrementing
2. When `clear_timer_elapsed_ms >= autoclear_delay_sec × 1000`, alarm auto-clears
3. Auto-clear happens regardless of whether the fault condition is still present
4. If fault persists after auto-clear, the alarm will re-trigger on the next cycle

```c
if (alarm->auto_clear) {
    alarm->clear_timer_elapsed_ms += cycle_time_ms;
    uint32_t clear_delay_ms = EM_ALARM_CONFIG.autoclear_delay_sec * 1000U;
    if (alarm->clear_timer_elapsed_ms >= clear_delay_ms) {
        ClearAlarm((AlarmId_t)i, CLEAR_MODE_AUTO);
    }
}
```

**Code Explanation:**
- This code runs **only in ACTIVE state** when `auto_clear` is enabled for the alarm
- `cycle_time_ms`: The elapsed time since the last `AlarmManager_Update()` call (derived from tick timer)
- `clear_timer_elapsed_ms`: Accumulates total time the alarm has been ACTIVE
- `EM_ALARM_CONFIG.autoclear_delay_sec * 1000U`: Converts seconds to milliseconds for comparison
- When the accumulated time exceeds the configured delay, `ClearAlarm()` is called with `CLEAR_MODE_AUTO`
- **Important**: Auto-clear happens regardless of fault state — even if the fault is still present, the alarm clears
- If the fault persists after auto-clear, the state machine will re-trigger the alarm on the next evaluation cycle (NORMAL → PENDING → ACTIVE again)

---

## 6. Configuration System

### 6.1 Configuration Parameters

| Parameter | Type | Range | Description |
|-----------|------|-------|-------------|
| `enabled_mask` | uint16_t | Bitmask | Bit N enables alarm ID N |
| `setpoint[i]` | float | Application-specific | Per-alarm threshold value |
| `sustain_time_msec` | uint32_t | 0-4,294,967,295 | Global delay before activation |
| `autoclear` | bool | true/false | Global auto-clear mode |
| `autoclear_delay_sec` | uint32_t | 0-4,294,967,295 | Delay before auto-clear |
| `do_map` | uint8_t | 0, 1, 2 | 0=NONE, 1=DO1, 2=DO2 |

### 6.2 Configuration Sources

1. **BLE/WiFi (ESP32)**: Via `AlarmManager_SetConfig()` API
2. **UART/Modbus**: Via `config_ParsAlarmConfig()` in config_handler.c
3. **NV Storage**: Loaded at boot via `AlarmManager_LoadFromNV()`
4. **Test Mode**: Compile-time override when `ALARM_TEST_MODE` is enabled

### 6.3 Set Configuration API

```c
void AlarmManager_SetConfig(AlarmId_t id, bool enabled, float setpoint,
                            uint32_t delay_msec, bool auto_clear,
                            AlarmDOConfig_t do_config);
```

**Behavior:**
1. Updates runtime `alarms[id]` structure
2. Applies global parameters (delay, auto_clear, DO map) to ALL alarms
3. Resets alarm state to CLEAR for fresh evaluation
4. Mirrors configuration to `EM_ALARM_CONFIG`
5. Persists to BKPSRAM and SPI Flash

---

## 7. Non-Volatile Storage

### 7.1 Storage Architecture

The alarm system uses **four-copy redundancy** across two storage tiers:

```
┌─────────────────────────────────────────────────────────────┐
│                    BKPSRAM (VBAT-backed)                    │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Alarm Config    │    │ Alarm Config    │                │
│  │ Copy 1          │    │ Copy 2          │                │
│  │ Offset: 1280    │    │ Offset: 1480    │                │
│  └─────────────────┘    └─────────────────┘                │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Alarm History   │    │ Alarm History   │                │
│  │ Copy 1          │    │ Copy 2          │                │
│  │ Offset: 1548    │    │ Offset: 2353    │                │
│  └─────────────────┘    └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 SPI Flash (W25Q64FV, 8MB)                   │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Alarm Config    │    │ Alarm Config    │                │
│  │ Sector 6 (4KB)  │    │ Sector 7 (4KB)  │                │
│  │ Page 96         │    │ Page 112        │                │
│  └─────────────────┘    └─────────────────┘                │
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Alarm History   │    │ Alarm History   │                │
│  │ Sector 8 (4KB)  │    │ Sector 9 (4KB)  │                │
│  │ Page 128        │    │ Page 144        │                │
│  └─────────────────┘    └─────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Load Priority Chain

On boot, configuration is loaded using this priority (first valid source wins):

```
BKPSRAM Copy 1 → BKPSRAM Copy 2 → Flash Copy 1 → Flash Copy 2 → Safe Defaults
```

### 7.3 Data Integrity

Each copy includes a **1-byte checksum**:

```c
uint8_t checksum = 0;
for (size_t i = 0; i < data_size; i++) {
    checksum += buffer[i];
}
checksum += 1;  // Prevents valid all-zero checksum
```

**Code Explanation:**
- This is a simple **additive checksum** algorithm used for all NV data
- The loop sums all bytes in the data buffer, with natural overflow to 8 bits
- `checksum += 1`: Critical addition that prevents an all-zero buffer from having a valid checksum of 0
  - Without this, erased flash (all 0xFF) or zeroed RAM would pass validation
- The checksum byte is stored **immediately after** the data buffer in memory
- On read, the checksum is recalculated and compared to the stored value
- If mismatch: data is corrupt → try next copy in priority chain
- This catches single-bit errors, partial writes, and flash corruption

### 7.4 Save Paths

| Trigger | Flash | BKPSRAM |
|---------|-------|---------|
| `AlarmManager_SaveToNV()` | ✓ | ✓ |
| `AlarmHistory_SaveToNV()` | ✓ | ✓ |
| Brownout callback (PVD IRQ) | ✓ | ✓ |
| Factory reset | ✓ | ✓ |

### 7.5 Brownout Protection

When the **Programmable Voltage Detector (PVD)** detects a power droop:

```c
void NV_Storage_Brownout_Callback(void)
{
    // Startup guard: ignore first 5 seconds
    // Re-entrancy guard: prevent overlapping saves
    
    // Emergency save sequence:
    SPIF_WriteHrCounterData();
    SPIF_WriteEnergyData();
    SPIF_WriteConfigData();
    SPIF_WriteAlarmConfigData();    // Alarm config to flash
    saveAlarmConfigToBKPSRAM();     // Alarm config to BKPSRAM
}
```

**Code Explanation:**
- This callback is triggered by the **PVD (Programmable Voltage Detector)** interrupt when supply voltage drops below threshold
- **Startup guard**: Ignores PVD triggers in the first 5 seconds after boot to avoid false triggers during power-up transients
- **Re-entrancy guard**: A flag (`u32WritingInProgress`) prevents concurrent calls if voltage dips multiple times
- **Emergency save sequence**: Writes ALL critical data in priority order:
  1. HR Counter — equipment runtime tracking
  2. Energy — accumulated kWh values
  3. Config — calibration and system settings
  4. Alarm Config — both to flash AND BKPSRAM
- The window for saving is typically 3-5ms before capacitors discharge and MCU loses power
- BKPSRAM save is faster than flash (no erase cycle), providing better brownout survival

---

## 8. Digital Output Integration

### 8.1 DO Configuration

```c
typedef enum {
    ALARM_DO_NONE = 0,  // No DO output for this alarm
    ALARM_DO_1    = 1,  // Assert DO1 when active
    ALARM_DO_2    = 2   // Assert DO2 when active
} AlarmDOConfig_t;
```

**Code Explanation:**
- Defines which physical digital output pin to control when an alarm is ACTIVE
- `ALARM_DO_NONE`: Alarm is informational only — no physical output action
- `ALARM_DO_1`: Assert DO1 (GPIO PA8 on PCB v3.0) when this alarm activates
- `ALARM_DO_2`: Assert DO2 (GPIO PA9 on PCB v3.0) when this alarm activates
- The DO typically drives a relay or contactor for load disconnect on fault
- This is stored in `EM_ALARM_CONFIG.do_map` as a **global** setting (same DO for all alarms)

### 8.2 DO Mode Reservation

When an alarm is configured to use a DO, that DO is **reserved for alarm use exclusively**:

```c
void AlarmManager_LoadFromNV(void) {
    // ... load config ...
    
    // Reserve DO for alarm use
    switch (EM_ALARM_CONFIG.do_map) {
        case 1U:
            EM_CONFIG.IO_config.DO1_Mode = DO_MODE_ALARM;
            break;
        case 2U:
            EM_CONFIG.IO_config.DO2_Mode = DO_MODE_ALARM;
            break;
    }
}
```

**Code Explanation:**
- This code runs during alarm manager initialization after loading config from NV
- `EM_CONFIG.IO_config.DO1_Mode` / `DO2_Mode`: Global DO mode configuration used by `do_control.c`
- `DO_MODE_ALARM`: A special mode value that tells the DO Control module "this output is reserved for alarms"
- **Why reservation matters**: The DO Control module supports multiple modes:
  - Scheduler mode (time-based on/off)
  - Parameter mode (threshold-based switching)
  - Pulse mode (energy pulse output)
  - Alarm mode (fault-triggered)
- Without reservation, these modes could conflict — e.g., scheduler turns DO OFF while alarm wants it ON
- Setting `DO_MODE_ALARM` ensures the DO Control module ignores other mode logic for this output

### 8.3 DO Update Logic

```c
void AlarmManager_UpdateDO(void) {
    bool do1_active = false;
    bool do2_active = false;

    // Scan all alarms for DO requirements
    for (uint8_t i = 0U; i < ALARM_COUNT; i++) {
        if (alarms[i].state == ALARM_STATE_ACTIVE) {
            switch (alarms[i].do_config) {
                case ALARM_DO_1: do1_active = true; break;
                case ALARM_DO_2: do2_active = true; break;
            }
        }
    }

    // Apply DO states
    DO_Control_SetDO1(do1_active);
    DO_Control_SetDO2(do2_active);
}
```

**Code Explanation:**
- Called at the end of every `AlarmManager_Update()` cycle to synchronize DO states
- **Scan phase**: Iterates through all 11 alarms looking for any in ACTIVE state
- If an ACTIVE alarm has `do_config == ALARM_DO_1`, sets `do1_active = true`
- If an ACTIVE alarm has `do_config == ALARM_DO_2`, sets `do2_active = true`
- **Apply phase**: Calls DO Control module to physically set GPIO states
- `DO_Control_SetDO1(true)` / `SetDO2(true)`: Drives the output HIGH (relay energized)
- `DO_Control_SetDO1(false)` / `SetDO2(false)`: Drives the output LOW (relay de-energized)
- **Multiple alarm support**: If multiple alarms share the same DO, any one being ACTIVE keeps it asserted
- DO only goes LOW when **no** ACTIVE alarms require that output

---

## 9. API Reference

### 9.1 Initialization

```c
/**
 * @brief Initialize the Alarm Manager module
 * @pre   DO_Control and RTC must be initialized
 * @post  All alarms in NORMAL state, history loaded, DOs OFF
 */
void AlarmManager_Init(void);
```

### 9.2 Periodic Update

```c
/**
 * @brief Run alarm evaluation cycle (call every 100ms)
 * @details Evaluates all alarms, manages state machines, updates DOs
 */
void AlarmManager_Update(void);
```

### 9.3 Configuration

```c
/**
 * @brief Set configuration for a specific alarm
 * @param id Alarm ID (0-10)
 * @param enabled Enable/disable flag
 * @param setpoint Threshold value
 * @param delay_msec Delay before activation (milliseconds)
 * @param auto_clear Auto-clear mode flag
 * @param do_config DO output assignment
 */
void AlarmManager_SetConfig(AlarmId_t id, bool enabled, float setpoint,
                            uint32_t delay_msec, bool auto_clear,
                            AlarmDOConfig_t do_config);

void AlarmManager_SaveToNV(void);
void AlarmManager_LoadFromNV(void);
```

### 9.4 Query Functions

```c
bool AlarmManager_IsAlarmActive(void);
uint8_t AlarmManager_GetActiveAlarmId(void);
const MasterAlarm_t* AlarmManager_GetAlarm(AlarmId_t id);
const MasterAlarm_t* AlarmManager_GetAlarms(void);
const char* AlarmManager_GetAlarmName(AlarmId_t id);
```

### 9.5 Clear Functions

```c
/**
 * @brief Clear the currently active alarm (manual clear)
 * @return true if an alarm was cleared, false if none active
 */
bool AlarmManager_ClearActiveAlarm(void);
```

### 9.6 History Functions

```c
const alarm_history_entry_t* AlarmHistory_GetEntry(uint8_t index);
uint8_t AlarmHistory_GetCount(void);
void AlarmHistory_SaveToNV(uint8_t alarm_id, const RTC_TimeTypeDef* time,
                           const RTC_DateTypeDef* date);
void AlarmHistory_LoadFromNV(void);
```

---

## 10. Integration Points

### 10.1 UI Integration (TouchGFX)

When an alarm becomes ACTIVE:
```c
ui_alarm_active(true);   // Switch to alarm screen
```

**Explanation:** Called from `ActivateAlarm()` — signals TouchGFX to immediately display the alarm notification screen, interrupting normal display cycle.

When alarm is cleared:
```c
ui_alarm_active(false);  // Return to normal screen
```

**Explanation:** Called from `ClearAlarm()` — signals TouchGFX to return to the normal measurement display screens.

UI can query alarm data:
```c
const MasterAlarm_t* alarms = AlarmManager_GetAlarms();
uint8_t active_id = AlarmManager_GetActiveAlarmId();
const char* name = AlarmManager_GetAlarmName(active_id);
```

**Explanation:**
- `AlarmManager_GetAlarms()`: Returns pointer to the internal `alarms[]` array for read-only access to all alarm states
- `AlarmManager_GetActiveAlarmId()`: Returns 0-10 for active alarm, or -1 (cast to uint8_t = 255) if none active
- `AlarmManager_GetAlarmName()`: Returns human-readable string like "Over Voltage" for display on alarm screen

### 10.2 Joystick Integration

Manual alarm clear via joystick press:
```c
// In joystick handler:
if (joystick_pressed && AlarmManager_IsAlarmActive()) {
    AlarmManager_ClearActiveAlarm();
}
```

**Code Explanation:**
- This pattern is used in the joystick/button handler (typically in `Joystick.c` or UI code)
- `joystick_pressed`: Boolean indicating the physical button was pressed
- `AlarmManager_IsAlarmActive()`: Returns true only if an alarm is currently in ACTIVE state
- `AlarmManager_ClearActiveAlarm()`: Clears the alarm that holds `global_ack`, logs `CLEAR_MODE_MANUAL` to history
- **User workflow**: Operator sees alarm screen → presses joystick → alarm clears → normal screen resumes
- Manual clear is **always available**, even if auto_clear is also enabled

### 10.3 BLE/UART Integration

Configuration commands via config_handler:
```c
// SET ALARM_CONFIG command handler:
AlarmManager_SetConfig(alarm_id, enabled, setpoint,
                       delay_ms, auto_clear, do_config);
```

**Code Explanation:**
- `config_handler.c` parses incoming BLE/UART commands (e.g., "SET ALARM_CONFIG 2 1 250.0 5000 1 1")
- Parsed values are passed to `AlarmManager_SetConfig()` which:
  1. Updates the specific alarm's runtime configuration
  2. Applies global parameters to all alarms
  3. Persists changes to NV storage immediately
- Parameters: `alarm_id` (0-10), `enabled` (0/1), `setpoint` (float), `delay_ms` (uint32), `auto_clear` (0/1), `do_config` (0/1/2)
- Remote configuration via smartphone app or PC utility uses this path

### 10.4 Modbus Integration

The `EM_PARAMS.LastActiveAlarm` field is updated when alarms activate:
```c
// When alarm activates:
EM_PARAMS.LastActiveAlarm = (uint16_t)(1U << active_alarm_id);

// When alarm clears:
EM_PARAMS.LastActiveAlarm = 0;
```

**Code Explanation:**
- `EM_PARAMS` is the global structure exposed via Modbus holding registers
- `LastActiveAlarm`: A 16-bit bitmask register readable by external Modbus masters (PLCs, SCADA systems)
- `(1U << active_alarm_id)`: Sets a single bit corresponding to the alarm type:
  - Bit 0 = Phase Reversal active
  - Bit 2 = Over Voltage active
  - Bit 8 = Max Demand active
  - etc.
- When alarm clears, field is set to 0 indicating "no active alarm"
- External systems poll this register to detect alarm conditions and trigger their own responses
- **Note**: Only one bit can be set at a time (due to global_ack lockout)

---

## 11. Test Mode

### 11.1 Enabling Test Mode

In `Device_SKU.h`:
```c
#define ALARM_TEST_MODE  1  // Set to 1 to enable test mode
```

**Code Explanation:**
- `ALARM_TEST_MODE` is a compile-time preprocessor macro
- When set to `1`, test-specific code is included via `#if (ALARM_TEST_MODE)` blocks
- When set to `0` (production), test code is excluded entirely from the binary
- Changing this requires recompilation — it's not a runtime setting
- Used for factory testing and development debugging

### 11.2 Test Configuration

When test mode is enabled, `AlarmManager_Init()` applies:
```c
AlarmManager_SetConfig(ALARM_ID_OVER_VOLTAGE,
    true,           // enabled
    120.0f,         // setpoint: 120V threshold
    5000U,          // 5 second sustain time
    true,           // auto_clear enabled
    ALARM_DO_NONE); // no DO output

EM_ALARM_CONFIG.autoclear_delay_sec = 3U;  // 3 second auto-clear delay
```

**Code Explanation:**
- This code runs inside `AlarmManager_Init()` only when `ALARM_TEST_MODE == 1`
- **Purpose**: Provides a known, repeatable alarm configuration for testing without modifying NV storage
- Test setup:
  - Over Voltage alarm enabled with 120V threshold (easy to exceed with test equipment)
  - 5-second sustain time — fault must persist 5 seconds before triggering
  - Auto-clear enabled with 3-second delay — alarm clears itself after 3 seconds
  - No DO output — safe for bench testing without relay clicking
- **Testing workflow**: Inject voltage > 120V → wait 5s → alarm activates → wait 3s → alarm auto-clears
- `EM_ALARM_CONFIG.autoclear_delay_sec` is set directly (not via SetConfig) to avoid persisting to NV

### 11.3 Testing Procedure

1. Enable `ALARM_TEST_MODE` in `Device_SKU.h`
2. Build and flash firmware
3. Generate simulated voltage > 120V in `EM_PARAMS.VR`
4. Observe alarm sequence:
   - NORMAL → PENDING (for 5 seconds)
   - PENDING → ACTIVE (alarm screen appears)
   - ACTIVE → NORMAL (after 3 seconds auto-clear)

---

## 12. Maintenance & Troubleshooting

### 12.1 Common Issues

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| Alarm never triggers | Disabled in config | Check `enabled_mask` |
| Alarm triggers immediately | Sustain time = 0 | Set appropriate delay |
| Multiple alarms not working | Global ACK lockout | Clear first alarm |
| History not persisting | Flash write failure | Check SPI flash health |
| DO not asserting | Wrong DO mapping | Verify `do_map` config |

### 12.2 Debugging Tips

1. **Check alarm state**: Use debugger to inspect `alarms[]` array
2. **Verify measurements**: Confirm `EM_PARAMS` values are correct
3. **Monitor state transitions**: Add breakpoints in `ActivateAlarm()`, `ClearAlarm()`
4. **NV validation**: Check checksum validation in `EM_ReadAlarmConfigData()`

### 12.3 Factory Reset

Factory reset clears alarm configuration:
```c
// In config_ui.c:
void perform_factory_reset(void) {
    // Reset EM_CONFIG
    ResetToFactorySettings();
    
    // Reset alarm config to defaults
    EM_ALARM_CONFIG.enabled_mask = 0;
    memset(EM_ALARM_CONFIG.setpoint, 0, sizeof(EM_ALARM_CONFIG.setpoint));
    EM_ALARM_CONFIG.sustain_time_msec = 0;
    EM_ALARM_CONFIG.autoclear = false;
    EM_ALARM_CONFIG.autoclear_delay_sec = 0;
    EM_ALARM_CONFIG.do_map = 0;
    
    // Persist to both storage tiers
    SPIF_WriteAlarmConfigData();
    saveAlarmConfigToBKPSRAM();
```

**Code Explanation:**
- This function is triggered from the UI "Factory Reset" menu option
- `ResetToFactorySettings()`: Resets main `EM_CONFIG` (calibration, Modbus params, etc.)
- **Alarm config reset**:
  - `enabled_mask = 0`: All 11 alarms disabled
  - `memset(...setpoint, 0, ...)`: All thresholds set to 0.0
  - `sustain_time_msec = 0`: No delay (though alarms are disabled anyway)
  - `autoclear = false`: Manual clear mode
  - `autoclear_delay_sec = 0`: No auto-clear delay
  - `do_map = 0`: No DO output assigned
- **Persistence**: Writes to BOTH SPI flash and BKPSRAM to ensure old settings don't survive
- After factory reset, the device boots with all alarms disabled until reconfigured
}
```

---

## 13. File Reference

### 13.1 Source Files

| File | Location | Purpose |
|------|----------|---------|
| `alarm_manager.c` | `Core/Src/` | Main implementation |
| `alarm_manager.h` | `Core/Inc/` | Public interface |
| `nv_storage.c` | `Core/Src/` | NV read/write functions |
| `nv_storage.h` | `Core/Inc/` | NV function prototypes |
| `EM.h` | `Core/Inc/` | Data structure definitions |
| `do_control.c` | `Core/Src/` | DO control integration |
| `config_handler.c` | `Core/Src/` | BLE/UART config parsing |

### 13.2 Related Documentation

| Document | Purpose |
|----------|---------|
| `ALARM_CONFIG_DETAILED_EXPLANATION.md` | In-depth NV storage explanation |
| `ALARM_CONFIG_NV_STORAGE.md` | Config storage overview |
| `ALARM_HISTORY_NV_STORAGE.md` | History storage details |
| `DO_CONTROL_GUIDE.md` | DO control integration guide |

### 13.3 Key Macros

| Macro | Value | Location |
|-------|-------|----------|
| `ALARM_COUNT` | 11 | `alarm_manager.h` |
| `ALARM_HISTORY_SIZE` | 100 | `alarm_manager.h` |
| `EM_ALARM_COUNT` | 11 | `EM.h` |
| `EM_ALARM_HISTORY_SIZE` | 100 | `EM.h` |

---

## 14. Revision History

| Date | Version | Author | Description |
|------|---------|--------|-------------|
| 11-Feb-2026 | 1.0 | Kshitij Vaze | Initial implementation |
| 18-Feb-2026 | 1.0 | Kshitij Vaze | Added history persistence |
| May 2026 | 1.0 | Kshitij Vaze | Handover documentation |

---

## Appendix A: Quick Reference Card

### Alarm IDs
```
0 = Phase Reversal      5 = Under Current       10 = Phase Current Imbalance
1 = Phase Failure       6 = Over Frequency
2 = Over Voltage        7 = Under Frequency
3 = Under Voltage       8 = Max Demand
4 = Over Current        9 = Phase Voltage Imbalance
```

### State Machine
```
NORMAL (0) → PENDING (1) → ACTIVE (2) → NORMAL (0)
```

### DO Mapping
```
0 = ALARM_DO_NONE
1 = ALARM_DO_1
2 = ALARM_DO_2
```

### Storage Locations
```
BKPSRAM: Config @ 1280/1480, History @ 1548/2353
Flash:   Config @ Sector 6/7, History @ Sector 8/9
```

---

**End of Document**

*This handover document provides complete technical reference for maintaining and extending the Alarm Manager system.*
