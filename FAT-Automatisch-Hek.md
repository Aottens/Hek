# Factory Acceptance Test (FAT)
## Automatisch Hek Kerk

---

## 1. Document Informatie

### 1.1 Documentgegevens

| Veld | Waarde |
|------|--------|
| Document | FAT-Automatisch-Hek |
| Versie | v1.0 |
| Datum | ______________ |
| Project | Automatisch Hek Kerk |
| PLC Platform | Siemens S7-1212C |
| Referentie FDS | FDS-Automatisch-Hek v0.2 |

### 1.2 Testpersoneel

| Rol | Naam | Handtekening | Datum |
|-----|------|--------------|-------|
| Tester | ________________ | ________________ | ________ |
| Getuige | ________________ | ________________ | ________ |
| Klant | ________________ | ________________ | ________ |

### 1.3 Apparatuur Identificatie

| Apparaat | Serienummer | Firmware Versie |
|----------|-------------|-----------------|
| PLC (S7-1212C) | ________________ | ________________ |
| HMI (indien aanwezig) | ________________ | ________________ |
| Frequentieregelaar | ________________ | ________________ |

### 1.4 Testcondities

| Conditie | Status |
|----------|--------|
| PLC programma geladen | [ ] Ja |
| I/O bedrading compleet | [ ] Ja |
| Motor aangesloten | [ ] Ja |
| Veiligheidssensoren actief | [ ] Ja |

---

## 2. Input Verificatie

**Doel:** Verifieer dat alle 7 in-scope ingangen correct zijn aangesloten en gelezen worden door de PLC.

**Procedure:** Voor elke ingang: voer de test actie uit en controleer de waarde in de PLC.

| Test ID | Tag | Adres | Test Actie | Verwacht | Pass/Fail | Opmerkingen |
|---------|-----|-------|------------|----------|-----------|-------------|
| IO-INP-01 | i_SS_SleutelSchakelaarOpen | %I0.1 | Draai sleutel naar OPEN positie | PLC toont 1 (NO: 1 = contact) | [ ] | |
| IO-INP-02 | i_DK_Bediening | %I0.2 | Druk de drukknop in | PLC toont 0 (NC: 0 = ingedrukt) | [ ] | |
| IO-INP-03 | i_SS_SleutelSchakelaarDicht | %I0.3 | Draai sleutel naar DICHT positie | PLC toont 1 (NO: 1 = contact) | [ ] | |
| IO-INP-04 | i_BNS_Open | %I0.7 | Duw hek naar OPEN eindstand | PLC toont 0 (NC: 0 = bereikt) | [ ] | |
| IO-INP-05 | i_BNS_Dicht | %I1.0 | Duw hek naar DICHT eindstand | PLC toont 0 (NC: 0 = bereikt) | [ ] | |
| IO-INP-06 | i_PBV_BeweegbaarHek | %I1.2 | Druk rubber bumper in | PLC toont 1 (NC geinverteerd: 1 = trigger) | [ ] | |
| IO-INP-07 | i_VS_Voertuig | %I1.3 | Blokkeer de lichtsluis | PLC toont 0 (0 = onderbroken) | [ ] | |

**Input Verificatie Samenvatting:**

- Totaal tests: 7
- Geslaagd: ____
- Gefaald: ____
- Opmerkingen: ________________________________________________________________

---

## 3. Output Verificatie

**Doel:** Verifieer dat alle 4 in-scope uitgangen correct worden aangestuurd door de PLC.

**Procedure:** Breng de PLC in de aangegeven toestand en observeer de actuator response.

| Test ID | Tag | Adres | Test Conditie | Verwacht | Pass/Fail | Opmerkingen |
|---------|-----|-------|---------------|----------|-----------|-------------|
| IO-OUT-01 | q_AlarmKerk | %Q0.3 | Hek in IDLE_DICHT state | Output = 0 (fail-safe: 0 = alarm AAN = hek dicht) | [ ] | |
| IO-OUT-02 | q_MS_Open | %Q0.4 | Hek in MOVING_OPEN state | Output = 1 (motor draait richting OPEN) | [ ] | |
| IO-OUT-03 | q_MS_Dicht | %Q0.5 | Hek in MOVING_DICHT of HOMING | Output = 1 (motor draait richting DICHT) | [ ] | |
| IO-OUT-04 | q_MS_LaagToeren | %Q0.6 | Tijdens HOMING of eerste 5s beweging | Output = 1 (lage snelheid actief) | [ ] | |

**Output Verificatie Samenvatting:**

- Totaal tests: 4
- Geslaagd: ____
- Gefaald: ____
- Opmerkingen: ________________________________________________________________

---
