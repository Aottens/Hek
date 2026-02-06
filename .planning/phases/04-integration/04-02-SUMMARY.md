# Phase 4 Plan 2: VS/PBV Timers and FC_Outputs Mutual Exclusion Summary

**Completed:** 2026-02-06
**Duration:** 3 minutes

## One-Liner

VS resume delay, PBV two-phase pause/retract, and FC_Outputs motor mutual exclusion for defense in depth.

## What Was Built

### Task 1: Update VS_PAUSE with PBV Transition Reset
Added `m_PBV_Retracting := FALSE` when transitioning from VS_PAUSE to PBV_RETRACT:
- Ensures PBV_RETRACT always starts in pause phase
- VS resume delay logic (m_VS_ResumeDelayDone) already in place from 04-01
- VS low speed flag (m_VS_ResumeActive) already set on resume from 04-01

### Task 2: PBV_RETRACT Two-Phase Logic
Implemented TMR-02 pause before retract:
- **PAUSE PHASE:** Motor off, wait CFG_PBV_PauzeMs (500ms) via m_PBV_PauseDone
- **RETRACT PHASE:** Motor on toward OPEN after pause completes
- `m_PBV_Retracting` flag tracks phase (FALSE=pause, TRUE=retract)
- Updated PBV retract timer to only run during retract phase
- Updated motor output CASE to handle pause phase (motor off)
- Reset flag on all exit paths (FAULT, endstop, timeout)

### Task 3: FC_Outputs Mutual Exclusion
Implemented OUT-05 defense in depth:
- If both i_MS_Open AND i_MS_Dicht true (bug), block BOTH outputs
- Also disables low speed when in error state
- Catches any FB logic bugs that could damage motor
- Version bumped to 0.2

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| PBV pause before retract (500ms) | Gives time for obstacle detection before motor starts |
| Motor off during PBV pause | Safety - no movement until pause timer complete |
| Mutual exclusion blocks BOTH directions | Fail-safe: unknown state -> stop motor (never pick one direction) |
| Low speed also blocked in mutual exclusion | Motor stopped anyway, keep all motor outputs consistent |

## Files Modified

| File | Changes |
|------|---------|
| PLC/Sources/04_FB_HekBesturing.scl | VS_PAUSE PBV transition reset, PBV_RETRACT two-phase logic, motor output update |
| PLC/Sources/06_FC_Outputs.scl | OUT-05 mutual exclusion check, version 0.2 |

## Commits

| Hash | Message |
|------|---------|
| 20322c8 | feat(04-02): update VS_PAUSE with m_PBV_Retracting reset on PBV transition |
| 3057829 | feat(04-02): implement PBV_RETRACT two-phase pause and retract logic |
| 95bfe21 | feat(04-02): add motor mutual exclusion safety check to FC_Outputs |

## Verification Results

All success criteria verified:

1. **VS_PAUSE waits for m_VS_ResumeDelayDone** - Check at line 360 before resume
2. **VS_PAUSE sets m_VS_ResumeActive flag** - Set at line 362 on resume transition
3. **PBV_RETRACT pause phase (motor off)** - m_PBV_Retracting=FALSE in pause, motor output conditional
4. **PBV_RETRACT motor only during retract** - IF m_PBV_Retracting at line 446
5. **FC_Outputs mutual exclusion** - IF #i_MS_Open AND #i_MS_Dicht at line 39, blocks both
6. **FC_Outputs version 0.2** - Version bumped at line 16

## Requirements Coverage

All Phase 4 timer and output requirements now complete:

| Requirement | Status | Implementation |
|-------------|--------|----------------|
| TMR-01 | Complete | TONR low speed 5s (04-01) |
| TMR-02 | Complete | TON PBV pause 500ms (04-02) |
| TMR-03 | Complete | TON PBV retract 3s (existing, updated 04-02) |
| TMR-04 | Complete | TON VS resume delay 2s (04-01, verified 04-02) |
| TMR-05 | Complete | TONR VS low speed 2s (04-01) |
| TMR-06 | Complete | TON movement timeout 90s (existing) |
| TMR-07 | Complete | TON endstop debounce (existing Phase 2) |
| TMR-08 | Complete | TONR pause/reset on direction change (04-01) |
| TMR-09 | Complete | Timers before CASE (04-01) |
| OUT-01 | Complete | Motor direction outputs (existing) |
| OUT-02 | Complete | Low speed output (04-01) |
| OUT-03 | Complete | Alarm output (existing) |
| OUT-04 | Complete | Alarm fail-safe (existing) |
| OUT-05 | Complete | Motor mutual exclusion (04-02) |
| OUT-06 | Complete | Output separation via FC (existing) |

## Deviations from Plan

None - plan executed exactly as written.

## Phase Completion

**Phase 4 Integration complete.** All timer sequences and safety outputs implemented.

**Project ready for:**
- Final integration testing
- Commissioning on target PLC

---

*Plan: 04-02*
*Phase: 04-integration*
*Completed: 2026-02-06*
