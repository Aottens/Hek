# Phase 3 Plan 3: Safety Response States Summary

**Completed:** 2026-02-03
**Duration:** 2 minutes

## One-Liner

VS_PAUSE and PBV_RETRACT safety response states with auto-resume from saved direction, PBV retract timer (3s), operator input blocking (SAF-05/SAF-09), and complete 9-state machine.

## What Was Built

### Task 1: Add PBV Retract Timer Variable

Added TON timer for PBV retract duration control:
- **m_TON_PBV_Retract : TON** in VAR section
- Timer call before CASE statement: runs only in PBV_RETRACT state
- Uses CFG_PBV_TerugtrekMs (3s default) for maximum retract duration

### Task 2: Implement PBV_RETRACT State

Complete STATE_PBV_RETRACT implementation:
- **SAF-03:** Always moves toward OPEN (motor output q_MS_Open=TRUE handled in output section)
- **SAF-04:** Timer limits retract duration - if endstop not reached within CFG_PBV_TerugtrekMs, transitions to STOPPED
- **SAF-05:** Ignores ALL operator input (no key/push button checks in state)
- **FAULT > PBV:** Only FAULT can interrupt PBV_RETRACT
- **Direction memory cleared** on exit to STOPPED (operator must choose new direction)

Transitions:
- i_Fault -> STATE_FAULT
- NOT i_BNS_Open (NC: reached OPEN endstop) -> STATE_IDLE_OPEN
- m_TON_PBV_Retract.Q (timer elapsed) -> STATE_STOPPED with m_LaatsteRichting := DIR_NONE

### Task 3: Implement VS_PAUSE State

Complete STATE_VS_PAUSE implementation:
- **SAF-08:** Auto-resume uses m_VorigeRichtingVoorVS (saved when entering from MOVING states)
- **SAF-09:** Ignores ALL operator input (auto-resume has priority over key/push button)
- **PBV > VS:** PBV can interrupt VS_PAUSE (statemachine.mmd line 49)
- **Safety fallback:** If no saved direction (edge case), transitions to STOPPED

Transitions:
- i_Fault -> STATE_FAULT
- i_PBV_Trigger -> STATE_PBV_RETRACT
- NOT i_VS_Trigger (beam clear) -> MOVING_OPEN or MOVING_DICHT based on m_VorigeRichtingVoorVS

Note: Phase 4 will add CFG_VS_HervattingMs delay before resume. Phase 3 resumes immediately when beam clears.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| PBV_RETRACT clears direction memory on timeout exit | Safety: Operator must explicitly choose new direction after safety event |
| VS_PAUSE immediate resume | Phase 3 simplification - Phase 4 adds configurable delay timer |
| VS_PAUSE fallback to STOPPED if no saved direction | Safety: Edge case protection for unexpected state |
| Only FAULT interrupts PBV_RETRACT | PBV already highest safety response (FAULT > PBV in priority) |

## Files Modified

| File | Changes |
|------|---------|
| PLC/Sources/04_FB_HekBesturing.scl | m_TON_PBV_Retract timer, VS_PAUSE state, PBV_RETRACT state |

## Commits

| Hash | Message |
|------|---------|
| c7a39f6 | feat(03-03): add PBV retract timer variable |
| 20ffe4b | feat(03-03): implement PBV_RETRACT state |
| b7bac3a | feat(03-03): implement VS_PAUSE state |

## Verification Results

All success criteria verified:

1. **All 9 states fully implemented** - No more placeholders
   - STATE_HOMING (0), STATE_IDLE_OPEN (1), STATE_IDLE_DICHT (2)
   - STATE_MOVING_OPEN (3), STATE_MOVING_DICHT (4), STATE_STOPPED (5)
   - STATE_VS_PAUSE (6), STATE_PBV_RETRACT (7), STATE_FAULT (8)

2. **PBV_RETRACT always moves toward OPEN** - Output section line 335: q_MS_Open=TRUE

3. **PBV_RETRACT timer variable added and called** - Line 52 (VAR), line 78 (call)

4. **PBV_RETRACT ignores operator input (SAF-05)** - No key/push button checks in state

5. **VS_PAUSE saves and restores direction** - m_VorigeRichtingVoorVS saved in MOVING states, restored in VS_PAUSE

6. **VS_PAUSE ignores operator input (SAF-09)** - No key/push button checks in state

7. **VS_PAUSE allows PBV interrupt** - PBV > VS priority maintained (line 267-269)

8. **Motor outputs mutually exclusive** - CASE statement ensures only one direction active

9. **Alarm output correct** - FALSE only in IDLE_DICHT (line 352)

## Deviations from Plan

None - plan executed exactly as written.

## Phase 3 Complete

**State Machine Phase Finished:**
- All 9 states implemented
- All transitions per statemachine.mmd
- Safety priority (FAULT > PBV > VS) enforced throughout
- Motor and alarm outputs correct for all states

**Ready for Phase 4: Integration**
- Timer enhancements (VS resume delay, low speed near endstops)
- OB1 integration
- Final testing

---

*Plan: 03-03*
*Phase: 03-state-machine*
*Completed: 2026-02-03*
