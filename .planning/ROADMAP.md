# Roadmap: Automatisch Hek Kerk

## Overview

This roadmap delivers TIA Portal v20 SCL code for an S7-1212C controlling an automatic sliding gate at a church. The journey moves from foundational block structure through isolated safety logic, then state machine implementation, culminating in timer integration and output mapping. Each phase builds on the previous, with safety logic verified before control logic is added.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3, 4): Planned milestone work
- Decimal phases (e.g., 2.1): Urgent insertions (marked with INSERTED)

- [ ] **Phase 1: Foundation** - Block structure, DB_Config, SCL file templates
- [ ] **Phase 2: Safety Core** - FC_Veiligheid, sensor processing, safety priority
- [ ] **Phase 3: State Machine** - FB_HekBesturing, 9-state logic, control inputs
- [ ] **Phase 4: Integration** - FC_Outputs, timer sequences, OB1 orchestration

## Phase Details

### Phase 1: Foundation
**Goal**: Establish SCL block structure and configuration parameters for TIA Portal import
**Depends on**: Nothing (first phase)
**Requirements**: CODE-01, CODE-02, CODE-03, CODE-04, CODE-05, CODE-06
**Success Criteria** (what must be TRUE):
  1. SCL files import into TIA Portal v20 without errors
  2. DB_Config contains all CFG_ parameters with documented defaults
  3. Block call structure (OB1 -> FC_Veiligheid -> FB_HekBesturing -> FC_Outputs) compiles
  4. State constants defined as INT values in FB_HekBesturing
**Plans**: 2 plans

Plans:
- [ ] 01-01-PLAN.md — Create DB_States and DB_Config data blocks
- [ ] 01-02-PLAN.md — Create FC_Veiligheid, FB_HekBesturing, FC_Outputs, Main_OB1

### Phase 2: Safety Core
**Goal**: Implement isolated safety logic that can stop the gate on any trigger
**Depends on**: Phase 1
**Requirements**: SAF-01, SAF-02, SAF-03, SAF-04, SAF-05, SAF-06, SAF-07, SAF-08, SAF-09, SAF-10, SAF-11, SAF-12, SEN-01, SEN-02, SEN-03, SEN-04
**Success Criteria** (what must be TRUE):
  1. PBV trigger (bumper contact) produces immediate stop signal with correct priority
  2. VS trigger (beam interruption) produces pause signal that auto-resumes after delay
  3. All NC sensors produce trigger when wire is disconnected (fail-safe verified)
  4. Safety priority order enforced: FAULT > PBV > VS
**Plans**: TBD

Plans:
- [ ] 02-01: TBD

### Phase 3: State Machine
**Goal**: Implement 9-state machine with all transitions and control inputs
**Depends on**: Phase 2
**Requirements**: SM-01, SM-02, SM-03, SM-04, SM-05, SM-06, SM-07, SM-08, SM-09, CTL-01, CTL-02, CTL-03, CTL-04, CTL-05, CTL-06, CTL-07, CTL-08
**Success Criteria** (what must be TRUE):
  1. Key switch OPEN/DICHT commands transition to correct movement states
  2. Push button toggles between STOPPED and movement states
  3. HOMING state activates on power-up with unknown position and moves toward DICHT
  4. FAULT state activates when both endstops active, auto-recovers when cleared
**Plans**: TBD

Plans:
- [ ] 03-01: TBD

### Phase 4: Integration
**Goal**: Complete system with timer sequences, output mapping, and full OB1 orchestration
**Depends on**: Phase 3
**Requirements**: TMR-01, TMR-02, TMR-03, TMR-04, TMR-05, TMR-06, TMR-07, TMR-08, TMR-09, OUT-01, OUT-02, OUT-03, OUT-04, OUT-05, OUT-06
**Success Criteria** (what must be TRUE):
  1. Motor outputs are mutually exclusive (never both q_MS_Open and q_MS_Dicht active)
  2. Low speed timer pauses on stop and resets on direction change
  3. Alarm output inverted correctly (q_AlarmKerk = 0 when gate closed)
  4. All timer sequences execute with configurable durations from DB_Config
**Plans**: TBD

Plans:
- [ ] 04-01: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 0/2 | Planned | - |
| 2. Safety Core | 0/? | Not started | - |
| 3. State Machine | 0/? | Not started | - |
| 4. Integration | 0/? | Not started | - |

---
*Roadmap created: 2026-02-02*
*Phase 1 planned: 2026-02-03*
*Coverage: 42/42 v1 requirements mapped*
