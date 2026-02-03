# Phase 3: State Machine - Research

**Researched:** 2026-02-03
**Domain:** Siemens SCL state machine implementation for PLC gate control
**Confidence:** HIGH

## Summary

Phase 3 implements a 9-state machine in FB_HekBesturing that controls an automatic gate. The state machine must handle operator inputs (key switch, push button), respond to safety triggers from FB_Veiligheid (PBV, VS, FAULT), manage movement timeouts, and track direction for resume functionality.

The existing FB_HekBesturing is currently a stub with inputs/outputs defined but no implementation. The state constants (0-8) and direction constants (DIR_NONE, DIR_OPEN, DIR_DICHT) are already defined in DB_States. FB_Veiligheid already provides debounced, priority-masked safety triggers (q_PBV_Trigger, q_VS_Trigger, q_Fault).

**Primary recommendation:** Implement state machine using CASE statement with edge detection for inputs, TON timer for movement timeout, and direction memory for push button resume. Split into 3 plans: (1) Core state machine structure with initialization, (2) Control inputs and operator logic, (3) Safety responses and timeout handling.

## Standard Stack

The established libraries/tools for this domain:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| IEC CASE statement | SCL native | State machine dispatch | Clean, readable, efficient for discrete states |
| R_TRIG / F_TRIG | IEC library | Edge detection | Standard IEC function blocks, persistent state |
| TON | IEC library | Movement timeout | Already used in FB_Veiligheid for debounce |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| DB_States | Project | State/direction constants | All state comparisons and assignments |
| DB_Config | Project | Timer parameters | CFG_BewegingTimeoutS for timeout |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| CASE statement | IF-ELSEIF chain | CASE more readable for 9 states |
| R_TRIG/F_TRIG FBs | Edge memory with XOR | FBs are standard, cleaner, persist state |
| Single FB | Split into multiple FBs | Single FB simpler, state machine is coherent unit |

**No external libraries needed:** All functionality available in IEC standard and TIA Portal base system.

## Architecture Patterns

### Recommended FB_HekBesturing Structure
```
FB_HekBesturing
    VAR_INPUT (existing)
        i_SS_Open, i_SS_Dicht     // Key switch inputs
        i_DK_Bediening            // Push button (NC: 0=pressed)
        i_BNS_Open, i_BNS_Dicht   // Endstop sensors
        i_PBV_Trigger             // From FB_Veiligheid
        i_VS_Trigger              // From FB_Veiligheid
        i_Fault                   // From FB_Veiligheid

    VAR_OUTPUT (existing)
        q_MS_Open, q_MS_Dicht     // Motor direction
        q_MS_LaagToeren           // Low speed (Phase 4)
        q_AlarmKerk               // Church alarm
        q_HuidigeToestand         // Diagnostic output

    VAR (static - to be added)
        m_Toestand                // Current state (existing)
        m_VorigeToestand          // Previous state (existing)
        m_LaatsteRichting         // Last direction (existing)
        m_VorigeRichtingVoorVS    // Direction before VS pause
        m_EdgeSS_Open : R_TRIG    // Rising edge key OPEN
        m_EdgeSS_Dicht : R_TRIG   // Rising edge key DICHT
        m_EdgeDK : F_TRIG         // Falling edge push button (NC)
        m_TON_Timeout : TON       // Movement timeout (90s)
        m_KeyReleased : Bool      // CTL-08: Key released after timeout
```

### Pattern 1: CASE-Based State Machine
**What:** Single CASE statement dispatching on m_Toestand
**When to use:** Always for state machines with discrete states
**Example:**
```scl
// Source: Siemens SCL programming standard
CASE #m_Toestand OF
    "DB_States".STATE_HOMING:
        // HOMING logic

    "DB_States".STATE_IDLE_OPEN:
        // IDLE_OPEN logic

    "DB_States".STATE_IDLE_DICHT:
        // IDLE_DICHT logic

    // ... remaining states

    ELSE
        // Unexpected state - recover to safe state
        #m_Toestand := "DB_States".STATE_STOPPED;
END_CASE;
```

### Pattern 2: Edge Detection Before CASE
**What:** Call all edge detection triggers before CASE statement
**When to use:** Always - ensures edges detected once per scan regardless of state
**Example:**
```scl
// Edge detection - call BEFORE CASE statement
#m_EdgeSS_Open(CLK := #i_SS_Open);
#m_EdgeSS_Dicht(CLK := #i_SS_Dicht);
#m_EdgeDK(CLK := #i_DK_Bediening);

// Timer update - call BEFORE CASE statement
#m_TON_Timeout(IN := <running>, PT := "DB_Config".Timers.CFG_BewegingTimeoutS);

CASE #m_Toestand OF
    // Use #m_EdgeSS_Open.Q, #m_EdgeSS_Dicht.Q, #m_EdgeDK.Q
    // Use #m_TON_Timeout.Q for timeout detection
END_CASE;
```

### Pattern 3: Safety-First Check in Movement States
**What:** Check safety triggers before any other logic in movement states
**When to use:** MOVING_OPEN, MOVING_DICHT, HOMING, PBV_RETRACT
**Example:**
```scl
"DB_States".STATE_MOVING_OPEN:
    // Safety checks FIRST (priority: FAULT > PBV > VS)
    IF #i_Fault THEN
        #m_Toestand := "DB_States".STATE_FAULT;
    ELSIF #i_PBV_Trigger THEN
        #m_Toestand := "DB_States".STATE_PBV_RETRACT;
    ELSIF #i_VS_Trigger THEN
        #m_VorigeRichtingVoorVS := "DB_States".DIR_OPEN;
        #m_Toestand := "DB_States".STATE_VS_PAUSE;
    ELSIF <timeout> THEN
        #m_Toestand := "DB_States".STATE_STOPPED;
        #m_KeyReleased := FALSE;  // CTL-08: require key release
    ELSIF <endstop reached> THEN
        #m_Toestand := "DB_States".STATE_IDLE_OPEN;
    ELSIF <push button edge> THEN
        #m_Toestand := "DB_States".STATE_STOPPED;
    END_IF;
```

### Pattern 4: Initialization at Block Start
**What:** Determine initial state based on endstop sensors on first scan
**When to use:** Power-on / first scan detection
**Example:**
```scl
// At block start, before CASE
IF #m_Toestand = 0 AND #m_VorigeToestand = 0 THEN
    // First scan - determine initial state based on sensors
    IF (NOT #i_BNS_Open) AND (NOT #i_BNS_Dicht) THEN
        #m_Toestand := "DB_States".STATE_FAULT;
    ELSIF NOT #i_BNS_Open THEN
        #m_Toestand := "DB_States".STATE_IDLE_OPEN;
    ELSIF NOT #i_BNS_Dicht THEN
        #m_Toestand := "DB_States".STATE_IDLE_DICHT;
    ELSE
        #m_Toestand := "DB_States".STATE_HOMING;
    END_IF;
END_IF;
```

### Anti-Patterns to Avoid
- **Nested CASE:** Do not nest CASE statements for sub-states; use separate variables if needed
- **Output assignment in CASE:** Assign outputs AFTER CASE for cleaner code (or use dedicated output section)
- **Edge detection inside CASE:** Call R_TRIG/F_TRIG once before CASE, not inside state logic
- **Timer reset in multiple places:** Timer IN parameter should be a single expression, not scattered assignments

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Edge detection | XOR with memory variable | R_TRIG / F_TRIG | Standard IEC, handles all edge cases |
| Movement timeout | Counter with comparison | TON timer | Consistent with existing debounce pattern |
| State comparison | Magic numbers (IF state = 3) | DB_States constants | Readable, maintainable, single source of truth |
| Direction storage | Multiple booleans | INT with DIR_NONE/OPEN/DICHT | Clean, already defined in DB_States |

**Key insight:** All timing and edge detection needs are covered by IEC standard function blocks already present in TIA Portal. Do not implement custom versions.

## Common Pitfalls

### Pitfall 1: Edge Detection Called Multiple Times
**What goes wrong:** R_TRIG/F_TRIG called inside CASE branches means edge may be detected in wrong state or not at all
**Why it happens:** Intuition says "only check key in states where key matters"
**How to avoid:** Call ALL edge triggers once before CASE, use .Q result in state logic
**Warning signs:** Key press sometimes ignored, or triggers twice

### Pitfall 2: Timer Not Running During Intended Period
**What goes wrong:** Movement timeout timer not ticking during actual movement
**Why it happens:** Timer IN parameter tied to wrong condition or reset prematurely
**How to avoid:** Timer IN := TRUE in movement states, captured timer.Q checked for timeout
**Warning signs:** Movement never times out, or times out immediately

### Pitfall 3: Direction Memory Not Cleared Appropriately
**What goes wrong:** Push button resumes in wrong direction after FAULT or after reaching endstop
**Why it happens:** Direction memory persists across state transitions that should reset it
**How to avoid:** Clear m_LaatsteRichting when entering IDLE states (reached endstop) and on FAULT recovery
**Warning signs:** Push button resumes toward wall (wrong direction)

### Pitfall 4: NC Sensor Logic Inverted
**What goes wrong:** State machine responds backwards to endstops or push button
**Why it happens:** NC sensors have inverted logic (0 = active)
**How to avoid:** Document NC logic clearly, use `NOT i_BNS_xxx` for "endstop reached" checks
**Warning signs:** Gate moves toward closed endstop when OPEN requested

### Pitfall 5: Safety Triggers Not Checked in All Movement States
**What goes wrong:** PBV or VS ignored in some states
**Why it happens:** Copy-paste error or forgetting to add safety check in new state
**How to avoid:** Template all movement states with safety-first pattern
**Warning signs:** Gate continues moving when safety should have triggered

### Pitfall 6: Initialization Race with FB_Veiligheid
**What goes wrong:** Initial state determined before FB_Veiligheid debounce completes
**Why it happens:** First scan logic runs before debounce timers settle
**How to avoid:** Initial state determination uses raw sensor inputs (already debounced upstream or accept immediate check)
**Warning signs:** Starts in wrong state on power-up, especially with intermittent sensors

## Code Examples

Verified patterns from project context and IEC standards:

### Edge Detection Declaration
```scl
// Source: IEC 61131-3 standard function blocks
VAR
    m_EdgeSS_Open : R_TRIG;    // Rising edge for key switch OPEN
    m_EdgeSS_Dicht : R_TRIG;   // Rising edge for key switch DICHT
    m_EdgeDK : F_TRIG;         // Falling edge for push button (NC: 0=pressed)
END_VAR
```

### Edge Detection Usage
```scl
// Call edge detection BEFORE CASE - once per scan
#m_EdgeSS_Open(CLK := #i_SS_Open);
#m_EdgeSS_Dicht(CLK := #i_SS_Dicht);
#m_EdgeDK(CLK := #i_DK_Bediening);  // NC: falling edge = press

// In CASE branches, use .Q
IF #m_EdgeSS_Open.Q THEN
    // Key OPEN rising edge detected
END_IF;
```

### Timeout Timer Pattern
```scl
// Movement states that need timeout: HOMING, MOVING_OPEN, MOVING_DICHT
// Timer runs when in movement state
VAR
    m_TON_Timeout : TON;
    m_MovementActive : Bool;  // Helper for timer IN
END_VAR

// Before CASE - determine if movement active
#m_MovementActive := (#m_Toestand = "DB_States".STATE_HOMING) OR
                     (#m_Toestand = "DB_States".STATE_MOVING_OPEN) OR
                     (#m_Toestand = "DB_States".STATE_MOVING_DICHT);

#m_TON_Timeout(IN := #m_MovementActive, PT := "DB_Config".Timers.CFG_BewegingTimeoutS);

// Inside movement states - check for timeout
IF #m_TON_Timeout.Q THEN
    #m_Toestand := "DB_States".STATE_STOPPED;
    #m_KeyReleased := FALSE;  // CTL-08
END_IF;
```

### NC Endstop Check Pattern
```scl
// NC sensors: 0 = contact closed = endstop reached
// Source: IO-LIJST.md and FDS

// Check if at OPEN endstop
IF NOT #i_BNS_Open THEN  // NC: 0 = reached
    // At OPEN position
END_IF;

// Check if at DICHT endstop
IF NOT #i_BNS_Dicht THEN  // NC: 0 = reached
    // At DICHT position
END_IF;

// FAULT check (both active - impossible condition)
IF (NOT #i_BNS_Open) AND (NOT #i_BNS_Dicht) THEN
    // Sensor fault - already handled by FB_Veiligheid, but check i_Fault instead
END_IF;
```

### Motor Output Assignment Pattern
```scl
// Assign outputs based on current state AFTER CASE
// Phase 3 scope: direction only, low speed in Phase 4

CASE #m_Toestand OF
    "DB_States".STATE_MOVING_OPEN, "DB_States".STATE_PBV_RETRACT:
        #q_MS_Open := TRUE;
        #q_MS_Dicht := FALSE;

    "DB_States".STATE_MOVING_DICHT, "DB_States".STATE_HOMING:
        #q_MS_Open := FALSE;
        #q_MS_Dicht := TRUE;

    ELSE:
        #q_MS_Open := FALSE;
        #q_MS_Dicht := FALSE;
END_CASE;

// Alarm: only active (0) when in IDLE_DICHT
#q_AlarmKerk := NOT (#m_Toestand = "DB_States".STATE_IDLE_DICHT);
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Magic state numbers | Named constants in DB | Project design | Readable, maintainable |
| Debounce in state machine | Debounce in FB_Veiligheid | Phase 2 | Clean separation of concerns |
| FC for stateful logic | FB with instance DB | Phase 2 | Timers persist between scans |

**Deprecated/outdated:**
- Direct I/O access in FB: Use symbolic parameters, physical mapping in OB1 only
- Combined safety+state logic: Keep FB_Veiligheid separate from FB_HekBesturing

## Open Questions

Things that couldn't be fully resolved:

1. **VS_PAUSE direction during PBV_RETRACT**
   - What we know: PBV > VS in priority, so VS ignored during PBV_RETRACT
   - What's unclear: If VS triggers after PBV_RETRACT starts, should we save direction?
   - Recommendation: No - PBV_RETRACT exits to STOPPED, VS irrelevant. CONTEXT.md says "Claude's discretion"

2. **Push button with no direction memory (fresh power-up, never moved)**
   - What we know: m_LaatsteRichting starts as DIR_NONE (0)
   - What's unclear: What happens if push button pressed in STOPPED with no direction?
   - Recommendation: Ignore push button if m_LaatsteRichting = DIR_NONE (safest - no movement without explicit direction)

3. **Simultaneous key OPEN + key DICHT**
   - What we know: Both inputs could theoretically be 1 in same scan
   - What's unclear: Physical mechanism (3-position switch should prevent this)
   - Recommendation: Check OPEN first, DICHT second; first edge wins. Log as unusual if both edges detected.

## Planning Questions Answered

### What states need internal sub-states?
**None.** The 9 states in statemachine.mmd are sufficient:
- PBV_RETRACT handles both pause-before-retract and retract (timer-based in Phase 4)
- VS_PAUSE handles wait-for-clear and wait-for-resume (timer-based in Phase 4)
- No internal sub-states needed at Phase 3 scope (state machine structure)

### How many plans are needed?
**Recommended: 3 plans**

1. **Plan 03-01: Core Structure and Initialization**
   - Add static variables (edge detectors, timer, direction memory)
   - Implement power-on initialization logic
   - Implement CASE skeleton with FAULT checking in all states
   - Implement HOMING state (moves toward DICHT, safety aborts to STOPPED)
   - Implement IDLE_OPEN and IDLE_DICHT states
   - Implement basic FAULT state with auto-recovery

2. **Plan 03-02: Control Inputs and Movement States**
   - Implement MOVING_OPEN and MOVING_DICHT states
   - Key switch edge detection for starting movement
   - Push button edge detection for stop/resume
   - STOPPED state with resume logic
   - Direction memory tracking
   - CTL-07: Opposing key stops movement
   - CTL-08: Key release requirement after timeout

3. **Plan 03-03: Safety Response States**
   - Implement VS_PAUSE state (pause, auto-resume direction)
   - Implement PBV_RETRACT state (always toward OPEN)
   - Movement timeout integration (SAF-11, SAF-12)
   - State machine output assignments (motor direction, alarm)
   - Final verification against all requirements

### What are the dependencies between state implementations?
```
Plan 03-01: Foundation
    |
    +-- HOMING (uses initialization, FAULT check)
    +-- IDLE states (landing states for HOMING, movement)
    +-- FAULT (must exist for all safety checks)
    |
    v
Plan 03-02: Movement (depends on IDLE, FAULT from 03-01)
    |
    +-- MOVING_OPEN/DICHT (transitions to IDLE on endstop)
    +-- STOPPED (transition target for push button)
    +-- Direction memory (used by push button resume)
    |
    v
Plan 03-03: Safety (depends on MOVING, STOPPED from 03-02)
    |
    +-- VS_PAUSE (needs direction memory from movement)
    +-- PBV_RETRACT (transitions from movement states)
    +-- Timeout (applies to movement states)
```

### What needs to be tested to verify the phase?

**Core verification (matches FAT tests from FDS):**
1. Power-on with no endstop active -> HOMING -> IDLE_DICHT when reached
2. Power-on at OPEN endstop -> IDLE_OPEN
3. Power-on at DICHT endstop -> IDLE_DICHT
4. Power-on with both endstops -> FAULT
5. Key OPEN from IDLE_DICHT -> MOVING_OPEN -> IDLE_OPEN
6. Key DICHT from IDLE_OPEN -> MOVING_DICHT -> IDLE_DICHT
7. Push button during movement -> STOPPED
8. Push button from STOPPED -> resumes last direction
9. Key from STOPPED -> starts selected direction (overrides last)
10. PBV during MOVING_DICHT -> PBV_RETRACT (toward OPEN)
11. PBV during MOVING_OPEN -> PBV_RETRACT (continues toward OPEN)
12. VS during movement -> VS_PAUSE
13. VS cleared -> auto-resume in same direction
14. Movement timeout (90s) -> STOPPED
15. FAULT condition -> FAULT state -> auto-recovers when cleared
16. Opposing key during movement -> STOPPED
17. Key held after timeout -> no action until released and pressed again

## Sources

### Primary (HIGH confidence)
- Existing codebase: FB_HekBesturing.scl (stub structure)
- Existing codebase: DB_States.scl (state constants)
- Existing codebase: FB_Veiligheid.scl (safety trigger interface)
- Existing codebase: Main_OB1.scl (call structure)
- statemachine.mmd (canonical state machine definition)
- FDS-Automatisch-Hek-v0.2.md (detailed behavior specifications)
- 03-CONTEXT.md (user decisions)

### Secondary (MEDIUM confidence)
- IEC 61131-3 standard for R_TRIG, F_TRIG, TON behavior
- Siemens SCL programming patterns (CASE statement syntax)
- Phase 2 research and implementation (established patterns)

### Tertiary (LOW confidence)
- None - all findings verified against existing codebase and specifications

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - IEC standard, already used in project
- Architecture: HIGH - Based on existing FB_HekBesturing structure and FDS
- Pitfalls: HIGH - Derived from requirements analysis and state machine review

**Research date:** 2026-02-03
**Valid until:** Project completion (stable requirements, no external dependencies)
