# Risparmi & Patrimonio nel Profilo — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Aggiungere una lista risparmi liquidi al profilo, una card patrimonio aggregata, e bridge bidirezionale con il FIRE calculator.

**Architecture:** Tutto in `finanza_avallone.html`. Nuovi array `prof_risparmi` in localStorage (stesso pattern di `prof_impegni`). Le funzioni di analisi (`profAnalisiRefresh`, `profBuildGeminiContext`) vengono aggiornate per includere gli incrementi mensili nel segmento "risparmio". FIRE integrato via due bridge button espliciti.

**Tech Stack:** Vanilla JS, localStorage, HTML/CSS inline, Chart.js (già presente, non toccato).

---

## File coinvolti

- **Modify:** `finanza_avallone.html` — unico file dell'app
  - CSS: nessuna nuova classe, tutto riusa `imp-*`, `bk-row`, `prof-card-title`
  - HTML profilo: inserire card risparmi (dopo `#prof-impegni-card`) + card patrimonio (dopo risparmi)
  - HTML FIRE: inserire 2 bridge button
  - JS: nuove funzioni helpers, CRUD, render, bridge; aggiornamento profAnalisiRefresh / profileRefresh / profBuildGeminiContext

---

## Task 1: HTML — Card risparmi nel profilo

**File:** `finanza_avallone.html`
Inserimento: dopo la chiusura `</div>` di `#prof-impegni-card` (attualmente riga ~1586).

- [ ] **Step 1: Inserisci la card HTML dopo `<!-- PAC (visibile solo se configurato) -->`**

Inserire prima di `<!-- PAC (visibile solo se configurato) -->` (riga ~1588):

```html
  <!-- RISPARMI LIQUIDI -->
  <div class="card" id="prof-risparmi-card">
    <div class="prof-card-title">risparmi liquidi</div>
    <div class="prof-card-sub">conti, liquidità, sotto al materasso</div>
    <div id="prof-risparmi-list"></div>
    <button class="imp-add-btn" onclick="profRisparmiOpenForm()">+ aggiungi voce</button>
    <div class="imp-form" id="prof-risp-form">
      <div class="imp-form-row">
        <div class="imp-form-group full">
          <label class="imp-form-label">nome *</label>
          <input type="text" class="imp-form-input" id="prof-risp-nome" placeholder="es. Conto Fineco">
        </div>
      </div>
      <div class="imp-form-row">
        <div class="imp-form-group">
          <label class="imp-form-label">saldo attuale € *</label>
          <input type="number" class="imp-form-input" id="prof-risp-saldo" min="0" step="100" placeholder="5000">
        </div>
        <div class="imp-form-group">
          <label class="imp-form-label">incremento mensile €</label>
          <input type="number" class="imp-form-input" id="prof-risp-incremento" min="0" step="50" placeholder="0">
        </div>
      </div>
      <div class="imp-form-row">
        <div class="imp-form-group full">
          <label class="imp-form-label">note (opzionale)</label>
          <input type="text" class="imp-form-input" id="prof-risp-note" placeholder="es. per emergenze">
        </div>
      </div>
      <input type="hidden" id="prof-risp-edit-id">
      <div class="imp-form-btns">
        <button class="imp-form-cancel" onclick="profRisparmiCloseForm()">Annulla</button>
        <button class="imp-form-save" onclick="profRisparmiSave()">Salva</button>
      </div>
    </div>
  </div>

  <!-- PATRIMONIO -->
  <div class="card" id="prof-patrimonio-card">
    <div class="prof-card-title">patrimonio</div>
    <div class="prof-card-sub">liquidità inserita manualmente</div>
    <div class="bk-row" style="border-bottom:none;">
      <div class="bk-label">liquidità totale</div>
      <div class="bk-val" id="prof-pat-liquidita" style="color:var(--teal);">€ 0</div>
    </div>
    <div class="bk-row" id="prof-pat-crescita-row" style="display:none;border-bottom:none;margin-top:0.5rem;">
      <div class="bk-label">crescita mensile</div>
      <div class="bk-val" id="prof-pat-crescita" style="color:var(--teal-dim);">—</div>
    </div>
  </div>

```

- [ ] **Step 2: Verifica visiva** — apri il browser, naviga al profilo: devono comparire le due nuove card con titoli corretti, form nascosto, placeholder "Nessun risparmio inserito" non ancora visibile (il render arriva dopo il JS).

- [ ] **Step 3: Commit**
```bash
git add finanza_avallone.html
git commit -m "feat: add risparmi + patrimonio card HTML in profilo"
```

---

## Task 2: HTML — Bridge buttons nel FIRE

**File:** `finanza_avallone.html`

- [ ] **Step 1: Aggiungi bridge A (← Carica) nella card parametri FIRE**

Trovare `<div class="card-eyebrow">parametri personali</div>` (riga ~1823).
Inserire subito dopo:

```html
    <button class="sim-to-profilo-btn" style="margin-bottom:0.75rem;" onclick="fireBridgeFromProfilo()">← Carica dati reali dal profilo</button>
```

- [ ] **Step 2: Aggiungi bridge B (↓ Aggiungi) prima del disclaimer FIRE**

Trovare `<p class="note">FIRE Number integrato` (riga ~1900).
Inserire prima:

```html
  <button class="sim-to-profilo-btn" onclick="profBridgeFire()">↓ Aggiungi patrimonio al profilo come voce risparmio</button>
```

- [ ] **Step 3: Commit**
```bash
git add finanza_avallone.html
git commit -m "feat: add FIRE bridge buttons HTML"
```

---

## Task 3: JS — Helpers, storage e totali risparmi

**File:** `finanza_avallone.html` — sezione JS, subito dopo le funzioni `profGetImpegni` / `profSaveImpegni` / `profTotaleRate` (riga ~3233).

- [ ] **Step 1: Inserisci le 4 funzioni helper dopo `profTotaleRate`**

```javascript
function profGetRisparmi()           { try { return JSON.parse(lsGet('prof_risparmi','[]')); } catch(e) { return []; } }
function profSaveRisparmi(arr)        { lsSet('prof_risparmi', JSON.stringify(arr)); }
function profTotaleLiquidita()        { return profGetRisparmi().reduce((s,r) => s + (Number(r.saldo)||0), 0); }
function profTotaleIncrementoMensile(){ return profGetRisparmi().reduce((s,r) => s + (Number(r.incremento_mensile)||0), 0); }
```

- [ ] **Step 2: Verifica in console browser**

Aprire DevTools → Console, digitare:
```javascript
profGetRisparmi()       // deve restituire []
profTotaleLiquidita()   // deve restituire 0
profTotaleIncrementoMensile() // deve restituire 0
```
Expected: nessun errore, valori corretti.

- [ ] **Step 3: Commit**
```bash
git add finanza_avallone.html
git commit -m "feat: add profGetRisparmi/profSaveRisparmi/totals helpers"
```

---

## Task 4: JS — Render card risparmi e CRUD

**File:** `finanza_avallone.html` — subito dopo le funzioni `profRenderImpegniCard` / `profImpegniDelete` (riga ~3381), prima di `profBridgeMutuo`.

- [ ] **Step 1: Inserisci `profRenderRisparmiCard`**

```javascript
function profRenderRisparmiCard() {
  const list   = profGetRisparmi();
  const listEl = document.getElementById('prof-risparmi-list');
  if (!listEl) return;
  if (list.length === 0) {
    listEl.innerHTML = '<div style="font-size:12px;color:var(--text-muted);padding:0.5rem 0 0.25rem;">Nessun risparmio inserito — aggiungi la prima voce ↓</div>';
    return;
  }
  listEl.innerHTML = list.map((r, idx) => {
    const isLast = idx === list.length - 1;
    const safeId = String(r.id).replace(/'/g, "\\'");
    const parts  = [];
    if (Number(r.incremento_mensile) > 0) parts.push('+' + fmtEur(r.incremento_mensile) + '/mese');
    return `<div class="imp-row${isLast ? ' last' : ''}">
      <div class="imp-row-body">
        <div class="imp-row-tipo">${r.nome}</div>
        ${parts.length ? `<div class="imp-row-details">${parts.join(' · ')}</div>` : ''}
        ${r.note ? `<div class="imp-row-details" style="font-style:italic;opacity:0.7;">${r.note}</div>` : ''}
      </div>
      <div class="imp-row-rata" style="color:var(--teal);">${fmtEur(Math.round(Number(r.saldo)))}</div>
      <div class="imp-row-actions">
        <button class="imp-action-btn" onclick="profRisparmiOpenForm('${safeId}')" title="Modifica">✏</button>
        <button class="imp-action-btn" onclick="profRisparmiDelete('${safeId}')" title="Elimina" style="color:var(--red);">🗑</button>
      </div>
    </div>`;
  }).join('');
}
```

- [ ] **Step 2: Inserisci `profRenderPatrimonioCard`**

```javascript
function profRenderPatrimonioCard() {
  const liquidita = profTotaleLiquidita();
  const crescita  = profTotaleIncrementoMensile();
  const liqEl     = document.getElementById('prof-pat-liquidita');
  const crRow     = document.getElementById('prof-pat-crescita-row');
  const crEl      = document.getElementById('prof-pat-crescita');
  if (!liqEl) return;
  liqEl.textContent   = fmtEur(Math.round(liquidita));
  crRow.style.display = crescita > 0 ? '' : 'none';
  if (crescita > 0) crEl.textContent = '+' + fmtEur(crescita) + '/mese';
}
```

- [ ] **Step 3: Inserisci form open/close/save/delete**

```javascript
function profRisparmiOpenForm(id) {
  const form = document.getElementById('prof-risp-form');
  document.getElementById('prof-risp-nome').value       = '';
  document.getElementById('prof-risp-saldo').value      = '';
  document.getElementById('prof-risp-incremento').value = '';
  document.getElementById('prof-risp-note').value       = '';
  document.getElementById('prof-risp-edit-id').value    = '';
  if (id) {
    const item = profGetRisparmi().find(r => String(r.id) === String(id));
    if (item) {
      document.getElementById('prof-risp-nome').value       = item.nome              || '';
      document.getElementById('prof-risp-saldo').value      = item.saldo             || '';
      document.getElementById('prof-risp-incremento').value = item.incremento_mensile|| '';
      document.getElementById('prof-risp-note').value       = item.note              || '';
      document.getElementById('prof-risp-edit-id').value    = item.id;
    }
  }
  form.classList.add('open');
  form.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
}

function profRisparmiCloseForm() {
  document.getElementById('prof-risp-form').classList.remove('open');
}

function profRisparmiSave() {
  const nome  = document.getElementById('prof-risp-nome').value.trim();
  const saldo = parseFloat(document.getElementById('prof-risp-saldo').value);
  if (!nome)        { document.getElementById('prof-risp-nome').focus();  return; }
  if (isNaN(saldo)) { document.getElementById('prof-risp-saldo').focus(); return; }
  const editId = document.getElementById('prof-risp-edit-id').value;
  const voce   = {
    id:                editId || String(Date.now()),
    nome,
    saldo,
    incremento_mensile: parseFloat(document.getElementById('prof-risp-incremento').value) || 0,
    note:               document.getElementById('prof-risp-note').value.trim(),
  };
  let list = profGetRisparmi();
  if (editId) { list = list.map(r => String(r.id) === editId ? voce : r); }
  else        { list.push(voce); }
  profSaveRisparmi(list);
  profRisparmiCloseForm();
  profRenderRisparmiCard();
  profRenderPatrimonioCard();
  profAnalisiRefresh();
}

function profRisparmiDelete(id) {
  profSaveRisparmi(profGetRisparmi().filter(r => String(r.id) !== String(id)));
  profRenderRisparmiCard();
  profRenderPatrimonioCard();
  profAnalisiRefresh();
}
```

- [ ] **Step 4: Verifica manuale in browser**
  - Naviga al profilo → la card risparmi mostra il placeholder
  - Clicca "+ aggiungi voce" → il form appare
  - Inserisci nome="Conto Fineco", saldo=5000, incremento=200 → Salva
  - La riga appare con €5.000 in teal e "+€200/mese"
  - La card patrimonio mostra €5.000 e "+€200/mese"
  - Ricarica la pagina → la voce persiste (localStorage)
  - Testa modifica (matita) e cancella (cestino)

- [ ] **Step 5: Commit**
```bash
git add finanza_avallone.html
git commit -m "feat: add risparmi CRUD functions and patrimonio render"
```

---

## Task 5: JS — Integrazione con profilo e analisi

**File:** `finanza_avallone.html`

- [ ] **Step 1: Aggiorna `profileRefresh()` — aggiungi chiamate dopo `profRenderImpegniCard()`**

Trovare (riga ~3181):
```javascript
  // Impegni finanziari (lista da profilo, indipendente dai simulatori)
  profRenderImpegniCard();
```
Sostituire con:
```javascript
  // Impegni finanziari (lista da profilo, indipendente dai simulatori)
  profRenderImpegniCard();

  // Risparmi liquidi e patrimonio
  profRenderRisparmiCard();
  profRenderPatrimonioCard();
```

- [ ] **Step 2: Aggiorna `profAnalisiRefresh()` — segmento risparmio include incrementi**

Trovare (riga ~3406):
```javascript
  const risparmio  = pacPmt;            // PAC investimento libero
```
Sostituire con:
```javascript
  const risparmio  = pacPmt + profTotaleIncrementoMensile(); // PAC + risparmi liberi mensili
```

- [ ] **Step 3: Aggiorna `profBuildGeminiContext()` — aggiorna risparmio locale e aggiungi 3 campi**

Dentro `profBuildGeminiContext()`, trovare (riga ~3542):
```javascript
  const risparmio  = pacPmt;
```
Sostituire con:
```javascript
  const risparmio  = pacPmt + profTotaleIncrementoMensile(); // PAC + risparmi liberi mensili
```

Poi trovare la fine del return object prima della riga `pac_attivo:` e aggiungere prima di essa:
```javascript
    liquidita_totale:          Math.round(profTotaleLiquidita()),
    risparmio_libero_mensile:  Math.round(profTotaleIncrementoMensile()),
    n_voci_risparmio:          profGetRisparmi().length,
```

Nota: aggiornare `risparmio` in `profBuildGeminiContext` allinea `risparmio_mensile` e `tasso_risparmio_pct` con i valori reali del profilo — stessa logica già applicata in `profAnalisiRefresh`.

- [ ] **Step 4: Verifica in browser**
  - Con la voce "Conto Fineco" (incremento €200) inserita, il segmento "risparmio" nella barra cash flow deve essere aumentato di €200
  - Il segmento "disponibile" deve essersi ridotto di €200
  - Rimuovi la voce → barra torna ai valori precedenti

- [ ] **Step 5: Commit**
```bash
git add finanza_avallone.html
git commit -m "feat: wire risparmi into profileRefresh, cash flow bar and Gemini context"
```

---

## Task 6: JS — Bridge functions FIRE

**File:** `finanza_avallone.html` — aggiungere dopo `profBridgePensione` (riga ~3416).

- [ ] **Step 1: Aggiungi `fireBridgeFromProfilo` e `profBridgeFire`**

```javascript
function fireBridgeFromProfilo() {
  const pat0El = document.getElementById('fire-pat0');
  const rispEl = document.getElementById('fire-risp');
  const pat0   = Math.min(profTotaleLiquidita(), +pat0El.max);
  const risp   = Math.min(lsNum('pac-pmt', 0) + profTotaleIncrementoMensile(), +rispEl.max);
  pat0El.value = pat0; lsSet('fire-pat0', pat0); updateGradient(pat0El);
  rispEl.value = risp; lsSet('fire-risp', risp); updateGradient(rispEl);
  fireRefresh();
  profShowToast('✓ Dati reali caricati nel FIRE');
}

function profBridgeFire() {
  const saldo = lsNum('fire-pat0', 0);
  let   list  = profGetRisparmi();
  const idx   = list.findIndex(r => r._fire_bridge);
  const voce  = {
    id:                 idx >= 0 ? list[idx].id : String(Date.now()),
    nome:               'FIRE — patrimonio attuale',
    saldo,
    incremento_mensile: 0,
    note:               '',
    _fire_bridge:       true,
  };
  if (idx >= 0) { list[idx] = voce; } else { list.push(voce); }
  profSaveRisparmi(list);
  profShowToast('✓ Patrimonio aggiunto al profilo');
}
```

- [ ] **Step 2: Verifica bridge A (← Carica dal profilo)**
  - Vai al profilo, aggiungi voce risparmi con saldo €10.000, incremento €300
  - Torna al FIRE → clicca "← Carica dati reali dal profilo"
  - `fire-pat0` deve diventare 10.000, `fire-risp` deve diventare 300 + pac-pmt
  - Il grafico FIRE si aggiorna
  - Toast "✓ Dati reali caricati nel FIRE" appare

- [ ] **Step 3: Verifica bridge B (↓ Aggiungi al profilo)**
  - Nel FIRE, imposta `fire-pat0` a €15.000 con lo slider
  - Clicca "↓ Aggiungi patrimonio al profilo come voce risparmio"
  - Vai al profilo → nella card risparmi appare "FIRE — patrimonio attuale · €15.000"
  - Ripeti: modifica il valore a €20.000 e ri-clicca → la voce viene aggiornata (non duplicata)

- [ ] **Step 4: Commit e push**
```bash
git add finanza_avallone.html
git commit -m "feat: add fireBridgeFromProfilo and profBridgeFire bridge functions"
git push
```
