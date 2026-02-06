# Phase 4: Integration - Context

**Gathered:** 2026-02-06
**Status:** Ready for planning

<domain>
## Phase Boundary

Complete system with timer sequences, output mapping, and full OB1 orchestration. This phase adds:
- Timer logic for low speed, VS resume delay, PBV pause/retract
- FC_Outputs for physical output mapping with mutual exclusion
- Integration of all components through OB1

Requirements: TMR-01 through TMR-09, OUT-01 through OUT-06

</domain>

<decisions>
## Implementation Decisions

### Timer Architecture
- Timer location: Claude's discretion (choose cleanest architecture for TIA Portal)
- Timer pause/reset behavior: Match FDS exactly (TMR-08: pause on stop, reset on direction change)
- Timer call location: Outside CASE statement (TMR-09) — call at top of FB, capture .Q once per scan
- Timer count: Match FDS spec — implement all timers the FDS defines

### Output Location
- Motor outputs determined in: FC_Outputs (not FB_HekBesturing)
- FB_HekBesturing provides: Best practice / safest approach (Claude determines interface)
- Alarm output (q_AlarmKerk): Also in FC_Outputs — single place for all physical output mapping
- Mutual exclusion: Defensive check in FC_Outputs (even if FB sends invalid combination, FC blocks both)

### Low Speed Behavior
- Low speed start: Match FDS exactly (section 2.4)
- VS resume low speed: Match FDS exactly (CFG_VS_LaagToerenMs as separate timer)
- PBV_RETRACT low speed: Match FDS exactly (section 2.5.2)
- STOPPED state timer: Match TMR-08 exactly (pause timer, resume on push button)

### VS Resume Timing
- CFG_VS_HervattingMs start: Match FDS exactly (section 2.5.1)
- Beam re-trigger during delay: Match FDS exactly
- Resume transition: Match FDS/statemachine.mmd — use state diagram transitions
- Operator input during VS_PAUSE: Match FDS/CTL requirements

### Claude's Discretion
- Timer instance location (in FB or separate in DB_HekData)
- Interface between FB_HekBesturing and FC_Outputs (state+direction, intent, or flags)
- Implementation details within FDS constraints

</decisions>

<specifics>
## Specific Ideas

- FDS v0.2 and statemachine.mmd are canonical — implement behavior exactly as specified
- Defense in depth: FC_Outputs enforces mutual exclusion even if state machine is correct
- Single responsibility: All physical output mapping in FC_Outputs

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 04-integration*
*Context gathered: 2026-02-06*
