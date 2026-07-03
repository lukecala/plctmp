# EDI — Montaggio FBD dei 3 PLC Zelio (documento unico)

Automazione di un elettrodeionizzatore (CEDI) su **3 PLC identici**, programmati
in **ZelioSoft2 · FBD**. Questo file contiene *tutto*: hardware, mappa I/O e la
logica di ogni PLC **blocco per blocco** (tabelle: un blocco per riga).

Come si legge una tabella-logica: ogni riga è **un blocco** da posare sul foglio
FBD. Colonne = **tipo del componente** (nome esatto nella palette) · **nome
dell'uscita** del blocco · **parametri** da impostare · **ingressi** da collegare
(per nome del segnale o del terminale). *I segnali con lo stesso nome vanno
cablati insieme, anche tra PLC diversi (via SCADA).*

---

## 1. Hardware (uguale su tutti e 3 i PLC)

| Modulo | Ruolo | I/O |
|---|---|---|
| **SR3B261BD** (base 24 VDC) | CPU + I/O di campo | 16 ingressi (di cui **6 analogici 0–10 V**), **10 uscite a relè** |
| **SR3XT43BD** (estensione) | I/O analogici | **+2 ingressi** e **+2 uscite** analogici 0–10 V |
| **SR3NET01BD** (rete) | Comunicazione SCADA | **Ethernet / Modbus TCP** (1 IP per PLC) → Vijeo Citect |

Capienza per PLC: **8 AI · 2 AO · 10 relè**. Le 2 uscite analogiche esistono
**solo** grazie all'estensione.

---

## 2. Legenda dei blocchi (palette ZelioSoft2 FBD)

| Sigla nel doc | Blocco reale | Cosa fa |
|---|---|---|
| `AN` | INGRESSO ANALOGICO | ingresso 0–10 V (sensore) |
| `NUM` | COSTANTE NUMERICA | valore fisso (soglia / setpoint) |
| `GAIN` | GAIN | uscita = (Gain · ingresso) / 100 + Offset |
| `SUB` / `ADD` | ADD/SUB (modo −/+) | sottrazione / somma di 2 analogici |
| `COMP` | COMPARATORE | confronto (≥, ≤, >, <) con **isteresi** → booleano |
| `MUX` | MUX (multiplexer) | sceglie A o B secondo il bit `sel` (0→A, 1→B) |
| `AND`/`OR`/`NOT` | funzioni booleane | logica |
| `TON` | TIMER (ritardo all'inserzione) | ritarda l'uscita di N secondi |
| `SR` | SET / RESET | memoria (latch) di un allarme |
| `Q` | USCITA A RELÈ | comanda pompa / valvola / generatore |
| `AQ` | USCITA ANALOGICA | segnale 0–10 V (estensione) |
| `RETE←` / `RETE→` | variabile Modbus | scritta da SCADA / pubblicata a SCADA |

> **Nota GAIN:** il guadagno è /100. Per moltiplicare ×20 → Gain = 2000; per ×0,5
> → Gain = 50. Così si fanno rapporti e scale senza blocco di divisione.

---

## 3. PLC 1 — IDRAULICA: Pressioni & Livelli

### Ingressi analogici (8/8)
| Terminale | Segnale |
|---|---|
| Base AI 1 | PT_ConcIn (pressione ingresso concentrato) |
| Base AI 2 | PT_DilIn (pressione ingresso diluito) |
| Base AI 3 | PT_ConcOut (pressione uscita concentrato) |
| Base AI 4 | PT_DilOut (pressione uscita diluito) |
| Base AI 5 | LT_Feed (livello Feed tank) |
| Base AI 6 | LT_Dil (livello tanica diluito) |
| Est. AI 1 | LT_ERS (livello tanica elettrodi) |
| Est. AI 2 | LT_Conc (livello tanica concentrato) |

### Uscite a relè
| Uscita | Funzione |
|---|---|
| Q1 | Pompa (Motore) |
| Q2 | Allarme pressione |
| Q3 | Allarme livello |
| Q4…QA | — riserva — |

### 1.1 Gradiente di pressione **Pdil ≥ Pconc** (sicurezza anti-contaminazione)
`PRESS_OK = (PT_DilIn ≥ PT_ConcIn) AND (PT_DilOut ≥ PT_ConcOut)`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| SUB | `d1` | A − B | `AN PT_DilIn`, `AN PT_ConcIn` |
| COMP | `bP1` | A ≥ B, isteresi | `d1`, `NUM 0` |
| SUB | `d2` | A − B | `AN PT_DilOut`, `AN PT_ConcOut` |
| COMP | `bP2` | A ≥ B, isteresi | `d2`, `NUM 0` |
| AND | `PRESS_OK` | — | `bP1`, `bP2` |
| RETE→ | `PRESS_OK` | pubblica a SCADA + PLC3 | `PRESS_OK` |
| NOT | `pAlarm` | — | `PRESS_OK` |
| Q | `Q2` | Allarme pressione | `pAlarm` |

### 1.2 Livelli 4 taniche — soglia bassa / alta
`x_LOW = LT_x < Lmin` · `x_HIGH = LT_x > Lmax`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| COMP | `x_LOW` | A < B, isteresi | `AN LT_x`, `NUM Lmin` |
| COMP | `x_HIGH` | A > B, isteresi | `AN LT_x`, `NUM Lmax` |
| RETE→ | `x_LOW / x_HIGH` | a SCADA | `x_LOW`, `x_HIGH` |

**Ripeti per x = Feed · Dil · ERS · Conc** (8 comparatori). `Feed_LOW` serve al
consenso pompa; i vari `x_HIGH` vanno a SCADA e (Dil_HIGH) al PLC3.

### 1.3 Consenso pompa
`Pompa = cmd_Pump AND (NOT Feed_LOW) AND (NOT dest_full)`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| OR | `dest_full` | — | `Dil_HIGH`, `Conc_HIGH`, `ERS_HIGH` |
| NOT | `nFeed` | — | `Feed_LOW` |
| NOT | `nFull` | — | `dest_full` |
| RETE← | `cmd_Pump` | da SCADA | (operatore) |
| AND | `runPump` | 3 ingressi | `cmd_Pump`, `nFeed`, `nFull` |
| Q | `Q1` | Pompa | `runPump` |

---

## 4. PLC 2 — ELETTRICO/QUALITÀ: Tensione, Corrente & Conducibilità

### Ingressi analogici (5/8)
| Terminale | Segnale |
|---|---|
| Base AI 1 | CT_Feed (conducibilità alimentazione = disturbo) |
| Base AI 2 | CT_Conc (conducibilità concentrato) |
| Base AI 3 | CT_Dil (conducibilità diluito — target 18 MΩ·cm) |
| Base AI 4 | VT_Gen (tensione misurata) |
| Base AI 5 | IT_Gen (corrente misurata) |
| Base AI 6 + Est. | — riserva — |

### Uscite
| Uscita | Funzione |
|---|---|
| AQ1 (Est.) | SP_I — Corrente richiesta |
| AQ2 (Est.) | SP_V — Tensione richiesta |
| Q1 | Generatore ON/OFF |
| Q2 | Allarme qualità (opzionale) |

### 2.1 Setpoint corrente **SP_I** (feedback qualità + feed-forward disturbo)
`SP_I = SP_base + Kp·(CT_Dil − CT_tgt) + Kff·CT_Feed` → clamp a `Imax`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| SUB | `ERRq` | A − B | `AN CT_Dil`, `NUM CT_tgt` |
| GAIN | `p` | Gain = Kp | `ERRq` |
| GAIN | `ff` | Gain = Kff | `AN CT_Feed` |
| ADD | `s1` | A + B | `p`, `ff` |
| ADD | `SP_I_calc` | A + B | `s1`, `NUM SP_base` |
| COMP | `OverI` | A > B | `SP_I_calc`, `NUM Imax` |
| MUX | `SP_I_cap` | sel = OverI | A:`SP_I_calc` · B:`NUM Imax` · sel:`OverI` |
| MUX | `SP_I` | sel = Man_mode | A:`SP_I_cap` · B:`RETE SP_I_man` · sel:`RETE Man_mode` |
| AQ | `AQ1` | SP_I → generatore | `SP_I` |

### 2.2 Setpoint tensione **SP_V**
`SP_V = Man ? SP_V_set(SCADA) : SP_V_auto`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| MUX | `SP_V` | sel = Man_mode | A:`NUM SP_V_auto` · B:`RETE SP_V_set` · sel:`RETE Man_mode` |
| AQ | `AQ2` | SP_V → generatore | `SP_V` |

### 2.3 Bit qualità **QUALITY_OK**
`QUALITY_OK = CT_Dil ≤ CT_ok`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| COMP | `QUALITY_OK` | A ≤ B, isteresi | `AN CT_Dil`, `NUM CT_ok` |
| RETE→ | `QUALITY_OK` | a SCADA + PLC3 | `QUALITY_OK` |

### 2.4 Generatore ON/OFF + sicurezza elettrodi
`Gen_ON = cmd_Gen AND ERS_OK AND (NOT OverV)`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| COMP | `OverV` | A > B | `AN VT_Gen`, `NUM Vmax` |
| NOT | `nOverV` | — | `OverV` |
| RETE← | `ERS_OK` | da PLC3 (via SCADA) | (PLC3) |
| RETE← | `cmd_Gen` | da SCADA | (operatore) |
| AND | `genRun` | 3 ingressi | `cmd_Gen`, `ERS_OK`, `nOverV` |
| Q | `Q1` | Generatore ON/OFF | `genRun` |
| RETE→ | `V · A` | display SCADA | `AN VT_Gen`, `AN IT_Gen` |

---

## 5. PLC 3 — ATTUAZIONE: Valvole & Portate

### Ingressi analogici (5/8)
| Terminale | Segnale |
|---|---|
| Base AI 1 | FT_Feed (portata alimentazione) |
| Base AI 2 | FT_Dil (portata diluito ~600 L/h) |
| Base AI 3 | FT_Conc (portata concentrato ~30 L/h) |
| Base AI 4 | FT_Feedback (portata ricircolo) |
| Base AI 5 | FT_ERS (portata ERS — sicurezza) |
| Base AI 6 + Est. | — riserva — |

### Uscite
| Uscita | Funzione |
|---|---|
| AQ1 (Est.) | AY_InC — ampiezza valvola concentrato |
| AQ2 (Est.) | AY_InD — ampiezza valvola diluito |
| Q1 | V_InD | 
| Q2 | V_InC |
| Q3 | V_OutD |
| Q4 | V_OutC |
| Q5 | V_OutE |
| Q6 | V_Feedback |
| Q7…QA | — riserva — |

### 3.1 Bilancio idraulico **600/30** → ampiezza valvola concentrato (P-control)
Obiettivo `Q_Dil ≈ 20·Q_Conc`.
`AY_InC = AY_bias + Kh·(FT_Dil − 20·FT_Conc)` · se `¬PRESS_OK` → `AY_safe`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| GAIN | `q20` | Gain = 2000 (×20) | `AN FT_Conc` |
| SUB | `ERR` | A − B | `AN FT_Dil`, `q20` |
| GAIN | `corr` | Gain = Kh | `ERR` |
| ADD | `AY_calc` | A + B | `corr`, `NUM AY_bias` |
| NOT | `nPress` | — | `RETE PRESS_OK` |
| MUX | `AY_auto` | sel = nPress | A:`AY_calc` · B:`NUM AY_safe` · sel:`nPress` |
| MUX | `AY_InC` | sel = Man_mode | A:`AY_auto` · B:`RETE AY_InC_man` · sel:`RETE Man_mode` |
| AQ | `AQ1` | AY_InC → valvola | `AY_InC` |

### 3.2 Ampiezza valvola diluito **AY_InD**
`AY_InD = Man ? AY_InD_set(SCADA) : AY_InD_auto` (apertura ampia)

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| MUX | `AY_InD` | sel = Man_mode | A:`NUM AY_InD_auto` · B:`RETE AY_InD_set` · sel:`RETE Man_mode` |
| AQ | `AQ2` | AY_InD → valvola | `AY_InD` |

### 3.3 Linea di feedback (ricircolo se qualità scarsa)
`V_Feedback = ¬QUALITY_OK OR cmd_Feedback`
`V_OutD = (QUALITY_OK AND ¬Dil_HIGH) OR cmd_OutD`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| NOT | `badQ` | — | `RETE QUALITY_OK` |
| OR | `vFB` | — | `badQ`, `RETE cmd_Feedback` |
| Q | `Q6` | V_Feedback | `vFB` |
| NOT | `nDilH` | — | `RETE Dil_HIGH` |
| AND | `autoD` | — | `RETE QUALITY_OK`, `nDilH` |
| OR | `vOutD` | — | `autoD`, `RETE cmd_OutD` |
| Q | `Q3` | V_OutD | `vOutD` |

### 3.4 Sicurezza ERS (flusso elettrodi sempre presente)
`ERS_OK = FT_ERS ≥ ERS_min` · `V_OutE = SystemRun OR cmd_OutE`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| COMP | `ERS_OK` | A ≥ B, isteresi | `AN FT_ERS`, `NUM ERS_min` |
| RETE→ | `ERS_OK` | a PLC2 | `ERS_OK` |
| OR | `vOutE` | — | `RETE SystemRun`, `RETE cmd_OutE` |
| Q | `Q5` | V_OutE | `vOutE` |

### 3.5 Consensi valvole ON/OFF
`V_InD/InC = cmd OR (Auto AND SystemRun)`
`V_OutC = cmd OR (Auto AND SystemRun AND ¬Conc_HIGH)`

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| AND | `autoRun` | — | `RETE Auto`, `RETE SystemRun` |
| OR | `vInD` | — | `RETE cmd_InD`, `autoRun` |
| Q | `Q1` | V_InD | `vInD` |
| OR | `vInC` | — | `RETE cmd_InC`, `autoRun` |
| Q | `Q2` | V_InC | `vInC` |
| NOT | `nConcH` | — | `RETE Conc_HIGH` |
| AND | `autoC` | — | `autoRun`, `nConcH` |
| OR | `vOutC` | — | `RETE cmd_OutC`, `autoC` |
| Q | `Q4` | V_OutC | `vOutC` |

### 3.6 Verifica valvole (diagnostica)
`FAULT_OutD` = SET quando `V_OutD AND (FT_Dil < soglia)` per **> 5 s**

| Blocco | Uscita | Parametri | Ingressi ← |
|---|---|---|---|
| COMP | `noflowD` | A < B | `AN FT_Dil`, `NUM soglia` |
| AND | `cfD` | — | `Q V_OutD (stato)`, `noflowD` |
| TON | `tD` | ritardo 5 s | `cfD` |
| SR | `FAULT_OutD` | SET = tD · RESET = ack | S:`tD` · R:`RETE ack_OutD` |
| RETE→ | `FAULT_OutD` | a SCADA | `FAULT_OutD` |

**Ripeti lo stesso schema per:** `V_OutC ↔ FT_Conc` · `V_Feedback ↔ FT_Feedback`
· `V_InC ↔ FT_Conc`.

---

## 6. Coordinamento tra i PLC (ponte via SCADA / Modbus)

| Bit | Prodotto da | Usato da |
|---|---|---|
| `PRESS_OK` | PLC1 | PLC3 (override sicurezza), SCADA |
| `QUALITY_OK` | PLC2 | PLC3 (linea feedback), SCADA |
| `ERS_OK` | PLC3 | PLC2 (abilita generatore), SCADA |
| `Dil_HIGH` (e altri livelli) | PLC1 | PLC3 (chiude V_OutD), SCADA |

Vijeo Citect legge il bit dallo slave che lo produce e lo scrive nel `RETE←` del
PLC che lo consuma. Tutte le misure (AI), gli stati (Q) e i bit sono leggibili
da SCADA; i comandi operatore (`cmd_*`, setpoint, `Auto/Man`, `SystemRun`) sono
scritti da SCADA.

---

## 7. Note per l'esame + parametri da tarare

- **Niente PID nativo** in Zelio FBD: i loop sono **P + feed-forward**
  (GAIN/SUB/ADD) o **on/off a isteresi** (COMP). Se la tua versione ha l'AFB
  “PID”, puoi sostituire il ramo GAIN. Dichiaralo in sede d'esame.
- **Nessuna divisione analogica**: i rapporti si fanno con GAIN ×N + SUB.
- **Da definire (prossimo passo):** scalatura 0–10 V ↔ unità fisiche di ogni
  sensore e i valori numerici:

| Parametro | Significato | Valore |
|---|---|---|
| `CT_tgt`, `CT_ok` | target/soglia conducibilità diluito | _da tarare_ |
| `Kp`, `Kff` | guadagni feedback / feed-forward corrente | _da tarare_ |
| `Kh`, `AY_bias`, `AY_safe` | guadagno rapporto, apertura base, apertura sicura | _da tarare_ |
| `Lmin`, `Lmax` | soglie livello taniche | _da tarare_ |
| `Imax`, `Vmax` | limiti generatore | _da tarare_ |
| `ERS_min`, `soglia` portate | flusso minimo ERS, soglia “no-flow” diagnostica | _da tarare_ |
