# Phase 1: Foundation - Context

**Gathered:** 2026-02-03
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish SCL block structure and configuration parameters for TIA Portal v20 import. This includes:
- SCL file templates for all blocks
- DB_Config with CFG_ parameters
- Block call structure (OB1 → FC_Veiligheid → FB_HekBesturing → FC_Outputs)
- State constants defined as INT values

</domain>

<decisions>
## Implementation Decisions

### File organization
- One SCL file per block (separate files for each)
- Numbered prefix naming: 01_DB_Config.scl, 02_FC_Veiligheid.scl, etc.
- TIA-style folder structure: PLC/Sources/
- OB1 handling: Claude's discretion

### Configuration parameters
- Timer values use native TIME type (T#30S format)
- Grouped into STRUCTs: DB_Config.Timers.OpenTijd, DB_Config.Safety.PauzeMs
- Comprehensive categories from start: Timers, Safety, Motor, Limits, Debug
- Optimized DB (TIA Portal default, symbolic access)

### Code style
- Dutch language for all variable and block names (matches FDS terminology)
- Hungarian + PascalCase naming: i_Sensor, q_Motor, m_Toestand
- Prefixes: i_ (input), q_ (output), m_ (memory/internal)
- Verbose comments in Dutch — every block, state, and transition documented

### State constants
- Separate global DB (DB_States) for all state constants
- State names match FDS exactly: HOMING, IDLE_OPEN, IDLE_DICHT, MOVING_OPEN, MOVING_DICHT, STOPPED, VS_PAUSE, PBV_RETRACT, FAULT
- Prefix STATE_: STATE_HOMING, STATE_IDLE_OPEN, etc.
- Numbering matches FDS section 2.2 order (0=HOMING, 1=IDLE_OPEN, 2=IDLE_DICHT, etc.)
- Full Dutch descriptions as comments for each state

### Claude's Discretion
- OB1 as SCL file vs TIA Portal manual configuration
- Exact file numbering order
- STRUCT organization within DB_Config categories
- Debug parameter specifics

</decisions>

<specifics>
## Specific Ideas

- "Match FDS terminology" — use exact Dutch terms from FDS document
- State values should reflect FDS section 2.2 table order for traceability
- TIA Portal v20 compatibility is critical — files must import without errors

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation*
*Context gathered: 2026-02-03*
