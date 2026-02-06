# Automatisch Hek Kerk

## What This Is

TIA Portal v20 PLC code for a Siemens S7-1212C controlling an automatic sliding gate at a church. Complete 9-state control system with priority-enforced safety (FAULT > PBV > VS), configurable timers, and motor mutual exclusion. Operated via 3-position momentary key switch with push-button stop/resume.

## Core Value

The gate must stop immediately on any safety trigger (bumper contact or beam interruption) and never trap a person or vehicle.

## Current State

**Shipped:** v1.0 MVP (2026-02-06)
**LOC:** 894 lines SCL across 7 files
**Tech stack:** Siemens S7-1212C, TIA Portal V20, SCL

**What was built:**
- 9-state machine (HOMING, IDLE_OPEN, IDLE_DICHT, MOVING_OPEN, MOVING_DICHT, STOPPED, VS_PAUSE, PBV_RETRACT, FAULT)
- Safety system with priority (FAULT > PBV > VS) and debounced sensors
- Configurable timers (low speed, PBV pause/retract, VS resume delay)
- Motor mutual exclusion in FC_Outputs

## Requirements

### Validated

- 9-state machine per statemachine.mmd -- v1.0
- Key switch OPEN/DICHT commands with rising edge detection -- v1.0
- Push button toggle stop/resume on falling edge -- v1.0
- HOMING on power-up, FAULT auto-recovery -- v1.0
- Priority FAULT > PBV > VS with debounce -- v1.0
- PBV retract: 500ms pause then timed retract toward OPEN -- v1.0
- VS auto-resume after beam clear + delay with low speed -- v1.0
- Motor mutual exclusion (OUT-05) -- v1.0
- All 42 v1 requirements satisfied -- v1.0

### Active

(None - awaiting next milestone definition)

### Out of Scope

- PBV vast hek (i_PBV_VastHek) -- not connected in v0.1
- Signaallamp (q_SL_Open) -- not connected
- Reset storing knop -- no input assigned
- Diagnostics/fault logging -- future version
- TIA Portal XML export -- SCL import sufficient

## Context

**Platform:**
- PLC: Siemens S7-1212C
- Engineering: TIA Portal V20
- Output format: SCL external source files (.scl)

**Existing documentation:**
- FDS-Automatisch-Hek-v0.2.md (canonical functional spec)
- IO-LIJST.md (I/O addresses and logic)
- statemachine.mmd (canonical state machine)

**Safety design:**
- All safety sensors NC (fail-safe)
- Alarm output inverted (0 = alarm active = gate closed)
- Priority: FAULT > PBV > VS

## Constraints

- **Platform**: Siemens S7-1212C / TIA Portal V20 -- existing hardware
- **Format**: SCL external source files -- direct import
- **Safety**: All NC sensors, fail-safe alarm logic -- industry standard
- **States**: 9 states per statemachine.mmd -- canonical source

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| SCL external source files | Human-readable, direct TIA Portal import | Good |
| statemachine.mmd as canonical | Single source for state transitions | Good |
| PBV always retracts toward OPEN | Bumper on closing edge, safest direction | Good |
| Timer pauses on stops | Position unknown during stop | Good |
| Automatic FAULT recovery | No manual reset needed | Good |
| Priority masking with AND NOT | Clear hierarchy, no double-counting | Good |
| TONR for pausable timers | Built-in pause/resume | Good |
| PBV pause before retract (500ms) | Obstacle detection time | Good |
| Motor mutual exclusion in FC | Defense in depth | Good |

---
*Last updated: 2026-02-06 after v1.0 milestone*
