---
phase: 01-foundation
plan: 02
subsystem: plc
tags: [scl, tia-portal, s7-1200, state-machine, safety]

# Dependency graph
requires:
  - phase: 01-01
    provides: DB_States and DB_Config global data blocks
provides:
  - FC_Veiligheid safety processing function stub
  - FB_HekBesturing state machine function block stub
  - DB_HekData instance data block
  - FC_Outputs output mapping function stub
  - Main_OB1 organization block with complete call chain
affects: [02-safety, 03-statemachine, 04-integration]

# Tech tracking
tech-stack:
  added: []
  patterns: [SCL block structure, OB1 call chain pattern, instance DB pattern]

key-files:
  created:
    - PLC/Sources/03_FC_Veiligheid.scl
    - PLC/Sources/04_FB_HekBesturing.scl
    - PLC/Sources/05_DB_HekData.scl
    - PLC/Sources/06_FC_Outputs.scl
    - PLC/Sources/07_Main_OB1.scl
  modified: []

key-decisions:
  - "Intermediate variables in OB1 for clean data flow between blocks"
  - "Physical I/O addresses in OB1 only (blocks use symbolic parameters)"
  - "Fail-safe default: alarm active until state machine sets hek dicht"

patterns-established:
  - "Call order pattern: FC_Veiligheid -> FB_HekBesturing -> FC_Outputs (CODE-01)"
  - "Naming convention: i_ (input), q_ (output), m_ (memory/static), t_ (temp)"
  - "Instance DB pattern: DATA_BLOCK with FB type reference"

# Metrics
duration: 2min
completed: 2026-02-03
---

# Phase 1 Plan 02: Block Structure Summary

**Complete SCL block scaffold with OB1 call chain: FC_Veiligheid -> FB_HekBesturing.DB_HekData -> FC_Outputs per CODE-01**

## Performance

- **Duration:** 2 min
- **Started:** 2026-02-03T16:39:26Z
- **Completed:** 2026-02-03T16:41:30Z
- **Tasks:** 2
- **Files created:** 5

## Accomplishments
- FC_Veiligheid stub with 4 safety inputs (PBV, VS, BNS_Open, BNS_Dicht) and 3 trigger outputs
- FB_HekBesturing stub with 8 inputs, 5 outputs, and 3 static variables for 9-state machine
- DB_HekData instance DB correctly referencing FB_HekBesturing
- FC_Outputs stub mapping state machine outputs to physical addresses
- Main_OB1 with complete call structure using named parameters and physical I/O addresses

## Task Commits

Each task was committed atomically:

1. **Task 1: Create FC_Veiligheid, FB_HekBesturing, and DB_HekData** - `90c32a7` (feat)
2. **Task 2: Create FC_Outputs and Main_OB1 with call structure** - `159a111` (feat)

## Files Created
- `PLC/Sources/03_FC_Veiligheid.scl` - Safety processing function stub (4 inputs, 3 outputs)
- `PLC/Sources/04_FB_HekBesturing.scl` - State machine function block stub (8 inputs, 5 outputs, 3 statics)
- `PLC/Sources/05_DB_HekData.scl` - Instance DB for FB_HekBesturing
- `PLC/Sources/06_FC_Outputs.scl` - Output mapping function stub (5 inputs, 4 outputs)
- `PLC/Sources/07_Main_OB1.scl` - Main program with CODE-01 call chain

## Decisions Made
- **Intermediate variables in OB1:** Used VAR_TEMP for passing data between blocks rather than global variables, keeping blocks modular and testable
- **Physical I/O only in OB1:** All physical addresses (%I, %Q) are in Main_OB1 only; blocks use symbolic parameters for portability
- **Fail-safe alarm default:** q_AlarmKerk defaults to TRUE (alarm active) until state machine confirms hek is dicht

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- All 7 SCL files exist in numbered order for TIA Portal import
- Compilation order correct: DBs (01, 02) -> FC (03) -> FB (04) -> IDB (05) -> FC (06) -> OB (07)
- Phase 1 Foundation complete: ready for Phase 2 Safety implementation
- FC_Veiligheid stub ready for debounce and fault detection logic
- FB_HekBesturing stub ready for 9-state machine implementation

---
*Phase: 01-foundation*
*Completed: 2026-02-03*
