# Profilo Finanziario — Analisi & Gemini AI
**Data:** 2026-03-30
**File target:** `C:\Users\thomas.avallone\Downloads\finanza_avallone.html`
**Pagina:** `page-profilo` (già esistente)

---

## Obiettivo

Trasformare la pagina profilo da semplice riepilogo a strumento di analisi finanziaria personale, con 5 nuove card analitiche e integrazione Gemini AI (briefing automatico + chat contestuale).

---

## Sezione 1 — Card analisi (5 nuove card)

### 1.1 Cash flow mensile stimato

**Calcolo:**
- Stipendio lordo mensile = `lsNum('i-ral', 30316) / 12`
- Stima IRPEF + contributi ≈ 30% (forfait ragionevole per RAL ~30k)
- Netto mensile stimato = lordo × 0.70
- Impegni = rata mutuo (se > 0) + rata prestito (se > 0)
- Risparmio = PAC pmt + contributo volontario + TFR (PEN_TFR = 96.71 — contributo datoriale fisso al fondo pensione, non accessibile come liquidità mensile, incluso qui per completezza del risparmio previdenziale)
- Disponibile = netto − impegni − risparmio

**UI:** barra stacked orizzontale con 4 segmenti colorati:
- Impegni (blu)
- Risparmio (oro)
- Disponibile (verde)
- (resto = tasse, implicito)

Valori numerici sotto ogni segmento.

### 1.2 Tasso di risparmio

**Calcolo:**
- Risparmio mensile = `lsNum('pac-pmt', 0)` + `lsNum('pen-pmt', 0)` + `PEN_TFR` (96.71)
  - Nota: PEN_TFR è il versamento TFR mensile al fondo pensione (costante 96.71), incluso nel risparmio previdenziale totale ma non è liquidità disponibile al lavoratore
- Netto mensile = RAL/12 × 0.70
- Tasso = risparmio / netto × 100

**UI:** grande % centrale + barra progresso verso benchmark 20%.
- < 10%: rosso
- 10–20%: arancio
- ≥ 20%: verde

### 1.3 Gap previdenziale

**Calcolo:**
- Reddito netto attuale mensile = RAL/12 × 0.70
- Pensione totale = `lsNum('cache_inps_rend', 0)` + `lsNum('cache_pen_rend', 0)`
- Gap mensile = max(0, reddito_netto − pensione_totale)
- Tasso sostituzione = pensione / reddito_netto × 100
- Capitale aggiuntivo per colmare gap = gap × 12 × 25 (SWR 4%)

**UI:** due valori affiancati (reddito oggi / pensione stimata) + gap in rosso + capitale necessario.

### 1.4 Ottimizzazione fiscale IRPEF

**Calcolo (già parzialmente nel simulatore pensione):**
- Contributo volontario annuo = `lsNum('pen-pmt', 0)` × 12
- Limite deducibile = €5.164,57
- Margine residuo = max(0, 5164.57 − vol_annuo)
- Risparmio aggiuntivo ottenibile = margine × `lsNum('penIRPEF', 0.35)`

**UI:** barra progresso verso €5.164,57 + "potresti risparmiare ancora €X di IRPEF".

### 1.5 Score salute finanziaria

**4 componenti, punteggio 0–10 ciascuno:**

| Componente | Logica |
|---|---|
| Tasso risparmio (25%) | 0 = 0%, 10 = ≥25% |
| Copertura pensione (25%) | tasso sostituzione: 0 = <30%, 10 = ≥80% |
| Avanzamento FIRE (25%) | `lsNum('fire_pct', 0)` / 10 |
| Debiti (25%) | 10 = nessun debito, 5 = solo mutuo, 3 = mutuo+prestito, 0 = solo prestito ad alto interesse |

**Score totale** = media pesata × 10 → 0–100
**Colore:** 0–39 rosso, 40–69 arancio, 70–100 verde
**Label testuale:** "Da migliorare" / "Sulla buona strada" / "Ottima forma"

---

## Sezione 2 — Gemini AI

### 2.1 Configurazione API Key

- Campo `type="password"` con toggle visibilità (👁)
- Pulsante "Salva" → `lsSet('gemini_key', value)` (usa prefisso `fa_` come tutte le chiavi dell'app, quindi salvata come `fa_gemini_key`)
- Se chiave assente: card briefing e chat mostrano CTA "Inserisci API key →"
- Chiave mai inviata fuori dal dispositivo (tutto client-side)

### 2.2 Briefing automatico

**Trigger:** prima visita alla pagina profilo per sessione (debounce: se il briefing è già stato generato in questa sessione — variabile module-scoped `geminiLastBriefing` — o se il timestamp in sessionStorage è < 5 minuti fa, non rifarlo. "↻ Aggiorna" bypassa sempre il debounce.)
**Modello:** `gemini-2.0-flash`
**Endpoint:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=KEY`

**System prompt:**
```
Sei un consulente finanziario personale per Thomas Avallone, 36 anni, italiano.
Hai accesso ai suoi dati finanziari reali elencati di seguito.
Rispondi sempre in italiano, con tono professionale ma diretto.
Usa i numeri specifici nei tuoi ragionamenti.
Produce un'analisi in 3 sezioni: "Punti di forza", "Aree da migliorare", "Suggerimento prioritario".
Ogni sezione: 2-3 frasi concise. Nessun elenco puntato, solo testo fluido.
```

**Contesto dati** (JSON strutturato inviato come user message):
```json
{
  "eta": 36,
  "ral_annuo": X,
  "netto_mensile_stimato": X,
  "tasso_risparmio_pct": X,
  "risparmio_mensile": X,
  "pensione_inps_mensile": X,
  "pensione_complementare_mensile": X,
  "gap_previdenziale_mensile": X,
  "tasso_sostituzione_pct": X,
  "montante_allianz": PEN_MONTANTE_REALE,
  "fire_year": X,
  "fire_anni_mancanti": X,
  "fire_completamento_pct": X,
  "contributo_volontario_annuo": X,
  "margine_irpef_residuo": X,
  "score_salute": X,
  "ha_mutuo": bool,
  "ha_prestito": bool,
  "pac_attivo": bool
}
```
Nota: `montante_allianz` viene popolato dalla costante JS `PEN_MONTANTE_REALE` (attualmente 931.24), non hardcoded nel codice.

**Gestione errori:**
- Network error (fetch fallisce): mostrare nella card "⚠ Impossibile raggiungere Gemini. Controlla la connessione."
- HTTP 400 / 403: "⚠ API key non valida o accesso negato. Verifica la chiave inserita."
- HTTP 429: "⚠ Limite di richieste raggiunto. Riprova tra qualche minuto."
- Altri errori HTTP (5xx, etc.): "⚠ Errore dal servizio Gemini (codice X). Riprova più tardi."
- In tutti i casi di errore: mostrare il messaggio nella card al posto del testo del briefing, con pulsante "↻ Aggiorna" ancora disponibile.

**UI:** card con titolo "🤖 Analisi AI · Gemini", testo del briefing (loading spinner durante chiamata), pulsante "↻ Aggiorna".

### 2.3 Chat contestuale

- Cronologia scrollabile (max 10 messaggi visibili, scroll automatico all'ultimo)
- Input textarea + pulsante Invia (anche Enter)
- Ogni richiesta invia: system prompt + contesto dati + intera history
- Bubble: utente (allineato destra, sfondo card-alt) / AI (allineato sinistra, sfondo card)
- Loading indicator durante risposta
- History azzerata alla navigazione fuori dalla pagina (no persistenza)
- Max history in memoria: 20 messaggi (per contenere token)
- Gestione errori: stessi casi del briefing, messaggio di errore come bubble AI con testo esplicativo

---

## Ordine di implementazione

1. CSS nuovi componenti (stacked bar, score gauge, chat bubbles)
2. Card cash flow
3. Card tasso di risparmio
4. Card gap previdenziale
5. Card ottimizzazione IRPEF
6. Card score salute
7. HTML API key field
8. JS buildGeminiContext()
9. JS geminiChat() — fetch API con gestione errori completa
10. Briefing automatico in profileRefresh() con debounce sessionStorage
11. Chat HTML + JS

---

## Costanti e dipendenze

- `PROF_TAX_RATE = 0.30` — forfait tasse+contributi per RAL ~30k
- `PEN_TFR = 96.71` — già definita (versamento TFR mensile al fondo pensione)
- `PEN_MONTANTE_REALE = 931.24` — già definita (montante reale Allianz al 31/12/2025)
- `PEN_DED_MAX = 5164.57` — già definita
- Tutte le chiavi localStorage usano il prefisso `fa_` (gestito automaticamente da lsSet/lsGet)
- Legge tutte le variabili da localStorage (nessun nuovo stato globale, eccetto `geminiLastBriefing` module-scoped per il debounce)
- Nessuna dipendenza esterna oltre a Gemini API (già CDN Chart.js incluso)
