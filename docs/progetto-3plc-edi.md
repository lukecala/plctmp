# Automazione EDI — Progetto su 3 PLC Zelio Logic

Progetto d'esame (simulato) per l'automazione di un impianto pilota di
**elettrodeionizzazione continua (CEDI)**, Università di Palermo.
Ambiente di sviluppo: **ZelioSoft2**, programmazione in **FBD** (Function Block
Diagram), file progetto `.zm2` (uno per PLC → 3 progetti separati).

---

## 1. Architettura hardware

Tre PLC **identici**, uno per dominio funzionale. Ogni nodo:

| Modulo | Ruolo | I/O disponibili |
|---|---|---|
| **SR3B261BD** (base, 24 V DC) | CPU + I/O di campo | 16 ingressi (di cui **6 analogici 0–10 V**), **10 uscite a relè**, orologio+display |
| **SR3XT43BD** (estensione) | I/O analogici | **2 ingressi + 2 uscite** analogici 0–10 V (10 bit); ingressi anche Pt100 |
| **SR3NET01BD** (rete) | Comunicazione SCADA | **Ethernet / Modbus TCP** (1 IP per PLC) |

**Capienza per PLC:** 8 AI (6 base + 2 estensione) · 2 AO (solo estensione) ·
10 uscite relè · fino a 16 DI (6 condivisi con gli AI).

> ⚠️ Il modulo base **non ha uscite analogiche**: le uniche 2 AO del nodo
> vengono dall'estensione SR3XT43BD. Questo è il vincolo che detta la
> ripartizione (vedi §3).

Comunicazione: i 3 PLC stanno sulla stessa rete **Ethernet**; **Vijeo Citect**
li interroga in **Modbus TCP**. Ogni PLC è uno slave con la sua tabella Modbus
(stati I/Q, valori analogici, bit/word interni esposti). Il coordinamento tra i
PLC passa **attraverso lo SCADA** (Vijeo legge un bit da un PLC e lo riscrive
sull'altro) — coerente col requisito «tutto comandato da Vijeo».

---

## 2. Inventario segnali dell'impianto

Ricavato da relazione tecnica + lista componenti + sinottico SCADA.

### Misure (ingressi analogici, 0–10 V)
| Tag | Descrizione | Dominio |
|---|---|---|
| CT_Feed | Conducibilità alimentazione (disturbo) | Qualità |
| CT_Conc | Conducibilità concentrato | Qualità |
| CT_Dil | Conducibilità diluito (target 18 MΩ·cm ≈ 0.055 µS/cm) | Qualità |
| PT_ConcIn | Pressione ingresso concentrato | Idraulica |
| PT_DilIn | Pressione ingresso diluito | Idraulica |
| PT_ConcOut | Pressione uscita concentrato | Idraulica |
| PT_DilOut | Pressione uscita diluito | Idraulica |
| FT_Feed | Portata alimentazione | Idraulica |
| FT_Dil | Portata diluito (~600 L/h) | Idraulica |
| FT_Conc | Portata concentrato (~30 L/h) | Idraulica |
| FT_Feedback | Portata linea di ricircolo | Idraulica |
| FT_ERS | Portata soluzione elettrodica (sicurezza) | Sicurezza |
| LT_Feed | Livello Feed tank | Livelli |
| LT_Dil | Livello Tanica Diluito | Livelli |
| LT_ERS | Livello Tanica Elettrodi | Livelli |
| LT_Conc | Livello Tanica Concentrato | Livelli |
| VT_Gen | Tensione misurata generatore DC | Elettrico |
| IT_Gen | Corrente misurata generatore DC | Elettrico |

### Attuatori (uscite)
| Tag | Tipo | Descrizione |
|---|---|---|
| Pump | DO relè | Pompa volumetrica (Motore) |
| Gen_ON | DO relè | Abilitazione generatore DC |
| V_InD | DO relè | Valvola ingresso diluito (proporzionale, consenso ON/OFF) |
| V_InC | DO relè | Valvola ingresso concentrato (proporzionale, consenso ON/OFF) |
| V_OutD | DO relè | Valvola uscita diluito → tanica |
| V_OutC | DO relè | Valvola uscita concentrato → tanica |
| V_OutE | DO relè | Valvola uscita ERS → tanica |
| V_Feedback | DO relè | Valvola linea di ricircolo |
| AY_InD | AO 0–10V | Ampiezza apertura valvola InD |
| AY_InC | AO 0–10V | Ampiezza apertura valvola InC |
| SP_V | AO 0–10V | Tensione richiesta al generatore |
| SP_I | AO 0–10V | Corrente richiesta al generatore |

**Totali:** 18 AI · 4 AO · 8 DO. Distribuiti su 3 PLC → ~6 AI e 2 AO ciascuno.

---

## 3. Ripartizione sui 3 PLC

Criterio: raggruppare per **dominio di misura** (facile da gestire e
diagnosticare) rispettando il vincolo delle **2 sole AO** per nodo, lasciando
**I/O di riserva** su ogni modulo.

### PLC 1 — IDRAULICA: Pressioni & Livelli
Sorveglianza pressoria (sicurezza anti-contaminazione) e riempimento taniche.

**Ingressi analogici (8/8 — al limite):**
| Canale | Tag |
|---|---|
| Base AI1 | PT_ConcIn |
| Base AI2 | PT_DilIn |
| Base AI3 | PT_ConcOut |
| Base AI4 | PT_DilOut |
| Base AI5 | LT_Feed |
| Base AI6 | LT_Dil |
| Est. AI7 | LT_ERS |
| Est. AI8 | LT_Conc |

**Uscite relè:** Q1 = Pump · Q2 = Allarme pressione · Q3 = Allarme livello ·
Q4…QA **liberi (riserva)**.
**Uscite analogiche:** nessuna (2 AO estensione = riserva).

**Logica FBD:**
- **Gradiente pressorio** (requisito tassativo **Pdil ≥ Pconc**): comparatori
  `PT_DilIn ≥ PT_ConcIn` **AND** `PT_DilOut ≥ PT_ConcOut`, con isteresi →
  bit `PRESS_OK` (esposto a Modbus, verso PLC3 e SCADA). Se 0 → `Q2` allarme.
- **Livelli:** ogni `LT_x` con 2 comparatori (soglia bassa/alta) → bit
  `x_LOW` / `x_HIGH`. `Feed_LOW` → blocca la pompa; `Dil_HIGH` (tanica prodotto
  piena) → segnale a PLC3 per chiudere `V_OutD`.
- **Consenso pompa:** `Pump = cmd_SCADA AND NOT Feed_LOW AND NOT dest_full`.

### PLC 2 — ELETTRICO/QUALITÀ: Tensione/Corrente & Conducibilità
Loop di qualità: modula il generatore DC sulla conducibilità del diluito.

**Ingressi analogici (5/8):**
| Canale | Tag |
|---|---|
| Base AI1 | CT_Feed |
| Base AI2 | CT_Conc |
| Base AI3 | CT_Dil |
| Base AI4 | VT_Gen |
| Base AI5 | IT_Gen |
| Base AI6, Est. AI7–8 | **liberi (riserva)** |

**Uscite analogiche (2/2):** AO1 = SP_V · AO2 = SP_I.
**Uscite relè:** Q1 = Gen_ON · Q2 = Allarme qualità · resto riserva.

**Logica FBD:**
- **Feedback qualità (conducibilità → corrente):** target diluito
  ≈ 0.055 µS/cm. Comparatore su `CT_Dil`: se qualità scarsa (conducibilità
  alta) → aumenta `SP_I` (più migrazione ionica). **Feed-forward** dal disturbo
  `CT_Feed` via blocco **GAIN** sommato al setpoint. Controllo pseudo-P
  (vedi §5: niente PID nativo).
- **Limiti:** `SP_V`, `SP_I` clampati al massimo con blocchi limite/comparatore.
- `QUALITY_OK` = `CT_Dil ≤ soglia_OK` (con isteresi) → bit verso PLC3.
- **Sicurezza elettrodi:** `Gen_ON = cmd_SCADA AND ERS_OK` (ERS_OK da PLC3):
  il generatore non parte / si spegne se manca il flusso ERS.

### PLC 3 — ATTUAZIONE: Valvole & Portate
Comanda tutte le valvole, regola il bilancio idraulico e **verifica che le
valvole facciano ciò che devono** (comando vs portata attesa).

**Ingressi analogici (5/8):**
| Canale | Tag |
|---|---|
| Base AI1 | FT_Feed |
| Base AI2 | FT_Dil |
| Base AI3 | FT_Conc |
| Base AI4 | FT_Feedback |
| Base AI5 | FT_ERS |
| Base AI6, Est. AI7–8 | **liberi (riserva)** |

**Uscite analogiche (2/2):** AO1 = AY_InD · AO2 = AY_InC.
**Uscite relè (6/10):** Q1 = V_InD · Q2 = V_InC · Q3 = V_OutD · Q4 = V_OutC ·
Q5 = V_OutE · Q6 = V_Feedback · Q7…QA **liberi (riserva)**.

**Logica FBD:**
- **Bilancio idraulico 600/30 (≈20:1):** errore = `FT_Dil / FT_Conc` vs 20;
  poiché la pompa è a portata fissa, si agisce sull'apertura relativa delle
  proporzionali (GAIN sull'errore → `AY_InC` / `AY_InD`). **Override sicurezza:**
  se `PRESS_OK = 0` (da PLC1, Pdil<Pconc) → strozza `AY_InC` fino a ripristino
  del gradiente. La sicurezza pressoria **prevale** sul bilancio.
- **Linea di feedback (ricircolo):** se `QUALITY_OK = 0` (da PLC2) →
  `V_Feedback = ON`, `V_OutD = OFF` (ricircolo, non contamina la tanica
  diluito); se qualità OK → `V_OutD = ON`, `V_Feedback = OFF`. Comando manuale
  SCADA in OR.
- **Sicurezza ERS:** `V_OutE = ON` sempre quando pompa/generatore attivi;
  `ERS_OK = FT_ERS ≥ soglia_min` → bit verso PLC2 (se cade, PLC2 spegne il gen).
- **Verifica valvole (diagnostica, per ognuna):** temporizzatore **TON** su
  «comando attivo ma portata attesa assente»:
  - `V_OutD=ON` per >Ts **e** `FT_Dil < soglia` → `FAULT_OutD`
  - `V_OutC=ON` **e** `FT_Conc < soglia` → `FAULT_OutC`
  - `V_Feedback=ON` **e** `FT_Feedback < soglia` → `FAULT_Feedback`
  - `V_InC` aperta/`AY_InC>0` **e** nessuna portata a valle → `FAULT_InC`
  Tutti i `FAULT_*` → bit esposti a SCADA (allarmi + eventuale blocco).

---

## 4. Mappa SCADA / Modbus (Vijeo Citect)

Ogni PLC espone su Modbus TCP una tabella di oggetti Zelio. Convenzione d'uso:

**Vijeo LEGGE** (monitoraggio): tutte le misure analogiche (18 AI), stati delle
6 valvole + pompa + generatore, VT_Gen/IT_Gen, i bit di stato
`PRESS_OK · QUALITY_OK · ERS_OK`, tutti i `FAULT_*`, e i bit livello
`x_LOW/x_HIGH`.

**Vijeo SCRIVE** (comandi operatore, «tutto comandato da Vijeo»):
- Modo `AUTO/MAN` per PLC.
- `cmd_Pump`, `cmd_Gen_ON`.
- Setpoint `SP_V`, `SP_I` (Tensione/Corrente richiesta del pannello).
- In manuale: comandi diretti valvole + `AY_InD/InC`.
- Soglie e setpoint di rapporto/qualità (opzionale, se regolabili da HMI).

I «selettori» del CONTROL PANEL (Conduttività Conc/Feed, Manometri Conc/Dil
In/Out, Portata Feed/Feedback) sono solo **scelte di visualizzazione** lato
Vijeo: il PLC pubblica tutti i valori, il selettore sceglie quale mostrare.

**Ponte inter-PLC via SCADA:** `PRESS_OK` (PLC1→PLC3), `QUALITY_OK` (PLC2→PLC3),
`ERS_OK` (PLC3→PLC2). Vijeo legge il bit dal PLC sorgente e lo scrive nel word
di comando del PLC destinazione.

---

## 5. Note di realismo Zelio (importanti per l'esame)

1. **Nessun PID nativo** in FBD su Zelio Logic. I «loop PID» citati nella
   relazione si realizzano con **controllo on/off a isteresi** (comparatori) e
   **GAIN** per la componente proporzionale/feed-forward. Va dichiarato in sede
   d'esame: si approssima il PID, non lo si implementa alla lettera.
2. **Uscite analogiche solo dall'estensione** (2 per PLC). La ripartizione
   sfrutta esattamente questo budget: PLC2 = 2 setpoint generatore, PLC3 = 2
   ampiezze; PLC1 = 0 (riserva).
3. **PLC1 al limite degli 8 AI** (4 pressioni + 4 livelli): nessuna riserva
   analogica su quel nodo. Se serve margine, portare 1–2 livelli a
   **galleggianti digitali** (soglia min/max su DI) libera ingressi analogici.
4. **Risoluzione 10 bit** su AI/AO estensione: adeguata per setpoint e ampiezze;
   tenerne conto nella scalatura ingegneristica (0–10 V ↔ unità fisiche).
5. **Programmazione FBD**, un progetto `.zm2` per PLC. Riutilizzare gli stessi
   blocchi di scalatura/allarme tra i tre progetti per coerenza.

---

## 6. Prossimi passi
- [ ] Confermare le soglie fisiche (range sensori, valori di scala 0–10 V).
- [ ] Disegnare i 3 schemi FBD in ZelioSoft2 (blocchi come da §3).
- [ ] Configurare la tabella Modbus di ogni PLC + i 3 IP.
- [ ] Costruire il sinottico Vijeo Citect collegato ai 3 slave.
