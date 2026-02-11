[[Titan]]


# CODE DESIGN

## 1. Enums and Master Struct
An Enum to track the states of the alarm at any given time.
```
typedef enum {
    ALARM_STATE_NORMAL,
    ALARM_STATE_PENDING,
    ALARM_STATE_ACTIVE,
    ALARM_STATE_ACKED,
    ALARM_STATE_CLEAR
} AlarmState_t;
```

An Enum to track the nature of the alarm.
```
typedef enum {
    ALARM_TYPE_OVER,
    ALARM_TYPE_UNDER,
    ALARM_TYPE_BINARY
} AlarmThresholdType_t;
```

A master structure for the alarm that can be replicated for each.
```
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

Global State
```
static MasterAlarm_t alarms[ALARM_COUNT];
static bool global_ack = false;
static int8_t active_alarm_id = -1;  // -1 = none
```

## 2. Alarm-to-Measurement Mapping

| Alarm ID | Alarm Name      | Threshold Type | Measurement Source              |
| -------- | --------------- | -------------- | ------------------------------- |
| 0        | Phase Reversal  | BINARY         | `phase_reversal_flag`           |
| 1        | Phase Failure   | BINARY         | `phase_failure_flag`            |
| 2        | Over Voltage    | OVER           | `voltage_max` (max of 3 phases) |
| 3        | Under Voltage   | UNDER          | `voltage_min` (min of 3 phases) |
| 4        | Over Current    | OVER           | `current_max` (max of 3 phases) |
| 5        | Under Current   | UNDER          | `current_min` (min of 3 phases) |
| 6        | Over Frequency  | OVER           | `frequency`                     |
| 7        | Under Frequency | UNDER          | `frequency`                     |
| 8        | Max Demand      | OVER           | `demand_kw` or `demand_kva`     |
| 9        | Phase Imbalance | OVER           | `imbalance_percent`             |

## 3. Threshold Evaluation Logic

For each alarm, determine if a fault condition is present:

|Threshold Type|Fault Condition|
|---|---|
|`ALARM_TYPE_OVER`|`measured_value > setpoint`|
|`ALARM_TYPE_UNDER`|`measured_value < setpoint`|
|`ALARM_TYPE_BINARY`|`measured_value != 0`|

## 4. State Machine

### 4.1 State Diagram
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
└────┬─────┘
     │ manual clear (joystick)
     │ OR auto_clear AND fault clears
     ▼
┌──────────┐
│  CLEAR   │──── transitions back to NORMAL
└──────────┘

### 4.2 State Transitions

#### NORMAL → PENDING

- **Condition:** `fault_present == true`
- **Actions:**
    - Set `state = ALARM_STATE_PENDING`
    - Set `timer_elapsed_ms = 0`

#### PENDING → NORMAL

- **Condition:** `fault_present == false` (recovered before delay)
- **Actions:**
    - Set `state = ALARM_STATE_NORMAL`
    - Set `timer_elapsed_ms = 0`

#### PENDING → ACTIVE

- **Condition:** `timer_elapsed_ms >= delay_ms` AND `global_ack == false`
- **Actions:**
    - Set `state = ALARM_STATE_ACTIVE`
    - Capture `last_value = current_measurement`
    - Capture `trigger_hour`, `trigger_min`, `trigger_sec` from RTC
    - Set `global_ack = true`
    - Set `owns_ack = true`
    - Set `active_alarm_id = this_alarm_id`
    - **Trigger:** UI navigation to alarm screen
    - **Trigger:** DO2 activation
    - **Trigger:** WiFi ALARM_ACTIVATED payload

#### PENDING while global_ack == true (Lockout)

- Alarm remains in PENDING state.
- Timer continues to run.
- Cannot transition to ACTIVE until `global_ack` is cleared.
- Once `global_ack` clears: if still faulted and `timer_elapsed_ms >= delay_ms`, immediately transition to ACTIVE.

#### ACTIVE → CLEAR (Manual)

- **Condition:** Joystick CLEAR event AND `owns_ack == true`
- **Actions:**
    - Set `state = ALARM_STATE_NORMAL`
    - Set `global_ack = false`
    - Set `owns_ack = false`
    - Set `active_alarm_id = -1`
    - **Trigger:** DO2 deactivation (if no other active alarms)
    - **Trigger:** WiFi ALARM_CLEARED payload
    - **Trigger:** UI return to normal screen

#### ACTIVE → CLEAR (Auto)

- **Condition:** `auto_clear == true` AND `fault_present == false`
- **Actions:** Same as manual clear.


## 6. DO2 Output Logic

After each evaluation cycle:

```
bool any_active = false;
for (int i = 0; i < ALARM_COUNT; i++) {
    if (alarms[i].state == ALARM_STATE_ACTIVE) {
        any_active = true;
        break;
    }
}
DO2_Set(any_active);
```

- DO2 asserts when any alarm is ACTIVE.
- DO2 de-asserts when no alarm is ACTIVE.
- On startup/reset: DO2 defaults to OFF.
## 7. Joystick Integration

In joystick event handler or main polling loop:
```
if (joystick_clear_event_detected) {
    if (global_ack && active_alarm_id >= 0) {
        AlarmManager_ClearActiveAlarm();
    }
}
```

`AlarmManager_ClearActiveAlarm()`:

- Finds alarm with `owns_ack == true`.
- Transitions it to CLEAR state.
- Executes all clear actions (reset flags, notify DO2, WiFi, UI).

## 8. WiFi Payload Integration

### 8.1 Events

|Transition|Event Type|
|---|---|
|PENDING → ACTIVE|`ALARM_ACTIVATED`|
|ACTIVE → CLEAR|`ALARM_CLEARED`|

### 8.2 Payload Contents

```
ALARM_ACTIVATED:
  - alarm_id (0-9)
  - alarm_name (string)
  - measured_value (float)
  - setpoint (float)
  - trigger_time (HH:MM:SS)

ALARM_CLEARED:
  - alarm_id (0-9)
  - alarm_name (string)
  - clear_time (HH:MM:SS)
  - clear_mode ("MANUAL" or "AUTO")
```
### 8.3 Integration Point

On state transition to ACTIVE or CLEAR, call:
```
Wifi_SendAlarmEvent(alarm_id, event_type, &alarm_data);
```

WiFi module handles encoding and transmission.

## 9. UI Integration

### 9.1 Alarm Screen Trigger

On transition to ACTIVE:

- Signal UI layer to navigate to alarm screen.
- Pass `active_alarm_id` for display.

### 9.2 Alarm Screen Data

Alarm screen reads from Alarm Manager:

- Alarm name (from `active_alarm_id`)
- `last_value`
- `setpoint`
- `trigger_hour`, `trigger_min`, `trigger_sec`
- Current state

### 9.3 Clear Confirmation

On joystick CLEAR:

- Alarm Manager processes the clear.
- UI returns to previous or home screen.
## 10. Evaluation Cycle (Runs in DO_Control Task)

Pseudocode for each cycle:
```
void AlarmManager_Update(uint32_t cycle_time_ms) {
    // 1. Read all measurements into local variables
    float measurements[ALARM_COUNT] = GetAlarmMeasurements();

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
                if (alarm->auto_clear && !fault_present) {
                    ClearAlarm(i, CLEAR_MODE_AUTO);
                }
                // Manual clear handled via joystick event
                break;

            default:
                break;
        }
    }

    // 3. Update DO2
    DO2_Set(IsAnyAlarmActive());
}
```


## 11. Function Signatures
```
// Initialization
void AlarmManager_Init(void);

// Called every cycle from DO_Control task
void AlarmManager_Update(uint32_t cycle_time_ms);

// Called from joystick handler
void AlarmManager_ClearActiveAlarm(void);

// Query functions for UI
int8_t AlarmManager_GetActiveAlarmId(void);
const MasterAlarm_t* AlarmManager_GetAlarm(AlarmId_t id);
bool AlarmManager_IsAnyAlarmActive(void);

// Configuration (called when BLE config received)
void AlarmManager_SetConfig(AlarmId_t id, bool enabled, float setpoint,
                            uint32_t delay_ms, bool auto_clear);

// Internal helpers
static bool EvaluateThreshold(AlarmThresholdType_t type, float value, float setpoint);
static void ActivateAlarm(AlarmId_t id, float measured_value);
static void ClearAlarm(AlarmId_t id, ClearMode_t mode);
static float GetMeasurementForAlarm(AlarmId_t id);
```