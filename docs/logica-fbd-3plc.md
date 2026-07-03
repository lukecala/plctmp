# Logica FBD dei 3 PLC — schema a blocchi (ZelioSoft2)

Programmazione **FBD**, un progetto `.zm2` per PLC. Blocchi usati (tutti reali
nella palette ZelioSoft2, tab IN · LOGIC · FBD · AFB · OUT):

| Blocco | Tab | Funzione |
|---|---|---|
| `AI` / `DI` | IN | ingresso analogico 0–10V / digitale |
| `NUM` | IN | costante numerica (soglia/setpoint) |
| `MB»` | IN | valore/bit **scritto da SCADA** via Modbus TCP |
| `GAIN` | AFB | out = (Gain·in)/100 + Offset |
| `SUB` / `ADD` | AFB | sottrazione / somma analogica |
| `COMP` | AFB | comparatore (>, <, ≥, in-zona) con **isteresi** → booleano |
| `MUX` | AFB | multiplexer: sel=0→A, sel=1→B (sceglie un analogico) |
| `PWM` / `CNA` | AFB | analogico → uscita modulata / conversione D→A |
| `AND/OR/NOT` | LOGIC | logica booleana |
| `RS` | FBD | flip-flop set/reset (latch allarmi) |
| `TON` | FBD | timer ritardo all'eccitazione |
| `AQ` | OUT | uscita analogica (estensione SR3XT43BD) |
| `Q` | OUT | uscita a relè |
| `»MB` | OUT | valore/bit **pubblicato a SCADA** |

> Nota GAIN: per moltiplicare ×20 → Gain=2000; ×0.5 → Gain=50 (divisione per 100
> interna). Così si fanno rapporti/scale senza blocco di divisione.

---

## PLC 1 — IDRAULICA: Pressioni & Livelli

### A) Gradiente pressorio  Pdil ≥ Pconc  (sicurezza anti-contaminazione)
```
 ⟦AI PT_DilIn ⟧──►┌─────┐
                  │ SUB │─ d1 = PT_DilIn−PT_ConcIn ─►┌──────────┐
 ⟦AI PT_ConcIn⟧──►└─────┘                            │ COMP ≥0  │─ bP1 ─┐
                                          ⟦NUM 0⟧──► │ hyst=h   │       │
                                                     └──────────┘       │
 ⟦AI PT_DilOut⟧──►┌─────┐                                               ├─►┌─────┐
                  │ SUB │─ d2 = PT_DilOut−PT_ConcOut ►┌──────────┐      │  │ AND │─ PRESS_OK ─►⟦»MB⟧
 ⟦AI PT_ConcOut⟧─►└─────┘                             │ COMP ≥0  │─ bP2─┘  └─────┘     │
                                           ⟦NUM 0⟧──► │ hyst=h   │                     └─►┌─────┐
                                                      └──────────┘         PRESS_OK ──────►│ NOT │─►⟦Q2 AllarmeP⟧
                                                                                           └─────┘
```
`PRESS_OK` è il bit-chiave: pubblicato a SCADA e (via SCADA) inviato al PLC3.

### B) Livelli 4 taniche  (per ognuna: soglia bassa e alta)
```
 ⟦AI LT_x⟧─┬─►┌───────────────┐
           │  │ COMP < Lmin,h │─ x_LOW  ─►⟦»MB⟧
           │  └───────────────┘
           └─►┌───────────────┐
              │ COMP > Lmax,h │─ x_HIGH ─►⟦»MB⟧
              └───────────────┘
   x = Feed, Dil, ERS, Conc     (Feed_LOW → interlock pompa; Dil_HIGH → al PLC3)
```

### C) Consenso pompa  (protezione marcia a secco + tanica piena)
```
 ⟦MB» cmd_Pump⟧────────────────►┌─────┐
 Feed_LOW ───►┌─────┐           │     │
              │ NOT │──────────►│ AND │─────►⟦Q1 Pompa⟧
 dest_full ──►│ NOT │──────────►│     │
              └─────┘           └─────┘
   dest_full = Dil_HIGH OR Conc_HIGH OR ERS_HIGH   (OR gate, opzionale)
```
Tutti gli AI sono comunque leggibili da SCADA (display pressioni/livelli).
**Uscite libere Q4…QA = riserva.**

---

## PLC 2 — ELETTRICO/QUALITÀ: Tensione/Corrente & Conducibilità

### A) Setpoint corrente  SP_I = SP_base + Kp·(CT_Dil−CT_tgt) + Kff·CT_Feed
Feedback sulla qualità del diluito + feed-forward sul disturbo (conducibilità feed).
```
 ⟦AI CT_Dil⟧──►┌─────┐
               │ SUB │─ ERRq ─►┌───────────┐
 ⟦NUM CT_tgt⟧─►└─────┘         │ GAIN ·Kp  │─ p ─┐
                               └───────────┘     │
 ⟦AI CT_Feed⟧─►┌───────────┐                     ├─►┌─────┐
               │ GAIN ·Kff │─ ff ────────────────┘  │ ADD │─ s1 ─►┌─────┐
               └───────────┘                        └─────┘       │ ADD │─ SP_I_calc
                                        ⟦NUM SP_base⟧────────────►└─────┘        │
                                                                                 ▼
                                                              ┌───────────────────────┐
                                                              │ COMP > Imax → clamp    │  (limite)
                                                              └───────────────────────┘
   SP_I_calc ─A─►┌─────┐
                 │ MUX │ sel = MB» Man_mode ─►⟦AQ SP_I⟧      (auto vs manuale da SCADA)
 ⟦MB» SP_I_man⟧B►└─────┘
```
`SP_V` più semplice (impostata dall'operatore o valore fisso di lavoro):
```
 ⟦MB» SP_V_set⟧─A─►┌─────┐
                   │ MUX │ sel=Man_mode ─►⟦AQ SP_V⟧
 ⟦NUM SP_V_auto⟧─B►└─────┘
```

### B) Bit qualità
```
 ⟦AI CT_Dil⟧──►┌────────────────┐
               │ COMP ≤ CT_ok,h │─ QUALITY_OK ─►⟦»MB⟧   (→ PLC3 via SCADA)
               └────────────────┘
```

### C) Generatore ON/OFF con sicurezza elettrodi
```
 ⟦MB» cmd_Gen⟧──────────►┌─────┐
 ⟦MB» ERS_OK ⟧──────────►│ AND │─────►⟦Q1 Gen_ON⟧      (ERS_OK arriva dal PLC3)
 OverV ─►┌─────┐          │     │
         │ NOT │─────────►│     │
         └─────┘          └─────┘
   OverV:  ⟦AI VT_Gen⟧─►[COMP > Vmax]     (blocca in sovratensione)
   ⟦AI VT_Gen⟧,⟦AI IT_Gen⟧ ─►⟦»MB⟧  (display V e A sul pannello)
```

---

## PLC 3 — ATTUAZIONE: Valvole & Portate

### A) Bilancio idraulico 600/30  → ampiezza valvola concentrato (P-control)
Obiettivo: Q_Dil ≈ 20·Q_Conc. Errore positivo ⇒ apri di più il concentrato.
```
 ⟦AI FT_Conc⟧─►┌──────────────┐
               │ GAIN ·20     │─ q20 ─►┌─────┐
               │ (Gain=2000)  │        │ SUB │─ ERR=FT_Dil−20·FT_Conc ─►┌──────────┐
               └──────────────┘        │     │                          │ GAIN ·Kh │─ corr ─┐
 ⟦AI FT_Dil⟧──────────────────────────►└─────┘                          └──────────┘        │
                                                                                             ├─►┌─────┐
                                                                     ⟦NUM AY_bias⟧───────────┘  │ ADD │─ AY_calc
                                                                                                └─────┘   │
   ── Override sicurezza pressione ──                                                                     ▼
   AY_calc ─────A─►┌─────┐
                   │ MUX │ sel = NOT PRESS_OK ─ AY_InC_auto ─┐   (Pdil<Pconc ⇒ B = strozza)
 ⟦NUM AY_safe⟧──B─►└─────┘                                   │
                                                             ▼
   AY_InC_auto ─A─►┌─────┐
                   │ MUX │ sel = MB» Man_mode ─►⟦AQ AY_InC⟧
 ⟦MB» AY_InC_man⟧B►└─────┘
```
`PRESS_OK` = `MB»` (dal PLC1). Valvola diluito `AY_InD`: uguale schema con
setpoint proprio (di norma apertura ampia, ~600 L/h) o impostata da SCADA:
```
 ⟦MB» AY_InD_set⟧─A─►┌─────┐
                     │ MUX │ sel=Man_mode ─►⟦AQ AY_InD⟧
 ⟦NUM AY_InD_auto⟧─B►└─────┘
```

### B) Linea di feedback (ricircolo se qualità scarsa)
```
 ⟦MB» QUALITY_OK⟧─►┌─────┐
                   │ NOT │─ badQ ─┬─►┌─────┐
                   └─────┘        │  │ OR  │─────►⟦Q6 V_Feedback⟧
 ⟦MB» cmd_Feedback⟧──────────────────►└─────┘

 ⟦MB» QUALITY_OK⟧─────────────►┌─────┐
 ⟦MB» Dil_HIGH⟧─►┌─────┐       │ AND │─ autoD ─┬─►┌─────┐
                 │ NOT │──────►│     │         │  │ OR  │─────►⟦Q3 V_OutD⟧
                 └─────┘       └─────┘         │  └─────┘
 ⟦MB» cmd_OutD⟧──────────────────────────────────►┘
```
(Costruzione: `badQ` e `QUALITY_OK` mutuamente esclusivi ⇒ OutD e Feedback mai
aperte insieme in automatico.)

### C) Sicurezza ERS (flusso elettrodi sempre presente)
```
 ⟦AI FT_ERS⟧─►┌────────────────┐
              │ COMP ≥ ERS_min │─ ERS_OK ─►⟦»MB⟧   (→ PLC2: se cade, spegne Gen)
              └────────────────┘
 ⟦MB» SystemRun⟧─┬─►┌─────┐
 ⟦MB» cmd_OutE⟧───►│ OR  │─────►⟦Q5 V_OutE⟧        (ERS aperta di default in marcia)
                   └─────┘
```

### D) Valvole ingresso/uscita ON/OFF (consenso)
```
 ⟦MB» cmd_InD⟧ ─OR─► (Auto AND SystemRun)               ─►⟦Q1 V_InD⟧
 ⟦MB» cmd_InC⟧ ─OR─► (Auto AND SystemRun)               ─►⟦Q2 V_InC⟧
 ⟦MB» cmd_OutC⟧─OR─► (Auto AND SystemRun AND NOT Conc_HIGH) ─►⟦Q4 V_OutC⟧
```

### E) Verifica valvole (diagnostica: comando ON ma portata assente)
Per ogni valvola, `TON` filtra i transitori, `RS` mantiene l'allarme fino ad ack.
```
 ⟦Q3 V_OutD⟧──────────────►┌─────┐
 ⟦AI FT_Dil⟧─►[COMP < s]─► │ AND │─►[TON 5s]─►┌────┐
                           └─────┘            │ RS │─ FAULT_OutD ─►⟦»MB⟧
             ⟦MB» ack_OutD⟧──────────────► R  │ S  │
                                              └────┘
   Stesso schema per:
     V_OutC   & FT_Conc<s      → FAULT_OutC
     V_Feedback & FT_Feedback<s→ FAULT_Feedback
     (V_InC & AY_InC>min) & FT_Conc<s → FAULT_InC
```
**Uscite libere Q7…QA = riserva.**

---

## Coordinamento inter-PLC (ponte via SCADA/Modbus)
| Bit | Prodotto da | Consumato da |
|---|---|---|
| `PRESS_OK` | PLC1 | PLC3 (override), SCADA |
| `QUALITY_OK` | PLC2 | PLC3 (feedback), SCADA |
| `ERS_OK` | PLC3 | PLC2 (gen safety), SCADA |
| `Dil_HIGH` (e altri livelli) | PLC1 | PLC3 (chiude OutD), SCADA |

Vijeo legge il bit dallo slave sorgente e lo scrive nel `MB»` del destinatario.

## Realismo (da dichiarare all'esame)
- Loop = **P + feed-forward** (GAIN/SUB/ADD) o **on/off a isteresi** (COMP): Zelio
  FBD **non ha PID** completo. In alcune versioni SR3 c'è un AFB "PID-lite": se
  presente nel tuo ZelioSoft2 puoi sostituire il ramo GAIN con quello.
- Nessuna divisione analogica: i rapporti si fanno con GAIN (×N) + SUB + COMP.
- Scalatura 0–10V ↔ unità fisiche va impostata nei parametri di ogni `AI`/`AQ`.
