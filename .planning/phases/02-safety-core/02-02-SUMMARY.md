---
phase: 02-safety-core
plan: 02
subsystem: safety-processing
tags: [scl, tia-portal, siemens, s7-1200, priority-logic, safety-stop, debug-flags]

# Dependency graph
requires:
  - 02-01 (FB_Veiligheid with TON timers and debounced outputs)
provides:
  - Priority-masked safety outputs (FAULT > PBV > VS)
  - Combined q_SafetyStop signal for state machine consumption
  - Debug flags for troubleshooting raw debounced states
affects: [03-state-machine]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Priority masking with cascaded AND NOT conditions
    - Dual output pattern (priority-masked + raw debug)
    - Combined safety OR output for simplified consumption

key-files:
  created: []
  modified:
    - PLC/Sources/03_FC_Veiligheid.scl (priority logic and new outputs)
    - PLC/Sources/07_Main_OB1.scl (capture additional safety outputs)

key-decisions:
  - "Priority order FAULT > PBV > VS with AND NOT masking"
  - "Debug flags bypass priority for raw debounced values"
  - "SafetyStop is OR of priority-masked triggers, not raw"

patterns-established:
  - "Priority masking: q_X := condition AND NOT q_higher_priority"
  - "Debug outputs named q_Debug_* for raw unmasked state"
  - "Combined output for simplified consumer logic"

# Metrics
duration: 1min
completed: 2026-02-03
---

# Phase 2 Plan 02: Priority Logic and SafetyStop Summary

**Priority-enforced safety outputs (FAULT > PBV > VS) with combined SafetyStop signal and debug flags for state machine consumption and troubleshooting**

## Performance

- **Duration:** 1 min
- **Started:** 2026-02-03T17:52:09Z
- **Completed:** 2026-02-03T17:53:06Z
- **Tasks:** 3
- **Files modified:** 2

## Accomplishments

- Implemented priority masking: FAULT masks PBV, PBV masks VS
- Added combined q_SafetyStop output (TRUE when any safety active)
- Added three debug flags showing raw debounced states before priority masking
- Updated Main_OB1 to capture all new FB_Veiligheid outputs

## Task Commits

Each task was committed atomically:

1. **Task 1: Add combined SafetyStop output and debug flags** - `3287b86` (feat)
2. **Task 2: Implement priority logic per SAF-01** - `b1346de` (feat)
3. **Task 3: Update Main_OB1 to receive new outputs** - `e1273d4` (feat)

## Files Created/Modified

- `PLC/Sources/03_FC_Veiligheid.scl` - Added q_SafetyStop, q_Debug_* outputs; implemented priority masking logic
- `PLC/Sources/07_Main_OB1.scl` - Added VAR_TEMP for new outputs; updated FB_Veiligheid call to capture all 7 outputs

## Decisions Made

- **Priority masking cascade:** FAULT evaluated first, then PBV masked by FAULT, then VS masked by both. This ensures clear diagnostic hierarchy.
- **Debug vs production outputs:** Debug flags (q_Debug_*) show raw debounced state for troubleshooting; production outputs (q_*_Trigger, q_Fault) are priority-masked for correct behavior.
- **SafetyStop uses masked values:** Combined OR uses priority-masked triggers, not raw values, ensuring no double-counting.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None

## User Setup Required

None - no external service configuration required.

## Requirements Satisfied

This plan satisfies the following FDS requirements:

| ID | Requirement | Implementation |
|----|-------------|----------------|
| SAF-01 | Priority FAULT > PBV > VS | AND NOT masking in priority order |
| SAF-02 | PBV immediate stop | q_SafetyStop TRUE when q_PBV_Trigger active |
| SAF-07 | VS immediate pause | q_SafetyStop TRUE when q_VS_Trigger active |

## Next Phase Readiness

- FB_Veiligheid outputs ready for consumption by FB_HekBesturing state machine
- q_SafetyStop provides single "motor must stop" signal for simple transition logic
- Debug flags available for HMI or diagnostic purposes
- Phase 3 (State Machine) can proceed immediately

---
*Phase: 02-safety-core*
*Completed: 2026-02-03*
