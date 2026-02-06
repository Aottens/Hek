# Phase 4: Integration - Research

**Researched:** 2026-02-06
**Domain:** TIA Portal SCL timer patterns, output mapping, state machine integration
**Confidence:** HIGH

## Summary

Phase 4 integrates all components: timer sequences for low speed control, VS resume delay, PBV pause/retract timing, and FC_Outputs for physical output mapping with mutual exclusion. The current codebase has a complete 9-state machine in FB_HekBesturing with placeholder outputs, and a stub FC_Outputs that passes through values without interlock logic.

The critical implementation decisions are:
1. **TONR timer** for low speed (pause on stop, reset on direction change) - S7-1200 native support
2. **TON timers** for delays (VS resume, PBV pause) - standard IEC timers
3. **Defensive mutual exclusion** in FC_Outputs - both outputs blocked if invalid combination
4. **Timer calls outside CASE** - call at FB entry, capture .Q once per scan for deterministic behavior

**Primary recommendation:** Use TONR timer for the pausable low speed timer (TMR-08 behavior built-in), TON timers for fixed delays, and implement defensive interlock pattern in FC_Outputs that blocks both motor outputs if somehow both are requested.

## Standard Stack

The established timers and patterns for this domain:

### Core Timer Types (S7-1200)
| Timer | Purpose | When to Use | Key Feature |
|-------|---------|-------------|-------------|
| TONR | Low speed duration | TMR-08: pause on stop, reset on direction change | Accumulates time, pauses when IN=FALSE, reset via R input |
| TON | Fixed delays | VS resume delay, PBV pause | Resets when IN goes FALSE |
| TOF | Off-delay | Not needed in this phase | Output stays on after input drops |
| TP | Pulse | Not needed in this phase | Fixed pulse regardless of input |

### TONR Timer Parameters
| Parameter | Type | Direction | Description |
|-----------|------|-----------|-------------|
| IN | Bool | Input | When TRUE, timer accumulates; when FALSE, timer pauses |
| PT | Time | Input | Preset time target |
| R | Bool | Input | Reset - clears ET and Q when TRUE |
| Q | Bool | Output | TRUE when accumulated time >= PT |
| ET | Time | Output | Current accumulated elapsed time |

**SCL Syntax:**
```scl
// TONR declaration in FB VAR section
m_TONR_LaagToeren : TONR;

// TONR call (outside CASE, at FB entry)
#m_TONR_LaagToeren(
    IN := #MotorRunning,      // Accumulate while motor running
    PT := "DB_Config".Timers.CFG_LaagToerenMs,
    R := #DirectionChanged,   // Reset on direction change
    Q => #LowSpeedComplete,   // Output: low speed period done
    ET => #LowSpeedElapsed    // Current accumulated time
);
```

### Supporting Timers
| Timer Instance | Purpose | Trigger Condition | Reset Condition |
|----------------|---------|-------------------|-----------------|
| m_TONR_LaagToeren | Start low speed (5s) | Motor moving | Direction change or endstop |
| m_TON_VS_Hervatting | VS resume delay (2s) | Beam cleared in VS_PAUSE | Beam blocked again |
| m_TONR_VS_LaagToeren | Post-VS low speed (2s) | After VS resume | Direction change or endstop |
| m_TON_PBV_Pauze | Pre-retract pause (500ms) | PBV_RETRACT entry | Timer complete or FAULT |

**Note:** Two separate low speed timers recommended:
- `m_TONR_LaagToeren` for normal start (5000ms)
- `m_TONR_VS_LaagToeren` for post-VS resume (2000ms)

Both need pausable behavior, hence TONR.

## Architecture Patterns

### Recommended Timer Location

**Decision:** Timer instances in FB_HekBesturing VAR section (not separate in DB_HekData)

**Rationale:**
- Timers are internal implementation detail of state machine
- Keeps FB self-contained with all its state
- DB_HekData already contains FB instance which includes timer state
- Matches existing pattern (m_TON_Timeout, m_TON_PBV_Retract already in FB)

### Timer Call Pattern (TMR-09)

```scl
// At TOP of FB_HekBesturing, BEFORE state machine CASE
// This ensures:
// 1. Timers called every scan regardless of state
// 2. .Q captured once per scan for deterministic state transitions

// Low speed timer - accumulates during movement, pauses on stop
#m_TONR_LaagToeren(
    IN := (#m_Toestand = "DB_States".STATE_MOVING_OPEN) OR
          (#m_Toestand = "DB_States".STATE_MOVING_DICHT),
    PT := "DB_Config".Timers.CFG_LaagToerenMs,
    R := #m_DirectionChanged OR #m_ReachedEndstop,
    Q => #m_LowSpeedComplete
);

// Capture timer outputs ONCE per scan for use in state logic
#t_LowSpeedDone := #m_TONR_LaagToeren.Q;

// THEN: Main state machine CASE statement
CASE #m_Toestand OF
    // ... states use #t_LowSpeedDone
END_CASE;
```

### Interface Between FB_HekBesturing and FC_Outputs

**Recommended Interface:** State + Direction + Low Speed Flag

```scl
// FB_HekBesturing outputs (existing + new)
VAR_OUTPUT
    q_HuidigeToestand : Int;       // State for FC_Outputs
    q_LaatsteRichting : Int;       // Direction for FC_Outputs
    q_LowSpeedActive : Bool;       // Low speed flag (new)

    // REMOVE direct motor outputs from FB - move to FC_Outputs
    // q_MS_Open : Bool;           // DELETE
    // q_MS_Dicht : Bool;          // DELETE
    // q_MS_LaagToeren : Bool;     // DELETE
    // q_AlarmKerk : Bool;         // DELETE
END_VAR
```

**Alternative (simpler):** FB keeps motor outputs, FC_Outputs adds interlock layer

The second approach requires less refactoring and maintains existing interface:
```scl
// FC_Outputs receives FB outputs and applies defensive checks
// FB still computes motor intent, FC enforces mutual exclusion
```

**Recommendation:** Keep existing FB output interface, add defensive checks in FC_Outputs. This minimizes changes to tested code while adding safety layer.

### Output Mutual Exclusion Pattern

```scl
// FC_Outputs.scl - Defensive mutual exclusion
FUNCTION "FC_Outputs" : Void

VAR_INPUT
    i_MS_Open : Bool;
    i_MS_Dicht : Bool;
    i_MS_LaagToeren : Bool;
    i_AlarmKerk : Bool;
END_VAR

VAR_OUTPUT
    q_MS_Open : Bool;
    q_MS_Dicht : Bool;
    q_MS_LaagToeren : Bool;
    q_AlarmKerk : Bool;
END_VAR

BEGIN
    // OUT-05: Motor outputs mutually exclusive
    // DEFENSIVE: If both requested (bug in FB), block BOTH
    IF #i_MS_Open AND #i_MS_Dicht THEN
        // Invalid combination - fail safe, stop motor
        #q_MS_Open := FALSE;
        #q_MS_Dicht := FALSE;
        #q_MS_LaagToeren := FALSE;
    ELSE
        // Normal operation - pass through
        #q_MS_Open := #i_MS_Open;
        #q_MS_Dicht := #i_MS_Dicht;
        #q_MS_LaagToeren := #i_MS_LaagToeren;
    END_IF;

    // Alarm always passed through (fail-safe is inherent)
    // 0 = alarm active (hek dicht), 1 = alarm inactive
    #q_AlarmKerk := #i_AlarmKerk;

END_FUNCTION
```

### Project Structure After Phase 4

```
PLC/Sources/
    01_DB_States.scl        # State constants (unchanged)
    02_DB_Config.scl        # Config params (unchanged - all timer values present)
    03_FC_Veiligheid.scl    # Now FB_Veiligheid (from Phase 2)
    04_FB_HekBesturing.scl  # State machine + timers (MODIFY)
    05_DB_HekData.scl       # FB instances (unchanged)
    06_FC_Outputs.scl       # Output mapping + interlock (MODIFY)
    07_Main_OB1.scl         # Orchestration (minor changes)
```

## Don't Hand-Roll

Problems with existing solutions in TIA Portal:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Pausable timer | Custom elapsed time tracking | TONR timer | Built-in pause/resume, tested, handles edge cases |
| Timer reset | Manual ET := T#0ms | TONR R input | Native reset, atomic operation |
| Motor interlock | Complex state tracking | Simple IF check in FC | Defensive, catches bugs, single responsibility |
| Timer call timing | Calling in CASE branches | Call before CASE | Deterministic, timer always runs, .Q stable for entire scan |

**Key insight:** TONR is specifically designed for the pausable timer pattern. Don't implement custom pause logic with TON + separate TIME variable - the TONR does exactly this.

## Common Pitfalls

### Pitfall 1: Calling Timers Inside CASE Branches
**What goes wrong:** Timer only called when in specific state, misses scans, inconsistent .Q values within same cycle
**Why it happens:** Natural instinct to group timer with related state logic
**How to avoid:** Call ALL timers at FB entry, BEFORE CASE statement (TMR-09)
**Warning signs:** Timer seems slow, state transitions happen at wrong times

### Pitfall 2: Using TON Instead of TONR for Pausable Timer
**What goes wrong:** Timer resets when motor stops, low speed starts from zero on resume
**Why it happens:** TON is more common, TONR less familiar
**How to avoid:** TMR-08 explicitly requires pause behavior - use TONR
**Warning signs:** Low speed duration restarts after push button stop/resume

### Pitfall 3: Trusting FB Motor Outputs Without Defensive Check
**What goes wrong:** Logic bug in FB could enable both motor directions simultaneously
**Why it happens:** Assumption that FB is always correct
**How to avoid:** Defensive check in FC_Outputs (OUT-05)
**Warning signs:** Motor damage, contactor short circuit

### Pitfall 4: Forgetting TONR Reset on Direction Change
**What goes wrong:** Low speed timer continues accumulating across direction changes
**Why it happens:** Reset condition not wired to direction change detection
**How to avoid:** Add edge detection for direction change, wire to TONR R input
**Warning signs:** No low speed after direction change

### Pitfall 5: VS Resume Without Low Speed
**What goes wrong:** Motor goes full speed immediately after VS resume
**Why it happens:** Using same timer for start and VS resume, or forgetting VS has separate timer
**How to avoid:** Use separate TONR for VS low speed (CFG_VS_LaagToerenMs)
**Warning signs:** Jerky movement after beam clears

### Pitfall 6: Alarm Output Sense Inversion
**What goes wrong:** Alarm active when gate open, inactive when closed
**Why it happens:** Forgetting q_AlarmKerk = 0 means alarm ON (fail-safe)
**How to avoid:** Comment clearly, test explicitly
**Warning signs:** Church alarm triggers when gate opens instead of when it should

## Code Examples

### Example 1: Complete Low Speed Timer Logic

```scl
// In FB_HekBesturing VAR section
VAR
    // Timers
    m_TONR_LaagToeren : TONR;        // Normal start low speed
    m_TONR_VS_LaagToeren : TONR;     // Post-VS resume low speed
    m_TON_VS_Hervatting : TON;       // VS resume delay

    // Edge detection for direction change
    m_VorigeRichting : Int;          // Previous direction for change detection

    // Timer output captures (captured once per scan)
    m_LowSpeedDone : Bool;
    m_VS_LowSpeedDone : Bool;
    m_VS_ResumeDelayDone : Bool;
END_VAR

// At FB entry, BEFORE CASE statement
BEGIN
    // Direction change detection
    VAR_TEMP
        t_DirectionChanged : Bool;
        t_MotorMoving : Bool;
        t_ReachedEndstop : Bool;
    END_VAR

    // Detect direction change (for timer reset)
    #t_DirectionChanged := (#m_LaatsteRichting <> #m_VorigeRichting) AND
                           (#m_LaatsteRichting <> "DB_States".DIR_NONE);
    #m_VorigeRichting := #m_LaatsteRichting;

    // Motor moving in normal movement states
    #t_MotorMoving := (#m_Toestand = "DB_States".STATE_MOVING_OPEN) OR
                      (#m_Toestand = "DB_States".STATE_MOVING_DICHT);

    // Reached endstop (timer reset condition)
    #t_ReachedEndstop := (NOT #i_BNS_Open) OR (NOT #i_BNS_Dicht);

    // Normal start low speed timer (TMR-01)
    #m_TONR_LaagToeren(
        IN := #t_MotorMoving,
        PT := "DB_Config".Timers.CFG_LaagToerenMs,
        R := #t_DirectionChanged OR #t_ReachedEndstop,
        Q => #m_LowSpeedDone
    );

    // Post-VS resume low speed timer (TMR-05)
    #m_TONR_VS_LaagToeren(
        IN := #t_MotorMoving AND #m_VS_ResumeActive,
        PT := "DB_Config".Timers.CFG_VS_LaagToerenMs,
        R := #t_DirectionChanged OR #t_ReachedEndstop OR NOT #m_VS_ResumeActive,
        Q => #m_VS_LowSpeedDone
    );

    // VS resume delay timer (TMR-04)
    #m_TON_VS_Hervatting(
        IN := (#m_Toestand = "DB_States".STATE_VS_PAUSE) AND NOT #i_VS_Trigger,
        PT := "DB_Config".Timers.CFG_VS_HervattingMs,
        Q => #m_VS_ResumeDelayDone
    );
```

### Example 2: Low Speed Output Determination

```scl
// After CASE statement, determine low speed output
// Low speed active when:
// 1. In HOMING (always low speed)
// 2. In PBV_RETRACT (always low speed)
// 3. In MOVING_* and start timer not complete
// 4. In MOVING_* after VS resume and VS timer not complete

VAR_TEMP
    t_InMovement : Bool;
    t_NormalLowSpeed : Bool;
    t_VS_LowSpeed : Bool;
END_VAR

#t_InMovement := (#m_Toestand = "DB_States".STATE_MOVING_OPEN) OR
                 (#m_Toestand = "DB_States".STATE_MOVING_DICHT);

// Normal start low speed: during movement, timer not complete
#t_NormalLowSpeed := #t_InMovement AND NOT #m_LowSpeedDone;

// VS resume low speed: after VS, timer not complete
#t_VS_LowSpeed := #t_InMovement AND #m_VS_ResumeActive AND NOT #m_VS_LowSpeedDone;

// Final low speed determination
#q_MS_LaagToeren := (#m_Toestand = "DB_States".STATE_HOMING) OR
                    (#m_Toestand = "DB_States".STATE_PBV_RETRACT) OR
                    #t_NormalLowSpeed OR
                    #t_VS_LowSpeed;
```

### Example 3: FC_Outputs Complete Implementation

```scl
FUNCTION "FC_Outputs" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.2

VAR_INPUT
    i_HuidigeToestand : Int;
    i_MS_Open : Bool;
    i_MS_Dicht : Bool;
    i_MS_LaagToeren : Bool;
    i_AlarmKerk : Bool;
END_VAR

VAR_OUTPUT
    q_MS_Open : Bool;          // %Q0.4
    q_MS_Dicht : Bool;         // %Q0.5
    q_MS_LaagToeren : Bool;    // %Q0.6
    q_AlarmKerk : Bool;        // %Q0.3
END_VAR

BEGIN
    // =========================================================================
    // OUT-05: Motor outputs mutually exclusive (defensive check)
    // =========================================================================
    // Even if FB sends invalid combination, FC blocks both (fail-safe)
    IF #i_MS_Open AND #i_MS_Dicht THEN
        // SAFETY: Both requested = bug, stop everything
        #q_MS_Open := FALSE;
        #q_MS_Dicht := FALSE;
        #q_MS_LaagToeren := FALSE;  // Also disable low speed
    ELSE
        // Normal operation
        #q_MS_Open := #i_MS_Open;
        #q_MS_Dicht := #i_MS_Dicht;
        #q_MS_LaagToeren := #i_MS_LaagToeren;
    END_IF;

    // =========================================================================
    // OUT-04: Alarm output (fail-safe: 0 = hek dicht = alarm active)
    // =========================================================================
    // FB already computes: TRUE when not in IDLE_DICHT
    // FC passes through - fail-safe is inherent (PLC off = output 0 = alarm on)
    #q_AlarmKerk := #i_AlarmKerk;

END_FUNCTION
```

### Example 4: VS_PAUSE State with Resume Delay

```scl
"DB_States".STATE_VS_PAUSE:
    // Safety checks FIRST (priority: FAULT > PBV > VS)
    IF #i_Fault THEN
        #m_Toestand := "DB_States".STATE_FAULT;
    ELSIF #i_PBV_Trigger THEN
        // PBV > VS: PBV can interrupt VS_PAUSE
        #m_Toestand := "DB_States".STATE_PBV_RETRACT;
    ELSIF NOT #i_VS_Trigger THEN
        // Beam cleared - wait for resume delay (TMR-04)
        IF #m_VS_ResumeDelayDone THEN
            // TMR-04 complete: resume movement
            #m_VS_ResumeActive := TRUE;  // Flag for VS low speed timer
            IF #m_VorigeRichtingVoorVS = "DB_States".DIR_OPEN THEN
                #m_Toestand := "DB_States".STATE_MOVING_OPEN;
            ELSIF #m_VorigeRichtingVoorVS = "DB_States".DIR_DICHT THEN
                #m_Toestand := "DB_States".STATE_MOVING_DICHT;
            ELSE
                #m_Toestand := "DB_States".STATE_STOPPED;
            END_IF;
        END_IF;
        // Note: If beam re-triggers during delay, timer resets (TON behavior)
    END_IF;
    // Operator input ignored (SAF-09)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual ET tracking | TONR timer | Available since S7-1200 V4 | Built-in pause/resume, more reliable |
| Interlocks in FB | Defensive check in FC | Best practice | Single responsibility, defense in depth |
| Timer in CASE | Timer before CASE | IEC 61131-3 pattern | Deterministic, timer always called |

**Deprecated/outdated:**
- S5 timer format (T#5s is correct, not S5T#5s)
- Putting timer instances in separate global DB (keep in FB for encapsulation)

## Open Questions

Things that couldn't be fully resolved:

1. **VS Resume During Delay Re-trigger**
   - What we know: FDS says beam re-trigger during delay restarts wait
   - What's unclear: Exact TON behavior when IN goes FALSE during timing
   - Recommendation: TON.IN FALSE resets ET to 0, which is correct behavior

2. **PBV_RETRACT Entry Pause Timer**
   - What we know: FDS says wait CFG_PBV_PauzeMs before starting retract
   - What's unclear: Current code starts retract immediately (Phase 3)
   - Recommendation: Add m_TON_PBV_Pauze in PBV_RETRACT state, sub-states needed

3. **Low Speed Timer Interaction with STOPPED**
   - What we know: Timer pauses on stop (TMR-08)
   - What's unclear: Should timer pause in VS_PAUSE too?
   - Recommendation: Yes, TONR IN should only be TRUE in MOVING states

## Sources

### Primary (HIGH confidence)
- Current codebase SCL files (analyzed directly)
- FDS-Automatisch-Hek-v0.2.md (FDS sections 2.4, 5.2, 5.3, 8)
- statemachine.mmd (canonical state transitions)
- 04-CONTEXT.md (user decisions)

### Secondary (MEDIUM confidence)
- [Siemens SCL IEC Timers](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-iec-timer-programming-tp-ton-tof-examples.php) - TON, TONR, TP, TOF behavior
- [PLC Timer Examples (TONR)](https://instrumentationtools.com/plc-program-for-iec-timers-ton-tof-tp-tonr-used-in-s7-1200/) - TONR parameters
- [Motor Interlock Theory](https://www.theengineeringprojects.com/2022/02/interlock-in-ladder-logic-programming.html) - Mutual exclusion patterns
- [Siemens Forum TONR](https://support.industry.siemens.com/forum/WW/en/posts/retentive-timer-in-s7-1200/46488) - TONR pause/resume behavior

### Tertiary (LOW confidence)
- [Automation Notes State Machine](https://abedgnu.github.io/Automation-Notes/chapters/PLC/Siemens/fsm.html) - State machine patterns in SCL

## Metadata

**Confidence breakdown:**
- Timer patterns (TONR for pause): HIGH - verified in Siemens documentation and forums
- Mutual exclusion pattern: HIGH - standard industrial practice, well documented
- Timer call location (before CASE): HIGH - IEC 61131-3 best practice, Siemens guides
- Interface design: MEDIUM - based on current codebase patterns, some discretion

**Research date:** 2026-02-06
**Valid until:** 2026-03-06 (stable domain, TIA Portal patterns well established)
