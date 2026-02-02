# Research Summary: AutomatischHek (Church Gate Control)

**Project:** TIA Portal v20 XML/SCL Export for S7-1212C Automatic Gate Control
**Synthesized:** 2026-02-02
**Overall Confidence:** MEDIUM-HIGH

---

## Executive Summary

This project requires generating TIA Portal v20-compatible code for an S7-1212C PLC controlling an automatic gate at a church. The research strongly recommends **using SCL External Source Files (.scl) rather than SimaticML XML**. SCL files are human-readable, directly importable via TIA Portal's "External Sources" feature, and don't require the Openness API. XML is more complex and primarily intended for automated tooling integration.

The gate control requires a **9-state state machine** with safety sensors (NC wiring for fail-safe operation), multiple TON timers for pause/retract sequences, and a fail-safe alarm output. The recommended architecture follows a clear separation: OB1 orchestrates calls to FC_Veiligheid (safety validation), FB_HekBesturing (state machine with instance DB), and FC_Outputs (state-to-output mapping). This pattern ensures testable, maintainable code with explicit data flow.

The most critical risks are **inverted safety logic errors** (NC sensors must be handled correctly or wire breaks create hazards) and **timer multiple-evaluation race conditions** unique to S7-1200. Both have clear prevention strategies that must be implemented from the start.

---

## Key Findings

### From STACK.md: Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| Code Format | `.scl` files | Human-readable, direct import, no API needed |
| Function Blocks | `.scl` with FB syntax | State retention for timers and state machine |
| Data Blocks | `.db` files | Configuration parameters, instance DBs |
| Import Method | External Sources > Generate blocks | Simple, documented, V20 supported |
| Block Optimization | Optimized Access (default) | Symbolic addressing, S7-1200 standard |

**SCL files are definitively the right choice.** XML adds complexity without benefit for this single-gate project.

### From FEATURES.md: Required SCL Constructs

**Must Implement:**
- State machine with CASE statement and INT constants (not ENUMs - limited S7-1200 support)
- TON timers for all delays (pause, retract, resume)
- R_TRIG edge detection for push button (one instance per signal)
- Safety priority logic (emergency stop overrides all states)
- Fail-safe output pattern (default FALSE, explicit TRUE only)

**Defer to V2:**
- Diagnostic logging
- Watchdog monitoring
- HMI-configurable parameters

### From ARCHITECTURE.md: Component Structure

```
OB1 [Main]
  |
  +-- FC_Veiligheid (safety validation, stateless)
  |     +-- PBV (pushbutton validation)
  |     +-- VS (sensor validation)
  |
  +-- FB_HekBesturing (state machine, stateful)
  |     +-- Instance DB: DB_HekData
  |     +-- Multi-instance timers in Static section
  |     +-- 9 states with CASE statement
  |
  +-- FC_Outputs (state-to-output mapping, stateless)
        +-- Motor open/close commands
        +-- Warning lamp
```

**Key Pattern:** Timers declared in FB Static section as multi-instance (share FB's instance DB). Timer calls OUTSIDE the CASE statement, controlled by state-derived enable flags.

### From PITFALLS.md: Critical Risks

| Pitfall | Impact | Prevention |
|---------|--------|------------|
| Timer Multiple-Evaluation | Gate fails to complete cycle | Capture timer.Q to BOOL once per scan, never reference multiple times |
| Inverted Safety Logic | Wire break undetected, safety failure | Document NC sensors, TRUE = path clear, test wire disconnection |
| Edge Detection Reuse | Button presses missed | One R_TRIG instance per signal, never in loops |
| CASE Without ELSE | State machine freezes | Always include ELSE clause, transition to FAULT state |
| XML ID Collisions | Import fails | Use global counter (if XML route taken) |

---

## Implications for Roadmap

### Suggested Phase Structure

**Phase 1: Project Setup and Block Structure**
- Create folder structure with `/source/` for SCL files
- Capture TIA Portal v20 version strings from test export (for future XML needs)
- Define state constants and interface contracts
- Create empty SCL file templates

*Rationale:* Foundation must be solid before logic. Establishes naming conventions and validates import procedure.

*Features:* Block templates (FB, FC, OB1, DB), import verification
*Pitfalls to avoid:* Optimized vs non-optimized mismatch, interface section structure

---

**Phase 2: Safety Logic (FC_Veiligheid)**
- Implement safety sensor validation
- NC sensor handling with correct inversion (TRUE = path clear)
- Emergency stop detection
- Fail-safe alarm output (inverted: 0 = active)

*Rationale:* Safety must be bulletproof before adding movement. Separating safety into FC allows testing in isolation.

*Features:* Safety priority logic, fail-safe outputs
*Pitfalls to avoid:* Inverted safety logic errors (test wire breaks, not just sensor activation)

**NEEDS PHASE RESEARCH:** Specific sensor wiring for this installation must be validated.

---

**Phase 3: State Machine Core (FB_HekBesturing)**
- Implement 9-state CASE statement with INT constants
- R_TRIG for push button edge detection
- State transition logic (HOMING, IDLE_OPEN, IDLE_DICHT, MOVING_OPEN, MOVING_DICHT, STOPPED, VS_PAUSE, PBV_RETRACT, FAULT)
- ELSE clause for undefined states -> FAULT

*Rationale:* Core control logic. Can be tested with simulated inputs before integrating real safety and timers.

*Features:* State machine, edge detection
*Pitfalls to avoid:* Edge detection instance reuse, CASE without ELSE, multiple BOOL flags for states

**Standard patterns available** - well-documented, minimal research needed.

---

**Phase 4: Timer Sequences**
- TON timers for VS_PAUSE, PBV_RETRACT, resume delays
- Timer calls outside CASE, controlled by state-derived enable flags
- Capture timer.Q to BOOL variable once per scan
- Configuration parameters in DB_Config

*Rationale:* Timers have S7-1200-specific quirks. Adding after state logic works allows testing transitions without timing complexity.

*Features:* TON timers, configurable timing parameters
*Pitfalls to avoid:* Timer multiple-evaluation race condition (critical for S7-1200)

**NEEDS PHASE RESEARCH:** Verify timer capture pattern works with specific TIA V20 behavior.

---

**Phase 5: Output Mapping and Integration (FC_Outputs + OB1)**
- FC_Outputs maps state to physical outputs
- Motor interlock (never both open and close active)
- Warning lamp logic
- OB1 orchestrates call sequence
- Integration testing with all components

*Rationale:* Final assembly. All components tested individually before integration.

*Features:* Output interlock, call sequence orchestration
*Pitfalls to avoid:* Direct I/O in FBs (use parameters), output interlock logic

---

**Phase 6: Validation and Documentation**
- Test all safety sensor failure modes (wire disconnection)
- Test emergency stop hardware independence
- Test gate reversal on obstacle during close
- Test alarm sounds on PLC power loss
- Document import procedure

*Rationale:* Safety verification cannot be skipped. Hardware testing validates software assumptions.

*Features:* Safety verification checklist
*Pitfalls to avoid:* Skipping wire-break tests, assuming simulation equals reality

---

### Research Flags

| Phase | Research Needed? | Notes |
|-------|------------------|-------|
| Phase 1: Setup | NO | Standard patterns, well-documented |
| Phase 2: Safety | YES | Site-specific sensor wiring must be validated |
| Phase 3: State Machine | NO | Standard IEC 61131-3 patterns |
| Phase 4: Timers | MAYBE | Verify S7-1200 timer quirks in V20 |
| Phase 5: Integration | NO | Standard orchestration patterns |
| Phase 6: Validation | NO | Checklist-based, no new patterns |

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack (SCL vs XML) | HIGH | Official Siemens docs confirm SCL import procedure |
| Features (SCL constructs) | HIGH | Official Siemens docs for TON, R_TRIG, CASE |
| Architecture | HIGH | Siemens programming guidelines for S7-1200/1500 |
| Pitfalls | MEDIUM-HIGH | Mix of official docs and verified community experience |

### Gaps to Address During Planning

1. **Site-specific sensor wiring** - NC/NO configuration for this specific installation not yet documented
2. **Exact 9-state definitions** - States are named but exact transitions need requirements definition
3. **Timer duration values** - Pause, retract, resume times need stakeholder input
4. **Alarm hardware** - Physical alarm behavior (continuous, intermittent) not specified

---

## Sources

### Official Siemens Documentation
- [TIA Portal V20 - External Source Files](https://docs.tia.siemens.cloud/r/en-us/v20/creating-and-managing-blocks/using-external-source-files-for-stl-and-scl/basics-of-using-external-source-files)
- [Siemens Programming Guideline S7-1200/1500](https://support.industry.siemens.com/cs/document/81318674)
- [Siemens Safety Programming Guideline](https://cache.industry.siemens.com/dl/files/255/109750255/att_1132641/v1/109750255_Programming-Guideline-Safety_DOC_V1_3_en.pdf)
- [Siemens TON Timer Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/timer-operations-s7-1200-s7-1500/ton-generate-on-delay-s7-1200-s7-1500)
- [Siemens R_TRIG Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/bit-logic-operations-s7-1200-s7-1500/r_trig-detect-positive-signal-edge-s7-1200-s7-1500)

### Technical Resources
- [DMC - Troubleshooting S7-1200 Timers](https://www.dmcinfo.com/latest-thinking/blog/id/8862/troubleshooting-your-siemens-simatic-s7-1200-timers)
- [Stefan Henneken - IEC 61131-3 State Pattern](https://stefanhenneken.net/2018/11/17/iec-61131-3-the-state-pattern/)
- [SolisPLC - TIA Portal Programming Tutorials](https://www.solisplc.com/tutorials)

### Safety Standards
- [UL 325 Gate Safety Standards](https://gatedepot.com/safety-sensors)
- [Rockwell Safety Guidelines](https://literature.rockwellautomation.com/idc/groups/literature/documents/at/safety-at061_-en-p.pdf)
