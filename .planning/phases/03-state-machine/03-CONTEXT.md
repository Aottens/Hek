# Phase 3: State Machine - Context

**Gathered:** 2026-02-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Implement 9-state machine with all transitions, control inputs, and safety response behaviors. The state machine receives operator commands (key switch, push button), responds to safety triggers from FB_Veiligheid (PBV retract, VS pause), and manages transitions between HOMING, IDLE_OPEN, IDLE_DICHT, OPENING, CLOSING, STOPPED, and FAULT states.

Timer sequences (low speed, pause durations) and output mapping are Phase 4 scope.

</domain>

<decisions>
## Implementation Decisions

### State Transition Behavior
- Key switch is edge-triggered only — holding key does nothing extra, gate responds to rising edge
- Push button during HOMING aborts to STOPPED (manual stop works in all movement states)
- After movement timeout (90s), require key release before accepting new input (CTL-08 intent)

### Safety Response Behavior
- PBV at IDLE_OPEN: stay at endstop, no retract attempt (already at safest position)
- After PBV retract completes: STOPPED if mid-position, IDLE_OPEN if retract reached endstop
- VS direction tracking: resume based on which movement state was active (OPENING/CLOSING)
- VS during HOMING: follow FDS SM-04 — abort to STOPPED (no auto-resume for HOMING)

### Direction Memory
- Direction memory is volatile — resets on power cycle (matches HOMING behavior)
- Memory cleared vs preserved across PBV/FAULT: Claude decides based on safety considerations

### FAULT Recovery
- When FAULT clears: always transition to STOPPED (safe default, operator decides next action)
- Direction memory on FAULT: Claude decides (likely clear for safety)
- FAULT during PBV retract: Claude decides based on priority (FAULT > PBV)
- Safety triggers during FAULT: Claude decides (FAULT is highest priority, gate already stopped)

### Claude's Discretion
- Simultaneous conflicting inputs (key OPEN + key DICHT same scan): pick safest approach
- Post-timeout key acceptance timing details
- No-direction push button behavior (fresh power-up): pick safest default
- Direction memory persistence across PBV retract
- Direction memory clearing on FAULT
- Safety trigger handling while in FAULT state
- FAULT interrupting PBV retract

</decisions>

<specifics>
## Specific Ideas

- Edge detection must be clean — no repeated triggers from held keys
- Push button abort in HOMING gives operator control in unexpected situations
- STOPPED as post-FAULT state is conservative and predictable
- Volatile direction memory simplifies reasoning about power-cycle behavior

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 03-state-machine*
*Context gathered: 2026-02-03*
