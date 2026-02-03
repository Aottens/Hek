# Phase 3 Plan 1: Core State Machine Structure Summary

**Completed:** 2026-02-03
**Duration:** 2 minutes

## One-Liner

FB_HekBesturing core structure with edge detection, timeout timer, first-scan initialization, and HOMING/IDLE/FAULT state logic.

## What Was Built

### Task 1: Static Variables for Edge Detection and Timers
Added required static variables to FB_HekBesturing:
- `m_EdgeSS_Open` (R_TRIG): Rising edge detection for key switch OPEN
- `m_EdgeSS_Dicht` (R_TRIG): Rising edge detection for key switch DICHT
- `m_EdgeDK` (F_TRIG): Falling edge detection for push button (NC sensor)
- `m_TON_Timeout` (TON): Movement timeout timer (90s)
- `m_VorigeRichtingVoorVS`: Direction before VS pause (for auto-resume)
- `m_KeyReleased`: Key release tracking after timeout (CTL-08)
- `m_FirstScan`: Power-on detection for initialization

### Task 2: Initialization and CASE Skeleton
Implemented complete code structure:
- Edge detection calls before CASE statement (ensures single detection per scan)
- Timeout timer running during HOMING, MOVING_OPEN, MOVING_DICHT states
- First-scan initialization determines initial state from endstop positions:
  - Both endstops active (NC=0) -> FAULT
  - BNS_Dicht active -> IDLE_DICHT
  - BNS_Open active -> IDLE_OPEN
  - Neither active -> HOMING
- CASE statement with all 9 states (4 implemented, 5 as placeholders)
- Motor output section based on state
- Alarm logic (q_AlarmKerk=FALSE only in IDLE_DICHT)

### Task 3: HOMING, IDLE, and FAULT State Logic

**STATE_HOMING:**
- Safety checks FIRST (priority: FAULT > PBV > VS)
- i_Fault -> FAULT state
- PBV or VS trigger -> STOPPED (SM-04: HOMING aborts on safety)
- Timeout elapsed -> STOPPED with m_KeyReleased := FALSE
- BNS_Dicht reached (NC=0) -> IDLE_DICHT
- Push button press -> STOPPED (manual abort)
- Motor: q_MS_Dicht=TRUE (moves toward DICHT)

**STATE_IDLE_OPEN:**
- Clears direction memory (no accidental resume toward wall)
- i_Fault -> FAULT state
- PBV trigger -> stay (already at safest position)
- Key DICHT edge -> MOVING_DICHT with direction set

**STATE_IDLE_DICHT:**
- Clears direction memory
- i_Fault -> FAULT state
- Key OPEN edge -> MOVING_OPEN with direction set

**STATE_FAULT:**
- Clears direction memory (safety decision)
- Motor forced off
- Auto-recovery when i_Fault clears (SM-08):
  - BNS_Dicht active -> IDLE_DICHT
  - BNS_Open active -> IDLE_OPEN
  - Neither -> STOPPED (safe default)

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| Direction memory cleared in FAULT | Safety: prevents accidental resume with stale direction after sensor fault |
| HOMING aborts to STOPPED on any safety trigger | FDS SM-04: HOMING is not critical enough to ignore safety |
| FAULT auto-recovers based on sensor state | Minimizes operator intervention while maintaining safety |
| Edge detection called before CASE | Ensures edges detected exactly once per scan cycle |

## Files Modified

| File | Changes |
|------|---------|
| PLC/Sources/04_FB_HekBesturing.scl | Added edge detection, timers, initialization, CASE skeleton, state logic |

## Commits

| Hash | Message |
|------|---------|
| 9415125 | feat(03-01): add static variables for edge detection and timers |
| bfe65a5 | feat(03-01): implement initialization and CASE skeleton with edge detection |
| f2a6c7b | feat(03-01): implement HOMING, IDLE, and FAULT state logic |

## Verification Results

All success criteria verified:

1. **FB_HekBesturing.scl compiles** - Valid SCL syntax, all constructs correct
2. **All 9 states in CASE** - HOMING, IDLE_OPEN, IDLE_DICHT, MOVING_OPEN, MOVING_DICHT, STOPPED, VS_PAUSE, PBV_RETRACT, FAULT
3. **Edge detection before CASE** - Lines 62-64 (R_TRIG/F_TRIG calls)
4. **Timeout timer for movement states** - Lines 69-72 (TON with OR condition)
5. **First scan initialization** - Lines 77-93 (endstop-based state selection)
6. **HOMING moves toward DICHT** - Line 208 (q_MS_Dicht=TRUE)
7. **FAULT auto-recovers** - Lines 183-192 (sensor-based recovery)
8. **q_AlarmKerk=FALSE only in IDLE_DICHT** - Line 221

## Deviations from Plan

None - plan executed exactly as written.

## Next Phase Readiness

**Ready for 03-02:** MOVING states, STOPPED state, control inputs, direction memory

**Dependencies satisfied:**
- Edge detection working (key switch and push button)
- Timeout timer configured
- Direction memory variable available (m_LaatsteRichting)
- IDLE states transition to MOVING states on key switch

---

*Plan: 03-01*
*Phase: 03-state-machine*
*Completed: 2026-02-03*
