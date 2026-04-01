# Spec: Risparmi & Patrimonio nel Profilo

**Data**: 2026-04-01
**Stato**: approvato

---

## Contesto

Il profilo personale già traccia impegni finanziari (debiti/affitto) e il contributo pensionistico. Manca la capacità di registrare i risparmi liquidi ("sotto al materasso") e visualizzare un patrimonio aggregato. Il FIRE calculator ha `fire-pat0` e `fire-risp` ma restano isolati dal profilo.

---

## Obiettivo

1. Permettere l'inserimento manuale di voci di risparmio liquido nel profilo (lista strutturata)
2. Mostrare un patrimonio aggregato basato solo su valori inseriti manualmente o bridged esplicitamente
3. Integrare il FIRE calculator con il profilo tramite bridge bidirezionale esplicito

---

## 1. Lista Risparmi nel Profilo

### Storage
- Chiave localStorage: `prof_risparmi` (via `lsSet`/`lsGet`, prefisso `fa_` automatico)
- Struttura voce:
  ```json
  {
    "id": "1712000000000",
    "nome": "Conto Fineco",
    "saldo": 5000,
    "incremento_mensile": 200,
    "note": ""
  }
  ```
- `incremento_mensile` e `note` sono opzionali (default 0 / stringa vuota)

### UX Card "risparmi" nel profilo
- Posizione: dopo la card "impegni finanziari", prima del patrimonio
- Stessa struttura della card impegni: lista voci + form inline
- Se lista vuota: mostra placeholder "Nessun risparmio inserito — aggiungi la prima voce ↓"
- Ogni riga mostra: nome, saldo formattato, incremento mensile (se > 0), note (se presente)
- Azioni per riga: modifica (matita) / elimina (×)
- Form campi:
  - Nome (text, obbligatorio)
  - Saldo attuale € (number, obbligatorio)
  - Incremento mensile € (number, opzionale)
  - Note (text, opzionale)
  - Tasto "Salva"
- Funzioni JS:
  - `profGetRisparmi()` — legge array da localStorage
  - `profSaveRisparmi(arr)` — salva array
  - `profTotaleIncrementoMensile()` — Σ incrementi mensili
  - `profTotaleLiquidita()` — Σ saldi
  - `profRenderRisparmiCard()` — render lista
  - `profRisparmiOpenForm(id?)` — apre form (vuoto o pre-compilato per edit)
  - `profRisparmiSave()` — salva voce (add o update)
  - `profRisparmiDelete(id)` — elimina voce

### Impatto sul cash flow bar
- Segmento "risparmio" (teal): da `pac-pmt` a `pac-pmt + profTotaleIncrementoMensile()`
- Semantica: `incremento_mensile` è denaro che esce mensilmente dal reddito disponibile verso un conto risparmio — è un cash outflow reale, non una proiezione. Giusto includerlo nel segmento risparmio insieme a PAC.
- `disponibile` si riduce coerentemente: `netto - impegni - previdenza - (pac-pmt + incrementi)`
- Sia `profAnalisiRefresh()` che `profBuildGeminiContext()` aggiornati

---

## 2. Card Patrimonio nel Profilo

### Posizione
Dopo la card risparmi, prima della sezione analisi AI.

### Contenuto
- **Liquidità totale**: `profTotaleLiquidita()` — somma saldi risparmi
- **Crescita mensile**: `profTotaleIncrementoMensile()` — mostrata solo se > 0
- Nessun valore simulato automatico — solo dati inseriti manualmente o bridged esplicitamente
- Nessun grafico né proiezione — valori reali di oggi

### Gemini context
Aggiunti a `profBuildGeminiContext()`:
- `liquidita_totale`: somma saldi
- `risparmio_libero_mensile`: somma incrementi mensili
- `n_voci_risparmio`: numero voci

---

## 3. Integrazione FIRE — Bridge Bidirezionale

### Tasto A: "← Carica dati reali dal profilo" (FIRE → legge profilo)
- Posizione: nella card "parametri personali" del FIRE, sopra i controlli
- Comportamento: imposta `fire-pat0` = `profTotaleLiquidita()`, `fire-risp` = `pac-pmt + profTotaleIncrementoMensile()`
- I valori vengono clampati al `max` dei rispettivi slider prima di essere assegnati (evita overflow silenzioso)
- Aggiorna i slider, chiama `updateGradient()` su entrambi, poi `fireRefresh()`
- Funzione JS: `fireBridgeFromProfilo()`

### Tasto B: "↓ Aggiungi al profilo come voce risparmio" (FIRE → scrive profilo)
- Posizione: in fondo alla pagina FIRE, prima del disclaimer (stesso pattern mutuo/prestito)
- Comportamento: crea o aggiorna una voce in `prof_risparmi` con `nome: "FIRE — patrimonio attuale"`, `saldo: fire-pat0`, `incremento_mensile: 0`
  - Il valore `fire-pat0` è inserito manualmente dall'utente nel FIRE, non è un output simulato — è l'utente che stima il proprio patrimonio attuale
  - Deduplicazione tramite flag `_fire_bridge: true` nella voce (non tramite nome, che è fragile): se esiste già una voce con quel flag, viene aggiornata; altrimenti ne viene creata una nuova
- Funzione JS: `profBridgeFire()`

---

## 4. Vincoli di design

- Nessun accoppiamento automatico tra FIRE e profilo — tutto esplicito via tasto
- `prof_risparmi` è totalmente indipendente da `fire-pat0`, `fire-risp`, `pac-pmt`
- Il bridge "← Carica dal profilo" è un'azione one-shot sul FIRE, non una sincronizzazione continua
- Il patrimonio nel profilo non include mai valori simulati non confermati

---

## 5. Chiavi localStorage coinvolte

| Chiave (bare) | Tipo | Descrizione |
|---|---|---|
| `prof_risparmi` | JSON array | Lista voci risparmio del profilo |
| `fire-pat0` | number | Patrimonio attuale nel FIRE (slider) |
| `fire-risp` | number | Risparmio mensile nel FIRE (slider) |
| `pac-pmt` | number | Versamento mensile PAC (condiviso con simulatore) |
| `prof_pen_pmt` | number | Contributo pensionistico reale (già decoupled) |
