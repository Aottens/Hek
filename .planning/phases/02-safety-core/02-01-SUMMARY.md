---
phase: 02-safety-core
plan: 01
subsystem: safety-processing
tags: [scl, tia-portal, siemens, s7-1200, function-block, debounce, ton-timer, safety]

# Dependency graph
requires:
  - 01-01 (DB_Config timer parameters)
  - 01-02 (FC_Veiligheid stub, DB_HekData, Main_OB1 structure)
provides:
  - FB_Veiligheid function block with TON timer instances
  - Debounced sensor outputs (PBV, VS, FAULT)
  - Timer state persistence between PLC scan cycles
affects: [02-02, 03-state-machine]

# Tech tracking
tech-stack:
  added:
    - IEC TON timer (on-delay timer for debounce)
  patterns:
    - Function Block (FB) for stateful logic with internal timers
    - Multi-instance FB call syntax via DB STRUCT
    - TON timer debounce pattern for NC sensors

key-files:
  created: []
  modified:
    - PLC/Sources/03_FC_Veiligheid.scl (converted FC to FB)
    - PLC/Sources/05_DB_HekData.scl (added m_Veiligheid instance)
    - PLC/Sources/07_Main_OB1.scl (updated FB call syntax)

key-decisions:
  - "FC_Veiligheid converted to FB_Veiligheid to enable internal timer instances"
  - "DB_HekData restructured as STRUCT containing FB instances (m_Veiligheid, m_HekBesturing)"
  - "TON timers used for all three debounce cases (PBV 50ms, VS 100ms, FAULT 50ms)"
  - "Priority logic deferred to Plan 02 - this plan provides raw debounced outputs"

patterns-established:
  - "FB call syntax: \"FB_Name\".\"DB_Name\".instance_name(...)"
  - "TON timer call: #timer(IN := signal, PT := \"DB_Config\".path); output := #timer.Q"
  - "NC sensor inversion: m_RawVS_Blocked := NOT #i_VS_Voertuig"

# Metrics
duration: 3min
completed: 2026-02-03
---

# Phase 2 Plan 01: FB Conversion and Timer Setup Summary

**FB_Veiligheid with TON timer instances for 50ms PBV/FAULT debounce and 100ms VS debounce using DB_Config parameters**

## Performance

- **Duration:** 3 min
- **Started:** 2026-02-03
- **Completed:** 2026-02-03
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments

- Converted FC_Veiligheid to FB_Veiligheid for timer state retention
- Implemented three TON timer instances: m_TON_PBV, m_TON_VS, m_TON_Fault
- Updated DB_HekData to combined STRUCT format with both FB instances
- Updated Main_OB1 with correct FB instance call syntax

## Task Commits

Each task was committed atomically:

1. **Task 1: Convert FC_Veiligheid to FB_Veiligheid with timer instances** - `198b4eb` (feat)
2. **Task 2: Update DB_HekData and Main_OB1 call structure** - `e1586f2` (feat)

## Files Created/Modified

- `PLC/Sources/03_FC_Veiligheid.scl` - Converted to FUNCTION_BLOCK with VAR section containing TON timers
- `PLC/Sources/05_DB_HekData.scl` - Changed to STRUCT with m_Veiligheid and m_HekBesturing instances
- `PLC/Sources/07_Main_OB1.scl` - Updated FB call syntax and comments

## Decisions Made

- **FC to FB conversion:** Required for timer state persistence between PLC cycles (FCs cannot hold timer instances)
- **Combined DB structure:** DB_HekData now contains both FB instances in a STRUCT (cleaner than separate DBs)
- **Raw debounced outputs:** This plan outputs debounced signals directly; priority masking deferred to Plan 02

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
| SAF-06 | PBV debounce 50ms | m_TON_PBV with CFG_EndstopDebounceMs |
| SAF-10 | VS debounce 100ms | m_TON_VS with CFG_VS_DebounceMs |
| SEN-01 | Endstop debounce | m_TON_Fault with CFG_EndstopDebounceMs |
| SEN-02 | FAULT requires stable signal | 50ms debounce on both-endstops-active condition |

## Next Phase Readiness

- FB_Veiligheid ready for priority logic implementation (Plan 02)
- Timer instances retain state between cycles (verified by TON in static VAR)
- Debounce times configurable via DB_Config
- Plan 02 (Priority Logic) can proceed immediately

---
*Phase: 02-safety-core*
*Completed: 2026-02-03*
