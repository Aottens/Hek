# Roadmap: Automatisch Hek Kerk

## Milestones

- [x] **v1.0 MVP** - Phases 1-4 (shipped 2026-02-06)
- [ ] **v1.1 FAT Documentation** - Phases 5-6 (in progress)

## Phases

<details>
<summary>v1.0 MVP (Phases 1-4) - SHIPPED 2026-02-06</summary>

See: .planning/milestones/v1.0-ROADMAP.md

- [x] **Phase 1: Foundation** - Block structure and configuration
- [x] **Phase 2: Safety Core** - Priority logic and debouncing
- [x] **Phase 3: State Machine** - 9 states with transitions
- [x] **Phase 4: Integration** - Timers and output mapping

</details>

### v1.1 FAT Documentation (In Progress)

**Milestone Goal:** Create comprehensive FAT document and update FDS to reflect implementation

- [ ] **Phase 5: FAT Document** - Comprehensive Factory Acceptance Test document
- [ ] **Phase 6: FDS Update** - Update FDS-Automatisch-Hek to v0.3

## Phase Details

### Phase 5: FAT Document
**Goal**: Tester can verify complete gate system using structured Markdown document
**Depends on**: Phase 4 (v1.0 complete)
**Requirements**: IO-01, IO-02, IO-03, STATE-01, STATE-02, STATE-03, CTL-01, CTL-02, CTL-03, CTL-04, CTL-05, CTL-06, CTL-07, TMR-01, TMR-02, TMR-03, TMR-04, TMR-05, TMR-06, TMR-07, TMR-08, EDGE-01, EDGE-02, EDGE-03, EDGE-04, EDGE-05, EDGE-06, EDGE-07, SAF-01, SAF-02, SAF-03, SAF-04, SAF-05, SAF-06, SAF-07, SAF-08, SAF-09, SAF-10, DOC-01, DOC-02, DOC-03, DOC-04, DOC-05
**Success Criteria** (what must be TRUE):
  1. Tester can verify each of the 7 in-scope inputs by toggling physical sensor and checking PLC
  2. Tester can verify each of the 4 in-scope outputs by checking physical actuator response
  3. Tester can walk through all 9 states with clear expected motor/alarm behavior
  4. Tester can verify all timer behaviors with documented trigger conditions and durations
  5. Tester can record Pass/Fail and notes in Remarks column for each test
**Plans**: 3 plans

Plans:
- [ ] 05-01-PLAN.md - Document structure with I/O and State Machine sections
- [ ] 05-02-PLAN.md - Control Behavior and Timer Behavior sections
- [ ] 05-03-PLAN.md - Edge Cases and Safety sections with traceability

### Phase 6: FDS Update
**Goal**: FDS v0.3 accurately reflects implemented behavior including edge cases
**Depends on**: Phase 5 (FAT identifies any gaps)
**Requirements**: FDS-01, FDS-02, FDS-03, FDS-04, FDS-05, FDS-06, FDS-07, FDS-08
**Success Criteria** (what must be TRUE):
  1. Movement timeout behavior documented in section 4.4
  2. Opposing key behavior and key release after timeout documented in section 4.3
  3. Debounce timer values documented in section 5
  4. FB_Veiligheid naming corrected in section 10
  5. Motor mutual exclusion documented in safety section
**Plans**: TBD

Plans:
- [ ] 06-01: TBD

## Progress

**Execution Order:** 5 -> 6

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Foundation | v1.0 | 2/2 | Complete | 2026-02-03 |
| 2. Safety Core | v1.0 | 2/2 | Complete | 2026-02-03 |
| 3. State Machine | v1.0 | 3/3 | Complete | 2026-02-03 |
| 4. Integration | v1.0 | 2/2 | Complete | 2026-02-06 |
| 5. FAT Document | v1.1 | 0/3 | Planned | - |
| 6. FDS Update | v1.1 | 0/? | Not started | - |

---
*Roadmap created: 2026-02-06*
*Last updated: 2026-02-06*
