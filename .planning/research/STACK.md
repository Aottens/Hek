# Technology Stack: TIA Portal V20 XML Export for S7-1212C

**Project:** Automatisch Hek Kerk
**Researched:** 2026-02-02
**Overall Confidence:** MEDIUM (verified approach, some details require validation during implementation)

## Executive Summary

TIA Portal provides two primary approaches for programmatically generating PLC code:

1. **SCL External Source Files** (RECOMMENDED) - Plain text files using IEC 61131-3 SCL syntax
2. **SimaticML XML Files** - XML format via TIA Portal Openness API

**Recommendation: Use SCL External Source Files (.scl)**

SCL source files are simpler to generate, human-readable, and directly importable into TIA Portal V20 without requiring the Openness API. XML is more complex and primarily intended for automated tooling integration.

---

## Recommended Stack

### Primary Approach: SCL External Source Files

| Component | Format | Why |
|-----------|--------|-----|
| Function Blocks (FB) | `.scl` | Human-readable, direct import, full SCL syntax support |
| Functions (FC) | `.scl` | Same file can contain multiple blocks |
| Organization Blocks (OB) | `.scl` | OB1 main cycle import supported |
| Data Blocks (DB) | `.db` or `.scl` | Struct/Array declarations in text format |
| User Data Types (UDT) | `.udt` | Type definitions for structured data |

**Confidence: HIGH** - Verified via [Siemens Support Article 79168964](https://support.industry.siemens.com/cs/document/79168964/) and [TIA Portal V20 Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/creating-and-managing-blocks/using-external-source-files-for-stl-and-scl/basics-of-using-external-source-files)

### Alternative: SimaticML XML Format

| Component | XML Element | Use When |
|-----------|-------------|----------|
| Function Blocks | `SW.Blocks.FB` | Automated tooling, Openness API integration |
| Functions | `SW.Blocks.FC` | Programmatic code generation |
| Organization Blocks | `SW.Blocks.OB` | Version Control Interface export |
| Global Data Blocks | `SW.Blocks.GlobalDB` | Complex data structure import |
| Instance Data Blocks | `SW.Blocks.InstanceDB` | FB instance data |

**Confidence: MEDIUM** - Verified structure via [DMC Openness Guide](https://www.dmcinfo.com/latest-thinking/blog/id/9908/how-to-read-an-openness-xml-file) but requires TIA Portal validation

---

## SCL Source File Format Specification

### File Extensions

| Block Type | Extension | Notes |
|------------|-----------|-------|
| Functions/Function Blocks/OBs | `.scl` | Contains SCL code |
| Data Blocks | `.db` | Contains DB declaration |
| User Data Types | `.udt` | Contains TYPE declaration |

### FUNCTION Block Template

```scl
FUNCTION "FC_FunctionName" : Void
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      inputVar1 : Bool;
      inputVar2 : Int;
   END_VAR

   VAR_OUTPUT
      outputVar1 : Bool;
   END_VAR

   VAR_TEMP
      tempVar1 : Int;
   END_VAR

BEGIN
    // SCL code here
    #outputVar1 := #inputVar1 AND (#inputVar2 > 0);
END_FUNCTION
```

**Source:** [Automation Notes - SCL Examples](https://abedgnu.github.io/Automation-Notes/chapters/PLC/Siemens/exercises-solutions.html)

### FUNCTION_BLOCK Template

```scl
FUNCTION_BLOCK "FB_BlockName"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_Enable : Bool;
      i_Command : Int;
   END_VAR

   VAR_OUTPUT
      q_Active : Bool;
      q_State : Int;
   END_VAR

   VAR
      // Static variables (retained between calls)
      internalState : Int;
      timer1 : TON_TIME;
   END_VAR

   VAR_TEMP
      // Temporary variables (not retained)
      tempCalc : Int;
   END_VAR

BEGIN
    // SCL code here
    #timer1(IN := #i_Enable, PT := T#1000ms);

    IF #i_Enable THEN
        #q_Active := TRUE;
        #q_State := #internalState;
    END_IF;
END_FUNCTION_BLOCK
```

### ORGANIZATION_BLOCK Template (OB1 Main)

```scl
ORGANIZATION_BLOCK "Main"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_TEMP
      tempVar : Int;
   END_VAR

BEGIN
    // Call function blocks and functions
    "FB_HekBesturing_DB"(
        i_SS_Open := "i_SS_SleutelSchakelaarOpen",
        i_SS_Dicht := "i_SS_SleutelSchakelaarDicht"
        // ... more parameters
    );

    "FC_Veiligheid"();
    "FC_Outputs"();
END_ORGANIZATION_BLOCK
```

### DATA_BLOCK Template (Global DB)

```scl
DATA_BLOCK "DB_Config"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
   STRUCT
      CFG_PBV_PauzeMs : Time := T#500ms;
      CFG_PBV_TerugtrekMs : Time := T#3000ms;
      CFG_VS_HervattingMs : Time := T#2000ms;
      CFG_VS_LaagToerenMs : Time := T#1000ms;
      CFG_LaagToerenMs : Time := T#2000ms;
   END_STRUCT;

BEGIN
   CFG_PBV_PauzeMs := T#500ms;
   CFG_PBV_TerugtrekMs := T#3000ms;
   CFG_VS_HervattingMs := T#2000ms;
   CFG_VS_LaagToerenMs := T#1000ms;
   CFG_LaagToerenMs := T#2000ms;
END_DATA_BLOCK
```

### Instance Data Block Template

```scl
DATA_BLOCK "FB_HekBesturing_DB"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
NON_RETAIN
"FB_HekBesturing"

BEGIN

END_DATA_BLOCK
```

---

## SimaticML XML Format Specification

### XML Document Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<Document>
  <Engineering version="V20" />
  <DocumentInfo>
    <Created>2026-02-02T00:00:00</Created>
    <ExportSetting>WithDefaults</ExportSetting>
    <InstalledProducts>
      <Product>
        <DisplayName>TIA Portal</DisplayName>
        <DisplayVersion>V20</DisplayVersion>
      </Product>
    </InstalledProducts>
  </DocumentInfo>

  <!-- Block definition goes here -->

</Document>
```

**Namespace for Interface sections:**
```
xmlns="http://www.siemens.com/automation/Openness/SW/Interface/v2"
```

### SW.Blocks.FB Example (Function Block)

```xml
<Document>
  <SW.Blocks.FB ID="0">
    <AttributeList>
      <AutoNumber>true</AutoNumber>
      <HeaderAuthor />
      <HeaderFamily />
      <HeaderName />
      <HeaderVersion>0.1</HeaderVersion>
      <Interface>
        <Sections xmlns="http://www.siemens.com/automation/Openness/SW/Interface/v2">
          <Section Name="Input">
            <Member Name="i_Enable" Datatype="Bool" Remanence="NonRetain" Accessibility="Public">
              <Comment>
                <MultiLanguageText Lang="en-US">Enable input</MultiLanguageText>
              </Comment>
            </Member>
          </Section>
          <Section Name="Output">
            <Member Name="q_Active" Datatype="Bool" Remanence="NonRetain" Accessibility="Public" />
          </Section>
          <Section Name="Static">
            <Member Name="internalState" Datatype="Int" Remanence="NonRetain" Accessibility="Public" />
          </Section>
          <Section Name="Temp">
            <Member Name="tempVar" Datatype="Int" />
          </Section>
        </Sections>
      </Interface>
      <MemoryLayout>Optimized</MemoryLayout>
      <Name>FB_HekBesturing</Name>
      <Number>1</Number>
      <ProgrammingLanguage>SCL</ProgrammingLanguage>
    </AttributeList>
    <ObjectList>
      <MultilingualText ID="1" CompositionName="Comment">
        <ObjectList>
          <MultilingualTextItem ID="2" CompositionName="Items">
            <AttributeList>
              <Culture>en-US</Culture>
              <Text>Gate control state machine</Text>
            </AttributeList>
          </MultilingualTextItem>
        </ObjectList>
      </MultilingualText>
      <MultilingualText ID="3" CompositionName="Title">
        <ObjectList>
          <MultilingualTextItem ID="4" CompositionName="Items">
            <AttributeList>
              <Culture>en-US</Culture>
              <Text>FB_HekBesturing</Text>
            </AttributeList>
          </MultilingualTextItem>
        </ObjectList>
      </MultilingualText>
      <SW.Blocks.CompileUnit ID="5" CompositionName="CompileUnits">
        <AttributeList>
          <NetworkSource>
            <FlgNet xmlns="http://www.siemens.com/automation/Openness/SW/NetworkSource/FlgNet/v4">
              <Parts>
                <!-- SCL source or LAD/FBD elements -->
              </Parts>
              <Wires>
                <!-- Connections between elements -->
              </Wires>
            </FlgNet>
          </NetworkSource>
          <ProgrammingLanguage>SCL</ProgrammingLanguage>
        </AttributeList>
      </SW.Blocks.CompileUnit>
    </ObjectList>
  </SW.Blocks.FB>
</Document>
```

### SW.Blocks.GlobalDB Example (Data Block)

```xml
<Document>
  <SW.Blocks.GlobalDB ID="0">
    <AttributeList>
      <AutoNumber>true</AutoNumber>
      <HeaderVersion>0.1</HeaderVersion>
      <Interface>
        <Sections xmlns="http://www.siemens.com/automation/Openness/SW/Interface/v2">
          <Section Name="Static">
            <Member Name="CFG_PBV_PauzeMs" Datatype="Time">
              <StartValue>T#500ms</StartValue>
            </Member>
            <Member Name="CFG_PBV_TerugtrekMs" Datatype="Time">
              <StartValue>T#3000ms</StartValue>
            </Member>
          </Section>
        </Sections>
      </Interface>
      <IsOnlyStoredInLoadMemory>false</IsOnlyStoredInLoadMemory>
      <MemoryLayout>Optimized</MemoryLayout>
      <Name>DB_Config</Name>
      <Number>1</Number>
      <ProgrammingLanguage>DB</ProgrammingLanguage>
    </AttributeList>
    <ObjectList>
      <MultilingualText ID="1" CompositionName="Comment">
        <ObjectList>
          <MultilingualTextItem ID="2" CompositionName="Items">
            <AttributeList>
              <Culture>en-US</Culture>
              <Text>Configuration parameters</Text>
            </AttributeList>
          </MultilingualTextItem>
        </ObjectList>
      </MultilingualText>
    </ObjectList>
  </SW.Blocks.GlobalDB>
</Document>
```

### XML ID Rules

- ID attributes must be unique within the entire XML document
- Valid range: 0 to 2147483647
- UId within networks must be unique and in range 21 to 2147483647

**Source:** [Siemens Support 109480446](https://support.industry.siemens.com/cs/document/109480446/)

---

## Import Procedure

### Method 1: SCL External Source Files (Recommended)

1. **Prepare source files** with `.scl`, `.db`, or `.udt` extensions
2. **Open TIA Portal V20** and your project
3. **Navigate to** Project Tree > PLC > External Sources
4. **Right-click** "External Sources" > "Add external file"
5. **Select** your `.scl` files
6. **Right-click** the imported file > "Generate blocks from source"
7. **Verify** blocks appear under "Program blocks"

**Source:** [TIA Portal V20 Documentation](https://docs.tia.siemens.cloud/r/en-us/v20/creating-and-managing-blocks/using-external-source-files-for-stl-and-scl/generating-blocks-from-external-source-files)

### Method 2: XML via TIA Openness API

Requires TIA Portal Openness license and programmatic access:

```csharp
// C# example using Siemens.Engineering API
using Siemens.Engineering;
using Siemens.Engineering.SW.Blocks;

// Connect to TIA Portal instance
TiaPortal tiaPortal = new TiaPortal(TiaPortalMode.WithUserInterface);
Project project = tiaPortal.Projects.Open(new FileInfo(@"C:\Project.ap20"));

// Import XML file
PlcSoftware plcSoftware = ...;
plcSoftware.BlockGroup.Blocks.Import(new FileInfo(@"C:\FB_Block.xml"),
    ImportOptions.Override);
```

### Method 3: Version Control Interface (VCI)

Available in TIA Portal V16+:

1. **Enable VCI** in Project > Settings > Version Control Interface
2. **Configure workspace** folder for XML/SCL export
3. **Place generated files** in workspace following project structure
4. **Sync from workspace** to import changes

**Source:** [SolisPLC VCI Tutorial](https://www.solisplc.com/tutorials/an-introduction-to-tia-portals-version-control-interface-using-git-and-github)

---

## S7-1212C Specific Considerations

### Supported Features

| Feature | S7-1212C Support |
|---------|------------------|
| SCL Language | YES |
| Optimized DB Access | YES (default) |
| TON/TOF/TP Timers | YES |
| PID Control | YES |
| Web Server | YES |

### Limitations

| Limitation | Impact |
|------------|--------|
| Memory | 75 KB work memory - keep program compact |
| I/O Modules | Max 2 SIWAREX modules |
| OPC UA | Not available on standard 1212C |

**Source:** [TIA Portal S7-1200 Documentation](https://docs.tia.siemens.cloud/r/simatic_s7_1200_manual_collection_enus_20/technical-specifications/cpu-1212c)

---

## SCL Syntax Reference for S7-1200

### Data Types

| Type | Description | Example |
|------|-------------|---------|
| Bool | Boolean | `TRUE`, `FALSE` |
| Int | 16-bit signed | `-32768` to `32767` |
| DInt | 32-bit signed | `-2147483648` to `2147483647` |
| Real | 32-bit float | `3.14` |
| Time | Duration | `T#1s`, `T#500ms`, `T#1h30m` |
| String | Text | `'Hello'` |

### Control Structures

```scl
// IF statement
IF condition THEN
    // code
ELSIF otherCondition THEN
    // code
ELSE
    // code
END_IF;

// CASE statement
CASE #stateVar OF
    0: // State 0
        #q_Output := FALSE;
    1: // State 1
        #q_Output := TRUE;
    ELSE
        // Default
END_CASE;

// FOR loop
FOR #i := 0 TO 10 DO
    #array[#i] := 0;
END_FOR;

// WHILE loop
WHILE #condition DO
    // code
END_WHILE;
```

### Timer Usage

```scl
// TON (On-Delay Timer)
#myTimer(IN := #startCondition, PT := T#5s);
IF #myTimer.Q THEN
    // Timer elapsed
END_IF;

// Elapsed time access
#elapsedTime := #myTimer.ET;
```

### Function Block Calls

```scl
// Call with instance DB
"FB_Instance_DB"(
    i_Input1 := #localVar,
    i_Input2 := "GlobalTag",
    q_Output1 => #result
);

// Multi-instance call within FB
#internalFB(IN := #input, PT := T#1s);
```

---

## File Structure Recommendation

```
AutomatischHek/
├── source/
│   ├── FB_HekBesturing.scl      # Main state machine FB
│   ├── FC_Veiligheid.scl        # Safety function
│   ├── FC_Outputs.scl           # Output control function
│   ├── OB_Main.scl              # OB1 main cycle
│   ├── DB_Config.db             # Configuration data block
│   ├── DB_HekData.db            # Runtime data block
│   └── UDT_StateInfo.udt        # State information type (if needed)
├── xml/                          # Alternative XML format (if needed)
│   ├── FB_HekBesturing.xml
│   └── ...
└── docs/
    └── import-guide.md          # Import instructions
```

---

## Tools and Libraries

### For SCL Generation

No special tools required - generate plain text files.

### For XML Generation (if needed)

| Tool | Language | Notes |
|------|----------|-------|
| [tia-portal-xml-generator](https://github.com/Repsay/tia-portal-xml-generator) | Python | DB, FB, OB support |
| [TiaUtilities](https://github.com/Parozzz/TiaUtilities) | C# | LAD only, V16-V19 |
| [CodeGeneratorOpenness](https://github.com/mking2203/CodeGeneratorOpenness) | C# | Full block types |

### XSD Schema Location

Schema files are included with TIA Portal installation:
```
C:\Program Files\Siemens\Automation\Portal V20\PublicAPI\V20\
```

---

## Version Compatibility

| TIA Portal Version | SCL Import | XML Import | VCI |
|--------------------|------------|------------|-----|
| V12 | YES | Limited | NO |
| V13 SP1+ | YES | YES (SCL blocks) | NO |
| V16+ | YES | YES | YES |
| V20 | YES | YES | YES |

**Important:** When generating XML, version attributes must match target TIA Portal version.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| SCL Source Format | HIGH | Verified syntax against multiple sources |
| Import Procedure | HIGH | Documented in Siemens official docs |
| XML Structure | MEDIUM | General structure verified, full schema requires validation |
| S7-1212C Compatibility | HIGH | Confirmed SCL and optimized DB support |
| Timer Syntax | HIGH | Standard IEC 61131-3 compliant |

---

## Sources

### Official Siemens Documentation
- [TIA Portal V20 - External Source Files](https://docs.tia.siemens.cloud/r/en-us/v20/creating-and-managing-blocks/using-external-source-files-for-stl-and-scl/basics-of-using-external-source-files)
- [TIA Portal V20 - Generating Blocks from Source](https://docs.tia.siemens.cloud/r/en-us/v20/creating-and-managing-blocks/using-external-source-files-for-stl-and-scl/generating-blocks-from-external-source-files)
- [Siemens Support 109480446 - XML Block Structure](https://support.industry.siemens.com/cs/document/109480446/)
- [Siemens Support 79168964 - Import External Source Files](https://support.industry.siemens.com/cs/document/79168964/)
- [S7-1200 SCL Documentation](https://docs.tia.siemens.cloud/r/simatic_s7_1200_manual_collection_enus_20/programming-concepts/programming-language/scl)

### Community Resources
- [DMC - How to Read an Openness XML File](https://www.dmcinfo.com/latest-thinking/blog/id/9908/how-to-read-an-openness-xml-file)
- [Automation Notes - SCL Examples](https://abedgnu.github.io/Automation-Notes/chapters/PLC/Siemens/exercises-solutions.html)
- [SolisPLC - VCI Tutorial](https://www.solisplc.com/tutorials/an-introduction-to-tia-portals-version-control-interface-using-git-and-github)

### GitHub Repositories
- [tia-portal-xml-generator](https://github.com/Repsay/tia-portal-xml-generator)
- [TiaUtilities](https://github.com/Parozzz/TiaUtilities)
- [CodeGeneratorOpenness](https://github.com/mking2203/CodeGeneratorOpenness)

---

## Recommendation Summary

**For this project (Automatisch Hek):**

1. **Use SCL External Source Files** - Simpler, human-readable, direct import
2. **Generate `.scl` files** for FB_HekBesturing, FC_Veiligheid, FC_Outputs, OB_Main
3. **Generate `.db` files** for DB_Config, DB_HekData
4. **Follow import procedure** via External Sources > Generate blocks from source
5. **Keep XML as fallback** - Only if SCL import fails for specific blocks

The SCL approach aligns with the project's need for maintainable, version-controllable code that can be directly imported into TIA Portal V20.
