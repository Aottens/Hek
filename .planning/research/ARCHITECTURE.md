# Architecture Patterns: TIA Portal PLC Program Structure

**Domain:** Automatic Gate Control System (AutomatischHek)
**Researched:** 2026-02-02
**Confidence:** HIGH (based on official Siemens documentation and established patterns)

## Recommended Architecture

```
+------------------+
|     OB1 Main     |  Cyclic execution entry point
+------------------+
         |
         v
+------------------+     +------------------+     +------------------+
| FB_HekBesturing  |---->|   FC_Veiligheid  |---->|   FC_Outputs     |
| (State Machine)  |     | (Safety Logic)   |     | (Output Mapping) |
+------------------+     +------------------+     +------------------+
         |
         v
+------------------+     +------------------+
|   DB_HekData     |     |    DB_Config     |
| (Instance DB)    |     | (Global DB)      |
+------------------+     +------------------+
```

### Calling Hierarchy from OB1

```
OB1 [Main]
  |
  +-- FB_HekBesturing (Instance: DB_HekData)
  |     |
  |     +-- Internal state machine logic
  |     +-- Timer instances (multi-instance)
  |     +-- Counter instances (multi-instance)
  |
  +-- FC_Veiligheid
  |     |
  |     +-- PBV (Pushbutton Validation)
  |     +-- VS (Sensor Validation)
  |     +-- FAULT (Fault Detection)
  |
  +-- FC_Outputs
        |
        +-- Map state to physical outputs
```

## Block Type Selection Guidelines

### When to Use Function Block (FB)

Use FB when the logic requires **memory retention between scan cycles**:

| Use Case | Rationale |
|----------|-----------|
| State machines | Must remember current state |
| Timers and counters | Retain elapsed time/count values |
| Motor control | Track run status, fault history |
| Sequence control | Remember step position |
| Any logic needing "last value" | Static variables persist |

**FB_HekBesturing is correctly an FB because:**
- State machine requires remembering current state
- Timers for gate movement timing
- Potentially counters for cycle counting
- Internal flags that persist across scans

### When to Use Function (FC)

Use FC when the logic is **purely combinational** (no memory needed):

| Use Case | Rationale |
|----------|-----------|
| Safety validation | Evaluate current inputs only |
| Output mapping | Direct state-to-output conversion |
| Calculations | No history required |
| Data conversion | Stateless transformation |

**FC_Veiligheid and FC_Outputs are correctly FCs because:**
- Safety logic evaluates current sensor/button states
- No need to remember previous safety states
- Output mapping is direct: current state -> current outputs

## Component Boundaries

| Component | Responsibility | Data Access |
|-----------|---------------|-------------|
| OB1 | Orchestration, call sequence | Calls FBs/FCs |
| FB_HekBesturing | State machine, timing, core logic | Instance DB (DB_HekData) |
| FC_Veiligheid | Safety validation (PBV, VS, FAULT) | Parameters only (no global access) |
| FC_Outputs | State-to-output mapping | Parameters only (no global access) |
| DB_HekData | State data, timers, internal flags | Owned by FB_HekBesturing |
| DB_Config | Configuration parameters | Global read access |

## Data Block Organization

### Instance Data Block (DB_HekData)

Structure mirrors FB_HekBesturing interface:

```
DB_HekData (Instance DB for FB_HekBesturing)
|
+-- Input Area (copied from FB inputs)
|
+-- Output Area (copied from FB outputs)
|
+-- InOut Area (bidirectional parameters)
|
+-- Static Area
    |
    +-- stateNumber : INT          // Current state
    +-- statePrevious : INT        // Previous state (for transitions)
    +-- instTimerOpen : TON        // Multi-instance timer
    +-- instTimerClose : TON       // Multi-instance timer
    +-- instTimerPause : TON       // Multi-instance timer
    +-- extHmiStatus : STRUCT      // HMI-visible data (ext prefix)
    +-- statInternalFlags : STRUCT // Internal processing flags
```

### Global Data Block (DB_Config)

For configuration parameters accessed by multiple blocks:

```
DB_Config (Global DB)
|
+-- TimingParameters
|   +-- openTime_ms : DINT        // Gate open duration
|   +-- closeTime_ms : DINT       // Gate close duration
|   +-- pauseTime_ms : DINT       // Pause before auto-close
|
+-- SafetyParameters
|   +-- maxOpenAttempts : INT     // Retry limit
|   +-- sensorTimeout_ms : DINT   // Sensor response timeout
|
+-- OperationMode
    +-- autoCloseEnabled : BOOL   // Enable auto-close
    +-- manualOverride : BOOL     // Manual mode active
```

## Interface Design Patterns

### FB Interface (FB_HekBesturing)

```
FB_HekBesturing
|
+-- INPUT (from calling block, read-only)
|   +-- cmdOpen : BOOL            // Command to open
|   +-- cmdClose : BOOL           // Command to close
|   +-- cmdStop : BOOL            // Emergency stop
|   +-- sensorOpen : BOOL         // Gate fully open sensor
|   +-- sensorClosed : BOOL       // Gate fully closed sensor
|   +-- safetyOK : BOOL           // Safety validation passed
|   +-- configData : "typeConfig" // Configuration reference
|
+-- OUTPUT (to calling block, write-only)
|   +-- doOpen : BOOL             // Activate open motor
|   +-- doClose : BOOL            // Activate close motor
|   +-- currentState : INT        // Current state number
|   +-- fault : BOOL              // Fault active
|   +-- faultCode : INT           // Specific fault code
|
+-- INOUT (bidirectional, use sparingly)
|   +-- hmiData : "typeHmiData"   // HMI read/write structure
|
+-- STATIC (internal, persistent)
|   +-- statStateNumber : INT     // Current state (stat prefix)
|   +-- statPrevState : INT       // Previous state
|   +-- instTimerOpen : TON       // Multi-instance (inst prefix)
|   +-- instTimerClose : TON
|   +-- extDiagData : STRUCT      // External access (ext prefix)
|
+-- TEMP (internal, not persistent)
    +-- tempTransition : BOOL     // Temp calculation flags
    +-- tempNextState : INT       // Computed next state
```

### FC Interface (FC_Veiligheid)

```
FC_Veiligheid
|
+-- INPUT
|   +-- buttonOpen : BOOL         // Open button raw
|   +-- buttonClose : BOOL        // Close button raw
|   +-- buttonStop : BOOL         // Stop button raw
|   +-- sensorOpen : BOOL         // Open sensor raw
|   +-- sensorClosed : BOOL       // Closed sensor raw
|   +-- sensorObstacle : BOOL     // Obstacle detection
|
+-- OUTPUT
|   +-- cmdOpenValid : BOOL       // Validated open command
|   +-- cmdCloseValid : BOOL      // Validated close command
|   +-- cmdStopValid : BOOL       // Validated stop command
|   +-- safetyOK : BOOL           // Overall safety status
|   +-- faultActive : BOOL        // Fault detected
|   +-- faultCode : INT           // Fault code
|
+-- TEMP
    +-- tempPBV : BOOL            // Pushbutton validation result
    +-- tempVS : BOOL             // Sensor validation result
```

### FC Interface (FC_Outputs)

```
FC_Outputs
|
+-- INPUT
|   +-- currentState : INT        // Current state from FB
|   +-- doOpen : BOOL             // Open command from FB
|   +-- doClose : BOOL            // Close command from FB
|   +-- faultActive : BOOL        // Fault status
|
+-- OUTPUT
|   +-- motorOpen : BOOL          // Physical output: open motor
|   +-- motorClose : BOOL         // Physical output: close motor
|   +-- lampRed : BOOL            // Status lamp: fault/stop
|   +-- lampGreen : BOOL          // Status lamp: OK/moving
|   +-- lampYellow : BOOL         // Status lamp: warning
```

## State Machine Implementation Pattern

### Recommended SCL Pattern for FB_HekBesturing

```scl
// State constants (use CONSTANTS region in TIA Portal)
CONST
    STATE_IDLE := 0;
    STATE_OPENING := 10;
    STATE_OPEN := 20;
    STATE_CLOSING := 30;
    STATE_CLOSED := 40;
    STATE_STOPPED := 50;
    STATE_FAULT := 99;
END_CONST

// Main state machine using CASE
CASE #statStateNumber OF

    STATE_IDLE:
        // Entry actions
        #doOpen := FALSE;
        #doClose := FALSE;

        // Transitions
        IF #cmdOpen AND #safetyOK THEN
            #statStateNumber := STATE_OPENING;
        ELSIF #cmdClose AND #safetyOK THEN
            #statStateNumber := STATE_CLOSING;
        END_IF;

    STATE_OPENING:
        // Entry actions
        #doOpen := TRUE;
        #doClose := FALSE;

        // Timer for timeout
        #instTimerOpen(IN := TRUE, PT := T#30s);

        // Transitions
        IF #sensorOpen THEN
            #instTimerOpen(IN := FALSE);
            #statStateNumber := STATE_OPEN;
        ELSIF #instTimerOpen.Q THEN
            #faultCode := 1; // Timeout fault
            #statStateNumber := STATE_FAULT;
        ELSIF #cmdStop OR NOT #safetyOK THEN
            #instTimerOpen(IN := FALSE);
            #statStateNumber := STATE_STOPPED;
        END_IF;

    // ... additional states

END_CASE;

// Track state changes
IF #statStateNumber <> #statPrevState THEN
    #statPrevState := #statStateNumber;
    // Log state change, update diagnostics
END_IF;
```

### Key State Machine Principles

1. **Use CASE, not IF-ELSE chains** - Better readability, standard pattern
2. **Use named constants for states** - Never magic numbers
3. **State numbers with gaps (0, 10, 20...)** - Room for substates
4. **Multi-instance timers** - Each timer is part of FB's instance DB
5. **Track previous state** - Enables transition detection
6. **Single point of state change** - All transitions modify same variable

## Patterns to Follow

### Pattern 1: Multi-Instance for Timers/Counters

**What:** Declare timer/counter FBs as STATIC in your FB interface
**When:** Any FB that needs timers or counters
**Why:** Keeps timer data in parent FB's instance DB, avoids separate DBs

```scl
// In FB_HekBesturing interface, STATIC section:
instTimerOpen : TON;    // Multi-instance timer
instTimerClose : TON;
instCounter : CTU;
```

### Pattern 2: Parameter Passing at OB1 Level

**What:** Pass data between blocks via OB1 network connections
**When:** Blocks need to communicate
**Why:** Explicit data flow, visible in OB1, maintainable

```
// OB1 pseudocode (LAD/FBD network style)
Network 1: Call FC_Veiligheid
  Inputs: Raw I/O signals
  Outputs: cmdOpenValid, cmdCloseValid, safetyOK, faultActive

Network 2: Call FB_HekBesturing
  Inputs: cmdOpenValid, cmdCloseValid, safetyOK (from FC_Veiligheid)
  Outputs: doOpen, doClose, currentState

Network 3: Call FC_Outputs
  Inputs: doOpen, doClose, currentState (from FB_HekBesturing)
  Outputs: Physical output assignments
```

### Pattern 3: No Global Data Access in FBs/FCs

**What:** FBs and FCs only access data through their interface
**When:** Always
**Why:** Modular, testable, reusable blocks

```scl
// BAD - accessing global DB directly
#result := "DB_Config".TimingParameters.openTime_ms;

// GOOD - receive via input parameter
#result := #configOpenTime;  // Passed in from OB1
```

### Pattern 4: Prefix Convention for Variables

**What:** Use consistent prefixes to indicate variable scope/purpose
**When:** All variable declarations

| Prefix | Meaning | Example |
|--------|---------|---------|
| (none) | Input/Output/InOut parameters | cmdOpen, doClose |
| stat | Static internal variable | statStateNumber |
| inst | Multi-instance FB | instTimerOpen |
| ext | External access (HMI) | extHmiStatus |
| temp | Temporary variable | tempNextState |
| type | User-defined data type | typeHmiData |

## Anti-Patterns to Avoid

### Anti-Pattern 1: Boolean Variables for States

**What:** Using separate BOOL for each state
**Why bad:** Impossible to debug, no clear state, conflicting flags
**Instead:** Single INT variable with CASE statement

```scl
// BAD
#isOpening : BOOL;
#isOpen : BOOL;
#isClosing : BOOL;
// What if isOpening AND isOpen are both TRUE?

// GOOD
#stateNumber : INT;  // Only one state possible
```

### Anti-Pattern 2: Single Instance Inside FC

**What:** Calling FB with single instance from within an FC
**Why bad:** FC is not reusable - all calls share same instance
**Instead:** Use parameter instance (INOUT) for FB instance

```scl
// BAD - in FC
"FB_Timer"(IN := ..., DB := "DB_Timer_Instance");
// If FC called twice, both use same timer!

// GOOD - in FC interface, INOUT:
instTimer : TON;  // Parameter instance
// In FC body:
#instTimer(IN := ...);  // Each FC call gets own instance
```

### Anti-Pattern 3: Direct I/O Access in FBs

**What:** Reading/writing I/O addresses directly in FB
**Why bad:** FB not reusable, tied to specific hardware
**Instead:** Pass I/O via parameters

```scl
// BAD
IF %I0.0 THEN  // Direct I/O in FB
    %Q0.0 := TRUE;
END_IF;

// GOOD
IF #inputSignal THEN  // Via parameter
    #outputSignal := TRUE;
END_IF;
```

### Anti-Pattern 4: Nested Call Depth > 8

**What:** Calling FBs/FCs more than 8 levels deep
**Why bad:** Safety program limit, hard to debug
**Instead:** Flatten structure, use fewer call levels

## Data Flow Diagram

```
Physical Inputs           OB1                    Physical Outputs
     |                     |                           |
     v                     v                           |
+----------+    +-------------------+                  |
| I0.0-I0.7|--->| FC_Veiligheid     |                  |
+----------+    | - Validate inputs |                  |
                | - Safety checks   |                  |
                +--------+----------+                  |
                         |                             |
           cmdOpenValid, cmdCloseValid, safetyOK       |
                         |                             |
                         v                             |
                +-------------------+                  |
                | FB_HekBesturing   |                  |
                | - State machine   |                  |
                | - Timer control   |                  |
                | - Decision logic  |                  |
                +--------+----------+                  |
                         |                             |
              doOpen, doClose, currentState            |
                         |                             |
                         v                             |
                +-------------------+    +----------+  |
                | FC_Outputs        |--->| Q0.0-Q0.7|<-+
                | - Map to outputs  |    +----------+
                | - Status lamps    |
                +-------------------+
```

## Scalability Considerations

| Concern | Current Design | If Multiple Gates |
|---------|---------------|-------------------|
| Code reuse | Single FB_HekBesturing | Create instances per gate |
| Data isolation | Instance DB per FB | Each gate has own DB |
| Configuration | DB_Config global | Add array or struct per gate |
| Diagnostics | extHmiStatus in FB | Aggregate in global DB for HMI |

## Implementation Checklist

- [ ] OB1 calls blocks in correct order: Safety -> Logic -> Outputs
- [ ] FB_HekBesturing uses multi-instance for timers
- [ ] FC_Veiligheid has no static data (pure function)
- [ ] FC_Outputs has no static data (pure function)
- [ ] No direct I/O access in FBs/FCs
- [ ] State machine uses CASE with named constants
- [ ] Variable prefixes follow convention (stat, inst, ext, temp)
- [ ] Configuration passed via parameters, not global access

## Sources

- [Siemens Programming Style Guide for S7-1200/S7-1500](https://support.industry.siemens.com/cs/document/81318674)
- [SolisPLC: Function Block Instances in TIA Portal](https://www.solisplc.com/tutorials/function-block-instances-in-siemens-tia-portal)
- [SolisPLC: Practical Guide to TIA Portal Programming](https://www.solisplc.com/tutorials/a-practical-guide-to-siemens-tia-portal-programming)
- [Instrumentation Tools: FB vs FC Difference](https://instrumentationtools.com/difference-between-fc-and-fb-in-tia-portal/)
- [Instrumentation Tools: Instance Data Blocks](https://instrumentationtools.com/instances-of-calling-a-function-block/)
- [PLCHMIs: OB1 Main Cyclic Organization Block](https://www.plchmis.com/en-articles.html/plc-programming-learning/ob1-the-main-cyclic-organization-block-in-the-siemens-tia-portal-r146/)
- [Automation Notes: State Machine Pattern](https://abedgnu.github.io/Automation-Notes/chapters/PLC/Siemens/fsm.html)
- [PLCtalk: FB/FC Discussion](https://www.plctalk.net/forums/threads/siemens-s7-tia-portal-fc-fb.135499/)
- [Instrumentation Tools: Global Data Blocks](https://instrumentationtools.com/global-data-blocks-in-plc/)
