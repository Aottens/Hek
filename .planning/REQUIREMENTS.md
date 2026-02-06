# Requirements: Automatisch Hek Kerk v1.1

**Defined:** 2026-02-06
**Core Value:** Gate stops immediately on any safety trigger and never traps a person or vehicle

## v1.1 Requirements

Requirements for FAT Documentation milestone.

### FAT-IO: I/O Verification

- [ ] **IO-01**: FAT document includes checkbox section for all 7 in-scope inputs
- [ ] **IO-02**: FAT document includes checkbox section for all 4 in-scope outputs
- [ ] **IO-03**: Each I/O entry has: tag, address, expected behavior, checkbox

### FAT-STATE: State Machine

- [ ] **STATE-01**: FAT document covers all 9 states with expected motor/alarm behavior
- [ ] **STATE-02**: FAT document covers all state transitions from statemachine.mmd
- [ ] **STATE-03**: Each state test has: Test ID, Description, Expected Result, Pass/Fail, Remarks

### FAT-CTL: Control Behavior

- [ ] **CTL-01**: FAT covers key switch OPEN command (momentary, rising edge)
- [ ] **CTL-02**: FAT covers key switch DICHT command (momentary, rising edge)
- [ ] **CTL-03**: FAT covers push button stop (falling edge, NC sensor)
- [ ] **CTL-04**: FAT covers push button resume from STOPPED (last direction)
- [ ] **CTL-05**: FAT covers key switch overrides direction from STOPPED
- [ ] **CTL-06**: FAT covers opposing key during movement → STOPPED (CTL-07 in code)
- [ ] **CTL-07**: FAT covers key release required after timeout (CTL-08 in code)

### FAT-TMR: Timer Behavior

- [ ] **TMR-01**: FAT covers start low speed timer (CFG_LaagToerenMs = 5s)
- [ ] **TMR-02**: FAT covers PBV pause timer (CFG_PBV_PauzeMs = 500ms)
- [ ] **TMR-03**: FAT covers PBV retract timer (CFG_PBV_TerugtrekMs = 3s)
- [ ] **TMR-04**: FAT covers VS resume delay (CFG_VS_HervattingMs = 2s)
- [ ] **TMR-05**: FAT covers VS low speed after resume (CFG_VS_LaagToerenMs = 2s)
- [ ] **TMR-06**: FAT covers movement timeout (CFG_BewegingTimeoutS = 90s)
- [ ] **TMR-07**: FAT covers timer pause on stop, resume on restart
- [ ] **TMR-08**: FAT covers timer reset on direction change or endstop

### FAT-EDGE: Edge Cases

- [ ] **EDGE-01**: FAT covers push button at IDLE (no action, no direction memory)
- [ ] **EDGE-02**: FAT covers PBV at IDLE_OPEN (stay, already safest)
- [ ] **EDGE-03**: FAT covers direction memory cleared at endstops
- [ ] **EDGE-04**: FAT covers push button during HOMING → STOPPED
- [ ] **EDGE-05**: FAT covers bediening ignored during PBV_RETRACT
- [ ] **EDGE-06**: FAT covers bediening ignored during VS_PAUSE
- [ ] **EDGE-07**: FAT covers PBV during VS_PAUSE (PBV wins)

### FAT-SAF: Safety Tests

- [ ] **SAF-01**: FAT covers PBV detection and 500ms pause + retract sequence
- [ ] **SAF-02**: FAT covers PBV always retracts toward OPEN
- [ ] **SAF-03**: FAT covers VS detection and auto-resume with delay + low speed
- [ ] **SAF-04**: FAT covers FAULT detection (both endstops active)
- [ ] **SAF-05**: FAT covers FAULT auto-recovery to correct state
- [ ] **SAF-06**: FAT covers priority: FAULT > PBV > VS
- [ ] **SAF-07**: FAT covers debounce behavior (50ms endstop/PBV, 100ms VS)
- [ ] **SAF-08**: FAT covers motor mutual exclusion (never both outputs)
- [ ] **SAF-09**: FAT covers alarm fail-safe (0 = active = hek dicht)
- [ ] **SAF-10**: FAT includes wire-break test for each NC sensor

### FAT-DOC: Documentation Quality

- [ ] **DOC-01**: FAT document is in Markdown format
- [ ] **DOC-02**: FAT document has consistent table structure throughout
- [ ] **DOC-03**: All tables have Remarks column for tester notes
- [ ] **DOC-04**: FAT document is printable/usable during hardware testing
- [ ] **DOC-05**: Test IDs are traceable to FDS sections or code comments

### FDS-UPDATE: Specification Update

- [ ] **FDS-01**: Update FDS section 4.4 with movement timeout behavior
- [ ] **FDS-02**: Update FDS section 4.3 with opposing key behavior (CTL-07)
- [ ] **FDS-03**: Update FDS section 4.3 with key release after timeout (CTL-08)
- [ ] **FDS-04**: Update FDS section 5 with debounce timer values
- [ ] **FDS-05**: Update FDS section 6.2 with push button abort during HOMING
- [ ] **FDS-06**: Update FDS section 10 to reflect FB_Veiligheid (not FC)
- [ ] **FDS-07**: Add motor mutual exclusion to FDS safety section
- [ ] **FDS-08**: Document all edge cases discovered in code review

## Out of Scope

| Feature | Reason |
|---------|--------|
| Automated test scripts | Manual FAT with checkbox document |
| TIA Portal test integration | FAT is paper-based verification |
| i_PBV_VastHek testing | Not connected in v0.1 hardware |
| q_SL_Open testing | Not connected |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| IO-01 | Phase 5 | Complete |
| IO-02 | Phase 5 | Complete |
| IO-03 | Phase 5 | Complete |
| STATE-01 | Phase 5 | Complete |
| STATE-02 | Phase 5 | Complete |
| STATE-03 | Phase 5 | Complete |
| CTL-01 | Phase 5 | Complete |
| CTL-02 | Phase 5 | Complete |
| CTL-03 | Phase 5 | Complete |
| CTL-04 | Phase 5 | Complete |
| CTL-05 | Phase 5 | Complete |
| CTL-06 | Phase 5 | Complete |
| CTL-07 | Phase 5 | Complete |
| TMR-01 | Phase 5 | Complete |
| TMR-02 | Phase 5 | Complete |
| TMR-03 | Phase 5 | Complete |
| TMR-04 | Phase 5 | Complete |
| TMR-05 | Phase 5 | Complete |
| TMR-06 | Phase 5 | Complete |
| TMR-07 | Phase 5 | Complete |
| TMR-08 | Phase 5 | Complete |
| EDGE-01 | Phase 5 | Complete |
| EDGE-02 | Phase 5 | Complete |
| EDGE-03 | Phase 5 | Complete |
| EDGE-04 | Phase 5 | Complete |
| EDGE-05 | Phase 5 | Complete |
| EDGE-06 | Phase 5 | Complete |
| EDGE-07 | Phase 5 | Complete |
| SAF-01 | Phase 5 | Complete |
| SAF-02 | Phase 5 | Complete |
| SAF-03 | Phase 5 | Complete |
| SAF-04 | Phase 5 | Complete |
| SAF-05 | Phase 5 | Complete |
| SAF-06 | Phase 5 | Complete |
| SAF-07 | Phase 5 | Complete |
| SAF-08 | Phase 5 | Complete |
| SAF-09 | Phase 5 | Complete |
| SAF-10 | Phase 5 | Complete |
| DOC-01 | Phase 5 | Complete |
| DOC-02 | Phase 5 | Complete |
| DOC-03 | Phase 5 | Complete |
| DOC-04 | Phase 5 | Complete |
| DOC-05 | Phase 5 | Complete |
| FDS-01 | Phase 6 | Pending |
| FDS-02 | Phase 6 | Pending |
| FDS-03 | Phase 6 | Pending |
| FDS-04 | Phase 6 | Pending |
| FDS-05 | Phase 6 | Pending |
| FDS-06 | Phase 6 | Pending |
| FDS-07 | Phase 6 | Pending |
| FDS-08 | Phase 6 | Pending |

**Coverage:**
- v1.1 requirements: 48 total
- Mapped to phases: 48
- Unmapped: 0

---
*Requirements defined: 2026-02-06*
*Last updated: 2026-02-06 (Phase 5 complete - 40 requirements verified)*
