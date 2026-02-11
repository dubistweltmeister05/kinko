[[Titan]]
# Epic: Titan Alarm System

## 1. Overview

Implement a robust alarm system for Titan that monitors key electrical and demand parameters, applies configurable setpoints and delays, and coordinates UI, DO (configurable), and WiFi behavior under a global alarm acknowledgement (ACK) model.

### 1.1 Goals

- Detect and signal abnormal electrical/demand conditions using a consistent alarm framework.
- Avoid nuisance trips via configurable delay/buffer times and (optional) hysteresis.
- Provide clear operator feedback on the device (alarm screen) and to remote systems (WiFi).
- Enforce a global `alarm_ack` policy where only one alarm is "in focus" until acknowledged/cleared.
- **Maintain a history of the last 100 alarms (LIFO) for operator review.**
- **Provide configurable DO output (DO1, DO2, or disabled) for alarm signaling.**

### 1.2 Scope (Alarms)

In scope alarms (already defined):

- Phase Reversal  
- Phase Failure  
- Over Voltage  
- Under Voltage  
- Over Current  
- Under Current  
- Over Frequency  
- Under Frequency  
- Max Demand  
- Phase Imbalance  

Inputs, setpoint configuration via BLE, alarm screen, joystick APIs, and DO actuation APIs already exist and will be reused.

---

## 2. Functional Requirements

1. **Setpoints & Delays**
   - Each alarm has:
     - Enable/disable flag
     - One or more setpoints (thresholds)
     - A delay/buffer time before raising the alarm
     - Clear mode: auto-clear or manual-clear
   - Setpoints and delays are received over BLE (pipeline already in place).

2. **Alarm Detection & Stability Timer**
   - System evaluates all enabled alarms against incoming measured data.
   - When a parameter crosses its setpoint, a per-alarm delay timer starts.
   - If the parameter returns to normal before delay expiry, the alarm is cancelled.
   - If the parameter remains abnormal until delay expiry, the alarm becomes ACTIVE.

3. **Global ACK and Lockout**
   - When any alarm becomes ACTIVE:
     - The system switches the display to the alarm screen.
     - **Configured DO (DO1 or DO2, if enabled) is driven according to alarm policy.**
     - A WiFi alarm payload is generated.
     - A global `alarm_ack` flag is set.
   - While `alarm_ack` is set, **other alarms are not considered for activation** (no new ACTIVE alarms) until the ACK is cleared.

4. **Alarm Acknowledgement and Clearance**
   - **Manual clearance**:
     - Operator uses joystick input to acknowledge and/or clear the active alarm.
     - Clearing the alarm resets `alarm_ack` and returns system to monitoring of all alarms.
     - **Manual clear is ALWAYS available, even when alarm is configured for auto-clear mode.**
   - **Automatic clearance** (for configured alarms):
     - Once the underlying condition returns to normal (possibly with a small clear delay), the alarm clears automatically.
     - Clearing an alarm that owns `alarm_ack` resets `alarm_ack`.
     - **Auto mode does NOT block manual intervention - user can always manually clear.**
   - When `alarm_ack` is cleared:
     - Configured DO deactivates if no other alarm is active.
     - Monitoring resumes for all alarms.

5. **UI Behavior**
   - On alarm activation, device automatically transitions to the alarm screen.
   - Alarm screen shows:
     - Alarm type (e.g., Over Voltage)
     - RTC Timestamp when the alarm was triggered.
     - Alarm status (e.g., Pending, Active, Acked, Cleared)
   - Once an alarm is cleared, the UI returns according to defined UX policy (previous screen or home).
   - **A dedicated Alarm History screen displays the last 100 alarms in LIFO order.**

6. **Alarm History (LIFO)**
   - **Maintain a circular buffer of the last 100 alarm events.**
   - **Each history entry contains:**
     - Alarm type/ID
     - Trigger timestamp (RTC)
     - Clear timestamp (RTC)
     - Measured value at trigger
     - Setpoint at trigger
     - Clear mode (Manual/Auto)
   - **History is accessible via a dedicated alarm history screen.**
   - **History persists across power cycles (stored in NV storage).**

7. **DO Configuration & Behavior**
   - **DO output is fully configurable:**
     - Alarm output can be mapped to DO1, DO2, or disabled entirely.
     - There is NO reserved/hardcoded DO for alarms.
     - For all alarms, only ONE configured DO is used (not multiple).
   - **DO operates only if configured for alarms.**
   - DO asserts when one or more alarms are in ACTIVE/ACKED (latched) state.
   - DO deasserts when all alarms are CLEARED and `alarm_ack` is reset.
   - **Behavior on reset/power-up:**
     - DO default state is OFF unless explicitly restored from persisted state.
     - Recommended: Always start with DO OFF.

8. **WiFi Alarm Integration**
   - **A new alarm status variable must be maintained in the data structure.**
   - **This variable is sent automatically every 80 ms over WiFi as part of regular data.**
   - **On alarm state transitions, send detailed payloads via existing WiFi pathway:**
     - At minimum: alarm ID/type, state (ACTIVE/ACK/CLEAR), timestamp, measured value, setpoint.
   - **ESP32 WiFi Logic (new development required):**
     - ESP32 monitors the alarm status variable.
     - If the alarm variable value changes, an ad-hoc alarm payload is generated and sent immediately.
     - Payload format is consistent and documented for remote consumers.

9. **Startup and Power-Cycle Behavior**
   - On power-up:
     - Alarms start in NORMAL state.
     - `alarm_ack` is cleared.
     - Configured DO is OFF.
     - **Alarm history is restored from NV storage.**
   - Configuration (setpoints, delays, clear modes, DO assignment) is loaded from existing BLE pipeline / NV storage as applicable.

---

## 3. Epic Breakdown (Features & Stories)

### 3.1 Alarm Evaluation Engine

**Objective:** Implement a central Alarm Manager that evaluates all alarms and manages timers/state.

- **Tasks**
  - Define internal data structures for:
    - Per-alarm configuration (reference to BLE-provided setpoints and delays).
    - Per-alarm runtime state (current state, timers, last measured value).
  - Implement evaluation cycle that:
    - Reads latest measurements from existing measurement sources.
    - Evaluates each enabled alarm against thresholds.
    - Handles delay timers and (optional) hysteresis around setpoints.
    - Raises alarms after delay expiry and drives state transitions.
- **Acceptance Criteria**
  - Transient violations shorter than configured delay do not raise alarms.
  - Behavior is deterministic under repeated crossings around the setpoint.

### 3.2 Per-Alarm State Machine & Global ACK

**Objective:** Provide clear lifecycle and global lockout behavior.

- **Per-Alarm States**
  - `NORMAL` → `PENDING` → `ACTIVE` → `ACKED/LATCHED` → `CLEAR`.
- **Global ACK Rules**
  - On first alarm entering `ACTIVE`:
    - Set `alarm_ack`.
    - Lock out other alarms from becoming ACTIVE (they may remain NORMAL or PENDING).
  - While `alarm_ack` is set:
    - Only the active alarm is processed for acknowledge/clear.
  - Clearing the owning alarm:
    - Resets `alarm_ack`.
    - Allows other alarms to be evaluated for activation again.
- **Acceptance Criteria**
  - Clear definition of which alarm owns `alarm_ack`.
  - No conflicting or overlapping ACTIVE alarms while `alarm_ack` is set.

### 3.3 DO Integration (Configurable)

**Objective:** Drive configurable DO output from alarm outcomes.

- **Tasks**
  - **Add DO configuration parameter:**
    - `alarm_do_config`: NONE, DO1, or DO2.
  - Define a consolidated `alarm_active` or equivalent condition:
    - True if any alarm is in ACTIVE/ACKED state.
  - Use existing DO APIs to:
    - Assert configured DO when `alarm_active` is true (if DO is configured).
    - Deassert configured DO when `alarm_active` becomes false.
    - Skip DO operations if `alarm_do_config == NONE`.
- **Acceptance Criteria**
  - DO behavior is fully consistent with the alarm screen and WiFi notifications.
  - DO only operates if explicitly configured for alarms.
  - Startup behavior: DO defaults to OFF.

### 3.4 Joystick Acknowledge & Clear

**Objective:** Use existing joystick APIs for alarm ACK and manual clear.

- **Tasks**
  - Define user interactions:
    - E.g., single press to ACK, long press to CLEAR, or a confirm screen.
  - Map joystick events to:
    - Transition ACTIVE → ACKED or ACTIVE → CLEAR depending on design.
  - Ensure ignoring joystick ACK/CLEAR when no alarm is active.
  - **Ensure manual clear works regardless of auto_clear configuration.**
- **Acceptance Criteria**
  - No accidental clears (reasonable protection via UX pattern).
  - Operator can reliably acknowledge and clear alarms using joystick only.
  - **Manual clear is always available, even in AUTO mode.**

### 3.5 Alarm Screen Behavior

**Objective:** Drive existing alarm screen from new Alarm Manager.

- **Tasks**
  - Bind alarm runtime data to existing widgets/fields:
    - Alarm name/type
    - Value vs setpoint
    - Timer/progress indication for PENDING (if UI supports it)
    - Status text/icon
  - Ensure automatic navigation to alarm screen when alarm transitions to ACTIVE.
  - Define navigation on clear (e.g., return to previously active screen).
- **Acceptance Criteria**
  - Alarm screen always reflects the current alarm state.
  - No flicker or inconsistent states under rapid transitions.

### 3.6 Alarm History Feature

**Objective:** Implement LIFO alarm history storage and display.

- **Tasks**
  - **Implement circular buffer for 100 alarm history entries.**
  - **Define history entry structure (alarm ID, timestamps, values, clear mode).**
  - **Store history to NV storage for persistence across power cycles.**
  - **Create alarm history screen UI showing all stored alarms.**
  - **Implement scrollable list with joystick navigation.**
- **Acceptance Criteria**
  - Last 100 alarms are stored in LIFO order.
  - History survives power cycles.
  - All history entries are viewable on dedicated screen.

### 3.7 WiFi Alarm Payload Integration

**Objective:** Surface alarm events via existing WiFi channel with new status variable.

- **Tasks**
  - **Define alarm status variable in data structure:**
    - This variable is sent every 80 ms as part of regular WiFi data.
  - Define alarm event payload format:
    - Alarm ID/type, event type (ACTIVATE/ACK/CLEAR), timestamp, measured value, setpoint, optional additional metadata.
  - Hook alarm state transitions into WiFi send API.
  - **ESP32 Development (new):**
    - Monitor alarm status variable for changes.
    - Generate and send ad-hoc alarm payload on status change.
  - Apply basic debouncing or aggregation if signals are unstable.
- **Acceptance Criteria**
  - Alarm status is included in regular 80 ms WiFi transmissions.
  - ESP32 detects changes and sends immediate ad-hoc payloads.
  - Remote system receives exactly one activation event per alarm activation and corresponding ACK/CLEAR events.
  - No uncontrolled flooding in unstable conditions.

### 3.8 Startup / Reset Policy

**Objective:** Ensure predictable behavior at power-up or reset.

- **Tasks**
  - Policy:
    - Always start with `alarm_ack` cleared, no active alarms, configured DO off.
    - **Restore alarm history from NV storage.**
    - Load DO configuration from NV storage.
  - Implement minimal required persistence for alarm history and configuration.
- **Acceptance Criteria**
  - Documented, predictable behavior after power cycle.
  - No "stuck DO" or ghost alarms on startup.
  - Alarm history is preserved.

---

## 4. System Design

### 4.1 High-Level Architecture

- **Inputs**
  - Existing measurement modules providing:
    - Phase voltages and currents
    - Phase status (reversal/failure)
    - Frequency
    - Max demand / demand window values
  - Configuration via BLE:
    - Alarm enable flags
    - Setpoints
    - Delay times
    - Auto/manual clear modes
    - **DO assignment (DO1, DO2, or NONE)**
- **Core**
  - **Alarm Manager** (new logical component):
    - Periodically reads measurement snapshot.
    - Evaluates each alarm against its configuration.
    - Manages per-alarm state machines and delay timers.
    - Owns `alarm_ack` and global lockout rules.
    - **Maintains alarm history buffer.**
    - **Maintains alarm status variable for WiFi.**
- **Outputs**
  - UI / Alarm Screen:
    - Display active alarm, status, measurements vs setpoints.
  - **Alarm History Screen:**
    - Display last 100 alarms in LIFO order.
  - **Configured DO (DO1, DO2, or NONE):**
    - External signaling based on `alarm_active`.
  - WiFi:
    - Alarm status variable (every 80 ms).
    - Alarm event payloads for remote monitoring.

Conceptual data flow:

> Measurements → Alarm Manager → {Alarm Screen, Alarm History, Configured DO, WiFi}

---

### 4.2 Alarm Data Model

For each alarm, define:

- **Static Metadata**
  - ID (unique integer or enum)
  - Name and description
  - Signal source(s) (e.g., phase voltages, currents, frequency, demand)
  - Category (Voltage, Current, Frequency, Demand, Phase)

- **Configurable Parameters** (provided via BLE)
  - `enabled`: on/off
  - `setpoint_high` / `setpoint_low` / `limit` (as applicable)
  - `delay_ms` (or s)
  - `clear_mode`: `MANUAL` or `AUTO`
  - Optional: hysteresis band or percentage (for chatter reduction)

- **Runtime State**
  - Current state: `NORMAL`, `PENDING`, `ACTIVE`, `ACKED`, `CLEAR`
  - Active delay timer and elapsed time
  - Last measured value(s) at activation
  - Flag indicating ownership of `alarm_ack` (true/false)

- **Global Configuration**
  - `alarm_do_config`: `NONE`, `DO1`, or `DO2`

---

### 4.3 Alarm History Data Model

**History Entry Structure:**
```c
typedef struct {
    uint8_t alarm_id;           // Alarm type (0-9)
    RTC_TimeTypeDef trigger_time;
    RTC_DateTypeDef trigger_date;
    RTC_TimeTypeDef clear_time;
    RTC_DateTypeDef clear_date;
    float measured_value;       // Value at trigger
    float setpoint;             // Setpoint at trigger
    uint8_t clear_mode;         // MANUAL or AUTO
    bool valid;                 // Entry is valid
} AlarmHistoryEntry_t;
```

**History Buffer:**
- Circular buffer of 100 entries
- LIFO order (newest first when displayed)
- Persisted to NV storage

---

### 4.4 State Machine (Per Alarm)

Conceptual transitions:

- **NORMAL**
  - Condition out-of-range for any duration → go to `PENDING` and start delay timer.

- **PENDING**
  - If condition returns to normal before delay expires → go back to `NORMAL`.
  - If condition remains out-of-range until delay expires → transition to `ACTIVE`.

- **ACTIVE**
  - Set `alarm_ack` (if not already set).
  - Mark this alarm as owning `alarm_ack`.
  - **Add entry to alarm history.**
  - Trigger:
    - **Configured DO activation (if configured).**
    - Alarm screen navigation.
    - WiFi "alarm ACTIVE" payload.
    - **Update alarm status variable.**
  - **Await operator acknowledgement, auto-clear conditions, OR manual clear.**
  - **Manual clear is always valid, regardless of auto_clear setting.**

- **ACKED/LATCHED**
  - Optional state if ACK and CLEAR are separate:
    - Operator acknowledges alarm but does not yet clear it.
    - Configured DO may remain active depending on policy.
  - For auto-clear alarms:
    - When condition normal and any clear delay passes → transition to `CLEAR`.
  - **Manual clear is always available in this state.**

- **CLEAR**
  - **Update alarm history entry with clear timestamp and mode.**
  - Alarm returns to `NORMAL`.
  - If this alarm owned `alarm_ack`:
    - Clear `alarm_ack`.
    - If no other alarm is active, configured DO is deasserted.
  - **Update alarm status variable.**

**Global rule:**

- While `alarm_ack` is set:
  - No additional alarms can transition to `ACTIVE`.
  - They may stay in `NORMAL` or `PENDING` depending on final design choice.

---

### 4.5 Timing & Scheduling

- **Evaluation Period**
  - Alarm Manager runs periodically (e.g., on a fixed tick or in an RTOS task) synchronized with measurement updates.
- **Delay Implementation**
  - Use existing delay/timer or RTOS timing mechanisms:
    - Per-alarm delay counters based on system tick.
- **Performance Considerations**
  - Ensure the evaluation cycle completes within its time budget.
  - Minimize blocking inside Alarm Manager to avoid impacting UI or communication tasks.

---

### 4.6 UI Integration Design

- **Alarm Screen**
  - Inputs from Alarm Manager:
    - Active alarm ID/type
    - Measured value(s)
    - Setpoint(s)
    - Alarm state text/indicator
  - Behavior:
    - On transition to `ACTIVE`, UI layer is requested to show alarm screen.
    - On CLEAR, UI returns to previous or home screen (as defined).

- **Alarm History Screen**
  - **Displays last 100 alarms in LIFO order (newest first).**
  - **Scrollable list with joystick navigation.**
  - **Each entry shows: alarm type, trigger time, clear time, status.**

- **Operator Interaction**
  - Joystick inputs mapped to:
    - ACK (optional step)
    - CLEAR (always available, regardless of auto_clear mode)
  - UI provides visible feedback that ACK/CLEAR action was applied.

---

### 4.7 DO Integration Design (Configurable)

- **Configuration**
  - `alarm_do_config`: `NONE`, `DO1`, or `DO2`
  - Stored in NV storage, configurable via BLE.
  - **No reserved/hardcoded DO for alarms.**

- **Signal Source**
  - Alarm Manager exposes a boolean `alarm_active` (or equivalent).

- **Behavior**
  - If `alarm_do_config == NONE`: No DO operations for alarms.
  - If `alarm_do_config == DO1` or `DO2`:
    - DO module uses `alarm_active` boolean to drive the configured DO.
  - Define safe defaults:
    - On boot: Configured DO is OFF.
    - On error/reset in Alarm Manager: fail-safe behavior (DO OFF).

---

### 4.8 WiFi Integration Design

- **Alarm Status Variable**
  - **New variable in data structure representing current alarm state.**
  - **Sent automatically every 80 ms as part of regular WiFi data.**
  - **Contains: active alarm ID (or -1 if none), alarm state, timestamp.**

- **Events**
  - At minimum, emit events for:
    - Alarm activation (`ACTIVE`)
    - Alarm acknowledgement (if distinct)
    - Alarm clearance (`CLEAR`)

- **Payload Contents**
  - Alarm ID/type
  - Event type
  - Timestamp (RTC or relative time)
  - Measured value(s) and setpoint(s) at relevant moment
  - Optional: device ID, firmware version, location or feeder ID

- **ESP32 Development (New)**
  - **Monitor alarm status variable for changes.**
  - **On change detection:**
    - Generate ad-hoc alarm payload immediately.
    - Send via WiFi.
  - **Rate Limiting:**
    - Suppress repeated identical events within a short time window.
    - Aggregate multiple changes if they occur in very quick succession.

---

## 5. Risks, Assumptions, and Open Points

- **Assumptions**
  - Measurement accuracy and scaling are already validated.
  - BLE setpoint pipeline reliably provides up-to-date configuration before Alarm Manager uses it.
  - Existing alarm screen, joystick APIs, and DO APIs are stable and documented.
  - **NV storage has sufficient capacity for 100 alarm history entries.**

- **Risks**
  - Ambiguity on how strict the lockout (`alarm_ack`) should be if multiple serious alarms occur.
  - Possible need for hysteresis tuning to avoid chatter around thresholds.
  - WiFi payload changes may require coordination with external systems.
  - **ESP32 code changes require development and testing.**

- **Open Points to Finalize**
  - Exact operator UX for ACK vs CLEAR:
    - Separate actions or combined?
    - Short-press vs long-press semantics.
  - Startup policy:
    - Always reset alarm states, or restore last alarm on reboot.
  - Which alarms support auto-clear vs require manual clear only.
  - **History persistence strategy: save on every alarm or periodic flush.**

---

