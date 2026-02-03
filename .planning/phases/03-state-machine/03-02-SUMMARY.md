# Phase 3 Plan 2: Movement and Control States Summary

**Completed:** 2026-02-03
**Duration:** 2 minutes

## One-Liner

MOVING_OPEN, MOVING_DICHT, and STOPPED states with CTL-04/05/06/07/08 control logic - push button stop/resume, key switch direction override, and timeout key release tracking.

## What Was Built

### Task 1: MOVING_OPEN State with All Transitions

Implemented STATE_MOVING_OPEN with complete transition logic:
- **Safety priority:** FAULT > PBV > VS (checked first)
- **i_Fault** -> STATE_FAULT
- **i_PBV_Trigger** -> STATE_PBV_RETRACT (continues toward OPEN)
- **i_VS_Trigger** -> STATE_VS_PAUSE with direction saved (m_VorigeRichtingVoorVS)
- **m_TON_Timeout.Q** -> STATE_STOPPED with m_KeyReleased := FALSE (CTL-08)
- **i_BNS_Open=0** (NC: reached) -> STATE_IDLE_OPEN
- **m_EdgeDK.Q** (push button) -> STATE_STOPPED
- **m_EdgeSS_Dicht.Q** (opposing key, CTL-07) -> STATE_STOPPED

### Task 2: MOVING_DICHT State with All Transitions

Implemented STATE_MOVING_DICHT mirroring MOVING_OPEN structure:
- **Safety priority:** FAULT > PBV > VS (checked first)
- **i_Fault** -> STATE_FAULT
- **i_PBV_Trigger** -> STATE_PBV_RETRACT (SAF-03: always retract toward OPEN)
- **i_VS_Trigger** -> STATE_VS_PAUSE with direction saved
- **m_TON_Timeout.Q** -> STATE_STOPPED with m_KeyReleased := FALSE (CTL-08)
- **i_BNS_Dicht=0** (NC: reached) -> STATE_IDLE_DICHT
- **m_EdgeDK.Q** (push button) -> STATE_STOPPED
- **m_EdgeSS_Open.Q** (opposing key, CTL-07) -> STATE_STOPPED

### Task 3: STOPPED State with Resume Logic and CTL-08

Implemented STATE_STOPPED with complete control input handling:
- **i_Fault** -> STATE_FAULT (always checked)
- **CTL-08 Key Release Tracking:**
  - If m_KeyReleased=FALSE (after timeout), wait for both keys released
  - When i_SS_Open=0 AND i_SS_Dicht=0, set m_KeyReleased := TRUE
  - Ignore all input while waiting for release
- **CTL-06 Key Switch Start:**
  - Key OPEN edge -> MOVING_OPEN with m_LaatsteRichting := DIR_OPEN
  - Key DICHT edge -> MOVING_DICHT with m_LaatsteRichting := DIR_DICHT
- **CTL-04/05 Push Button Resume:**
  - Push button edge + m_LaatsteRichting=DIR_OPEN -> MOVING_OPEN
  - Push button edge + m_LaatsteRichting=DIR_DICHT -> MOVING_DICHT
  - Push button edge + m_LaatsteRichting=DIR_NONE -> ignored (safety)

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| Opposing key stops immediately | CTL-07: User changing direction intention while moving should stop first (safety) |
| Push button with no direction ignored | Safety: No movement without explicit direction from fresh power-up |
| Key release required after timeout | CTL-08: Prevents stuck key from immediately restarting movement |
| Direction memory preserved in MOVING states | Set when entering MOVING from IDLE, maintained through push button stop |

## Files Modified

| File | Changes |
|------|---------|
| PLC/Sources/04_FB_HekBesturing.scl | MOVING_OPEN, MOVING_DICHT, STOPPED states fully implemented |

## Commits

| Hash | Message |
|------|---------|
| 17bc074 | feat(03-02): implement MOVING_OPEN state with all transitions |
| 1666f95 | feat(03-02): implement MOVING_DICHT state with all transitions |
| 8ccd5d9 | feat(03-02): implement STOPPED state with resume logic and CTL-08 |

## Verification Results

All success criteria verified:

1. **MOVING_OPEN and MOVING_DICHT states fully implemented** - Lines 156-212
2. **STOPPED state with complete resume logic** - Lines 214-249
3. **Direction memory (m_LaatsteRichting) set on key switch entry** - Lines 137-138, 151-152, 234, 237
4. **Direction memory preserved across push button stop** - No modification when entering STOPPED via push button
5. **CTL-07: Opposing key stops movement** - Lines 178-180, 207-209
6. **CTL-08: Key release tracking after timeout** - Lines 171, 200, 223-227
7. **All safety checks (FAULT/PBV/VS) present in movement states** - Lines 159-167, 188-196
8. **Push button with no direction memory is safely ignored** - Lines 239-246 (DIR_NONE case)

## Deviations from Plan

None - plan executed exactly as written.

## Next Phase Readiness

**Ready for 03-03:** VS_PAUSE and PBV_RETRACT safety states

**Dependencies satisfied:**
- Direction saved before VS_PAUSE (m_VorigeRichtingVoorVS)
- PBV_RETRACT entered from both MOVING states
- Motor outputs already handle PBV_RETRACT (q_MS_Open=TRUE)
- All control inputs edge-detected

---

*Plan: 03-02*
*Phase: 03-state-machine*
*Completed: 2026-02-03*
