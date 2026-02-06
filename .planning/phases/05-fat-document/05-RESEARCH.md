# Phase 5: FAT Document - Research

**Researched:** 2026-02-06
**Domain:** Factory Acceptance Test documentation for PLC-based gate control system
**Confidence:** HIGH

## Summary

This phase produces a comprehensive Factory Acceptance Test (FAT) document for the Automatisch Hek v1.0 PLC system. The document must enable a tester to systematically verify all 7 inputs, 4 outputs, 9 states, and 40+ requirements using structured tables with Pass/Fail checkboxes and Remarks columns.

The FAT document is a **paper-based verification tool** that will be used during on-site hardware testing. It must be structured for sequential workflow (I/O first, then states, then features), with clear expected behaviors derived from the FDS and actual v1.0 SCL code. Each test must be traceable to requirement IDs (IO-01 through DOC-05) and FDS sections.

**Primary recommendation:** Create a single Markdown document with seven major sections (Header/Info, I/O Verification, State Machine, Control Behavior, Timer Behavior, Edge Cases, Safety Tests), each using consistent table format with Test ID, Description, Expected Result, Pass/Fail, Remarks columns.

## Standard Stack

The FAT document is a Markdown text document - no libraries or code required.

### Core
| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| Markdown | CommonMark | Document format | DOC-01 requirement, GitHub rendering, printable |
| Markdown Tables | GFM | Test tables | Consistent structure, easy to edit |

### Supporting
| Tool | Purpose | When to Use |
|------|---------|-------------|
| VS Code | Editing | Primary editor for .md files |
| Markdown Preview | Verification | Check rendering before commit |
| Print CSS | Printing | Browser print for paper FAT |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Markdown | Excel | Excel loses version control, harder to diff |
| Markdown | Word | Word loses version control, binary format |
| Markdown | PDF | PDF not editable, need source anyway |

## Architecture Patterns

### Recommended Document Structure
```
FAT-Automatisch-Hek.md
|
+-- Header Section
|   +-- Document info (version, date, project)
|   +-- Test personnel signatures
|   +-- Equipment identification
|
+-- I/O Verification Section
|   +-- Input Tests (7 in-scope inputs)
|   +-- Output Tests (4 in-scope outputs)
|
+-- State Machine Section
|   +-- All 9 states with motor/alarm behavior
|   +-- State transitions from statemachine.mmd
|
+-- Control Behavior Section
|   +-- Key switch tests (CTL-01, CTL-02, CTL-05, CTL-06, CTL-07)
|   +-- Push button tests (CTL-03, CTL-04)
|
+-- Timer Behavior Section
|   +-- All 8 timer tests (TMR-01 through TMR-08)
|
+-- Edge Case Section
|   +-- All 7 edge cases (EDGE-01 through EDGE-07)
|
+-- Safety Section
|   +-- All 10 safety tests (SAF-01 through SAF-10)
|   +-- Wire-break tests
```

### Pattern 1: Standard Test Table Format
**What:** Consistent 5-column table for all test sections
**When to use:** Every test table throughout the document
**Example:**
```markdown
| Test ID | Description | Expected Result | Pass/Fail | Remarks |
|---------|-------------|-----------------|-----------|---------|
| IO-INP-01 | Toggle i_SS_Open (%I0.1) | PLC shows 1 when key turned to OPEN | [ ] | |
| IO-INP-02 | Toggle i_DK_Bediening (%I0.2) | PLC shows 0 when pressed (NC sensor) | [ ] | |
```

### Pattern 2: I/O Test Entry Format
**What:** Include tag, address, action, and verification method
**When to use:** All I/O verification tests
**Example:**
```markdown
| Tag | Address | Test Action | Expected | Pass/Fail | Remarks |
|-----|---------|-------------|----------|-----------|---------|
| i_BNS_Open | %I0.7 | Push hek to OPEN limit | PLC input = 0 (NC reached) | [ ] | |
```

### Pattern 3: State Machine Test Entry Format
**What:** Include precondition, trigger, expected state and outputs
**When to use:** All state transition tests
**Example:**
```markdown
| Test ID | Precondition | Trigger | Expected State | Motor | Alarm | Pass/Fail | Remarks |
|---------|--------------|---------|----------------|-------|-------|-----------|---------|
| SM-01 | IDLE_DICHT | Key OPEN | MOVING_OPEN | OPEN | OFF | [ ] | |
```

### Pattern 4: Timer Test Entry Format
**What:** Include timer parameter, trigger condition, expected duration
**When to use:** All timer behavior tests
**Example:**
```markdown
| Test ID | Timer | Trigger | Duration | Verification | Pass/Fail | Remarks |
|---------|-------|---------|----------|--------------|-----------|---------|
| TMR-01 | CFG_LaagToerenMs | Start movement | 5s | Observe q_MS_LaagToeren active for 5s | [ ] | |
```

### Anti-Patterns to Avoid
- **Vague descriptions:** "Test the key switch" - must specify which position, what to observe
- **Missing verification method:** Must explain HOW to verify, not just WHAT to verify
- **Untraceable tests:** Every test must map to a requirement ID or FDS section
- **Inconsistent format:** All tables must use the same structure within their section
- **Missing remarks column:** Tester MUST have space for notes (DOC-03)

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Test ID numbering | Manual numbering | Prefix-based IDs (IO-INP-01, SM-01, TMR-01) | Traceable, unique, no gaps |
| Expected behaviors | Guess from memory | Extract from FDS + SCL code | Actual code may differ from FDS |
| State outputs | Assume per diagram | Check FB_HekBesturing output section | Code is truth |
| Timer values | Use FDS defaults | Check DB_Config actual values | Configuration is truth |

**Key insight:** The FAT document verifies the ACTUAL v1.0 implementation, not the FDS specification. When FDS and code differ, document the code behavior and note discrepancy.

## Common Pitfalls

### Pitfall 1: Testing FDS Instead of Code
**What goes wrong:** FAT expects behavior from FDS, but code implements slightly differently
**Why it happens:** FDS was written before code, code evolved during implementation
**How to avoid:** Cross-reference each FAT test with actual SCL code
**Warning signs:** "Movement timeout" not in FDS section 4.4 but exists in code (CTL-07, CTL-08)

### Pitfall 2: NC Sensor Logic Confusion
**What goes wrong:** Tester expects 1=active for NC sensors
**Why it happens:** NC logic is counterintuitive (0=contact, 1=break)
**How to avoid:** Explicitly state expected value AND meaning in every NC sensor test
**Example:** "PLC shows 0 (NC: 0 = position reached)"

### Pitfall 3: Missing Preconditions
**What goes wrong:** Test fails because system was in wrong starting state
**Why it happens:** Tester jumps to test without setting up precondition
**How to avoid:** Every test must explicitly state required starting state
**Warning signs:** "From IDLE" - which IDLE? IDLE_OPEN or IDLE_DICHT?

### Pitfall 4: Forgetting Edge Detection Requirements
**What goes wrong:** Test passes with held key but should require edge (rising/falling)
**Why it happens:** Edge detection invisible to tester
**How to avoid:** Specify "momentary key turn" or "press and release" for edge-triggered tests
**Warning signs:** Code uses R_TRIG/F_TRIG for key switch and push button

### Pitfall 5: Timer Pause Behavior Not Tested
**What goes wrong:** Timer appears to work but pause/resume not verified
**Why it happens:** TMR-07 requires specific pause scenario testing
**How to avoid:** Include explicit "stop mid-timer, resume, verify accumulated time" tests
**Warning signs:** TONR timer used in code for pausable timers

### Pitfall 6: Wire-Break Testing Omitted
**What goes wrong:** FAT passes but wire-break fail-safe not verified
**Why it happens:** SAF-10 easy to overlook, requires physical action
**How to avoid:** Dedicated wire-break section with each NC sensor
**Warning signs:** All safety sensors are NC (fail-safe design)

## Code Examples

Verified patterns from actual v1.0 SCL code:

### State Values (from DB_States.scl)
```
STATE_HOMING      = 0   // Power-on unknown position
STATE_IDLE_OPEN   = 1   // At OPEN endstop
STATE_IDLE_DICHT  = 2   // At DICHT endstop
STATE_MOVING_OPEN = 3   // Moving toward OPEN
STATE_MOVING_DICHT= 4   // Moving toward DICHT
STATE_STOPPED     = 5   // Manual stop, mid-position
STATE_VS_PAUSE    = 6   // Beam blocked, auto-resume pending
STATE_PBV_RETRACT = 7   // Bumper hit, retracting to OPEN
STATE_FAULT       = 8   // Both endstops active (impossible)
```

### Timer Configuration (from DB_Config.scl)
```
CFG_LaagToerenMs      = T#5S      // Low speed on start (5000ms)
CFG_PBV_PauzeMs       = T#500MS   // Pause before PBV retract (500ms)
CFG_PBV_TerugtrekMs   = T#3S      // Max PBV retract duration (3000ms)
CFG_VS_HervattingMs   = T#2S      // VS resume delay (2000ms)
CFG_VS_LaagToerenMs   = T#2S      // Low speed after VS resume (2000ms)
CFG_BewegingTimeoutS  = T#90S     // Movement timeout (90000ms)
CFG_EndstopDebounceMs = T#50MS    // Endstop/PBV debounce (50ms)
CFG_VS_DebounceMs     = T#100MS   // VS debounce (100ms)
```

### Motor Output Logic (from FB_HekBesturing output section)
| State | q_MS_Open | q_MS_Dicht | q_MS_LaagToeren |
|-------|-----------|------------|-----------------|
| HOMING | FALSE | TRUE | TRUE |
| IDLE_OPEN | FALSE | FALSE | FALSE |
| IDLE_DICHT | FALSE | FALSE | FALSE |
| MOVING_OPEN | TRUE | FALSE | (timer-based) |
| MOVING_DICHT | FALSE | TRUE | (timer-based) |
| STOPPED | FALSE | FALSE | FALSE |
| VS_PAUSE | FALSE | FALSE | FALSE |
| PBV_RETRACT (pause) | FALSE | FALSE | FALSE |
| PBV_RETRACT (retract) | TRUE | FALSE | TRUE |
| FAULT | FALSE | FALSE | FALSE |

### Alarm Logic (from FB_HekBesturing)
```scl
// q_AlarmKerk = NOT (m_Toestand = STATE_IDLE_DICHT)
// 0 = hek dicht = alarm ON (fail-safe)
// 1 = hek niet dicht = alarm OFF
```

### Key Release After Timeout (CTL-08, from code)
```scl
// After movement timeout, keys must be released before new input accepted
// m_KeyReleased := FALSE on timeout
// m_KeyReleased := TRUE when both keys released
// Input ignored while m_KeyReleased = FALSE
```

### Opposing Key Behavior (CTL-07, from code)
```scl
// In MOVING_OPEN: if m_EdgeSS_Dicht.Q then -> STOPPED
// In MOVING_DICHT: if m_EdgeSS_Open.Q then -> STOPPED
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Paper FAT forms | Markdown + print | Always | Version control, diffs visible |
| Excel checklists | Markdown tables | Always | Git-friendly, no binary |
| Separate FDS/FAT | FAT traces to FDS | v1.1 | Traceability required |

**Deprecated/outdated:**
- None for this domain (FAT documentation is stable practice)

## In-Scope I/O Summary (from IO-LIJST.md)

### Inputs (7 in-scope)
| Tag | Address | Type | Test Method |
|-----|---------|------|-------------|
| i_SS_SleutelSchakelaarOpen | %I0.1 | NO | Turn key left, verify 1 |
| i_DK_Bediening | %I0.2 | NC | Press button, verify 0 |
| i_SS_SleutelSchakelaarDicht | %I0.3 | NO | Turn key right, verify 1 |
| i_BNS_Open | %I0.7 | NC | At OPEN endstop, verify 0 |
| i_BNS_Dicht | %I1.0 | NC | At DICHT endstop, verify 0 |
| i_PBV_BeweegbaarHek | %I1.2 | NC | Press bumper, verify 1 (inverted!) |
| i_VS_Voertuig | %I1.3 | - | Block beam, verify 0 |

### Outputs (4 in-scope)
| Tag | Address | Test Method |
|-----|---------|-------------|
| q_AlarmKerk | %Q0.3 | At IDLE_DICHT = 0, else = 1 |
| q_MS_Open | %Q0.4 | In MOVING_OPEN = 1 |
| q_MS_Dicht | %Q0.5 | In MOVING_DICHT/HOMING = 1 |
| q_MS_LaagToeren | %Q0.6 | In HOMING/PBV = 1, first 5s of movement = 1 |

### Out of Scope (per REQUIREMENTS.md)
| Tag | Address | Reason |
|-----|---------|--------|
| i_PBV_VastHek | %I1.1 | Not connected in v0.1 hardware |
| q_SL_Open | %Q0.2 | Not connected |

## Open Questions

Things that couldn't be fully resolved:

1. **Movement timeout behavior in FDS**
   - What we know: Code implements 90s timeout with key-release requirement (CTL-07, CTL-08)
   - What's unclear: FDS section 4.4 doesn't document this behavior
   - Recommendation: Document actual code behavior in FAT, note FDS update needed (Phase 6)

2. **Push button abort during HOMING**
   - What we know: Code allows m_EdgeDK.Q to abort HOMING to STOPPED
   - What's unclear: Not explicitly in FDS section 6.2
   - Recommendation: Test actual behavior, note FDS update needed

3. **Low speed timer exact behavior**
   - What we know: TONR used, pauses on stop, resets on direction change or endstop
   - What's unclear: Exact accumulation behavior during VS_PAUSE
   - Recommendation: Test carefully, document observed behavior

## Sources

### Primary (HIGH confidence)
- `PLC/Sources/01_DB_States.scl` - State constants verified
- `PLC/Sources/02_DB_Config.scl` - Timer configurations verified
- `PLC/Sources/03_FC_Veiligheid.scl` - Safety logic verified
- `PLC/Sources/04_FB_HekBesturing.scl` - State machine verified
- `PLC/Sources/06_FC_Outputs.scl` - Output logic verified
- `FDS-Automatisch-Hek-v0.2.md` - Functional specification
- `IO-LIJST.md` - I/O addresses and logic
- `statemachine.mmd` - Canonical state transitions
- `.planning/REQUIREMENTS.md` - 40 FAT requirements

### Secondary (MEDIUM confidence)
- [ProcessNavigation FAT Template](https://processnavigation.com/insights/factory-acceptance-test-fat-template/) - FAT section structure
- [RealPars FAT Guide](https://www.realpars.com/blog/factory-acceptance-test) - PLC testing workflow

### Tertiary (LOW confidence)
- General FAT best practices from WebSearch - validated against project needs

## Metadata

**Confidence breakdown:**
- I/O list: HIGH - verified from IO-LIJST.md and code
- State behaviors: HIGH - verified from FB_HekBesturing.scl
- Timer values: HIGH - verified from DB_Config.scl
- FAT structure: HIGH - industry standard, adapted to project needs
- Edge case behaviors: HIGH - verified from code

**Research date:** 2026-02-06
**Valid until:** Indefinite (static code base for v1.0)

---
*Research complete. Ready for Phase 5 planning.*
