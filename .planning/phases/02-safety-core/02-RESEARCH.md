# Phase 2: Safety Core - Research

**Researched:** 2026-02-03
**Domain:** Siemens S7-1200 SCL safety sensor processing, debouncing, fail-safe patterns
**Confidence:** MEDIUM

## Summary

This phase implements isolated safety logic in FC_Veiligheid to process PBV (bumper), VS (beam), and FAULT conditions with correct priority ordering. The key technical challenges are:

1. **Timer instance management in FCs** - FC_Veiligheid is a Function (FC), not a Function Block (FB). IEC timers require instance data to retain state between cycles. FCs do not have instance data. This requires passing timer instances via InOut parameters from a DB.

2. **NC sensor fail-safe debouncing** - All safety sensors are NC (Normally Closed). A wire break reads the same as a trigger (fail-safe). Debouncing must filter noise without masking genuine triggers. The pattern requires TON timers with short preset times (50-100ms).

3. **Priority enforcement** - FAULT > PBV > VS must be evaluated in order with clear output signals for the state machine.

**Primary recommendation:** Convert FC_Veiligheid to FB_Veiligheid (Function Block) to support internal timer instances for debouncing, OR create a separate DB to hold timer instances passed as InOut parameters to FC_Veiligheid.

## Standard Stack

The established tools for safety sensor processing in S7-1200:

### Core
| Component | Type | Purpose | Why Standard |
|-----------|------|---------|--------------|
| TON | IEC Timer | On-delay for debounce (signal must be stable for duration) | Standard debounce pattern, filters bouncing contacts |
| TOF | IEC Timer | Off-delay for debounce (holds output after signal drops) | Used with TON for complete debounce |
| R_TRIG | FB | Rising edge detection | Detects transition 0->1 |
| F_TRIG | FB | Falling edge detection | Detects transition 1->0 |

### Supporting
| Component | Type | Purpose | When to Use |
|-----------|------|---------|-------------|
| TIME type | Data type | Timer preset values | All timer PT parameters |
| BOOL | Data type | Sensor states | All digital I/O |
| INT | Data type | State values | State machine interface |

### Configuration Parameters (from DB_Config)
| Parameter | Value | Purpose |
|-----------|-------|---------|
| CFG_EndstopDebounceMs | T#50MS | Debounce for endstops and PBV |
| CFG_VS_DebounceMs | T#100MS | Debounce for VS (beam) sensor |

## Architecture Patterns

### Critical Decision: FC vs FB for Timers

**Problem:** FC_Veiligheid is currently a Function (FC). IEC timers require instance data to retain state between PLC cycles. FCs have no instance data.

**Option A: Convert to FB (Recommended)**
```
FUNCTION_BLOCK "FB_Veiligheid"
   VAR
      // Timer instances as static variables (multi-instance)
      m_TON_PBV_Debounce : TON;
      m_TON_VS_Debounce : TON;
      m_TON_Fault_Debounce : TON;
   END_VAR
```

**Option B: Keep FC, pass timer DB via InOut**
```
FUNCTION "FC_Veiligheid" : Void
   VAR_IN_OUT
      ioTimers : UDT_SafetyTimers;  // Contains timer instances
   END_VAR
```

**Recommendation:** Option A (convert to FB) is cleaner. The call in Main_OB1 already uses FB call syntax and creates an instance via DB_HekData. Consistency favors FB_Veiligheid with its own timer instances.

### Recommended Block Structure

```
FB_Veiligheid (or FC with timer InOut)
  |
  +-- Inputs: Raw sensor signals (i_PBV_BeweegbaarHek, i_VS_Voertuig, i_BNS_Open, i_BNS_Dicht)
  +-- Config: Debounce times from DB_Config
  +-- Outputs: Debounced triggers (q_PBV_Trigger, q_VS_Trigger, q_Fault)
  |
  +-- Internal:
      +-- TON instances for each sensor debounce
      +-- Priority logic (evaluate FAULT first, then PBV, then VS)
```

### Debounce Pattern: TON for Rising Edge Confirmation

For NC sensors where 1 = trigger:

```scl
// Declare timer instance (in VAR for FB)
m_TON_PBV_Debounce : TON;

// Call timer - output Q is TRUE only after input stable for PT duration
m_TON_PBV_Debounce(
    IN := i_PBV_BeweegbaarHek,    // Raw sensor: 1 = trigger
    PT := "DB_Config".Timers.CFG_EndstopDebounceMs
);

// Use Q output as debounced signal
q_PBV_Trigger := m_TON_PBV_Debounce.Q;
```

### Full Debounce Pattern (Both Edges)

For complete debounce of both rising AND falling edges:

```scl
// Two-timer pattern for complete debounce
m_TON_Rising : TON;
m_TOF_Falling : TOF;

m_TON_Rising(IN := RawSignal, PT := DebounceTime);
m_TOF_Falling(IN := m_TON_Rising.Q, PT := DebounceTime);

DebouncedSignal := m_TOF_Falling.Q;
```

**Note:** For safety sensors, single TON debounce (rising edge only) is typically sufficient. The signal going FALSE immediately is acceptable for fail-safe behavior.

### NC Sensor Logic Pattern

```scl
// NC Sensor interpretation:
// Physical: 0 = circuit closed (OK), 1 = circuit open (trigger or wire break)
// i_PBV_BeweegbaarHek: NC sensor
//   0 = bumper OK (circuit closed)
//   1 = bumper trigger OR wire break (fail-safe: both treated as trigger)

// i_VS_Voertuig:
//   1 = beam intact (OK)
//   0 = beam blocked (TRIGGER)

// i_BNS_Open, i_BNS_Dicht: NC sensors
//   0 = position reached (circuit closed)
//   1 = not at position (circuit open)
```

### Priority Evaluation Pattern

```scl
// Priority order: FAULT > PBV > VS
// Evaluate in order, with early exit possible

// 1. FAULT detection (highest priority)
t_BothEndstops := (NOT i_BNS_Open) AND (NOT i_BNS_Dicht);
// Apply debounce...
q_Fault := DebouncedFault;

// 2. PBV detection (second priority)
// PBV is NC: 1 = trigger
// Apply debounce...
q_PBV_Trigger := DebouncedPBV AND NOT q_Fault;

// 3. VS detection (lowest priority)
// VS: 0 = blocked
// Apply debounce...
q_VS_Trigger := NOT DebouncedVS AND NOT q_Fault AND NOT q_PBV_Trigger;
```

### Anti-Patterns to Avoid

- **Using TEMP variables for timer instances in FC:** Timers need static storage. TEMP variables are reset each cycle, breaking timer functionality.
- **Single IDB for multiple FC calls:** Each timer needs its own instance data. Sharing causes state corruption.
- **Debounce time longer than expected response:** Keep debounce times short (50-100ms) to avoid masking real events.
- **Treating wire break differently from trigger for NC sensors:** The whole point of NC is that wire break = trigger (fail-safe).

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Debounce logic | Counter with thresholds | TON/TOF IEC timers | Edge cases around timer reset, overflow |
| Edge detection | Manual state comparison | R_TRIG/F_TRIG FBs | State storage handled correctly |
| Timer state storage | Manual BOOL flags | IEC timer internal state | Timer Q and ET handled by system |
| Fail-safe detection | Custom NC inversion | Direct NC sensor reading | NC = 1 on trigger is already fail-safe |

**Key insight:** IEC timers and edge triggers are system function blocks designed exactly for these use cases. They handle the edge cases (timer overflow, simultaneous events, state retention) that custom implementations often miss.

## Common Pitfalls

### Pitfall 1: Timer Instance in FC Without InOut

**What goes wrong:** Timer declared in FC TEMP section loses state each cycle, never reaches preset time.
**Why it happens:** FCs have no instance data. TEMP variables reset each call.
**How to avoid:** Use FB with VAR (static), or pass timer via VAR_IN_OUT from a global DB.
**Warning signs:** Timer Q never becomes TRUE, or becomes TRUE immediately.

### Pitfall 2: Wrong NC Sensor Interpretation

**What goes wrong:** Logic inverted, trigger when sensor OK.
**Why it happens:** Confusion between physical circuit state and logical meaning.
**How to avoid:** Document each sensor's interpretation in code comments.
**Warning signs:** Safety triggers during normal operation.

**Reference from FDS:**
| Sensor | NC Logic | 0 means | 1 means |
|--------|----------|---------|---------|
| i_PBV_BeweegbaarHek | NC | OK (intact) | TRIGGER |
| i_VS_Voertuig | - | BLOCKED | OK (intact) |
| i_BNS_Open | NC | REACHED | NOT reached |
| i_BNS_Dicht | NC | REACHED | NOT reached |

### Pitfall 3: FAULT Detection Without Debounce

**What goes wrong:** Transient noise triggers FAULT state incorrectly.
**Why it happens:** Both endstops momentarily reading active during electrical noise.
**How to avoid:** Apply short debounce (50ms) to FAULT condition too.
**Warning signs:** Random FAULT triggers with no actual sensor issue.

### Pitfall 4: Debounce Too Long

**What goes wrong:** Real safety events delayed, safety response too slow.
**Why it happens:** Using debounce time appropriate for buttons (200-500ms) instead of sensors (50ms).
**How to avoid:** Use 50ms for endstops/PBV, 100ms for VS per FDS.
**Warning signs:** Noticeable delay between physical trigger and system response.

### Pitfall 5: Priority Logic Not Enforced

**What goes wrong:** VS response overrides PBV, or both flags set simultaneously causing state machine confusion.
**Why it happens:** Parallel evaluation without priority masking.
**How to avoid:** Evaluate in order: FAULT first (masks all), PBV second (masks VS), VS last.
**Warning signs:** Multiple safety flags active simultaneously when only one trigger present.

## Code Examples

Verified patterns from Siemens SCL documentation:

### TON Timer Declaration and Call

```scl
// Source: Siemens TIA Portal SCL documentation
// In FB static section (VAR):
VAR
    m_Debounce_TON : TON;
END_VAR

// In code:
m_Debounce_TON(
    IN := InputSignal,
    PT := T#50MS
);

// Access outputs:
IF m_Debounce_TON.Q THEN
    // Signal has been stable TRUE for 50ms
END_IF;
```

### R_TRIG Edge Detection

```scl
// Source: Siemens IEC 61131-3 implementation
VAR
    m_RisingEdge : R_TRIG;
END_VAR

// In code:
m_RisingEdge(CLK := InputSignal);

IF m_RisingEdge.Q THEN
    // Rising edge detected this cycle
END_IF;
```

### Complete Debounce with Priority

```scl
// Source: Pattern synthesis from Siemens best practices
FUNCTION_BLOCK "FB_Veiligheid"
VAR_INPUT
    i_PBV_BeweegbaarHek : Bool;  // NC: 1 = trigger
    i_VS_Voertuig : Bool;        // 0 = blocked
    i_BNS_Open : Bool;           // NC: 0 = reached
    i_BNS_Dicht : Bool;          // NC: 0 = reached
END_VAR

VAR_OUTPUT
    q_PBV_Trigger : Bool;
    q_VS_Trigger : Bool;
    q_Fault : Bool;
END_VAR

VAR
    m_TON_PBV : TON;
    m_TON_VS : TON;
    m_TON_Fault : TON;
    m_RawFault : Bool;
    m_RawVS_Blocked : Bool;
END_VAR

BEGIN
    // FAULT: both endstops active (NC: 0 = reached)
    m_RawFault := (NOT i_BNS_Open) AND (NOT i_BNS_Dicht);
    m_TON_Fault(IN := m_RawFault, PT := T#50MS);
    q_Fault := m_TON_Fault.Q;

    // PBV: NC sensor, 1 = trigger
    m_TON_PBV(IN := i_PBV_BeweegbaarHek, PT := T#50MS);
    q_PBV_Trigger := m_TON_PBV.Q AND NOT q_Fault;  // Masked by FAULT

    // VS: 0 = blocked (trigger)
    m_RawVS_Blocked := NOT i_VS_Voertuig;
    m_TON_VS(IN := m_RawVS_Blocked, PT := T#100MS);
    q_VS_Trigger := m_TON_VS.Q AND NOT q_Fault AND NOT q_PBV_Trigger;  // Masked by both

END_FUNCTION_BLOCK
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| S5/S7 timers | IEC timers (TON/TOF/TP) | TIA Portal V11+ | Multi-instance support, standardized interface |
| Manual edge flags | R_TRIG/F_TRIG FBs | IEC 61131-3 | Cleaner code, standard pattern |
| FC with external timers | FB with multi-instance timers | Best practice | Encapsulation, reusability |

**Current best practice:** Use Function Blocks (FBs) for any logic requiring state retention (timers, edge detection, counters). Use Functions (FCs) only for pure calculations without state.

## Open Questions

Things that couldn't be fully resolved:

1. **VS direction-awareness in FC_Veiligheid**
   - What we know: CONTEXT.md says VS "only active in closing direction"
   - What's unclear: Should FC_Veiligheid receive current direction as input, or should state machine handle this?
   - Recommendation: FC_Veiligheid outputs raw debounced VS trigger. State machine (Phase 3) decides whether to act based on direction. This keeps safety logic isolated.

2. **VS timeout behavior**
   - What we know: CONTEXT.md mentions "if beam broken too long, transition to STOPPED"
   - What's unclear: Should timeout logic be in FC_Veiligheid or state machine?
   - Recommendation: Timeout is state transition logic, belongs in state machine. FC_Veiligheid just reports if VS is currently triggered.

3. **Wire break vs trigger distinction**
   - What we know: NC sensors make wire break indistinguishable from trigger (by design)
   - What's unclear: Should there be any diagnostic output indicating "possible wire break" (sustained trigger)?
   - Recommendation: Keep simple for Phase 2. NC behavior is intentionally fail-safe. Add diagnostics in future phase if needed.

4. **FC to FB conversion impact**
   - What we know: Changing FC_Veiligheid to FB_Veiligheid requires instance DB
   - What's unclear: Whether this breaks existing OB1 call structure
   - Recommendation: Review OB1 call syntax. May need to create DB_VeiligheidData or integrate into DB_HekData.

## Sources

### Primary (HIGH confidence)
- [Siemens SCL IEC Timers: TP, TON, TOF Programming](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-iec-timer-programming-tp-ton-tof-examples.php) - TON syntax and parameters
- [Timer instructions (IEC timers) - TIA Portal](https://docs.tia.siemens.cloud/r/simatic_s7_1200_manual_collection_eses_20/basic-instructions/timer-operations/timer-instructions-iec-timers) - Official timer documentation
- [Understanding Function Block Instances in Siemens TIA Portal](https://www.solisplc.com/tutorials/function-block-instances-in-siemens-tia-portal) - Instance patterns

### Secondary (MEDIUM confidence)
- [Timer variables in SCL for S7-1200 - Siemens Forum](https://support.industry.siemens.com/forum/WW/en/posts/timer-variables-in-scl-for-s7-1200/83666) - FC timer limitations
- [How to Program PLC Input Debounce On and Off](https://accautomation.ca/how-to-program-plc-input-debounce-on-and-off/) - Debounce pattern principles
- [R_TRIG: Detect positive signal edge (S7-1200, S7-1500)](https://docs.tia.siemens.cloud/r/en-us/v20/scl-s7-1200-s7-1500/bit-logic-operations-s7-1200-s7-1500/r_trig-detect-positive-signal-edge-s7-1200-s7-1500) - Edge detection

### Tertiary (LOW confidence - patterns synthesized)
- [Programming Guideline Safety for SIMATIC S7-1200/1500](https://cache.industry.siemens.com/dl/files/255/109750255/att_1132641/v1/109750255_Programming-Guideline-Safety_DOC_V1_3_en.pdf) - Safety programming patterns (not directly verified for this use case)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - IEC timers are well-documented
- Architecture (FC vs FB): MEDIUM - Best practice clear, but project constraints may differ
- Debounce patterns: HIGH - Standard industrial pattern
- Priority logic: MEDIUM - Synthesized from principles, not project-specific example
- NC sensor interpretation: HIGH - From FDS specification

**Research date:** 2026-02-03
**Valid until:** 60 days (stable domain, no fast-moving dependencies)
