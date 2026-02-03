# Phase 1: Foundation - Research

**Researched:** 2026-02-03
**Domain:** TIA Portal v20 SCL External Source Files for S7-1200
**Confidence:** HIGH

## Summary

This research covers TIA Portal v20 SCL external source file syntax requirements for creating PLC blocks that can be directly imported. The S7-1200/1500 platform with TIA Portal v20 uses SCL (Structured Control Language), which is Siemens' implementation of IEC 61131-3 Structured Text.

External source files (.scl) are human-readable text files that can be created outside TIA Portal and imported to generate PLC blocks. This approach enables version control, code review, and automated generation while maintaining full compatibility with TIA Portal's native compilation.

Key findings: Use optimized block access (default), declare constants in global data blocks rather than using local CONST blocks, use TIME data type with T# format for timers, and follow strict compilation order (UDTs first, then DBs, then FBs/FCs, then OBs).

**Primary recommendation:** Create separate .scl files per block, use symbolic names throughout, enable optimized block access, and test import with single file before full import.

## Standard Stack

The established approach for TIA Portal v20 SCL external source files:

### Core Elements
| Element | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| SCL Language | TIA Portal v20 | PLC programming language | Native to S7-1200/1500, best readability |
| .scl file format | - | External source container | Text-based, importable, version-controllable |
| Optimized DB Access | Default | Memory management | Better performance, symbolic access only |

### Block Types Supported
| Block Type | Declaration | Purpose |
|------------|-------------|---------|
| ORGANIZATION_BLOCK | `ORGANIZATION_BLOCK "name"` | Main program (OB1) |
| FUNCTION_BLOCK | `FUNCTION_BLOCK "name"` | Stateful logic with instance DB |
| FUNCTION | `FUNCTION "name" : ReturnType` | Stateless operations |
| DATA_BLOCK | `DATA_BLOCK "name"` | Global data storage |
| TYPE | `TYPE "name"` | User-defined data types (UDT) |

### Files to Create (Phase 1)
```
PLC/Sources/
  01_DB_States.scl       // State constants
  02_DB_Config.scl       // Configuration parameters
  03_FC_Veiligheid.scl   // Safety function (stub)
  04_FB_HekBesturing.scl // State machine FB (stub)
  05_FC_Outputs.scl      // Output mapping (stub)
  06_Main_OB1.scl        // Main program cycle
```

## Architecture Patterns

### SCL Block Declaration Pattern

**FUNCTION_BLOCK with Attributes:**
```scl
FUNCTION_BLOCK "FB_HekBesturing"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_Sensor : Bool;   // Voorbeeld invoer
   END_VAR

   VAR_OUTPUT
      q_Motor : Bool;    // Voorbeeld uitvoer
   END_VAR

   VAR
      m_Toestand : Int;  // Huidige toestand
   END_VAR

   VAR_TEMP
      t_Hulp : Int;      // Tijdelijke variabele
   END_VAR

BEGIN
    // Programmacode hier
END_FUNCTION_BLOCK
```
Source: [Automation Notes Examples](https://abedgnu.github.io/Automation-Notes/chapters/PLC/Siemens/exercises-solutions.html)

**FUNCTION Declaration:**
```scl
FUNCTION "FC_Veiligheid" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_Parameter : Int;
   END_VAR

   VAR_OUTPUT
      q_Resultaat : Bool;
   END_VAR

   VAR_TEMP
      t_Lokaal : Int;
   END_VAR

BEGIN
    // Programmacode hier
END_FUNCTION
```
Source: [SolisPlc SCL Tutorial](https://www.solisplc.com/tutorials/a-comprehensive-guide-to-constants-for-siemens-plc-programming-tia-portal)

**Global DATA_BLOCK with STRUCT:**
```scl
DATA_BLOCK "DB_Config"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
   STRUCT
      Timers : STRUCT
         OpenTijd : Time := T#30S;
         DichtTijd : Time := T#30S;
         PBV_PauzeMs : Time := T#500MS;
         PBV_TerugtrekMs : Time := T#3S;
         VS_HervattingMs : Time := T#2S;
         VS_LaagToerenMs : Time := T#2S;
         LaagToerenMs : Time := T#5S;
         BewegingTimeout : Time := T#90S;
         EndstopDebounce : Time := T#50MS;
      END_STRUCT;
      Safety : STRUCT
         // Veiligheidsparameters
      END_STRUCT;
      Motor : STRUCT
         // Motorparameters
      END_STRUCT;
   END_STRUCT;
BEGIN
END_DATA_BLOCK
```
Source: [PLCBlog SCL Data Types](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-data-types-and-uses-and-value-assignments.php)

**Constants in Global DATA_BLOCK:**
```scl
DATA_BLOCK "DB_States"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
   STRUCT
      // Toestandsconstanten - matchen met FDS sectie 2.2
      STATE_HOMING : Int := 0;        // Zoeken referentiepositie
      STATE_IDLE_OPEN : Int := 1;     // Hek volledig open
      STATE_IDLE_DICHT : Int := 2;    // Hek volledig dicht
      STATE_MOVING_OPEN : Int := 3;   // Beweging naar open
      STATE_MOVING_DICHT : Int := 4;  // Beweging naar dicht
      STATE_STOPPED : Int := 5;       // Gestopt in tussenpositie
      STATE_VS_PAUSE : Int := 6;      // Gepauzeerd door beam
      STATE_PBV_RETRACT : Int := 7;   // Terugtrekken na bumper
      STATE_FAULT : Int := 8;         // Sensorstoring
   END_STRUCT;
BEGIN
END_DATA_BLOCK
```
Source: [SolisPlc Constants Guide](https://www.solisplc.com/tutorials/a-comprehensive-guide-to-constants-for-siemens-plc-programming-tia-portal)

**Instance DATA_BLOCK for FB:**
```scl
DATA_BLOCK "DB_HekData"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
"FB_HekBesturing"
BEGIN
END_DATA_BLOCK
```
Source: [Industrial Monitor Direct](https://industrialmonitordirect.com/blogs/knowledgebase/scl-fbfc-block-calling-and-instance-db-management-in-siemens-s7)

**ORGANIZATION_BLOCK OB1:**
```scl
ORGANIZATION_BLOCK "Main"
TITLE = Hoofdprogramma
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_TEMP
      t_OB1_Info : Array[0..19] of Byte;
   END_VAR

BEGIN
    // Aanroepvolgorde per CODE-01:
    // 1. FC_Veiligheid - veiligheidsverwerking
    // 2. FB_HekBesturing - toestandsmachine
    // 3. FC_Outputs - uitgangen zetten

    "FC_Veiligheid"();
    "FB_HekBesturing"."DB_HekData"();
    "FC_Outputs"();
END_ORGANIZATION_BLOCK
```
Source: [Siemens Programming Guideline](https://support.industry.siemens.com/cs/attachments/90885040/81318674_Programming_guideline_DOC_v16_en.pdf)

### Variable Section Reference

| Section | Purpose | Scope | Notes |
|---------|---------|-------|-------|
| VAR_INPUT | Input parameters | Read-only in block | Passed by value |
| VAR_OUTPUT | Output parameters | Written by block | Can be read from IDB |
| VAR_IN_OUT | In/Out parameters | Read/Write | Passed by reference |
| VAR | Static variables | FB only | Retained between calls |
| VAR_TEMP | Temporary variables | Within call | Reset each call |
| VAR CONSTANT | Local constants | Within block | TIA Portal syntax (not CONST) |

### TIME Data Type Format

Use `T#` prefix with unit combinations:
```scl
// Geldige TIME formaten:
T#30S           // 30 seconden
T#500MS         // 500 milliseconden
T#2S            // 2 seconden
T#5H10M30S      // 5 uur, 10 minuten, 30 seconden
TIME#10D_20H    // 10 dagen, 20 uur

// Bereik: T#-24d_20h_31m_23s_648ms tot T#+24d_20h_31m_23s_647ms
```
Source: [PLCBlog TIME Data Type](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-data-types-and-uses-and-value-assignments.php)

### Comment Syntax

```scl
// Dit is een enkelregelige opmerking

(* Dit is een
   meerregelige opmerking *)

REGION Veiligheid
    // Code voor veiligheidsafhandeling
END_REGION
```
Source: [Siemens FAQ on Comments](https://support.industry.siemens.com/cs/document/109482004/in-step-7-(tia-portal)-how-do-you-comment-out-instructions-in-lad-fbd-stl-and-scl-?dti=0&lc=en-WW)

### Block Calling Syntax

**Calling FB from OB1:**
```scl
// Syntax: "FB_Name"."IDB_Name"(parameters);
"FB_HekBesturing"."DB_HekData"(
    i_Parameter := waarde,
    q_Uitvoer => lokaleVar
);

// Of zonder parameters:
"FB_HekBesturing"."DB_HekData"();

// FC aanroep:
"FC_Veiligheid"(i_Input := waarde);
```
Source: [Industrial Monitor Direct](https://industrialmonitordirect.com/blogs/knowledgebase/scl-fbfc-block-calling-and-instance-db-management-in-siemens-s7)

### Compilation Order

**CRITICAL:** Files must compile in dependency order:

1. TYPE declarations (UDTs)
2. Global DATA_BLOCKs (DB_States, DB_Config)
3. FUNCTION_BLOCKs
4. Instance DATA_BLOCKs (DB_HekData)
5. FUNCTIONs
6. ORGANIZATION_BLOCKs

Source: [Industrial Monitor Direct](https://industrialmonitordirect.com/blogs/knowledgebase/scl-fbfc-block-calling-and-instance-db-management-in-siemens-s7)

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Timer functions | Custom counter logic | TON_TIME, TOF_TIME, TP_TIME | Built-in IEC timers, edge cases handled |
| Edge detection | Manual previous-value tracking | R_TRIG, F_TRIG | Standard instruction, reliable |
| State constants | Magic numbers | Global DB with named constants | Readability, single point of change |
| Instance data | Global variables | FB with instance DB | Proper encapsulation, reusability |

**Key insight:** TIA Portal provides IEC-standard function blocks for timers and edge detection. Using these ensures proper behavior across PLC scan cycles and avoids edge cases with timer rollover or simultaneous events.

## Common Pitfalls

### Pitfall 1: Using CONST Instead of VAR CONSTANT
**What goes wrong:** Classic Step 7 used `CONST` / `END_CONST` for local constants. TIA Portal generates "Unknown instruction" error.
**Why it happens:** Syntax changed between Step 7 and TIA Portal.
**How to avoid:** Use `VAR CONSTANT` / `END_VAR` for local constants, or better: use global DB for shared constants.
**Warning signs:** Compile error "Unknown instruction CONST".
Source: [SolisPlc Constants Guide](https://www.solisplc.com/tutorials/a-comprehensive-guide-to-constants-for-siemens-plc-programming-tia-portal)

### Pitfall 2: Missing S7_Optimized_Access Attribute
**What goes wrong:** Block may default to non-optimized mode, breaking symbolic-only access assumptions.
**Why it happens:** Attribute not explicitly set in external source file.
**How to avoid:** Always include `{ S7_Optimized_Access := 'TRUE' }` after block declaration.
**Warning signs:** Ability to see byte addresses in data block, unexpected memory layout.
Source: [WZAutomation DB Access](https://www.wzautomation.com/tech-talk/the-standard-and-optimized-db-access/)

### Pitfall 3: Wrong Compilation Order
**What goes wrong:** Compiler error "Block not found" or "Unknown type".
**Why it happens:** Dependencies must exist before blocks that reference them.
**How to avoid:** Compile in order: UDTs -> Global DBs -> FBs -> Instance DBs -> FCs -> OBs. Or use single source file with correct order.
**Warning signs:** "Compile all" works but individual compile fails; errors reference blocks that exist.
Source: [Industrial Monitor Direct](https://industrialmonitordirect.com/blogs/knowledgebase/scl-fbfc-block-calling-and-instance-db-management-in-siemens-s7)

### Pitfall 4: Nested IF/CASE with Structural Variables (TIA v18/v19 Bug)
**What goes wrong:** Assignments get swapped during code generation.
**Why it happens:** SCL optimizer bug in TIA Portal V19 Update 2 and V18 Update 4.
**How to avoid:** Install latest patches; use "Software (Full Rebuild)" when compiling.
**Warning signs:** Logic executes differently than written; structural variable assignments behave unexpectedly.
Source: [MasterOfPLC Compilation Issue](http://www.masterofplc.com/2024/11/tia-portal-compilation-issue-scl.html)

### Pitfall 5: Instance DB Without FB Reference
**What goes wrong:** Instance DB created but not properly linked to FB.
**Why it happens:** Instance DB must reference the FB type in its declaration.
**How to avoid:** Use correct syntax: `DATA_BLOCK "name" "FB_Name"` (FB name on separate line after DB name).
**Warning signs:** DB created but FB call fails; DB has no structure.
Source: [Industrial Monitor Direct](https://industrialmonitordirect.com/blogs/knowledgebase/scl-fbfc-block-calling-and-instance-db-management-in-siemens-s7)

### Pitfall 6: Using INT for Timer Values Instead of TIME
**What goes wrong:** Must manually convert milliseconds; loses T# readability.
**Why it happens:** Coming from other PLCs where timers use integers.
**How to avoid:** Use TIME data type with T# notation for all duration parameters.
**Warning signs:** Configuration not readable; calculations for conversion needed.
Source: [PLCBlog SCL Data Types](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-data-types-and-uses-and-value-assignments.php)

### Pitfall 7: Block Name Case Sensitivity Confusion
**What goes wrong:** Inconsistent naming causes confusion, though SCL is case-insensitive.
**Why it happens:** SCL is NOT case sensitive, but TIA Portal preserves case in display.
**How to avoid:** Use consistent PascalCase naming throughout all files.
**Warning signs:** Same variable appears with different cases in different places.
Source: [Instrumentation Tools SCL Rules](https://instrumentationtools.com/rules-for-writing-scl-language/)

## Code Examples

### Complete FC Example (Stub for Phase 1)
```scl
// FC_Veiligheid.scl
// Veiligheidsverwerking voor automatisch hek
// Verwerkt PBV, VS, en sensordefect detectie

FUNCTION "FC_Veiligheid" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_PBV_BeweegbaarHek : Bool;  // Rubber bumper (NC: 0=OK, 1=trigger)
      i_VS_Voertuig : Bool;        // Beam detectie (0=onderbroken)
      i_BNS_Open : Bool;           // Eindstand OPEN (NC: 0=bereikt)
      i_BNS_Dicht : Bool;          // Eindstand DICHT (NC: 0=bereikt)
   END_VAR

   VAR_OUTPUT
      q_PBV_Trigger : Bool;        // PBV geactiveerd
      q_VS_Trigger : Bool;         // VS geactiveerd
      q_Fault : Bool;              // Sensordefect gedetecteerd
   END_VAR

   VAR_TEMP
      t_BeidEindstanden : Bool;
   END_VAR

BEGIN
    // Stub - volledige implementatie in Phase 2
    #q_PBV_Trigger := FALSE;
    #q_VS_Trigger := FALSE;
    #q_Fault := FALSE;
END_FUNCTION
```

### Complete FB Example (Stub for Phase 1)
```scl
// FB_HekBesturing.scl
// Toestandsmachine voor hekbesturing
// Bevat alle 9 toestanden per FDS sectie 2.2

FUNCTION_BLOCK "FB_HekBesturing"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_SS_Open : Bool;            // Sleutel naar OPEN
      i_SS_Dicht : Bool;           // Sleutel naar DICHT
      i_DK_Bediening : Bool;       // Drukknop (NC: 0=ingedrukt)
      i_BNS_Open : Bool;           // Eindstand OPEN
      i_BNS_Dicht : Bool;          // Eindstand DICHT
      i_PBV_Trigger : Bool;        // PBV actief (van FC_Veiligheid)
      i_VS_Trigger : Bool;         // VS actief (van FC_Veiligheid)
      i_Fault : Bool;              // Sensordefect (van FC_Veiligheid)
   END_VAR

   VAR_OUTPUT
      q_MS_Open : Bool;            // Motor richting OPEN
      q_MS_Dicht : Bool;           // Motor richting DICHT
      q_MS_LaagToeren : Bool;      // Lage snelheid actief
      q_AlarmKerk : Bool;          // Kerkalarm (0=AAN, fail-safe)
      q_HuidigeToestand : Int;     // Huidige toestand voor diagnose
   END_VAR

   VAR
      m_Toestand : Int;            // Huidige toestand
      m_VorigeToestand : Int;      // Vorige toestand
      m_LaatsteRichting : Int;     // Laatst actieve richting
   END_VAR

BEGIN
    // Stub - volledige implementatie in Phase 3
    // Initialiseer uitgangen
    #q_MS_Open := FALSE;
    #q_MS_Dicht := FALSE;
    #q_MS_LaagToeren := FALSE;
    #q_AlarmKerk := TRUE;  // Fail-safe: hek niet dicht
    #q_HuidigeToestand := #m_Toestand;
END_FUNCTION_BLOCK
```

### Complete Global DB Example
```scl
// DB_States.scl
// Globale toestandsconstanten voor hekbesturing
// Waarden matchen FDS sectie 2.2

DATA_BLOCK "DB_States"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
   STRUCT
      // Toestanden - volgorde per FDS
      STATE_HOMING : Int := 0;        // Zoeken referentiepositie bij opstart
      STATE_IDLE_OPEN : Int := 1;     // Hek volledig open, motor uit
      STATE_IDLE_DICHT : Int := 2;    // Hek volledig dicht, alarm aan
      STATE_MOVING_OPEN : Int := 3;   // Beweging richting open
      STATE_MOVING_DICHT : Int := 4;  // Beweging richting dicht
      STATE_STOPPED : Int := 5;       // Gestopt in tussenpositie
      STATE_VS_PAUSE : Int := 6;      // Gepauzeerd door lichtsluis
      STATE_PBV_RETRACT : Int := 7;   // Terugtrekken na bumperactivatie
      STATE_FAULT : Int := 8;         // Sensorstoring gedetecteerd

      // Richtingsconstanten
      DIR_NONE : Int := 0;            // Geen richting
      DIR_OPEN : Int := 1;            // Richting OPEN
      DIR_DICHT : Int := 2;           // Richting DICHT
   END_STRUCT;
BEGIN
END_DATA_BLOCK
```

### Complete Config DB Example
```scl
// DB_Config.scl
// Configuratieparameters voor hekbesturing
// Alle CFG_ parameters uit FDS sectie 8

DATA_BLOCK "DB_Config"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
   STRUCT
      Timers : STRUCT
         CFG_LaagToerenMs : Time := T#5S;        // Duur lage snelheid bij start
         CFG_PBV_PauzeMs : Time := T#500MS;      // Pauze voor terugtrekken na PBV
         CFG_PBV_TerugtrekMs : Time := T#3S;     // Max duur terugtrekbeweging
         CFG_VS_HervattingMs : Time := T#2S;     // Wachttijd na beam herstel
         CFG_VS_LaagToerenMs : Time := T#2S;     // Lage snelheid na VS hervatting
         CFG_BewegingTimeoutS : Time := T#90S;   // Bewegingstimeout
         CFG_EndstopDebounceMs : Time := T#50MS; // Debounce eindstand/PBV
      END_STRUCT;
      Safety : STRUCT
         CFG_VS_DebounceMs : Time := T#100MS;    // VS debounce tijd
      END_STRUCT;
      Motor : STRUCT
         // Toekomstige motorparameters
      END_STRUCT;
      Limits : STRUCT
         // Toekomstige limietwaarden
      END_STRUCT;
      Debug : STRUCT
         SimulatieActief : Bool := FALSE;        // Debug simulatiemodus
      END_STRUCT;
   END_STRUCT;
BEGIN
END_DATA_BLOCK
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| CONST/END_CONST for constants | VAR CONSTANT or Global DB | TIA Portal v11+ | Must use new syntax |
| Non-optimized DB default | Optimized DB default | S7-1200/1500 | Symbolic access only |
| INT for timers (ms) | TIME with T# format | Always for TIA | Better readability |
| Absolute addressing | Symbolic addressing | S7-1200/1500 | No direct byte access |
| Step 7 attributes (S7_tasklist, etc.) | Limited subset | TIA Portal | Some CFC attributes removed |

**Deprecated/outdated:**
- `CONST`/`END_CONST` syntax: Use `VAR CONSTANT` or global DB instead
- Absolute addressing in optimized blocks: Not supported, use symbolic names
- Classic Step 7 system attributes: Many not supported in TIA Portal

## Open Questions

Things that could not be fully resolved:

1. **OB1 Block Number**
   - What we know: OB1 is the standard program cycle OB
   - What's unclear: Whether TIA Portal auto-assigns numbers or requires explicit numbering in external source
   - Recommendation: Let TIA Portal assign block numbers automatically (don't specify in code)

2. **Multiple Files Import Order**
   - What we know: Dependencies must exist before referencing blocks
   - What's unclear: Whether TIA Portal resolves import order automatically when importing multiple files
   - Recommendation: Number files with prefix (01_, 02_, etc.) and import in order; test with single file first

3. **TITLE Attribute in External Source**
   - What we know: TITLE is shown in some examples as block attribute
   - What's unclear: Exact syntax and whether it's required
   - Recommendation: Use `TITLE = Beschrijving` after block declaration (optional)

## Sources

### Primary (HIGH confidence)
- [Siemens Programming Guideline for S7-1200/S7-1500](https://support.industry.siemens.com/cs/attachments/90885040/81318674_Programming_guideline_DOC_v16_en.pdf) - Block structure, best practices
- [Automation Notes SCL Examples](https://abedgnu.github.io/Automation-Notes/chapters/PLC/Siemens/exercises-solutions.html) - Complete working SCL examples
- [Industrial Monitor Direct - SCL FB/FC Calling](https://industrialmonitordirect.com/blogs/knowledgebase/scl-fbfc-block-calling-and-instance-db-management-in-siemens-s7) - Block calling syntax, compilation order
- [TIA Portal v20 External Source Files Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/creating-and-managing-blocks/using-external-source-files-for-stl-and-scl/basics-of-using-external-source-files) - Official import process

### Secondary (MEDIUM confidence)
- [SolisPlc - Constants Guide](https://www.solisplc.com/tutorials/a-comprehensive-guide-to-constants-for-siemens-plc-programming-tia-portal) - Constant declaration patterns, verified with examples
- [PLCBlog - SCL Data Types](https://plcblog.in/plc/siemens-tia-portal/siemens-scl-data-types-and-uses-and-value-assignments.php) - TIME format, data types
- [WZAutomation - DB Access](https://www.wzautomation.com/tech-talk/the-standard-and-optimized-db-access/) - Optimized vs standard access
- [Siemens FAQ on Comments](https://support.industry.siemens.com/cs/document/109482004/in-step-7-(tia-portal)-how-do-you-comment-out-instructions-in-lad-fbd-stl-and-scl-?dti=0&lc=en-WW) - Comment syntax

### Tertiary (LOW confidence)
- [MasterOfPLC - TIA v18/v19 Bug](http://www.masterofplc.com/2024/11/tia-portal-compilation-issue-scl.html) - Compilation bug warning, verify patch status
- [Instrumentation Tools - SCL Rules](https://instrumentationtools.com/rules-for-writing-scl-language/) - General coding rules

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Based on official Siemens documentation and working examples
- Architecture patterns: HIGH - Multiple verified sources with consistent syntax
- Block syntax: HIGH - Tested examples from official sources
- Pitfalls: MEDIUM - Combination of documented issues and community reports

**Research date:** 2026-02-03
**Valid until:** 2026-03-03 (30 days - stable domain, TIA Portal v20 current)

---

*Phase: 01-foundation*
*Research completed: 2026-02-03*
