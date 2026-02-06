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

## 4. State Machine Tests

**Referentie:** statemachine.mmd (canonieke bron)

### 4.1 State Gedrag Tests

**Doel:** Verifieer dat elke state de correcte motor en alarm outputs heeft.

**Procedure:** Breng het systeem in elke state en verifieer de outputs.

| Test ID | State | Motor Open | Motor Dicht | LaagToeren | Alarm | Pass/Fail | Opmerkingen |
|---------|-------|------------|-------------|------------|-------|-----------|-------------|
| SM-STATE-01 | HOMING | OFF (0) | ON (1) | ON (1) | OFF (1) | [ ] | Langzaam naar DICHT |
| SM-STATE-02 | IDLE_OPEN | OFF (0) | OFF (0) | OFF (0) | OFF (1) | [ ] | Hek volledig open |
| SM-STATE-03 | IDLE_DICHT | OFF (0) | OFF (0) | OFF (0) | ON (0) | [ ] | Fail-safe alarm actief |
| SM-STATE-04 | MOVING_OPEN | ON (1) | OFF (0) | (timer) | OFF (1) | [ ] | LaagToeren eerste 5s |
| SM-STATE-05 | MOVING_DICHT | OFF (0) | ON (1) | (timer) | OFF (1) | [ ] | LaagToeren eerste 5s |
| SM-STATE-06 | STOPPED | OFF (0) | OFF (0) | OFF (0) | OFF (1) | [ ] | In tussenpositie |
| SM-STATE-07 | VS_PAUSE | OFF (0) | OFF (0) | OFF (0) | OFF (1) | [ ] | Beam onderbroken |
| SM-STATE-08a | PBV_RETRACT (pauze fase) | OFF (0) | OFF (0) | OFF (0) | OFF (1) | [ ] | 500ms wachten |
| SM-STATE-08b | PBV_RETRACT (terugtrek fase) | ON (1) | OFF (0) | ON (1) | OFF (1) | [ ] | Richting OPEN, laag |
| SM-STATE-09 | FAULT | OFF (0) | OFF (0) | OFF (0) | OFF (1) | [ ] | Beide eindstanden |

**State Gedrag Samenvatting:**

- Totaal tests: 10
- Geslaagd: ____
- Gefaald: ____

---

### 4.2 State Transitie Tests

**Doel:** Verifieer alle state transities volgens statemachine.mmd.

**Procedure:** Breng het systeem in de voorwaarde state, activeer de trigger, en verifieer de nieuwe state.

#### Power-on Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-01 | Power On | BNS_Dicht actief (=0) | IDLE_DICHT | [ ] | |
| SM-TRANS-02 | Power On | BNS_Open actief (=0) | IDLE_OPEN | [ ] | |
| SM-TRANS-03 | Power On | Geen eindstand actief | HOMING | [ ] | |
| SM-TRANS-04 | Power On | Beide eindstanden (=0) | FAULT | [ ] | |

#### HOMING Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-05 | HOMING | Eindstand DICHT bereikt | IDLE_DICHT | [ ] | |
| SM-TRANS-06a | HOMING | PBV trigger | STOPPED | [ ] | Afgebroken |
| SM-TRANS-06b | HOMING | VS trigger (beam) | STOPPED | [ ] | Afgebroken |
| SM-TRANS-07 | HOMING | Beide eindstanden actief | FAULT | [ ] | |

#### IDLE Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-08 | IDLE_DICHT | Sleutel OPEN | MOVING_OPEN | [ ] | |
| SM-TRANS-09 | IDLE_OPEN | Sleutel DICHT | MOVING_DICHT | [ ] | |
| SM-TRANS-10 | IDLE_DICHT | Beide eindstanden actief | FAULT | [ ] | |
| SM-TRANS-11 | IDLE_OPEN | Beide eindstanden actief | FAULT | [ ] | |

#### Movement Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-12 | MOVING_OPEN | BNS_Open bereikt | IDLE_OPEN | [ ] | |
| SM-TRANS-13 | MOVING_DICHT | BNS_Dicht bereikt | IDLE_DICHT | [ ] | |
| SM-TRANS-14 | MOVING_OPEN | Drukknop | STOPPED | [ ] | |
| SM-TRANS-15 | MOVING_DICHT | Drukknop | STOPPED | [ ] | |
| SM-TRANS-16 | MOVING_OPEN | VS beam onderbroken | VS_PAUSE | [ ] | |
| SM-TRANS-17 | MOVING_DICHT | VS beam onderbroken | VS_PAUSE | [ ] | |
| SM-TRANS-18 | MOVING_OPEN | PBV bumper trigger | PBV_RETRACT | [ ] | |
| SM-TRANS-19 | MOVING_DICHT | PBV bumper trigger | PBV_RETRACT | [ ] | |
| SM-TRANS-20 | MOVING_OPEN | Beide eindstanden actief | FAULT | [ ] | |
| SM-TRANS-21 | MOVING_DICHT | Beide eindstanden actief | FAULT | [ ] | |

#### STOPPED Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-22 | STOPPED (richting was OPEN) | Drukknop | MOVING_OPEN | [ ] | Hervat richting |
| SM-TRANS-23 | STOPPED (richting was DICHT) | Drukknop | MOVING_DICHT | [ ] | Hervat richting |
| SM-TRANS-24 | STOPPED | Sleutel OPEN | MOVING_OPEN | [ ] | |
| SM-TRANS-25 | STOPPED | Sleutel DICHT | MOVING_DICHT | [ ] | |
| SM-TRANS-26 | STOPPED | Beide eindstanden actief | FAULT | [ ] | |

#### VS_PAUSE Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-27 | VS_PAUSE (richting was OPEN) | Beam OK + timer (2s) | MOVING_OPEN | [ ] | Auto-hervatting |
| SM-TRANS-28 | VS_PAUSE (richting was DICHT) | Beam OK + timer (2s) | MOVING_DICHT | [ ] | Auto-hervatting |
| SM-TRANS-29 | VS_PAUSE | PBV bumper trigger | PBV_RETRACT | [ ] | PBV > VS |
| SM-TRANS-30 | VS_PAUSE | Beide eindstanden actief | FAULT | [ ] | |

#### PBV_RETRACT Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-31 | PBV_RETRACT | Timer + BNS_Open bereikt | IDLE_OPEN | [ ] | Eindstand eerst |
| SM-TRANS-32 | PBV_RETRACT | Timer klaar (tussenpositie) | STOPPED | [ ] | Max 3s retract |
| SM-TRANS-33 | PBV_RETRACT | Beide eindstanden actief | FAULT | [ ] | |

#### FAULT Transities

| Test ID | Voorwaarde | Trigger | Verwacht State | Pass/Fail | Opmerkingen |
|---------|------------|---------|----------------|-----------|-------------|
| SM-TRANS-34 | FAULT | Sensor herstel + BNS_Dicht | IDLE_DICHT | [ ] | Auto recovery |
| SM-TRANS-35 | FAULT | Sensor herstel + BNS_Open | IDLE_OPEN | [ ] | Auto recovery |
| SM-TRANS-36 | FAULT | Sensor herstel + geen eindstand | STOPPED | [ ] | Auto recovery |

**State Transitie Samenvatting:**

- Totaal tests: 36
- Geslaagd: ____
- Gefaald: ____
- Opmerkingen: ________________________________________________________________

---

## Totaal Resultaten

| Sectie | Totaal | Geslaagd | Gefaald |
|--------|--------|----------|---------|
| 2. Input Verificatie | 7 | ____ | ____ |
| 3. Output Verificatie | 4 | ____ | ____ |
| 4.1 State Gedrag | 10 | ____ | ____ |
| 4.2 State Transities | 36 | ____ | ____ |
| 5.1 Sleutelschakelaar Tests | 11 | ____ | ____ |
| 5.2 Drukknop Tests | 6 | ____ | ____ |
| 6. Timer Tests | 26 | ____ | ____ |
| **TOTAAL** | **100** | ____ | ____ |

### Eindoordeel

- [ ] **GOEDGEKEURD** - Alle tests geslaagd
- [ ] **AFGEKEURD** - Zie opmerkingen

**Opmerkingen:**

________________________________________________________________

________________________________________________________________

________________________________________________________________

### Handtekeningen Goedkeuring

| Rol | Naam | Handtekening | Datum |
|-----|------|--------------|-------|
| Tester | ________________ | ________________ | ________ |
| Projectleider | ________________ | ________________ | ________ |
| Klant | ________________ | ________________ | ________ |

---

## 5. Control Behavior Tests

**Referentie:** FDS sectie 4.1-4.3

### 5.1 Sleutelschakelaar Tests

**Doel:** Verifieer dat de 3-standen sleutelschakelaar met veerretour correct werkt als momentary edge-triggered bediening.

| Test ID | Voorwaarde | Actie | Verwacht Resultaat | Pass/Fail | Opmerkingen |
|---------|------------|-------|-------------------|-----------|-------------|
| CTL-01a | IDLE_DICHT | Draai sleutel kort naar OPEN, laat direct los | Hek start MOVING_OPEN | [ ] | Momentary impuls |
| CTL-01b | CTL-01a uitgevoerd | Observeer beweging na loslaten sleutel | Beweging gaat autonoom door tot eindstand | [ ] | Geen continue input nodig |
| CTL-01c | IDLE_DICHT | Houd sleutel vast in OPEN positie | Zelfde resultaat als kort draaien (edge-triggered) | [ ] | Stijgende flank detectie |
| CTL-02a | IDLE_OPEN | Draai sleutel kort naar DICHT, laat direct los | Hek start MOVING_DICHT | [ ] | Momentary impuls |
| CTL-02b | CTL-02a uitgevoerd | Observeer beweging na loslaten sleutel | Beweging gaat autonoom door tot eindstand | [ ] | Geen continue input nodig |
| CTL-05a | STOPPED (laatste richting was OPEN) | Draai sleutel naar DICHT | Hek gaat MOVING_DICHT (overschrijft richting) | [ ] | Sleutel override |
| CTL-05b | STOPPED (laatste richting was DICHT) | Draai sleutel naar OPEN | Hek gaat MOVING_OPEN (overschrijft richting) | [ ] | Sleutel override |
| CTL-06a | MOVING_OPEN | Draai sleutel naar DICHT | Hek stopt (STOPPED state) | [ ] | Tegengestelde sleutel |
| CTL-06b | MOVING_DICHT | Draai sleutel naar OPEN | Hek stopt (STOPPED state) | [ ] | Tegengestelde sleutel |
| CTL-07a | Na 90s timeout in STOPPED | Houd sleutel in OPEN of DICHT | Input genegeerd zolang sleutel vastgehouden | [ ] | Key release vereist |
| CTL-07b | CTL-07a, sleutels losgelaten | Draai sleutel opnieuw naar OPEN of DICHT | Beweging wordt geaccepteerd | [ ] | Na release, input OK |

**Sleutelschakelaar Tests Samenvatting:**

- Totaal tests: 11
- Geslaagd: ____
- Gefaald: ____

---

### 5.2 Drukknop Tests

**Doel:** Verifieer dat de drukknop correct werkt als NC sensor met falling edge detectie (transitie 1→0).

| Test ID | Voorwaarde | Actie | Verwacht Resultaat | Pass/Fail | Opmerkingen |
|---------|------------|-------|-------------------|-----------|-------------|
| CTL-03a | MOVING_OPEN | Druk drukknop in (signaal 0) | Motor stopt direct (STOPPED state) | [ ] | Dalende flank |
| CTL-03b | MOVING_DICHT | Druk drukknop in | Motor stopt direct (STOPPED state) | [ ] | Dalende flank |
| CTL-03c | MOVING_OPEN | Observeer signaalverloop | Stop bij 1→0 transitie, niet bij vasthouden | [ ] | Edge detection |
| CTL-04a | STOPPED (laatste richting was OPEN) | Druk drukknop in en los | Hek hervat MOVING_OPEN | [ ] | Hervat laatste richting |
| CTL-04b | STOPPED (laatste richting was DICHT) | Druk drukknop in en los | Hek hervat MOVING_DICHT | [ ] | Hervat laatste richting |
| CTL-04c | STOPPED | Houd drukknop vast | Geen hervatting zolang knop ingedrukt | [ ] | Release vereist |

**Drukknop Tests Samenvatting:**

- Totaal tests: 6
- Geslaagd: ____
- Gefaald: ____

---

## 6. Timer Behavior Tests

**Referentie:** FDS sectie 4.4 + DB_Config parameters

### 6.1 Timer Parameters

| Parameter | Waarde | Beschrijving |
|-----------|--------|--------------|
| CFG_LaagToerenMs | 5000ms (5s) | Lage snelheid bij start beweging |
| CFG_PBV_PauzeMs | 500ms | Pauze voor PBV terugtrekken |
| CFG_PBV_TerugtrekMs | 3000ms (3s) | Max duur PBV terugtrekbeweging |
| CFG_VS_HervattingMs | 2000ms (2s) | Wachttijd na beam herstel |
| CFG_VS_LaagToerenMs | 2000ms (2s) | Lage snelheid na VS hervatting |
| CFG_BewegingTimeoutS | 90000ms (90s) | Maximale bewegingsduur |

### 6.2 Timer Tests

| Test ID | Timer Parameter | Trigger | Duur | Verificatie Methode | Pass/Fail | Opmerkingen |
|---------|-----------------|---------|------|---------------------|-----------|-------------|
| TMR-01a | CFG_LaagToerenMs | Start beweging vanuit IDLE | - | Observeer q_MS_LaagToeren actief na start | [ ] | Timer start |
| TMR-01b | CFG_LaagToerenMs | - | 5s | Meet duur met stopwatch | [ ] | Exacte timing |
| TMR-01c | CFG_LaagToerenMs | Timer verlopen | - | q_MS_LaagToeren gaat UIT (normale snelheid) | [ ] | Timer afgelopen |
| TMR-02a | CFG_PBV_PauzeMs | PBV trigger tijdens beweging | - | Motor stopt direct | [ ] | Pauze start |
| TMR-02b | CFG_PBV_PauzeMs | - | 500ms | Meet pauze duur | [ ] | Wachten |
| TMR-02c | CFG_PBV_PauzeMs | Pauze verlopen | - | Motor start richting OPEN | [ ] | Terugtrekken |
| TMR-03a | CFG_PBV_TerugtrekMs | Na PBV pauze | - | Motor start OPEN met lage snelheid | [ ] | Retract start |
| TMR-03b | CFG_PBV_TerugtrekMs | - | max 3s | Meet terugtrek duur | [ ] | Max duration |
| TMR-03c | CFG_PBV_TerugtrekMs | Timer OF BNS_Open bereikt | - | Motor stopt (wat het eerst komt) | [ ] | Stopcondities |
| TMR-04a | CFG_VS_HervattingMs | VS beam onderbroken tijdens beweging | - | Motor stopt (VS_PAUSE state) | [ ] | VS trigger |
| TMR-04b | CFG_VS_HervattingMs | Beam hersteld (1) | 2s | Meet wachttijd na beam herstel | [ ] | Resume delay |
| TMR-04c | CFG_VS_HervattingMs | Timer verlopen | - | Beweging hervat in zelfde richting | [ ] | Auto-resume |
| TMR-05a | CFG_VS_LaagToerenMs | Na VS hervatting | - | q_MS_LaagToeren actief | [ ] | Laag na VS |
| TMR-05b | CFG_VS_LaagToerenMs | - | 2s | Meet lage snelheid duur | [ ] | Timer check |
| TMR-05c | CFG_VS_LaagToerenMs | Timer verlopen | - | Normale snelheid hervat | [ ] | Timer afgelopen |
| TMR-06a | CFG_BewegingTimeoutS | Start beweging, blokkeer eindstand | - | Beweging start | [ ] | Timeout test |
| TMR-06b | CFG_BewegingTimeoutS | - | 90s | Meet totale bewegingsduur | [ ] | Max 90 seconden |
| TMR-06c | CFG_BewegingTimeoutS | Timeout bereikt | - | Motor stopt (STOPPED), key release vereist | [ ] | Zie CTL-07 |

### 6.3 Timer Pause/Resume Tests

**Doel:** Verifieer dat TONR timers pauzeren bij stop en hervatten bij restart.

| Test ID | Timer Parameter | Trigger | Duur | Verificatie Methode | Pass/Fail | Opmerkingen |
|---------|-----------------|---------|------|---------------------|-----------|-------------|
| TMR-07a | CFG_LaagToerenMs | Start beweging, wacht 2s | 2s | Observeer q_MS_LaagToeren actief | [ ] | Timer loopt |
| TMR-07b | CFG_LaagToerenMs | Druk stop knop na 2s | - | Timer pauzeert bij 2s geaccumuleerd | [ ] | TONR pause |
| TMR-07c | CFG_LaagToerenMs | Hervat beweging | - | Timer hervat vanaf 2s | [ ] | Resume |
| TMR-07d | CFG_LaagToerenMs | Na totaal 5s bewegingstijd | - | q_MS_LaagToeren gaat UIT | [ ] | Correct totaal |

### 6.4 Timer Reset Tests

**Doel:** Verifieer dat timers resetten bij richtingswisseling of bereiken eindstand.

| Test ID | Timer Parameter | Trigger | Duur | Verificatie Methode | Pass/Fail | Opmerkingen |
|---------|-----------------|---------|------|---------------------|-----------|-------------|
| TMR-08a | CFG_LaagToerenMs | Start OPEN, wacht 3s | 3s | q_MS_LaagToeren actief | [ ] | Timer loopt |
| TMR-08b | CFG_LaagToerenMs | Sleutel DICHT (richtingswisseling) | - | Timer reset naar 0 | [ ] | Direction change |
| TMR-08c | CFG_LaagToerenMs | Nieuwe beweging DICHT | - | q_MS_LaagToeren actief voor volle 5s | [ ] | Fresh timer |
| TMR-08d | CFG_LaagToerenMs | Bereik eindstand | - | Timer reset voor volgende beweging | [ ] | Endstop reset |

**Timer Tests Samenvatting:**

- Totaal tests: 26
- Geslaagd: ____
- Gefaald: ____

---

## 7. Edge Case Tests

**Referentie:** FB_HekBesturing edge cases

**Doel:** Verifieer het correct gedrag van het systeem in grensgevallen en bijzondere situaties.

### 7.1 Edge Case Verificatie

| Test ID | Voorwaarde | Actie | Verwacht Resultaat | Reden | Pass/Fail | Opmerkingen |
|---------|------------|-------|-------------------|-------|-----------|-------------|
| EDGE-01a | IDLE_DICHT | Druk drukknop in | Geen actie, hek blijft IDLE_DICHT | Geen richtingsgeheugen bij eindstand | [ ] | |
| EDGE-01b | IDLE_OPEN | Druk drukknop in | Geen actie, hek blijft IDLE_OPEN | Geen richtingsgeheugen bij eindstand | [ ] | |
| EDGE-01c | N.v.t. | Analyseer logica | Drukknop bij IDLE heeft geen effect | Niets te hervatten, sleutel vereist | [ ] | |
| EDGE-02a | IDLE_OPEN | Trigger PBV (druk rubber bumper) | Geen actie, hek blijft IDLE_OPEN | Al in veiligste positie (volledig open) | [ ] | |
| EDGE-02b | N.v.t. | Analyseer logica | PBV bij IDLE_OPEN heeft geen effect | Terugtrekken naar OPEN is zinloos, al daar | [ ] | |
| EDGE-03a | MOVING_OPEN | Wacht tot IDLE_OPEN bereikt | State wordt IDLE_OPEN | Normale transitie | [ ] | |
| EDGE-03b | IDLE_OPEN (na EDGE-03a) | Druk drukknop in | Geen actie, geen hervatting | Richtingsgeheugen gewist bij eindstand | [ ] | |
| EDGE-03c | IDLE_OPEN (na EDGE-03b) | Draai sleutel DICHT | Hek start MOVING_DICHT | Alleen sleutel kan nieuwe beweging starten | [ ] | |
| EDGE-04a | Power on, onbekende positie | Observeer | Hek start HOMING (langzaam naar DICHT) | Initialisatie procedure | [ ] | |
| EDGE-04b | HOMING actief | Druk drukknop in | Hek stopt, STOPPED state | Gebruiker veiligheids-override tijdens homing | [ ] | |
| EDGE-04c | N.v.t. | Analyseer logica | Drukknop tijdens HOMING toegestaan | Gebruiker moet kunnen ingrijpen bij homing | [ ] | |
| EDGE-05a | MOVING_DICHT | Trigger PBV | Hek gaat naar PBV_RETRACT | PBV gedetecteerd | [ ] | |
| EDGE-05b | PBV_RETRACT actief | Draai sleutel DICHT | Genegeerd, terugtrekken gaat door | Veiligheidsterugtrekking moet voltooien | [ ] | |
| EDGE-05c | PBV_RETRACT actief | Druk drukknop in | Genegeerd, terugtrekken gaat door | Veiligheidsterugtrekking moet voltooien | [ ] | |
| EDGE-05d | N.v.t. | Analyseer logica | Bediening genegeerd tijdens PBV_RETRACT | Veiligheid heeft prioriteit | [ ] | |
| EDGE-06a | MOVING_DICHT | Blokkeer lichtsluis | Hek stopt, VS_PAUSE state | VS gedetecteerd | [ ] | |
| EDGE-06b | VS_PAUSE actief | Draai sleutel (beide richtingen) | Genegeerd, wacht op beam herstel | Auto-hervatting heeft prioriteit | [ ] | |
| EDGE-06c | VS_PAUSE actief | Druk drukknop in | Genegeerd, wacht op beam herstel | Auto-hervatting heeft prioriteit | [ ] | |
| EDGE-06d | N.v.t. | Analyseer logica | Bediening genegeerd tijdens VS_PAUSE | Systeem wacht op veilige conditie | [ ] | |
| EDGE-07a | MOVING_DICHT | Blokkeer lichtsluis | Hek stopt, VS_PAUSE state | VS gedetecteerd | [ ] | |
| EDGE-07b | VS_PAUSE actief | Trigger PBV (druk rubber bumper) | Hek gaat naar PBV_RETRACT (niet blijven in VS_PAUSE) | PBV heeft hogere prioriteit dan VS | [ ] | |
| EDGE-07c | PBV_RETRACT (na EDGE-07b) | Observeer | Hek trekt terug richting OPEN | PBV > VS prioriteit bevestigd | [ ] | |
| EDGE-07d | N.v.t. | Analyseer logica | Prioriteitsvolgorde: FAULT > PBV > VS | Hogere veiligheidsprioriteit wint altijd | [ ] | |

**Edge Case Tests Samenvatting:**

- Totaal tests: 23
- Geslaagd: ____
- Gefaald: ____
- Opmerkingen: ________________________________________________________________

---

*Document gegenereerd voor FAT v1.0 - Automatisch Hek Kerk*
