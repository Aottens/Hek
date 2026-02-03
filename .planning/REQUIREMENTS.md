# Requirements: Automatisch Hek Kerk

**Defined:** 2026-02-02
**Core Value:** Gate stops immediately on any safety trigger and never traps a person or vehicle

## v1 Requirements

### State Machine (SM)

- [ ] **SM-01**: Implement 9-state state machine per statemachine.mmd
- [ ] **SM-02**: HOMING state on power-up with unknown position (no endstop active)
- [ ] **SM-03**: HOMING moves toward DICHT at low speed
- [ ] **SM-04**: HOMING aborts to STOPPED on PBV or VS trigger
- [ ] **SM-05**: IDLE_OPEN when i_BNS_Open active (=0)
- [ ] **SM-06**: IDLE_DICHT when i_BNS_Dicht active (=0)
- [ ] **SM-07**: FAULT when both endstops active simultaneously
- [ ] **SM-08**: FAULT auto-recovers when sensors return to valid state
- [ ] **SM-09**: STOPPED state for manual stop or after PBV retract in mid-position

### Control (CTL)

- [ ] **CTL-01**: Key switch OPEN (i_SS_SleutelSchakelaarOpen) starts movement toward OPEN
- [ ] **CTL-02**: Key switch DICHT (i_SS_SleutelSchakelaarDicht) starts movement toward DICHT
- [ ] **CTL-03**: Key switch uses rising edge detection (no debounce)
- [ ] **CTL-04**: Push button (i_DK_Bediening) toggles stop/resume on falling edge (NC: 0 = pressed)
- [ ] **CTL-05**: Push button from STOPPED resumes last active direction
- [ ] **CTL-06**: Key switch from STOPPED starts in selected direction (overrides last direction)
- [ ] **CTL-07**: Opposing key direction while moving -> stop and wait for new input
- [ ] **CTL-08**: After timeout/stop, key must be released before new input accepted

### Safety (SAF)

- [ ] **SAF-01**: Priority order: FAULT > PBV > VS
- [ ] **SAF-02**: PBV trigger (i_PBV_BeweegbaarHek = 1) -> immediate stop
- [ ] **SAF-03**: PBV always retracts toward OPEN (regardless of movement direction)
- [ ] **SAF-04**: PBV retract at low speed for max CFG_PBV_TerugtrekMs or until endstop
- [ ] **SAF-05**: PBV ignores all operator input during retract
- [ ] **SAF-06**: PBV debounce: CFG_EndstopDebounceMs (50ms)
- [ ] **SAF-07**: VS trigger (i_VS_Voertuig = 0, NC sensor) -> immediate pause
- [ ] **SAF-08**: VS auto-resumes in same direction after beam clear + CFG_VS_HervattingMs
- [ ] **SAF-09**: VS ignores operator input during pause (auto-resume has priority)
- [ ] **SAF-10**: VS debounce: 100ms
- [ ] **SAF-11**: Movement timeout: CFG_BewegingTimeoutS (90s) -> STOPPED
- [ ] **SAF-12**: HOMING uses same timeout as normal movement

### Timers (TMR)

- [ ] **TMR-01**: CFG_LaagToerenMs (5000ms) - low speed duration at movement start
- [ ] **TMR-02**: CFG_PBV_PauzeMs (500ms) - pause before PBV retract
- [ ] **TMR-03**: CFG_PBV_TerugtrekMs (3000ms) - max PBV retract duration
- [ ] **TMR-04**: CFG_VS_HervattingMs (2000ms) - delay after beam clear before resume
- [ ] **TMR-05**: CFG_VS_LaagToerenMs (2000ms) - low speed duration after VS resume
- [ ] **TMR-06**: CFG_BewegingTimeoutS (90s) - movement timeout (seconds)
- [ ] **TMR-07**: CFG_EndstopDebounceMs (50ms) - endstop/PBV debounce
- [ ] **TMR-08**: Low speed timer pauses on stop, resets on direction change
- [ ] **TMR-09**: Timers called outside CASE statement, Q captured once per scan

### Sensors (SEN)

- [ ] **SEN-01**: Endstop BNS_Open debounce: CFG_EndstopDebounceMs
- [ ] **SEN-02**: Endstop BNS_Dicht debounce: CFG_EndstopDebounceMs
- [ ] **SEN-03**: All safety sensors NC (fail-safe: wire break = trigger)
- [ ] **SEN-04**: VS sensor NC: 0 = beam blocked, 1 = beam intact

### Outputs (OUT)

- [ ] **OUT-01**: q_MS_Open = motor direction OPEN
- [ ] **OUT-02**: q_MS_Dicht = motor direction DICHT
- [ ] **OUT-03**: q_MS_LaagToeren = low speed active
- [ ] **OUT-04**: q_AlarmKerk = 0 when gate closed (fail-safe: PLC off = alarm active)
- [ ] **OUT-05**: Motor outputs mutually exclusive (never both active)
- [ ] **OUT-06**: All outputs default FALSE, only enabled in specific states

### Code Structure (CODE)

- [ ] **CODE-01**: Main OB1 calls blocks in order: FC_Veiligheid -> FB_HekBesturing -> FC_Outputs
- [ ] **CODE-02**: FB_HekBesturing contains state machine with instance DB
- [ ] **CODE-03**: FC_Veiligheid handles safety priority and sensor processing
- [ ] **CODE-04**: FC_Outputs maps state to physical outputs
- [ ] **CODE-05**: DB_Config holds all CFG_ parameters
- [ ] **CODE-06**: Generate SCL external source files (.scl) for TIA Portal import

## v2 Requirements

### Future Features

- **FUT-01**: PBV vast hek integration (i_PBV_VastHek)
- **FUT-02**: Signaallamp sturing (q_SL_Open)
- **FUT-03**: Manual FAULT reset button
- **FUT-04**: Diagnostics/fault logging
- **FUT-05**: Movement time learning (auto-adjust timeout)

## Out of Scope

| Feature | Reason |
|---------|--------|
| i_PBV_VastHek | Not connected in v0.1 |
| q_SL_Open | Not connected |
| Manual FAULT reset | Auto-recovery sufficient for v1 |
| HMI integration | No HMI in scope |
| Remote control | Local operation only |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| SM-01 | Phase 3 | Pending |
| SM-02 | Phase 3 | Pending |
| SM-03 | Phase 3 | Pending |
| SM-04 | Phase 3 | Pending |
| SM-05 | Phase 3 | Pending |
| SM-06 | Phase 3 | Pending |
| SM-07 | Phase 3 | Pending |
| SM-08 | Phase 3 | Pending |
| SM-09 | Phase 3 | Pending |
| CTL-01 | Phase 3 | Pending |
| CTL-02 | Phase 3 | Pending |
| CTL-03 | Phase 3 | Pending |
| CTL-04 | Phase 3 | Pending |
| CTL-05 | Phase 3 | Pending |
| CTL-06 | Phase 3 | Pending |
| CTL-07 | Phase 3 | Pending |
| CTL-08 | Phase 3 | Pending |
| SAF-01 | Phase 2 | Pending |
| SAF-02 | Phase 2 | Pending |
| SAF-03 | Phase 2 | Pending |
| SAF-04 | Phase 2 | Pending |
| SAF-05 | Phase 2 | Pending |
| SAF-06 | Phase 2 | Pending |
| SAF-07 | Phase 2 | Pending |
| SAF-08 | Phase 2 | Pending |
| SAF-09 | Phase 2 | Pending |
| SAF-10 | Phase 2 | Pending |
| SAF-11 | Phase 2 | Pending |
| SAF-12 | Phase 2 | Pending |
| TMR-01 | Phase 4 | Pending |
| TMR-02 | Phase 4 | Pending |
| TMR-03 | Phase 4 | Pending |
| TMR-04 | Phase 4 | Pending |
| TMR-05 | Phase 4 | Pending |
| TMR-06 | Phase 4 | Pending |
| TMR-07 | Phase 4 | Pending |
| TMR-08 | Phase 4 | Pending |
| TMR-09 | Phase 4 | Pending |
| SEN-01 | Phase 2 | Pending |
| SEN-02 | Phase 2 | Pending |
| SEN-03 | Phase 2 | Pending |
| SEN-04 | Phase 2 | Pending |
| OUT-01 | Phase 4 | Pending |
| OUT-02 | Phase 4 | Pending |
| OUT-03 | Phase 4 | Pending |
| OUT-04 | Phase 4 | Pending |
| OUT-05 | Phase 4 | Pending |
| OUT-06 | Phase 4 | Pending |
| CODE-01 | Phase 1 | Complete |
| CODE-02 | Phase 1 | Complete |
| CODE-03 | Phase 1 | Complete |
| CODE-04 | Phase 1 | Complete |
| CODE-05 | Phase 1 | Complete |
| CODE-06 | Phase 1 | Complete |

**Coverage:**
- v1 requirements: 42 total
- Mapped to phases: 42
- Unmapped: 0

---
*Requirements defined: 2026-02-02*
*Last updated: 2026-02-03 after Phase 1 completion*
