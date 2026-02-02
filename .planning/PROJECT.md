# Automatisch Hek Kerk

## What This Is

TIA Portal v20 PLC code for a Siemens S7-1212C controlling an automatic sliding gate at a church. The gate is operated via a 3-position momentary key switch with push-button stop/resume, integrating rubber bumper and beam safety systems, and providing fail-safe church alarm signaling.

## Core Value

The gate must stop immediately on any safety trigger (bumper contact or beam interruption) and never trap a person or vehicle.

## Requirements

### Validated

(Validated through FDS v0.2 review)

- FDS-Automatisch-Hek-v0.2.md defines complete functional behavior
- IO-LIJST.md defines hardware mapping
- statemachine.mmd defines canonical state transitions

### Active

- [ ] Generate TIA Portal XML export for all program blocks
- [ ] FB_HekBesturing: 9-state machine with all transitions per statemachine.mmd
- [ ] FC_Veiligheid: PBV retract, VS pause, FAULT detection with priority FAULT > PBV > VS
- [ ] FC_Outputs: Motor direction, speed, alarm outputs based on state
- [ ] DB_Config: Configurable timers (CFG_PBV_PauzeMs, CFG_PBV_TerugtrekMs, CFG_VS_HervattingMs, CFG_VS_LaagToerenMs, CFG_LaagToerenMs)
- [ ] Main OB1: Cyclic call structure
- [ ] HOMING sequence on power-up with unknown position
- [ ] Low-speed timer that pauses on stops, resets on direction change

### Out of Scope

- PBV vast hek (i_PBV_VastHek) — not connected in v0.1
- Signaallamp (q_SL_Open) — not connected
- Reset storing knop — no input assigned
- Diagnostics/fault logging — future version

## Context

**Platform:**
- PLC: Siemens S7-1212C
- Engineering: TIA Portal V20
- Output format: XML export for direct import

**Existing documentation:**
- FDS-Automatisch-Hek-v0.2.md (canonical functional spec)
- IO-LIJST.md (I/O addresses and logic)
- statemachine.mmd (canonical state machine)
- PLAN.md (project overview, SSOT)

**Safety design:**
- All safety sensors NC (fail-safe)
- Alarm output inverted (0 = alarm active = gate closed)
- Priority: FAULT > PBV > VS

**I/O Summary:**
| Input | Address | Function |
|-------|---------|----------|
| i_SS_SleutelSchakelaarOpen | %I0.1 | Key to OPEN |
| i_DK_Bediening | %I0.2 | Push button (NC) |
| i_SS_SleutelSchakelaarDicht | %I0.3 | Key to DICHT |
| i_BNS_Open | %I0.7 | End position OPEN (NC) |
| i_BNS_Dicht | %I1.0 | End position DICHT (NC) |
| i_PBV_BeweegbaarHek | %I1.2 | Rubber bumper (NC) |
| i_VS_Voertuig | %I1.3 | Beam detection |

| Output | Address | Function |
|--------|---------|----------|
| q_AlarmKerk | %Q0.3 | Church alarm (0=ON) |
| q_MS_Open | %Q0.4 | Motor OPEN |
| q_MS_Dicht | %Q0.5 | Motor DICHT |
| q_MS_LaagToeren | %Q0.6 | Low speed |

## Constraints

- **Platform**: Siemens S7-1212C / TIA Portal V20 — existing hardware
- **Format**: XML export compatible with TIA Portal import — user requirement
- **Safety**: All NC sensors, fail-safe alarm logic — industry standard
- **States**: 9 states per statemachine.mmd — canonical source

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| XML export format | Direct import to TIA Portal without manual translation | — Pending |
| statemachine.mmd as canonical | Single source for state transitions, FDS describes behavior | — Pending |
| PBV always retracts toward OPEN | Bumper is on closing edge, safest direction is always open | — Pending |
| Timer pauses on stops | Position unknown during stop, timer should measure movement time only | — Pending |
| Automatic FAULT recovery | No manual reset needed, recovers when sensors return to valid state | — Pending |

---
*Last updated: 2026-02-02 after initialization*
