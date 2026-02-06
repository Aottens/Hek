# Phase 4 Plan 1: Timer Infrastructure and Low Speed Logic Summary

**Completed:** 2026-02-06
**Duration:** 3 minutes

## One-Liner

TONR pausable timers for low speed control with direction change reset, TON timers for VS/PBV delays, all called before CASE per TMR-09.

## What Was Built

### Task 1: Timer Variables and Direction Change Detection
Added timer infrastructure to FB_HekBesturing VAR section:
- `m_TONR_LaagToeren` (TONR): Pausable 5s low speed timer for normal start
- `m_TONR_VS_LaagToeren` (TONR): Pausable 2s low speed timer for post-VS resume
- `m_TON_VS_Hervatting` (TON): 2s VS resume delay timer (TMR-04)
- `m_TON_PBV_Pauze` (TON): 500ms PBV pause before retract (TMR-02)
- `m_VorigeRichting`: Direction tracking for change detection
- Timer output captures: `m_LowSpeedDone`, `m_VS_LowSpeedDone`, `m_VS_ResumeDelayDone`, `m_PBV_PauseDone`
- State tracking flags: `m_VS_ResumeActive`, `m_PBV_Retracting`

### Task 2: Timer Calls Before CASE Statement (TMR-09)
Added timer logic after edge detection, before state machine:
- VAR_TEMP for computed conditions: `t_DirectionChanged`, `t_MotorMoving`, `t_ReachedEndstop`
- Direction change detection: compares current vs previous direction, both non-NONE
- `m_TONR_LaagToeren` call with:
  - IN := motor in MOVING states (accumulates during movement)
  - PT := CFG_LaagToerenMs (5s)
  - R := direction change OR endstop reached (TMR-08)
- `m_TONR_VS_LaagToeren` call with:
  - IN := motor moving AND m_VS_ResumeActive
  - PT := CFG_VS_LaagToerenMs (2s)
  - R := direction change OR endstop OR VS resume inactive
- `m_TON_VS_Hervatting` call with:
  - IN := in VS_PAUSE AND beam cleared
  - PT := CFG_VS_HervattingMs (2s)
- `m_TON_PBV_Pauze` call with:
  - IN := in PBV_RETRACT AND not yet retracting
  - PT := CFG_PBV_PauzeMs (500ms)

### Task 3: Low Speed Output Determination
Implemented complete low speed logic after motor output CASE:
- HOMING: always low speed (FDS 2.2)
- PBV_RETRACT: always low speed (FDS 5.2)
- MOVING states: low speed for first 5s (until m_LowSpeedDone)
- After VS resume: additional 2s low speed (until m_VS_LowSpeedDone)
- Updated VS_PAUSE state to use resume delay timer (TMR-04)
- Set `m_VS_ResumeActive` flag on VS resume transition
- Clear `m_VS_ResumeActive` when VS low speed complete or at endstop

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| TONR for pausable timers | Built-in pause/resume behavior, reset via R input - matches TMR-08 exactly |
| Two separate low speed timers | Normal start (5s) and VS resume (2s) have different durations and triggers |
| Timers before CASE (TMR-09) | Deterministic behavior, timer called every scan, .Q stable for entire cycle |
| Direction change detection excludes DIR_NONE | Prevents false reset when direction set from idle state |
| m_VS_ResumeActive flag | Tracks VS resume context for separate low speed timer activation |

## Files Modified

| File | Changes |
|------|---------|
| PLC/Sources/04_FB_HekBesturing.scl | Added TONR/TON timers, direction change detection, low speed output logic, VS resume delay |

## Commits

| Hash | Message |
|------|---------|
| b758acb | feat(04-01): add timer variables and direction change detection |
| 42fa63a | feat(04-01): add timer calls before CASE statement (TMR-09) |
| 9728417 | feat(04-01): implement low speed output determination |

## Verification Results

All success criteria verified:

1. **TONR timers declared and called** - m_TONR_LaagToeren (line 55, 133), m_TONR_VS_LaagToeren (line 56, 143)
2. **TON timers declared and called** - m_TON_VS_Hervatting (line 59, 153), m_TON_PBV_Pauze (line 60, 162)
3. **Direction change detection** - t_DirectionChanged computed from m_VorigeRichting comparison (lines 119-121)
4. **Timers before CASE** - Timer calls at lines 133-167, CASE at line 197
5. **Low speed output** - HOMING always (line 457), PBV_RETRACT always (line 458), first 5s of movement, 2s after VS resume
6. **m_VS_ResumeActive flag** - Set in VS_PAUSE resume (line 362), cleared after timer (line 463)

## Deviations from Plan

None - plan executed exactly as written.

## Next Phase Readiness

**Ready for 04-02:** FC_Outputs mutual exclusion, PBV pause integration

**Dependencies satisfied:**
- All timers in place and called correctly
- Low speed output computed per TMR-01, TMR-05
- VS resume delay implemented (TMR-04)
- PBV pause timer ready (TMR-02)
- Direction change detection working (TMR-08)

---

*Plan: 04-01*
*Phase: 04-integration*
*Completed: 2026-02-06*
