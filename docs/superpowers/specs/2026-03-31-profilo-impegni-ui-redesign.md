# Profilo — Impegni Finanziari + UI Redesign + AI Visual
**Data:** 2026-03-31
**File target:** `C:\Users\thomas.avallone\Downloads\finanza_avallone.html`
**Pagina principale:** `page-profilo`

---

## Obiettivo

Tre interventi coordinati sulla pagina profilo:
1. **Impegni finanziari** — lista strutturata add/edit/delete indipendente dai simulatori
2. **UI redesign** — titoli card leggibili, cash flow a 4 segmenti concettualmente corretti
3. **AI visual** — parsing sezioni Gemini e rendering strutturato

---

## 1. Impegni finanziari

### 1.1 Modello dati

Array JSON salvato in `localStorage` sotto la chiave `fa_prof_impegni` (via `lsSet`/`lsGet`).

Ogni voce:
```json
{
  "id": "1711900000000",
  "tipo": "Mutuo casa",
  "rata": 650,
  "importo": 120000,
  "tasso": 3.5,
  "mesi_rimanenti": 240,
  "giorno_addebito": 15
}
```

Campi:
- `id` — `String(Date.now())`, usato come chiave univoca
- `tipo` — stringa da select predefinita o "Altro: …"
- `rata` — **obbligatorio**, numero positivo
- `importo`, `tasso`, `mesi_rimanenti`, `giorno_addebito` — tutti opzionali

Tipi predefiniti: `['Affitto', 'Mutuo casa', 'Prestito auto', 'Prestito personale', 'Finanziamento', 'Leasing', 'Altro']`

Persistenza:
```js
function profGetImpegni()     { return JSON.parse(lsGet('fa_prof_impegni', '[]')); }
function profSaveImpegni(arr) { lsSet('fa_prof_impegni', JSON.stringify(arr)); }
function profTotaleRate()     { return profGetImpegni().reduce((s,i) => s + (i.rata||0), 0); }
```

Nota: `lsGet`/`lsSet` aggiungono automaticamente il prefisso `fa_` — passare `'fa_prof_impegni'` come chiave produrrà `fa_fa_prof_impegni`. Usare invece il bare key `'prof_impegni'`:
```js
function profGetImpegni()     { return JSON.parse(lsGet('prof_impegni', '[]')); }
function profSaveImpegni(arr) { lsSet('prof_impegni', JSON.stringify(arr)); }
```
La chiave effettiva in localStorage sarà `fa_prof_impegni`, coerente con il namespace dell'app.

### 1.2 Calcolo residuo stimato

Il residuo è calcolato come **valore attuale delle rate rimanenti** (PV annuity), non come differenza contabile:

- Se `tasso > 0` e `mesi_rimanenti > 0`:
  ```js
  const r = tasso / 100 / 12;
  residuo = rata * (1 - Math.pow(1 + r, -mesi_rimanenti)) / r;
  ```
- Se `tasso == 0` o assente ma `mesi_rimanenti > 0`: approssimazione lineare `rata × mesi_rimanenti`
- Altrimenti: non mostrato

**Nota:** Non mostrare mai `importo` come fallback per il residuo — `importo` è il capitale originale, non quello residuo. Il residuo va mostrato solo se calcolabile. Per le voci di tipo "Affitto" il residuo non viene mai mostrato (non ha senso concettuale).

### 1.3 UI lista + form inline

**Layout lista:**
```
┌─────────────────────────────────────────┐
│ impegni finanziari                      │  ← prof-card-title
│ DEBITI E RATE ATTIVE                    │  ← prof-card-sub
├─────────────────────────────────────────┤
│ Affitto           → € 900/mese    ✏ 🗑 │
│ giorno 1                                │
├─────────────────────────────────────────┤
│ Mutuo casa        → € 650/mese    ✏ 🗑 │
│ 3,5% · 240 mesi · residuo ~€ 98k       │
├─────────────────────────────────────────┤
│         [+ aggiungi impegno]            │
└─────────────────────────────────────────┘
```

**Form inline** (espanso sotto la lista su click `+` o ✏ edit; la lista rimane visibile sopra):

Campi:
1. Select tipo (valori in `TIPI_IMPEGNO`); se "Altro" mostra un secondo `<input type="text">` per label libera
2. Input rata mensile * (required, `type="number"`, min=1)
3. Input importo finanziato (nascosto se tipo === 'Affitto')
4. Input tasso % annuo (nascosto se tipo === 'Affitto')
5. Input mesi rimanenti (nascosto se tipo === 'Affitto')
6. Input giorno addebito (type="number", min=1, max=28)
7. Preview residuo calcolato in real-time (visibile solo se calcolabile e tipo !== 'Affitto')
8. Pulsanti **Salva** / **Annulla**

**Validazione:** Salva abilitato solo se `rata > 0`. Gli altri campi, se presenti, devono essere numeri positivi.

**Comportamento edit:** cliccando ✏ si popola il form con i valori dell'impegno selezionato. `id` viene conservato. Salva sovrascrive la voce esistente.

**Comportamento delete:** cliccando 🗑 rimuove immediatamente la voce dall'array (no conferma, per semplicità).

### 1.4 Scollegamento dai simulatori — migrazioni richieste

Le seguenti funzioni devono essere aggiornate per usare `profGetImpegni()` / `profTotaleRate()` invece di `lsNum('m-imp',0)` / `lsNum('p-imp',0)`:

**`profileRefresh()` — blocco impegni (da rimuovere/sostituire):**
Il blocco esistente che legge `m-imp`, `p-imp` e renderizza `prof-impegni-card` deve essere rimosso. Al suo posto: chiamata a `profRenderImpegniCard()` (nuova funzione che renderizza la lista da `profGetImpegni()`).

**`profAnalisiRefresh()` — cash flow e score:**

- `impegni` ora = `profTotaleRate()`
- Rimuovere lettura di `lsNum('m-imp',0)`, `lsNum('p-imp',0)`, `lsNum('m-tasso',3)`, etc.
- La logica `cDebt` (score salute) va riscritta come segue:
  ```js
  const imp = profGetImpegni();
  const prestitiCount = imp.filter(i => i.tipo !== 'Affitto').length;
  const hasAffitto    = imp.some(i => i.tipo === 'Affitto');
  // Affitto non conta come debito per il debt score (è un costo fisso, non un passivo)
  let cDebt;
  if (prestitiCount === 0)       cDebt = 10;
  else if (prestitiCount === 1)  cDebt = 5;
  else if (prestitiCount === 2)  cDebt = 3;
  else                            cDebt = 1;
  ```

**`profBuildGeminiContext()` — flag impegni:**
- Rimuovere `ha_mutuo`, `ha_prestito`
- Aggiungere:
  ```js
  const imp = profGetImpegni();
  ha_affitto:  imp.some(i => i.tipo === 'Affitto'),
  ha_mutuo:    imp.some(i => i.tipo === 'Mutuo casa'),
  ha_prestiti: imp.some(i => ['Prestito auto','Prestito personale','Finanziamento','Leasing'].includes(i.tipo)),
  n_impegni:   imp.length,
  totale_rate_mensili: Math.round(profTotaleRate()),
  ```

### 1.5 Bridge simulatori → profilo

In fondo alle pagine `page-mutuo` e `page-prestito`, pulsante discreto (stile `gemini-refresh-btn` o simile):

```
↓ Aggiungi al profilo come impegno reale
```

**Mutuo:**
```js
const rata = profCalcRataMutuo(lsNum('m-imp',0), lsNum('m-tasso',3), lsNum('m-dur',20));
// profCalcRataMutuo usa piano francese (ammortamento a rata costante)
const voce = {
  id: String(Date.now()),
  tipo: 'Mutuo casa',
  rata: Math.round(rata),
  importo: lsNum('m-imp', 0),
  tasso: lsNum('m-tasso', 3),
  mesi_rimanenti: Math.round(lsNum('m-dur', 20) * 12),
  giorno_addebito: null
};
```

**Prestito:**
```js
const rata = profCalcRataPrestito(lsNum('p-imp',0), lsNum('p-tan',7), lsNum('p-dur',36));
const voce = {
  id: String(Date.now()),
  tipo: 'Prestito personale',
  rata: Math.round(rata),
  importo: lsNum('p-imp', 0),
  tasso: lsNum('p-tan', 7),
  mesi_rimanenti: lsNum('p-dur', 36),
  giorno_addebito: null
};
```

Aggiunge sempre come nuovo elemento (non sovrascrive). Dopo il salvataggio: toast breve "✓ Aggiunto al profilo" (div temporaneo che scompare dopo 2s).

---

## 2. Card titles redesign

Ogni card del profilo passa da `card-eyebrow` come unico label a un header strutturato. Il `card-eyebrow` esistente rimane come pattern generale dell'app; nelle card del profilo viene sostituito da:

```html
<div class="prof-card-header">
  <div class="prof-card-title">titolo principale</div>
  <div class="prof-card-sub">contesto · dettagli</div>
</div>
```

CSS da aggiungere:
```css
.prof-card-title {
  font-size: 16px; font-weight: 500; color: var(--text);
  letter-spacing: -0.01em; margin-bottom: 2px;
}
.prof-card-sub {
  font-family: var(--mono); font-size: 10px; color: var(--text-muted);
  letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 1rem;
}
```

Titoli e sottotitoli per ogni card del profilo:

| Card | `prof-card-title` | `prof-card-sub` |
|---|---|---|
| Anagrafica | Thomas Avallone | profilo personale |
| Previdenza | pensione stimata | INPS · complementare |
| FIRE | indipendenza finanziaria | obiettivo · completamento |
| Impegni | impegni finanziari | debiti e rate attive |
| PAC | piano di accumulo | capitale stimato · scenario base |
| Cash flow | cash flow mensile | stima netto disponibile |
| Tasso risparmio | tasso di risparmio | obiettivo 20% del netto |
| Gap previdenziale | gap previdenziale | reddito oggi vs pensione |
| IRPEF | ottimizzazione fiscale | deducibilità fondo pensione |
| Score | salute finanziaria | score composito 0–100 |
| API key | Gemini API key | configurazione |
| Briefing | analisi AI | briefing finanziario personale |
| Chat | chat finanziaria | assistente contestuale |

---

## 3. Cash flow a 4 segmenti

Barra stacked aggiornata da 3 a 4 segmenti. Nuova variabile CSS: `--teal: #4AB8A0` (aggiunta in `:root`).

Segmenti in ordine:

| Pos | ID segmento | ID label valore | Etichetta | Colore | Calcolo |
|---|---|---|---|---|---|
| 1 | `prof-cf-seg-imp`  | `prof-cf-val-imp`  | impegni    | `--blue`  | `profTotaleRate()` |
| 2 | `prof-cf-seg-prev` | `prof-cf-val-prev` | previdenza | `--gold`  | `PEN_TFR + lsNum('pen-pmt', 0)` |
| 3 | `prof-cf-seg-risp` | `prof-cf-val-risp` | risparmio  | `--teal`  | `lsNum('pac-pmt', 0)` |
| 4 | `prof-cf-seg-disp` | `prof-cf-val-disp` | disponibile| `--green` | `netto - imp - prev - risp` |

I 4 segmenti e le 4 label sostituiscono i 3 esistenti sia nell'HTML che in `profAnalisiRefresh()`.

---

## 4. AI visual — parsing sezioni

La funzione `profBriefing()` aggiorna il rendering del testo ricevuto. Il contenitore `#prof-briefing-text` passerà da `textContent` a `innerHTML`.

**Funzione di parsing:**
```js
function profParseBriefing(text) {
  const SECTIONS = [
    { key: 'punti di forza',           icon: '💪', color: 'var(--green)'  },
    { key: 'aree da migliorare',        icon: '⚠',  color: 'var(--orange)' },
    { key: 'suggerimento prioritario',  icon: '💡', color: 'var(--gold)'   },
  ];
  // Regex: riga che inizia con qualsiasi combinazione di emoji, numeri, *, #, spazi
  // seguita dal titolo atteso (case-insensitive)
  // Esempio match: "1. Punti di forza", "**Punti di forza**", "💪 Punti di forza:"
  const headerRegex = /^[\d\.\s\*#💪⚠️💡]*\s*(punti di forza|aree da migliorare|suggerimento prioritario)\s*[:*]?\s*$/im;

  // Split sul testo cercando le righe-header
  const lines = text.split('\n');
  const segments = []; // { title, icon, color, body }
  let current = null;

  for (const line of lines) {
    const trimmed = line.trim();
    let matched = null;
    for (const s of SECTIONS) {
      const re = new RegExp('^[\\d\\.\\s\\*#💪⚠️💡]*\\s*' + s.key + '\\s*[:\\*]?\\s*$', 'i');
      if (re.test(trimmed)) { matched = s; break; }
    }
    if (matched) {
      if (current) segments.push(current);
      current = { title: matched.key, icon: matched.icon, color: matched.color, lines: [] };
    } else if (current) {
      current.lines.push(line);
    }
  }
  if (current) segments.push(current);

  if (segments.length === 0) return null; // fallback

  return segments.map(s => {
    const title = s.title.charAt(0).toUpperCase() + s.title.slice(1);
    const body     = s.lines.join('\n').trim();
    const safeBody = body.replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/\n/g,'<br>');
    return `<div class="gem-section">
      <div class="gem-section-header" style="color:${s.color}">
        <span>${s.icon}</span>
        <span class="gem-section-title">${title}</span>
      </div>
      <div class="gem-section-divider"></div>
      <div class="gem-section-body">${safeBody}</div>
    </div>`;
  }).join('');
}
```

In `profBriefing()`, dopo aver ricevuto `text`:
```js
const parsed = profParseBriefing(text);
if (parsed) {
  textEl.innerHTML = parsed;
} else {
  // fallback: plain text
  textEl.innerHTML = '<div style="white-space:pre-wrap;font-size:13px;line-height:1.7;color:var(--text-dim)">'
    + text.replace(/</g,'&lt;') + '</div>';
}
textEl.style.display = '';
```

CSS aggiuntivo:
```css
.gem-section { margin-bottom: 1.25rem; }
.gem-section:last-child { margin-bottom: 0; }
.gem-section-header {
  display: flex; align-items: center; gap: 0.4rem;
  font-family: var(--mono); font-size: 11px; font-weight: 600;
  letter-spacing: 0.08em; text-transform: uppercase; margin-bottom: 0.375rem;
}
.gem-section-divider { height: 1px; background: var(--border); margin-bottom: 0.5rem; }
.gem-section-body { font-size: 13px; line-height: 1.7; color: var(--text-dim); }
```

---

## Ordine di implementazione

1. **CSS**: aggiungere `--teal`, `.prof-card-title/sub`, `.gem-section-*`, campo form impegni
2. **JS helpers**: `profGetImpegni()`, `profSaveImpegni()`, `profTotaleRate()`, `profParseBriefing()`
3. **HTML card impegni**: sostituire il vecchio `prof-impegni-card` con la nuova struttura lista + form inline
4. **JS card impegni**: `profRenderImpegniCard()`, `profImpegniOpenForm(id?)`, `profImpegniSave()`, `profImpegniDelete(id)`
5. **Aggiornare `profileRefresh()`**: rimuovere il vecchio blocco impegni (legge da `m-imp`/`p-imp`), aggiungere `profRenderImpegniCard()`
6. **Aggiornare `profAnalisiRefresh()`**: cash flow 4 segmenti, `impegni = profTotaleRate()`, nuovo `cDebt`
7. **Aggiornare `profBuildGeminiContext()`**: nuovi flag impegni
8. **Aggiornare `profBriefing()`**: usare `profParseBriefing()` + `innerHTML`
9. **Titoli card**: sostituire `card-eyebrow` con `prof-card-header` in tutto `page-profilo`
10. **Bridge simulatori**: pulsante "Aggiungi al profilo" su `page-mutuo` e `page-prestito`
11. **Commit + push**

---

## Costanti e dipendenze

```js
const TIPI_IMPEGNO = ['Affitto','Mutuo casa','Prestito auto','Prestito personale','Finanziamento','Leasing','Altro'];
```

- `--teal: #4AB8A0` aggiunto in `:root`
- `fa_prof_impegni` — nuovo namespace localStorage (array JSON)
- I simulatori (`m-imp`, `p-imp`, ecc.) rimangono **invariati** — solo le funzioni del profilo smettono di leggerli
- `profCalcRataMutuo` e `profCalcRataPrestito` già esistenti, usate dal bridge
