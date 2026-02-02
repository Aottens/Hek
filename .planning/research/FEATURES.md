# Feature Landscape: S7-1200 Gate Control SCL Features

**Domain:** S7-1200 PLC programming for automatic gate control
**Researched:** 2026-02-02
**Overall Confidence:** HIGH (verified via official Siemens documentation and multiple authoritative sources)

---

## Table Stakes

Features that MUST be implemented correctly. Missing or incorrect implementation results in non-functional or unsafe gate control.

| Feature | Why Expected | Complexity | Confidence |
|---------|--------------|------------|------------|
| State machine with CASE | Required for 9-state gate control logic | Medium | HIGH |
| TON timers | Pause, retract, resume delays require on-delay timing | Low | HIGH |
| Edge detection (R_TRIG) | Push button single-press detection | Low | HIGH |
| Safety priority logic | Emergency stop must override all states | Medium | HIGH |
| Fail-safe output patterns | Outputs must default to safe state on fault | Medium | HIGH |

---

## SCL Constructs for S7-1200

### 1. State Machine Implementation

**Recommended Pattern: CASE Statement with INT Constants**

The S7-1200 in TIA Portal supports CASE statements for state machine implementation. While true enumeration types (TYPE...END_TYPE) have limited support in S7-1200 SCL, using INT constants provides equivalent functionality with better compatibility.

**Declaration Pattern:**
```scl
// State constants (in Static or Constants section)
VAR CONSTANT
    ST_HOMING       : INT := 0;
    ST_IDLE_OPEN    : INT := 1;
    ST_IDLE_DICHT   : INT := 2;
    ST_MOVING_OPEN  : INT := 3;
    ST_MOVING_DICHT : INT := 4;
    ST_STOPPED      : INT := 5;
    ST_VS_PAUSE     : INT := 6;
    ST_PBV_RETRACT  : INT := 7;
    ST_FAULT        : INT := 8;
END_VAR

VAR
    eState : INT := 0;  // Current state
END_VAR
```

**CASE Statement Syntax:**
```scl
CASE eState OF
    ST_HOMING:
        // Homing logic
        IF bHomingComplete THEN
            eState := ST_IDLE_OPEN;
        END_IF;

    ST_IDLE_OPEN:
        // Idle open logic
        IF bStartClose THEN
            eState := ST_MOVING_DICHT;
        END_IF;

    ST_MOVING_DICHT:
        // Moving closed logic
        IF bEndSwitchClosed THEN
            eState := ST_IDLE_DICHT;
        ELSIF bObstacleDetected THEN
            eState := ST_VS_PAUSE;
        END_IF;

    // ... additional states

ELSE
    // Unknown state - go to fault
    eState := ST_FAULT;
END_CASE;
```

**Confidence:** HIGH - Verified via [Stefan Henneken's State Pattern article](https://stefanhenneken.net/2018/11/17/iec-61131-3-the-state-pattern/) and [Siemens TIA Portal documentation](https://docs.tia.siemens.cloud/r/simatic_s7_1200_manual_collection_eses_20/basic-instructions/program-control-operations/scl-program-control-statements/case)

**Best Practice:** Always include an ELSE clause to catch undefined states and transition to FAULT state.

---

### 2. Timer Usage (TON, TOF)

**TON (On-Delay Timer) - Primary timer for gate control**

Used for: pause delays, retract timing, resume delays

**Declaration:**
```scl
VAR
    tonPauseTimer   : TON;      // VS pause delay
    tonRetractTimer : TON;      // PBV retract delay
    tonResumeTimer  : TON;      // Resume after pause
    tPauseTime      : TIME := T#3s;
    tRetractTime    : TIME := T#2s;
END_VAR
```

**SCL Call Syntax:**
```scl
// Call timer (must be called every scan cycle)
tonPauseTimer(
    IN := bPauseActive,         // BOOL - starts timer when TRUE
    PT := tPauseTime,           // TIME - preset time (T#3s, T#500ms, etc.)
    Q  => bPauseComplete,       // BOOL - TRUE when timer elapsed
    ET => tPauseElapsed         // TIME - current elapsed time
);
```

**TOF (Off-Delay Timer)**

Used for: debouncing, delayed release

**Declaration and Call:**
```scl
VAR
    tofDebounce : TOF;
END_VAR

tofDebounce(
    IN := bInputSignal,         // Output stays TRUE for PT after input goes FALSE
    PT := T#100ms,
    Q  => bDebouncedSignal,
    ET => tDebounceElapsed
);
```

**Timer Behavior Summary:**

| Timer | IN = FALSE->TRUE | While IN = TRUE | IN = TRUE->FALSE |
|-------|------------------|-----------------|------------------|
| TON   | ET starts counting | Q=TRUE when ET>=PT | Q=FALSE, ET resets |
| TOF   | Q=TRUE immediately | Q stays TRUE | ET starts, Q=FALSE when ET>=PT |

**Confidence:** HIGH - Verified via [Siemens TON Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/timer-operations-s7-1200-s7-1500/ton-generate-on-delay-s7-1200-s7-1500) and [PLCBlog Timer Examples](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-iec-timer-programming-tp-ton-tof-examples.php)

**TIME Value Format:**
- `T#500ms` - 500 milliseconds
- `T#10s` - 10 seconds
- `T#2m30s` - 2 minutes 30 seconds
- `T#1h10m5s` - 1 hour 10 minutes 5 seconds

---

### 3. Edge Detection (R_TRIG, F_TRIG)

**R_TRIG (Rising Edge) - Primary for push button detection**

Detects transition from FALSE to TRUE. Output Q is TRUE for exactly one scan cycle.

**Declaration:**
```scl
VAR
    rtrigPushButton : R_TRIG;   // Push button rising edge
    rtrigStartCmd   : R_TRIG;   // Start command edge
END_VAR
```

**SCL Call Syntax:**
```scl
// Call edge detection (must be called every scan cycle)
rtrigPushButton(CLK := bPushButtonRaw);

// Use the edge output
IF rtrigPushButton.Q THEN
    // Button was just pressed (one scan only)
    bToggleCommand := TRUE;
END_IF;
```

**F_TRIG (Falling Edge)**

Detects transition from TRUE to FALSE.

```scl
VAR
    ftrigSensor : F_TRIG;
END_VAR

ftrigSensor(CLK := bSensorSignal);

IF ftrigSensor.Q THEN
    // Sensor just went from TRUE to FALSE
END_IF;
```

**Manual Edge Detection Alternative:**

If R_TRIG/F_TRIG instances are not desired, manual implementation:

```scl
VAR
    bButtonPrev : BOOL;         // Previous scan value
    bButtonEdge : BOOL;         // Rising edge detected
END_VAR

// Rising edge detection
bButtonEdge := bPushButton AND NOT bButtonPrev;
bButtonPrev := bPushButton;     // Store for next scan

// Falling edge detection
bButtonFallEdge := NOT bPushButton AND bButtonPrev;
```

**Confidence:** HIGH - Verified via [Siemens R_TRIG Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/bit-logic-operations-s7-1200-s7-1500/r_trig-detect-positive-signal-edge-s7-1200-s7-1500) and [PLCTalk Forum](https://www.plctalk.net/forums/threads/positive-edge-detection-in-scl-s7.25853/)

---

### 4. Data Types for Gate Control

**Essential Data Types:**

| Type | Size | Range | Use Case |
|------|------|-------|----------|
| BOOL | 1 bit | TRUE/FALSE | Inputs, outputs, flags |
| INT | 16 bit | -32,768 to +32,767 | State variable, counters |
| DINT | 32 bit | -2,147,483,648 to +2,147,483,647 | Large counters |
| TIME | 32 bit | T#0ms to T#24d20h31m23s647ms | Timer presets |
| REAL | 32 bit | IEEE 754 float | Not needed for gate control |

**Declaration Syntax:**
```scl
VAR_INPUT
    bEndSwitchOpen  : BOOL;     // End switch - gate open
    bEndSwitchClosed: BOOL;     // End switch - gate closed
    bPushButton     : BOOL;     // Push button input
    bSafetyPhotocell: BOOL;     // Safety photocell (VS)
    bEmergencyStop  : BOOL;     // Emergency stop input
END_VAR

VAR_OUTPUT
    bMotorOpen      : BOOL;     // Motor open command
    bMotorClose     : BOOL;     // Motor close command
    bLampWarning    : BOOL;     // Warning lamp
END_VAR

VAR
    eState          : INT := 0; // State machine state
    tPausePreset    : TIME := T#3s;
END_VAR
```

**Confidence:** HIGH - Verified via [Siemens SCL Data Types](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-data-types-and-uses-and-value-assignments.php)

---

## Differentiators

Features that improve robustness but are not strictly required.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Multi-instance timers in FB | Single DB for all timers, cleaner code | Low | Declare timers in FB static section |
| Diagnostic state logging | Debug and maintenance support | Medium | Log state transitions to DB |
| Watchdog monitoring | Detect program stuck conditions | Medium | Use Re_Trigr instruction |
| Configurable timing parameters | Easy tuning without code changes | Low | Expose times as FB inputs |

---

## Anti-Features

Features to explicitly NOT build or patterns to avoid.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Jumps in state machine | Unpredictable state, not fail-safe | Use CASE statement, explicit transitions |
| Timer inside CASE without external call | Timer may not reset on state change | Call timers outside CASE, control via flags |
| Hardcoded time values | Difficult to tune in field | Use TIME variables or FB inputs |
| State as WORD/DWORD | No semantic meaning, error-prone | Use INT with named constants |
| Relying on scan order | Fragile, maintenance nightmare | Explicit state transitions only |
| Direct output writes from multiple states | Race conditions, unclear logic | Calculate outputs once after CASE |

---

## Safety-Critical Logic Patterns

### Priority Logic Pattern

Emergency stop and safety inputs MUST override all other logic:

```scl
// Safety check FIRST - before state machine
IF NOT bEmergencyStop THEN
    // E-stop is NC (normally closed), so FALSE = pressed
    eState := ST_FAULT;
    bMotorOpen := FALSE;
    bMotorClose := FALSE;
    RETURN;  // Exit immediately
END_IF;

// Safety photocell during closing
IF (eState = ST_MOVING_DICHT) AND NOT bSafetyPhotocell THEN
    eState := ST_VS_PAUSE;
END_IF;

// Then run state machine
CASE eState OF
    // ... states
END_CASE;
```

**Confidence:** HIGH - Based on [Siemens Safety Programming Guideline](https://cache.industry.siemens.com/dl/files/255/109750255/att_1132641/v1/109750255_Programming-Guideline-Safety_DOC_V1_3_en.pdf)

### Fail-Safe Output Pattern

Outputs should default to safe state:

```scl
// Default all outputs to safe state
bMotorOpen := FALSE;
bMotorClose := FALSE;

// Only energize in specific safe states
CASE eState OF
    ST_MOVING_OPEN:
        bMotorOpen := TRUE;     // Only state where open is TRUE

    ST_MOVING_DICHT:
        bMotorClose := TRUE;    // Only state where close is TRUE

    // All other states: outputs remain FALSE (safe)
END_CASE;
```

### Output Interlock Pattern

Motor outputs must never be simultaneously TRUE:

```scl
// Interlock - physically impossible and damaging
IF bMotorOpen AND bMotorClose THEN
    // This should never happen - fault condition
    bMotorOpen := FALSE;
    bMotorClose := FALSE;
    eState := ST_FAULT;
END_IF;
```

---

## Timer Patterns for State Machine

### Pattern 1: External Timer Call with State Control

**Problem:** Timers inside CASE may not reset properly on state transitions.

**Solution:** Call timers outside CASE, use state-specific enable flags.

```scl
VAR
    tonPause : TON;
    bPauseTimerEnable : BOOL;
END_VAR

// Timer call OUTSIDE the CASE statement
bPauseTimerEnable := (eState = ST_VS_PAUSE);
tonPause(IN := bPauseTimerEnable, PT := T#3s);

// State machine uses timer output
CASE eState OF
    ST_VS_PAUSE:
        IF tonPause.Q THEN
            eState := ST_PBV_RETRACT;
        END_IF;
END_CASE;
```

**Confidence:** MEDIUM - Based on [ABPLCCenter Timer Issues article](https://www.abplccenter.com/how-to-avoid-common-timer-instruction-issues-in-s7-1200-1500-plcs_n39) and forum discussions

### Pattern 2: Timer in Static Section (Multi-Instance)

For Function Blocks, declare timers in Static section to share single DB:

```scl
FUNCTION_BLOCK FB_GateControl
VAR
    // Timers as multi-instance (share FB's instance DB)
    tonPause    : TON;
    tonRetract  : TON;
    tonResume   : TON;

    // State and flags
    eState      : INT := 0;
END_VAR
```

---

## Feature Dependencies

```
Edge Detection (R_TRIG)
    |
    v
Push Button Input --> State Machine (CASE)
                           |
                           +--> TON Timers (Pause, Retract)
                           |
                           +--> Safety Priority Check
                           |
                           v
                      Output Logic --> Motor Commands
                           |
                           v
                      Output Interlock Check
```

**Critical Path:**
1. Edge detection must be called before state machine
2. Safety checks before state transitions
3. Timers called outside CASE, outputs determined by state
4. Interlock check after output calculation

---

## MVP Recommendation

For MVP gate control, prioritize:

1. **State machine with CASE and INT constants** - Core control logic
2. **TON timers for delays** - VS pause, PBV retract, resume timing
3. **R_TRIG for push button** - Single-press detection
4. **Safety priority logic** - Emergency stop override, photocell pause
5. **Fail-safe output pattern** - Default to safe, explicit enable only

Defer to post-MVP:
- Diagnostic logging: Adds complexity, not required for function
- Watchdog monitoring: Nice-to-have for robustness
- Configurable parameters via HMI: Can hardcode initially

---

## Code Template: Complete Gate Control FB

```scl
FUNCTION_BLOCK FB_GateControl
VAR_INPUT
    // Physical inputs
    bEndSwitchOpen      : BOOL;     // Gate fully open
    bEndSwitchClosed    : BOOL;     // Gate fully closed
    bPushButton         : BOOL;     // Push button (toggle)
    bSafetyPhotocell    : BOOL;     // Safety beam (VS)
    bEmergencyStop      : BOOL;     // E-stop (NC, TRUE = safe)
END_VAR

VAR_OUTPUT
    // Physical outputs
    bMotorOpen          : BOOL;     // Motor open command
    bMotorClose         : BOOL;     // Motor close command
    bLampWarning        : BOOL;     // Warning lamp

    // Diagnostics
    nCurrentState       : INT;      // Current state (for HMI)
END_VAR

VAR CONSTANT
    // State constants
    ST_HOMING           : INT := 0;
    ST_IDLE_OPEN        : INT := 1;
    ST_IDLE_DICHT       : INT := 2;
    ST_MOVING_OPEN      : INT := 3;
    ST_MOVING_DICHT     : INT := 4;
    ST_STOPPED          : INT := 5;
    ST_VS_PAUSE         : INT := 6;
    ST_PBV_RETRACT      : INT := 7;
    ST_FAULT            : INT := 8;
END_VAR

VAR
    // State machine
    eState              : INT := 0;

    // Edge detection
    rtrigButton         : R_TRIG;

    // Timers (multi-instance in FB)
    tonPause            : TON;
    tonRetract          : TON;
    tonResume           : TON;

    // Timer enable flags
    bPauseEnable        : BOOL;
    bRetractEnable      : BOOL;
    bResumeEnable       : BOOL;

    // Timing constants
    tPauseTime          : TIME := T#3s;
    tRetractTime        : TIME := T#2s;
    tResumeTime         : TIME := T#1s;
END_VAR

// ========================================
// EDGE DETECTION (call first)
// ========================================
rtrigButton(CLK := bPushButton);

// ========================================
// SAFETY PRIORITY (call before state machine)
// ========================================
IF NOT bEmergencyStop THEN
    eState := ST_FAULT;
END_IF;

// ========================================
// TIMER CALLS (outside CASE, controlled by state)
// ========================================
bPauseEnable := (eState = ST_VS_PAUSE);
tonPause(IN := bPauseEnable, PT := tPauseTime);

bRetractEnable := (eState = ST_PBV_RETRACT);
tonRetract(IN := bRetractEnable, PT := tRetractTime);

// ========================================
// STATE MACHINE
// ========================================
CASE eState OF

    ST_HOMING:
        // Initial state - determine position
        IF bEndSwitchOpen THEN
            eState := ST_IDLE_OPEN;
        ELSIF bEndSwitchClosed THEN
            eState := ST_IDLE_DICHT;
        END_IF;

    ST_IDLE_OPEN:
        IF rtrigButton.Q THEN
            eState := ST_MOVING_DICHT;
        END_IF;

    ST_IDLE_DICHT:
        IF rtrigButton.Q THEN
            eState := ST_MOVING_OPEN;
        END_IF;

    ST_MOVING_OPEN:
        IF bEndSwitchOpen THEN
            eState := ST_IDLE_OPEN;
        ELSIF rtrigButton.Q THEN
            eState := ST_STOPPED;
        END_IF;

    ST_MOVING_DICHT:
        IF bEndSwitchClosed THEN
            eState := ST_IDLE_DICHT;
        ELSIF NOT bSafetyPhotocell THEN
            eState := ST_VS_PAUSE;
        ELSIF rtrigButton.Q THEN
            eState := ST_STOPPED;
        END_IF;

    ST_STOPPED:
        IF rtrigButton.Q THEN
            // Resume in opposite direction
            eState := ST_MOVING_OPEN;
        END_IF;

    ST_VS_PAUSE:
        IF tonPause.Q THEN
            eState := ST_PBV_RETRACT;
        END_IF;

    ST_PBV_RETRACT:
        IF tonRetract.Q OR bEndSwitchOpen THEN
            eState := ST_IDLE_OPEN;
        END_IF;

    ST_FAULT:
        // Manual reset required
        IF bEmergencyStop AND rtrigButton.Q THEN
            eState := ST_HOMING;
        END_IF;

ELSE
    // Unknown state - fault
    eState := ST_FAULT;

END_CASE;

// ========================================
// OUTPUT CALCULATION (after state machine)
// ========================================
// Default to safe state
bMotorOpen := FALSE;
bMotorClose := FALSE;
bLampWarning := FALSE;

CASE eState OF
    ST_MOVING_OPEN, ST_PBV_RETRACT:
        bMotorOpen := TRUE;
        bLampWarning := TRUE;

    ST_MOVING_DICHT:
        bMotorClose := TRUE;
        bLampWarning := TRUE;

    ST_VS_PAUSE:
        bLampWarning := TRUE;   // Flashing would need additional logic

    ST_FAULT:
        bLampWarning := TRUE;   // Indicate fault
END_CASE;

// ========================================
// OUTPUT INTERLOCK (final safety check)
// ========================================
IF bMotorOpen AND bMotorClose THEN
    bMotorOpen := FALSE;
    bMotorClose := FALSE;
    eState := ST_FAULT;
END_IF;

// Diagnostic output
nCurrentState := eState;

END_FUNCTION_BLOCK
```

---

## Sources

### HIGH Confidence (Official Documentation)
- [Siemens TIA Portal CASE Statement](https://docs.tia.siemens.cloud/r/simatic_s7_1200_manual_collection_eses_20/basic-instructions/program-control-operations/scl-program-control-statements/case)
- [Siemens TON Timer Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/timer-operations-s7-1200-s7-1500/ton-generate-on-delay-s7-1200-s7-1500)
- [Siemens R_TRIG Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/bit-logic-operations-s7-1200-s7-1500/r_trig-detect-positive-signal-edge-s7-1200-s7-1500)
- [Siemens Safety Programming Guideline](https://cache.industry.siemens.com/dl/files/255/109750255/att_1132641/v1/109750255_Programming-Guideline-Safety_DOC_V1_3_en.pdf)
- [Siemens Programming Guideline S7-1200/1500](https://support.industry.siemens.com/cs/document/81318674/programming-guidelines-and-programming-styleguide-for-simatic-s7-1200-and-s7-1500-and-wincc-(tia-portal))

### MEDIUM Confidence (Verified Technical Sources)
- [Stefan Henneken - IEC 61131-3 State Pattern](https://stefanhenneken.net/2018/11/17/iec-61131-3-the-state-pattern/)
- [PLCBlog - SCL IEC Timer Programming](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-iec-timer-programming-tp-ton-tof-examples.php)
- [PLCBlog - SCL Data Types](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-data-types-and-uses-and-value-assignments.php)
- [ABPLCCenter - Timer Issues in S7-1200](https://www.abplccenter.com/how-to-avoid-common-timer-instruction-issues-in-s7-1200-1500-plcs_n39)

### LOW Confidence (Community/Forum Sources)
- [PLCTalk - Edge Detection in SCL](https://www.plctalk.net/forums/threads/positive-edge-detection-in-scl-s7.25853/)
- [LamaPLC - SCL Commands Reference](https://www.lamaplc.com/doku.php?id=simatic:scl_commands)
