# Phase 2: Safety Core - Context

**Gathered:** 2026-02-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Implement isolated safety logic in FC_Veiligheid that can stop the gate on any trigger. Process PBV (bumper), VS (beam), and FAULT conditions with correct priority ordering. The state machine transitions (Phase 3) will consume these safety signals.

</domain>

<decisions>
## Implementation Decisions

### PBV (bumper) behavior
- On contact: immediate stop + brief reverse to release pressure
- Reverse duration: configurable via DB_Config (default 0.5s)
- After reverse completes: stay stopped, requires operator input to resume
- No automatic resumption — bumper contact always needs manual intervention

### VS (beam) behavior
- Only active in closing direction — ignored when opening
- On interruption: immediate pause (stop motor)
- On clear: wait configurable delay (DB_Config), then auto-resume closing
- Timeout protection: if beam broken too long (configurable, e.g., 30s), transition to STOPPED state requiring operator input

### Safety priority & conflicts
- Priority order: FAULT > PBV > VS (as per FDS)
- When FAULT condition clears: go to HOMING state to re-establish position
- No diagnostic tracking needed — just process current state

### Fail-safe signal handling
- NC sensors: configurable debounce time via DB_Config to filter noise
- Output structure: combined SafetyStop signal + individual flags (PBV_Active, VS_Active, FAULT_Active) for debugging

### Claude's Discretion
- PBV repeated trigger handling (each vs fault after N triggers)
- Priority overlap behavior (PBV during VS pause)
- Operator override policy (key/button vs safety conditions)
- Wire break treatment (same as trigger vs distinct fault)
- Endstop edge vs level triggering

</decisions>

<specifics>
## Specific Ideas

- VS timeout to STOPPED prevents indefinite waiting when beam is permanently blocked
- PBV requires operator input because bumper contact means something was hit — should not auto-resume
- FAULT clears to HOMING because position may be unknown after fault condition

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-safety-core*
*Context gathered: 2026-02-03*
