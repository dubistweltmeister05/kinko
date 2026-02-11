[[Titan]]


# CODE DESIGN
## 1. Enums and Master Struct

### 1.1 Alarm State Enum
An Enum to track the states of the alarm at any given time.
```c
typedef enum {
    ALARM_STATE_NORMAL,
    ALARM_STATE_PENDING,
    ALARM_STATE_ACTIVE,
    ALARM_STATE_ACKED,
    ALARM_STATE_CLEAR
} AlarmState_t;
```

### 1.2 Threshold Type Enum
An Enum to track the nature of the alarm.
```c
typedef enum {
    ALARM_TYPE_OVER,
    ALARM_TYPE_UNDER,
    ALARM_TYPE_BINARY
} AlarmThresholdType_t;
```

### 1.3 DO Configuration Enum
**NEW: Configurable DO output for alarms.**
```c
typedef enum {
    ALARM_DO_NONE = 0,   // No DO output for alarms
    ALARM_DO_1,          // Use DO1 for alarms
    ALARM_DO_2           // Use DO2 for alarms
} AlarmDOConfig_t;
```

### 1.4 Clear Mode Enum
```c
typedef enum {
    CLEAR_MODE_MANUAL,
    CLEAR_MODE_AUTO
} ClearMode_t;
```

### 1.5 Master Alarm Structure
A master structure for the alarm that can be replicated for each.
```c
typedef struct {
    // Config (from BLE)
    bool enabled;
    AlarmThresholdType_t threshold_type;
    float setpoint;
    uint32_t delay_ms;
    bool auto_clear;

    // Runtime
    AlarmState_t state;
    uint32_t timer_elapsed_ms;
    float last_value;
    bool owns_ack;

    // Timestamp at activation
    RTC_TimeTypeDef trigger_time;
    RTC_DateTypeDef trigger_date;
} MasterAlarm_t;
```

### 1.6 Alarm History Entry Structure
**NEW: Structure for storing alarm history entries (LIFO buffer).**
```c
#define ALARM_HISTORY_SIZE  100

typedef struct {
    uint8_t alarm_id;               // Alarm type (0-9)
    RTC_TimeTypeDef trigger_time;   // Time when alarm was triggered
    RTC_DateTypeDef trigger_date;   // Date when alarm was triggered
    RTC_TimeTypeDef clear_time;     // Time when alarm was cleared
    RTC_DateTypeDef clear_date;     // Date when alarm was cleared
    float measured_value;           // Value at trigger
    float setpoint;                 // Setpoint at trigger
    ClearMode_t clear_mode;         // How the alarm was cleared
    bool valid;                     // Entry contains valid data
    bool cleared;                   // Alarm has been cleared
} AlarmHistoryEntry_t;
```

### 1.7 Alarm Status Variable for WiFi
**NEW: Structure for WiFi alarm status (sent every 80ms).**
```c
typedef struct {
    int8_t active_alarm_id;         // -1 if no active alarm
    AlarmState_t alarm_state;       // Current state of active alarm
    uint8_t trigger_hour;
    uint8_t trigger_min;
    uint8_t trigger_sec;
    float measured_value;
    float setpoint;
    uint8_t change_counter;         // Incremented on each state change (for ESP32 detection)
} AlarmStatusWiFi_t;
```

### 1.8 Global State
```c
static MasterAlarm_t alarms[ALARM_COUNT];
static bool global_ack = false;
static int8_t active_alarm_id = -1;  // -1 = none

// NEW: DO Configuration
static AlarmDOConfig_t alarm_do_config = ALARM_DO_NONE;

// NEW: Alarm History (Circular Buffer - LIFO)
static AlarmHistoryEntry_t alarm_history[ALARM_HISTORY_SIZE];
static uint8_t alarm_history_head = 0;   // Next write position
static uint8_t alarm_history_count = 0;  // Number of valid entries

// NEW: WiFi Status Variable
static AlarmStatusWiFi_t alarm_wifi_status = {
    .active_alarm_id = -1,
    .alarm_state = ALARM_STATE_NORMAL,
    .change_counter = 0
};
```

---

## 2. Alarm-to-Measurement Mapping

| Alarm ID | Alarm Name      | Threshold Type | Measurement Source              |
| -------- | --------------- | -------------- | ------------------------------- |
| 0        | Phase Reversal  | BINARY         | `phase_reversal_flag`           |
| 1        | Phase Failure   | BINARY         | `phase_failure_flag`            |
| 2        | Over Voltage    | OVER           | `voltage_max` (max of 3 phases) |
| 3        | Under Voltage   | UNDER          | `voltage_min` (min of 3 phases) |
| 4        | Over Current    | OVER           | `current_max` (max of 3 phases) |
| 5        | Under Current   | UNDER          | `current_min` (min of 3 phases) |
| 6        | Over Frequency  | OVER           | `frequency`                     |
| 7        | Under Frequency | UNDER          | `frequency`                     |
| 8        | Max Demand      | OVER           | `demand_kw` or `demand_kva`     |
| 9        | Phase Imbalance | OVER           | `imbalance_percent`             |

---

## 3. Threshold Evaluation Logic

For each alarm, determine if a fault condition is present:

| Threshold Type     | Fault Condition           |
| ------------------ | ------------------------- |
| `ALARM_TYPE_OVER`  | `measured_value > setpoint` |
| `ALARM_TYPE_UNDER` | `measured_value < setpoint` |
| `ALARM_TYPE_BINARY`| `measured_value != 0`       |

---

## 4. State Machine

### 4.1 State Diagram
```
┌──────────┐
│  NORMAL  │◄────────────────────────────────┐
└────┬─────┘                                 │
     │ fault_present == true                 │
     ▼                                       │
┌──────────┐                                 │
│ PENDING  │──── fault clears before delay ──┘
└────┬─────┘
     │ timer_elapsed_ms >= delay_ms
     │ AND global_ack == false
     ▼
┌──────────┐
│  ACTIVE  │◄── alarm now owns global_ack
└────┬─────┘    ◄── added to history
     │ manual clear (joystick) - ALWAYS AVAILABLE
     │ OR (auto_clear AND fault clears)
     ▼
┌──────────┐
│  CLEAR   │──── transitions back to NORMAL
└──────────┘    ◄── history entry updated
```

### 4.2 State Transitions

#### NORMAL → PENDING

- **Condition:** `fault_present == true`
- **Actions:**
    - Set `state = ALARM_STATE_PENDING`
    - Set `timer_elapsed_ms = 0`

#### PENDING → NORMAL

- **Condition:** `fault_present == false` (recovered before delay)
- **Actions:**
    - Set `state = ALARM_STATE_NORMAL`
    - Set `timer_elapsed_ms = 0`

#### PENDING → ACTIVE

- **Condition:** `timer_elapsed_ms >= delay_ms` AND `global_ack == false`
- **Actions:**
    - Set `state = ALARM_STATE_ACTIVE`
    - Capture `last_value = current_measurement`
    - Capture `trigger_time`, `trigger_date` from RTC
    - Set `global_ack = true`
    - Set `owns_ack = true`
    - Set `active_alarm_id = this_alarm_id`
    - **NEW:** Add entry to alarm history
    - **NEW:** Update `alarm_wifi_status` and increment `change_counter`
    - **Trigger:** UI navigation to alarm screen
    - **Trigger:** Configured DO activation (if `alarm_do_config != ALARM_DO_NONE`)
    - **Trigger:** WiFi ALARM_ACTIVATED payload

#### PENDING while global_ack == true (Lockout)

- Alarm remains in PENDING state.
- Timer continues to run.
- Cannot transition to ACTIVE until `global_ack` is cleared.
- Once `global_ack` clears: if still faulted and `timer_elapsed_ms >= delay_ms`, immediately transition to ACTIVE.

#### ACTIVE → CLEAR (Manual)

- **Condition:** Joystick CLEAR event AND `owns_ack == true`
- **IMPORTANT: Manual clear is ALWAYS available, even if `auto_clear == true`**
- **Actions:**
    - Set `state = ALARM_STATE_NORMAL`
    - Set `global_ack = false`
    - Set `owns_ack = false`
    - Set `active_alarm_id = -1`
    - **NEW:** Update history entry with clear time and `CLEAR_MODE_MANUAL`
    - **NEW:** Update `alarm_wifi_status` and increment `change_counter`
    - **Trigger:** Configured DO deactivation (if no other active alarms)
    - **Trigger:** WiFi ALARM_CLEARED payload
    - **Trigger:** UI return to normal screen

#### ACTIVE → CLEAR (Auto)

- **Condition:** `auto_clear == true` AND `fault_present == false`
- **Note:** This does NOT prevent manual clear - manual clear is always available
- **Actions:**
    - Same as manual clear, but with `CLEAR_MODE_AUTO` in history

---

## 5. Alarm History Management (LIFO)

### 5.1 Data Structure
```c
#define ALARM_HISTORY_SIZE  100

static AlarmHistoryEntry_t alarm_history[ALARM_HISTORY_SIZE];
static uint8_t alarm_history_head = 0;
static uint8_t alarm_history_count = 0;
```

### 5.2 Adding an Entry
```c
void AlarmHistory_Add(uint8_t alarm_id, float measured_value, float setpoint) {
    AlarmHistoryEntry_t *entry = &alarm_history[alarm_history_head];
    
    entry->alarm_id = alarm_id;
    entry->measured_value = measured_value;
    entry->setpoint = setpoint;
    entry->valid = true;
    entry->cleared = false;
    
    // Get current time from RTC
    HAL_RTC_GetTime(&hrtc, &entry->trigger_time, RTC_FORMAT_BIN);
    HAL_RTC_GetDate(&hrtc, &entry->trigger_date, RTC_FORMAT_BIN);
    
    // Clear fields will be populated on clear
    memset(&entry->clear_time, 0, sizeof(RTC_TimeTypeDef));
    memset(&entry->clear_date, 0, sizeof(RTC_DateTypeDef));
    
    // Advance head (circular)
    alarm_history_head = (alarm_history_head + 1) % ALARM_HISTORY_SIZE;
    
    if (alarm_history_count < ALARM_HISTORY_SIZE) {
        alarm_history_count++;
    }
    
    // Persist to NV storage
    AlarmHistory_SaveToNV();
}
```

### 5.3 Updating Entry on Clear
```c
void AlarmHistory_MarkCleared(uint8_t alarm_id, ClearMode_t mode) {
    // Find the most recent uncleated entry for this alarm
    for (int i = 0; i < alarm_history_count; i++) {
        int idx = (alarm_history_head - 1 - i + ALARM_HISTORY_SIZE) % ALARM_HISTORY_SIZE;
        AlarmHistoryEntry_t *entry = &alarm_history[idx];
        
        if (entry->valid && entry->alarm_id == alarm_id && !entry->cleared) {
            entry->cleared = true;
            entry->clear_mode = mode;
            HAL_RTC_GetTime(&hrtc, &entry->clear_time, RTC_FORMAT_BIN);
            HAL_RTC_GetDate(&hrtc, &entry->clear_date, RTC_FORMAT_BIN);
            
            // Persist to NV storage
            AlarmHistory_SaveToNV();
            break;
        }
    }
}
```

### 5.4 Reading History for UI (LIFO Order)
```c
// Get entry at index (0 = newest, count-1 = oldest)
const AlarmHistoryEntry_t* AlarmHistory_GetEntry(uint8_t index) {
    if (index >= alarm_history_count) {
        return NULL;
    }
    
    // LIFO: index 0 is the most recent entry
    int actual_idx = (alarm_history_head - 1 - index + ALARM_HISTORY_SIZE) % ALARM_HISTORY_SIZE;
    return &alarm_history[actual_idx];
}

uint8_t AlarmHistory_GetCount(void) {
    return alarm_history_count;
}
```

### 5.5 NV Storage Persistence
```c
void AlarmHistory_SaveToNV(void) {
    // Save alarm_history array, alarm_history_head, alarm_history_count
    // to NV storage using existing nv_storage API
    NV_Storage_Write(NV_ADDR_ALARM_HISTORY, alarm_history, sizeof(alarm_history));
    NV_Storage_Write(NV_ADDR_ALARM_HISTORY_HEAD, &alarm_history_head, sizeof(alarm_history_head));
    NV_Storage_Write(NV_ADDR_ALARM_HISTORY_COUNT, &alarm_history_count, sizeof(alarm_history_count));
}

void AlarmHistory_LoadFromNV(void) {
    NV_Storage_Read(NV_ADDR_ALARM_HISTORY, alarm_history, sizeof(alarm_history));
    NV_Storage_Read(NV_ADDR_ALARM_HISTORY_HEAD, &alarm_history_head, sizeof(alarm_history_head));
    NV_Storage_Read(NV_ADDR_ALARM_HISTORY_COUNT, &alarm_history_count, sizeof(alarm_history_count));
    
    // Validate
    if (alarm_history_head >= ALARM_HISTORY_SIZE) {
        alarm_history_head = 0;
    }
    if (alarm_history_count > ALARM_HISTORY_SIZE) {
        alarm_history_count = 0;
    }
}
```

---

## 6. DO Output Logic (Configurable)

### 6.1 Configuration
```c
// Set DO configuration (called from BLE config handler)
void AlarmManager_SetDOConfig(AlarmDOConfig_t config) {
    alarm_do_config = config;
    
    // Persist to NV storage
    NV_Storage_Write(NV_ADDR_ALARM_DO_CONFIG, &alarm_do_config, sizeof(alarm_do_config));
}

AlarmDOConfig_t AlarmManager_GetDOConfig(void) {
    return alarm_do_config;
}
```

### 6.2 DO Control Function
After each evaluation cycle:
```c
void AlarmManager_UpdateDO(void) {
    // Skip if no DO is configured for alarms
    if (alarm_do_config == ALARM_DO_NONE) {
        return;
    }
    
    // Check if any alarm is active
    bool any_active = false;
    for (int i = 0; i < ALARM_COUNT; i++) {
        if (alarms[i].state == ALARM_STATE_ACTIVE || 
            alarms[i].state == ALARM_STATE_ACKED) {
            any_active = true;
            break;
        }
    }
    
    // Drive the configured DO
    switch (alarm_do_config) {
        case ALARM_DO_1:
            DO1_Set(any_active);
            break;
        case ALARM_DO_2:
            DO2_Set(any_active);
            break;
        default:
            break;
    }
}
```

### 6.3 Startup Behavior
```c
void AlarmManager_Init(void) {
    // ... other initialization ...
    
    // Load DO config from NV storage
    NV_Storage_Read(NV_ADDR_ALARM_DO_CONFIG, &alarm_do_config, sizeof(alarm_do_config));
    
    // Validate
    if (alarm_do_config > ALARM_DO_2) {
        alarm_do_config = ALARM_DO_NONE;
    }
    
    // Ensure DO is OFF on startup
    DO1_Set(false);
    DO2_Set(false);
}
```

---

## 7. Joystick Integration

### 7.1 Clear Handler
**IMPORTANT: Manual clear is ALWAYS available, regardless of auto_clear setting**

In joystick event handler or main polling loop:
```c
if (joystick_clear_event_detected) {
    if (global_ack && active_alarm_id >= 0) {
        // Manual clear is always allowed
        AlarmManager_ClearActiveAlarm(CLEAR_MODE_MANUAL);
    }
}
```

### 7.2 Clear Function
```c
void AlarmManager_ClearActiveAlarm(ClearMode_t mode) {
    if (active_alarm_id < 0 || active_alarm_id >= ALARM_COUNT) {
        return;
    }
    
    MasterAlarm_t *alarm = &alarms[active_alarm_id];
    
    if (!alarm->owns_ack) {
        return;
    }
    
    // Update history with clear info
    AlarmHistory_MarkCleared(active_alarm_id, mode);
    
    // Clear the alarm
    alarm->state = ALARM_STATE_NORMAL;
    alarm->owns_ack = false;
    alarm->timer_elapsed_ms = 0;
    
    // Clear global state
    global_ack = false;
    int8_t cleared_alarm_id = active_alarm_id;
    active_alarm_id = -1;
    
    // Update WiFi status
    AlarmManager_UpdateWiFiStatus();
    
    // Update DO
    AlarmManager_UpdateDO();
    
    // Send WiFi clear notification
    Wifi_SendAlarmEvent(cleared_alarm_id, ALARM_EVENT_CLEARED, mode);
    
    // Notify UI to return to normal screen
    UI_ReturnFromAlarmScreen();
}
```

---

## 8. WiFi Integration

### 8.1 Alarm Status Variable (sent every 80ms)

```c
// Update the WiFi status variable (called on state changes)
void AlarmManager_UpdateWiFiStatus(void) {
    alarm_wifi_status.active_alarm_id = active_alarm_id;
    
    if (active_alarm_id >= 0) {
        MasterAlarm_t *alarm = &alarms[active_alarm_id];
        alarm_wifi_status.alarm_state = alarm->state;
        alarm_wifi_status.trigger_hour = alarm->trigger_time.Hours;
        alarm_wifi_status.trigger_min = alarm->trigger_time.Minutes;
        alarm_wifi_status.trigger_sec = alarm->trigger_time.Seconds;
        alarm_wifi_status.measured_value = alarm->last_value;
        alarm_wifi_status.setpoint = alarm->setpoint;
    } else {
        alarm_wifi_status.alarm_state = ALARM_STATE_NORMAL;
    }
    
    // Increment change counter (wraps at 255)
    alarm_wifi_status.change_counter++;
}

// Get pointer to WiFi status (for 80ms transmission)
const AlarmStatusWiFi_t* AlarmManager_GetWiFiStatus(void) {
    return &alarm_wifi_status;
}
```

### 8.2 Events

| Transition        | Event Type        |
| ----------------- | ----------------- |
| PENDING → ACTIVE  | `ALARM_ACTIVATED` |
| ACTIVE → CLEAR    | `ALARM_CLEARED`   |

### 8.3 Payload Contents
```c
typedef struct {
    uint8_t alarm_id;           // 0-9
    uint8_t event_type;         // ALARM_ACTIVATED or ALARM_CLEARED
    uint8_t trigger_hour;
    uint8_t trigger_min;
    uint8_t trigger_sec;
    float measured_value;
    float setpoint;
    uint8_t clear_mode;         // MANUAL or AUTO (only for CLEARED events)
} AlarmWiFiPayload_t;
```

### 8.4 ESP32 Integration (New Development Required)

**On ESP32 side, implement the following logic:**

```c
// ESP32 code - pseudo implementation
static uint8_t last_change_counter = 0;

void ESP32_ProcessAlarmStatus(AlarmStatusWiFi_t *status) {
    // Check if alarm status has changed
    if (status->change_counter != last_change_counter) {
        last_change_counter = status->change_counter;
        
        // Generate and send ad-hoc alarm payload
        ESP32_SendAlarmAdHocPayload(status);
    }
}

void ESP32_SendAlarmAdHocPayload(AlarmStatusWiFi_t *status) {
    // Build JSON or binary payload
    // Send via WiFi immediately
    // This is in addition to the regular 80ms data
}
```

---

## 9. UI Integration

### 9.1 Alarm Screen Trigger

On transition to ACTIVE:
- Signal UI layer to navigate to alarm screen.
- Pass `active_alarm_id` for display.

### 9.2 Alarm Screen Data

Alarm screen reads from Alarm Manager:
- Alarm name (from `active_alarm_id`)
- `last_value`
- `setpoint`
- `trigger_time`
- Current state

### 9.3 Clear Confirmation

On joystick CLEAR:
- **Manual clear is always processed, regardless of auto_clear setting**
- Alarm Manager processes the clear.
- UI returns to previous or home screen.

### 9.4 Alarm History Screen (NEW)

**Displays last 100 alarms in LIFO order (newest first).**

```c
// UI code to display history
void UI_DisplayAlarmHistory(void) {
    uint8_t count = AlarmHistory_GetCount();
    
    for (uint8_t i = 0; i < count && i < visible_rows; i++) {
        const AlarmHistoryEntry_t *entry = AlarmHistory_GetEntry(i + scroll_offset);
        if (entry == NULL || !entry->valid) continue;
        
        // Display: alarm name, trigger time, clear time, status
        UI_DrawHistoryRow(i, entry);
    }
}
```

**Joystick navigation for scrolling through history.**

---

## 10. Evaluation Cycle (Runs in DO_Control Task)

Pseudocode for each cycle:
```c
void AlarmManager_Update(uint32_t cycle_time_ms) {
    // 1. Read all measurements into local variables
    float measurements[ALARM_COUNT];
    GetAlarmMeasurements(measurements);

    // 2. For each alarm
    for (int i = 0; i < ALARM_COUNT; i++) {
        MasterAlarm_t *alarm = &alarms[i];

        // Skip disabled alarms
        if (!alarm->enabled) {
            alarm->state = ALARM_STATE_NORMAL;
            continue;
        }

        // Evaluate fault condition
        bool fault_present = EvaluateThreshold(
            alarm->threshold_type,
            measurements[i],
            alarm->setpoint
        );

        // Run state machine
        switch (alarm->state) {
            case ALARM_STATE_NORMAL:
                if (fault_present) {
                    alarm->state = ALARM_STATE_PENDING;
                    alarm->timer_elapsed_ms = 0;
                }
                break;

            case ALARM_STATE_PENDING:
                if (!fault_present) {
                    // Recovered
                    alarm->state = ALARM_STATE_NORMAL;
                    alarm->timer_elapsed_ms = 0;
                } else {
                    // Still faulted
                    alarm->timer_elapsed_ms += cycle_time_ms;
                    if (alarm->timer_elapsed_ms >= alarm->delay_ms) {
                        if (!global_ack) {
                            // Transition to ACTIVE
                            ActivateAlarm(i, measurements[i]);
                        }
                        // Else: stay PENDING (lockout)
                    }
                }
                break;

            case ALARM_STATE_ACTIVE:
                // Auto clear check (manual clear is always available via joystick)
                if (alarm->auto_clear && !fault_present) {
                    ClearAlarm(i, CLEAR_MODE_AUTO);
                }
                // Manual clear handled via joystick event
                break;

            default:
                break;
        }
    }

    // 3. Update configured DO
    AlarmManager_UpdateDO();
}
```

---

## 11. Function Signatures

```c
// Initialization
void AlarmManager_Init(void);

// Called every cycle from DO_Control task
void AlarmManager_Update(uint32_t cycle_time_ms);

// Called from joystick handler (manual clear always available)
void AlarmManager_ClearActiveAlarm(ClearMode_t mode);

// Query functions for UI
int8_t AlarmManager_GetActiveAlarmId(void);
const MasterAlarm_t* AlarmManager_GetAlarm(AlarmId_t id);
bool AlarmManager_IsAnyAlarmActive(void);

// Configuration (called when BLE config received)
void AlarmManager_SetConfig(AlarmId_t id, bool enabled, float setpoint,
                            uint32_t delay_ms, bool auto_clear);

// NEW: DO Configuration
void AlarmManager_SetDOConfig(AlarmDOConfig_t config);
AlarmDOConfig_t AlarmManager_GetDOConfig(void);

// NEW: WiFi Status
const AlarmStatusWiFi_t* AlarmManager_GetWiFiStatus(void);
void AlarmManager_UpdateWiFiStatus(void);

// NEW: Alarm History
void AlarmHistory_Add(uint8_t alarm_id, float measured_value, float setpoint);
void AlarmHistory_MarkCleared(uint8_t alarm_id, ClearMode_t mode);
const AlarmHistoryEntry_t* AlarmHistory_GetEntry(uint8_t index);
uint8_t AlarmHistory_GetCount(void);
void AlarmHistory_SaveToNV(void);
void AlarmHistory_LoadFromNV(void);

// Internal helpers
static bool EvaluateThreshold(AlarmThresholdType_t type, float value, float setpoint);
static void ActivateAlarm(AlarmId_t id, float measured_value);
static void ClearAlarm(AlarmId_t id, ClearMode_t mode);
static float GetMeasurementForAlarm(AlarmId_t id);
```

---

## 12. Summary of Changes from Original Design

| Requirement | Implementation |
|------------|----------------|
| **Alarm History (LIFO)** | Circular buffer of 100 entries with NV persistence and dedicated UI screen |
| **Configurable DO** | `AlarmDOConfig_t` enum with NONE/DO1/DO2 options, stored in NV |
| **Manual Clear in Auto Mode** | Manual clear via joystick always processed, regardless of `auto_clear` flag |
| **Alarm Status for WiFi** | `AlarmStatusWiFi_t` struct with `change_counter` for ESP32 change detection |
| **ESP32 Ad-hoc Payloads** | ESP32 monitors `change_counter` and sends immediate payload on change |
| **DO Startup State** | Always OFF on startup, no hardcoded DO assignment |