# Mileage Log Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-page, mobile-optimised web app that logs recurring business-travel mileage claims to `localStorage`, exports them as CSV and via the iOS share sheet, and wipes them on demand.

**Architecture:** One self-contained `index.html` file with inlined CSS and JS. No build step, no runtime network calls. State lives in three `localStorage` keys (`mileage.settings`, `mileage.trips`, `mileage.dirty`). Vanilla JS organised as a flat module with a small state object and explicit `render*()` functions called after each mutation.

**Tech Stack:** HTML, CSS, vanilla JS (ES2022). `localStorage`, `navigator.share`, `<input type="date">`, `URL.createObjectURL` + `Blob` for CSV download. No frameworks. No test framework — spec explicitly rules it out; each task ends with scripted hand-verification in Safari.

**Spec:** `docs/superpowers/specs/2026-04-23-mileage-log-design.md`

**Deliberate deviation from standard TDD:** The spec rules out a test framework (personal tool, single user, single device). Each task ends with a hand-verification step in Safari with specific "tap this, expect that" instructions. The `render*()` boundary keeps pure data transforms (CSV generation, total arithmetic, month grouping) hand-testable via the JS console.

---

## File structure

All work happens in `/Users/garry/Documents/ClaudeCode/expenselogger/`:

```
expenselogger/
  index.html              # THE app — HTML shell + <style> + <script>
  manifest.webmanifest    # PWA manifest for Add-to-Home-Screen
  icon.svg                # App icon (512×512, inline SVG)
  README.md               # Usage + hand-test checklist
  docs/
    superpowers/
      specs/2026-04-23-mileage-log-design.md
      plans/2026-04-23-mileage-log.md      (this file)
```

All implementation tasks touch `index.html`. Tasks 1 and 13 also touch the sibling files.

---

## Task 1: Project scaffold

Create the four files the browser needs to load the app at all. End state: opening `index.html` in Safari shows a titled, empty page with the manifest loaded.

**Files:**
- Create: `index.html`
- Create: `manifest.webmanifest`
- Create: `icon.svg`
- Create: `README.md`
- Create: `.gitignore`

- [ ] **Step 1: Write `icon.svg`**

A flat SVG icon — a road + car silhouette on the accent colour.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
  <rect width="512" height="512" rx="96" fill="#0f766e"/>
  <path d="M128 320 L384 320 L352 224 L160 224 Z" fill="white"/>
  <circle cx="176" cy="336" r="28" fill="white"/>
  <circle cx="336" cy="336" r="28" fill="white"/>
  <rect x="224" y="384" width="64" height="16" rx="4" fill="white"/>
  <rect x="160" y="416" width="192" height="8" rx="2" fill="white" opacity="0.6"/>
</svg>
```

- [ ] **Step 2: Write `manifest.webmanifest`**

```json
{
  "name": "Mileage Log",
  "short_name": "Mileage",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0f766e",
  "icons": [
    { "src": "icon.svg", "sizes": "any", "type": "image/svg+xml", "purpose": "any" }
  ]
}
```

- [ ] **Step 3: Write `index.html` skeleton**

```html
<!doctype html>
<html lang="en-GB">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <meta name="theme-color" content="#0f766e">
  <link rel="manifest" href="manifest.webmanifest">
  <link rel="apple-touch-icon" href="icon.svg">
  <title>Mileage Log</title>
  <style>
    /* filled in Task 2 */
  </style>
</head>
<body>
  <main id="app">
    <header class="app-header">
      <h1>Mileage Log</h1>
      <button id="open-settings" aria-label="Settings">⚙</button>
    </header>
    <div id="banner" class="banner" hidden></div>
    <section id="log-form" class="card"></section>
    <section id="months" class="months"></section>
    <section id="actions" class="actions"></section>
  </main>
  <dialog id="settings-sheet"></dialog>
  <script>
    /* filled in Task 3 onward */
  </script>
</body>
</html>
```

- [ ] **Step 4: Write `README.md` stub**

```markdown
# Mileage Log

Single-page web app for logging recurring business-travel mileage claims.
Open `index.html` in Safari on iPhone and "Add to Home Screen".

## Hand-test checklist

Populated at the end of implementation.
```

- [ ] **Step 5: Write `.gitignore`**

```
.DS_Store
*.swp
```

- [ ] **Step 6: Verify in Safari**

Open `index.html` in Safari (Mac Safari is fine for dev — mobile Safari for final checks).

Expected: title in tab reads "Mileage Log", page shows the "Mileage Log" heading and a ⚙ button, no console errors, Network tab shows `manifest.webmanifest` and `icon.svg` loaded with 200 status.

- [ ] **Step 7: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html manifest.webmanifest icon.svg README.md .gitignore
git commit -m "feat: project scaffold — empty shell loads in Safari"
```

---

## Task 2: App-shell CSS

Give the shell its look — system fonts, card layout, accent colour, mobile-first sizing, safe-area insets for the iPhone notch. End state: the empty sections render as spaced cards with proper typography.

**Files:**
- Modify: `index.html` (`<style>` block)

- [ ] **Step 1: Replace the `<style>` block with the shell CSS**

Replace the empty `<style>` block in `index.html` with the following:

```html
<style>
  :root {
    --accent: #0f766e;
    --accent-weak: #ccfbf1;
    --bg: #f8fafc;
    --card: #ffffff;
    --text: #0f172a;
    --text-muted: #64748b;
    --border: #e2e8f0;
    --danger: #b91c1c;
    --radius: 12px;
    --tap: 48px;
  }

  * { box-sizing: border-box; }

  html, body {
    margin: 0;
    padding: 0;
    background: var(--bg);
    color: var(--text);
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
    font-size: 17px;
    line-height: 1.4;
  }

  main#app {
    max-width: 560px;
    margin: 0 auto;
    padding: max(env(safe-area-inset-top), 12px) 16px max(env(safe-area-inset-bottom), 24px);
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  .app-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .app-header h1 {
    font-size: 22px;
    margin: 0;
  }
  #open-settings {
    min-width: var(--tap);
    min-height: var(--tap);
    background: transparent;
    border: none;
    font-size: 24px;
    cursor: pointer;
    color: var(--text-muted);
  }

  .card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 16px;
  }

  .banner {
    background: #fef3c7;
    border: 1px solid #fde68a;
    border-radius: var(--radius);
    padding: 12px 16px;
    color: #78350f;
    font-size: 14px;
  }
  .banner.error {
    background: #fee2e2;
    border-color: #fecaca;
    color: var(--danger);
  }

  .months { display: flex; flex-direction: column; gap: 8px; }

  .actions {
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  button {
    font: inherit;
    min-height: var(--tap);
    padding: 0 18px;
    border-radius: var(--radius);
    border: 1px solid var(--border);
    background: var(--card);
    color: var(--text);
    cursor: pointer;
  }
  button.primary {
    background: var(--accent);
    border-color: var(--accent);
    color: white;
    font-weight: 600;
  }
  button.danger {
    border-color: var(--danger);
    color: var(--danger);
    background: var(--card);
  }
  button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  input[type="date"],
  input[type="text"],
  input[type="number"] {
    font: inherit;
    min-height: var(--tap);
    padding: 0 12px;
    border-radius: var(--radius);
    border: 1px solid var(--border);
    background: var(--card);
    color: var(--text);
    width: 100%;
  }

  .row { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; }
  .label { font-size: 13px; color: var(--text-muted); margin-bottom: 4px; display: block; }
  .muted { color: var(--text-muted); font-size: 14px; }
</style>
```

- [ ] **Step 2: Verify in Safari**

Reload `index.html`.

Expected: max-width-constrained layout centred on large viewports, cards visible for the three empty sections (though content is empty so they collapse to small boxes), ⚙ button is a large tappable target, no horizontal scrolling on a narrow viewport (test at 375px DevTools mobile).

- [ ] **Step 3: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: app-shell CSS — system fonts, card layout, accent"
```

---

## Task 3: Storage layer and state init

Introduce the data layer: the three `localStorage` keys, a storage-availability probe, corrupt-JSON handling, and an initial `render()` that is a no-op for now but wires up the event-loop shape for later tasks. End state: opening the app seeds `mileage.settings` in `localStorage` on first run; subsequent reloads reuse it.

**Files:**
- Modify: `index.html` (`<script>` block)

- [ ] **Step 1: Write the storage + state code into the `<script>` block**

Replace the empty `<script>` block's content with:

```js
'use strict';

const K_SETTINGS = 'mileage.settings';
const K_TRIPS = 'mileage.trips';
const K_DIRTY = 'mileage.dirty';

const DEFAULT_SETTINGS = {
  tripRate: 27.45,
  passengerRate: 5.48,
  regulars: ['CE', 'DDR', 'HD']
};

const state = {
  settings: null,     // object
  trips: [],          // array
  dirty: false,       // boolean
  storageOk: true,
  corrupt: false      // true if any key failed to parse
};

function storageAvailable() {
  try {
    const probe = '__mileage_probe__';
    localStorage.setItem(probe, '1');
    localStorage.removeItem(probe);
    return true;
  } catch {
    return false;
  }
}

function loadJSON(key, fallback) {
  try {
    const raw = localStorage.getItem(key);
    if (raw === null) return { value: fallback, missing: true };
    return { value: JSON.parse(raw), missing: false };
  } catch (err) {
    console.error(`Failed to parse ${key}:`, err);
    state.corrupt = true;
    return { value: fallback, missing: true };
  }
}

function saveJSON(key, value) {
  if (!state.storageOk) return;
  try {
    localStorage.setItem(key, JSON.stringify(value));
  } catch (err) {
    console.error(`Failed to save ${key}:`, err);
  }
}

function setDirty(v) {
  state.dirty = !!v;
  saveJSON(K_DIRTY, state.dirty);
}

function genId() {
  return Math.random().toString(36).slice(2, 8);
}

function todayLocalISO() {
  const d = new Date();
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return `${y}-${m}-${day}`;
}

function init() {
  state.storageOk = storageAvailable();

  if (!state.storageOk) {
    state.settings = { ...DEFAULT_SETTINGS };
    state.trips = [];
    state.dirty = false;
    render();
    return;
  }

  const settingsRes = loadJSON(K_SETTINGS, { ...DEFAULT_SETTINGS });
  if (settingsRes.missing && !state.corrupt) saveJSON(K_SETTINGS, settingsRes.value);
  state.settings = settingsRes.value;

  const tripsRes = loadJSON(K_TRIPS, []);
  state.trips = Array.isArray(tripsRes.value) ? tripsRes.value : [];

  const dirtyRes = loadJSON(K_DIRTY, false);
  state.dirty = !!dirtyRes.value;

  render();
}

function render() {
  renderBanner();
  renderLogForm();
  renderMonths();
  renderActions();
}

function renderBanner() {
  const el = document.getElementById('banner');
  if (!state.storageOk) {
    el.hidden = false;
    el.className = 'banner error';
    el.textContent = "Storage unavailable — trips won't be saved. Turn off Private Browsing.";
    return;
  }
  if (state.corrupt) {
    el.hidden = false;
    el.className = 'banner error';
    el.textContent = 'Could not load some saved data — inspect localStorage in DevTools before continuing.';
    return;
  }
  el.hidden = true;
}

function renderLogForm() {
  /* filled in Task 5 */
}

function renderMonths() {
  /* filled in Task 7 */
}

function renderActions() {
  /* filled in Task 9 */
}

document.addEventListener('DOMContentLoaded', init);
```

- [ ] **Step 2: Verify first-run seeding**

In Safari, open DevTools → Application (or Storage) → Local Storage. Clear all mileage.* keys. Reload the page.

Expected: after reload, `mileage.settings` exists with the default JSON, `mileage.trips` does not exist (we only save it on first write), `mileage.dirty` does not exist. No console errors. No visible banner.

- [ ] **Step 3: Verify corrupt-JSON path**

In DevTools console:
```js
localStorage.setItem('mileage.settings', 'not json{');
```
Reload.

Expected: page loads, yellow-ish red banner shows "Could not load some saved data…", `state.corrupt` is true, default settings still available in memory. Clear the bad value to recover:
```js
localStorage.removeItem('mileage.settings');
```

- [ ] **Step 4: Verify storage-unavailable path**

Open a Safari Private Browsing window and load the file. (Mac Safari: File → New Private Window. Or on mobile: tap tabs → Private.)

Expected: page loads, red banner shows "Storage unavailable…", no console errors. Note: some Safari versions allow writes in private mode — if the banner doesn't appear, stub the probe temporarily by typing `state.storageOk = false; render();` in the console and confirm the banner renders.

- [ ] **Step 5: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: storage layer, state init, error banners"
```

---

## Task 4: Settings sheet

Open-close a `<dialog>` with editable trip rate, passenger rate, and a regulars list. Validate on save; persist to `localStorage`. End state: tapping ⚙ opens the sheet, edits persist, reloading reflects the new values.

**Files:**
- Modify: `index.html` (`<style>` and `<script>` blocks)

- [ ] **Step 1: Append settings-sheet CSS to the `<style>` block**

Add at the end of the existing `<style>` block:

```css
dialog#settings-sheet {
  border: none;
  border-radius: var(--radius);
  padding: 0;
  max-width: 480px;
  width: 90vw;
  color: var(--text);
  background: var(--card);
}
dialog#settings-sheet::backdrop {
  background: rgba(15, 23, 42, 0.4);
}
.sheet-inner { padding: 20px; display: flex; flex-direction: column; gap: 16px; }
.sheet-header { display: flex; justify-content: space-between; align-items: center; }
.sheet-header h2 { margin: 0; font-size: 18px; }
.regulars-list { display: flex; flex-direction: column; gap: 8px; }
.regulars-list .row input { flex: 1; }
.regulars-list button { min-width: auto; padding: 0 12px; }
.field-error { color: var(--danger); font-size: 13px; min-height: 1.2em; }
```

- [ ] **Step 2: Add the settings render + save functions to the `<script>` block**

Add before the `document.addEventListener('DOMContentLoaded', init)` line:

```js
function openSettings() {
  const dlg = document.getElementById('settings-sheet');
  dlg.innerHTML = settingsSheetHTML(state.settings);
  wireSettingsSheet(dlg);
  if (typeof dlg.showModal === 'function') dlg.showModal();
  else dlg.setAttribute('open', '');
}

function closeSettings() {
  const dlg = document.getElementById('settings-sheet');
  if (typeof dlg.close === 'function') dlg.close();
  else dlg.removeAttribute('open');
}

function settingsSheetHTML(s) {
  const regulars = s.regulars.map((name, i) => `
    <div class="row" data-regular-index="${i}">
      <input type="text" value="${escapeAttr(name)}" aria-label="Regular ${i + 1}">
      <button type="button" data-action="remove-regular">×</button>
    </div>
  `).join('');
  return `
    <form class="sheet-inner" id="settings-form">
      <div class="sheet-header">
        <h2>Settings</h2>
        <button type="button" data-action="close">Close</button>
      </div>
      <div>
        <label class="label" for="trip-rate">Trip rate (£)</label>
        <input type="number" id="trip-rate" step="0.01" min="0" value="${s.tripRate}">
        <div class="field-error" id="trip-rate-error"></div>
      </div>
      <div>
        <label class="label" for="passenger-rate">Passenger rate (£)</label>
        <input type="number" id="passenger-rate" step="0.01" min="0" value="${s.passengerRate}">
        <div class="field-error" id="passenger-rate-error"></div>
      </div>
      <div>
        <label class="label">Regulars</label>
        <div class="regulars-list" id="regulars-list">${regulars}</div>
        <button type="button" data-action="add-regular" style="margin-top: 8px;">+ Add regular</button>
      </div>
      <button type="submit" class="primary" id="save-settings">Save</button>
    </form>
  `;
}

function escapeAttr(s) {
  return String(s).replace(/[&<>"']/g, c => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
  })[c]);
}

function wireSettingsSheet(dlg) {
  const form = dlg.querySelector('#settings-form');
  form.addEventListener('click', e => {
    const action = e.target.dataset.action;
    if (action === 'close') { closeSettings(); return; }
    if (action === 'remove-regular') {
      e.target.closest('[data-regular-index]').remove();
      return;
    }
    if (action === 'add-regular') {
      const list = dlg.querySelector('#regulars-list');
      const row = document.createElement('div');
      row.className = 'row';
      row.dataset.regularIndex = 'new';
      row.innerHTML = '<input type="text" value="" aria-label="New regular"><button type="button" data-action="remove-regular">×</button>';
      list.appendChild(row);
      row.querySelector('input').focus();
      return;
    }
  });
  form.addEventListener('submit', e => {
    e.preventDefault();
    saveSettingsFromSheet(dlg);
  });
}

function saveSettingsFromSheet(dlg) {
  const tripInput = dlg.querySelector('#trip-rate');
  const passInput = dlg.querySelector('#passenger-rate');
  const tripErr = dlg.querySelector('#trip-rate-error');
  const passErr = dlg.querySelector('#passenger-rate-error');
  tripErr.textContent = '';
  passErr.textContent = '';

  const trip = Number(tripInput.value);
  const pass = Number(passInput.value);
  let ok = true;
  if (!Number.isFinite(trip) || trip < 0) { tripErr.textContent = 'Must be a positive number.'; ok = false; }
  if (!Number.isFinite(pass) || pass < 0) { passErr.textContent = 'Must be a positive number.'; ok = false; }
  if (!ok) return;

  const regulars = Array.from(dlg.querySelectorAll('.regulars-list input'))
    .map(i => i.value.trim())
    .filter(Boolean);

  state.settings = {
    tripRate: round2(trip),
    passengerRate: round2(pass),
    regulars
  };
  saveJSON(K_SETTINGS, state.settings);
  closeSettings();
  render();
}

function round2(n) { return Math.round(n * 100) / 100; }
```

- [ ] **Step 3: Wire the cog button to `openSettings` inside `init`**

Inside the `init()` function, add after the `render();` call but BEFORE the `}` that closes `init`:

```js
  document.getElementById('open-settings').addEventListener('click', openSettings);
```

(This line must be inside `init` so it runs once after storage is ready. Verify by reading the function end-to-end.)

- [ ] **Step 4: Verify in Safari**

Reload. Tap ⚙.

Expected:
- Dialog opens with trip rate 27.45, passenger rate 5.48, regulars CE / DDR / HD each with × buttons.
- Tap × on a regular — row disappears.
- Tap "+ Add regular", type "JP", save.
- Reload. Open ⚙. JP is present.
- Edit trip rate to `abc`, tap Save — "Must be a positive number" appears, sheet stays open.
- Edit trip rate to `-1`, tap Save — same error.
- Edit trip rate to `30`, passenger rate to `6`, Save. Reload, reopen — values persist as 30 and 6.
- DevTools localStorage shows `mileage.settings` with the edited values.
- Reset to defaults: in console, `localStorage.removeItem('mileage.settings'); location.reload()`.

- [ ] **Step 5: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: settings sheet — edit rates and regulars"
```

---

## Task 5: Log-a-trip form (render + passenger selection)

Render the log-a-trip card: date input with Today button, regulars as pill toggles, ad-hoc passenger rows with +/×, and a live total. Save button is not yet wired to persist (next task). End state: all UI controls work, total updates live, cap of 3 passengers enforced.

**Files:**
- Modify: `index.html` (`<style>` and `<script>` blocks)

- [ ] **Step 1: Append log-form CSS to the `<style>` block**

```css
.form-grid { display: flex; flex-direction: column; gap: 16px; }
.date-row { display: flex; gap: 8px; }
.date-row input[type="date"] { flex: 1; }
.today-btn { min-width: 90px; }
.today-btn.pressed {
  background: var(--accent);
  color: white;
  border-color: var(--accent);
}

.pill-row { display: flex; flex-wrap: wrap; gap: 8px; }
.pill {
  padding: 10px 16px;
  border: 1px solid var(--border);
  border-radius: 999px;
  background: var(--card);
  min-height: var(--tap);
  cursor: pointer;
}
.pill.on {
  background: var(--accent-weak);
  border-color: var(--accent);
  color: var(--accent);
  font-weight: 600;
}

.adhoc-row { display: flex; gap: 8px; align-items: center; }
.adhoc-row input { flex: 1; }
.adhoc-row button { min-width: var(--tap); }

.total-line {
  display: flex;
  justify-content: space-between;
  font-weight: 600;
  font-size: 20px;
  padding: 8px 0;
  border-top: 1px solid var(--border);
  padding-top: 12px;
}
```

- [ ] **Step 2: Implement `renderLogForm` and its helpers**

Replace the empty `function renderLogForm() { /* filled in Task 5 */ }` body with:

```js
const formState = {
  date: todayLocalISO(),
  regulars: new Set(),        // selected regular names
  adhoc: []                    // array of strings (each ad-hoc passenger)
};

function passengerCount() {
  return formState.regulars.size + formState.adhoc.length;
}

function formTotal() {
  return round2(state.settings.tripRate + state.settings.passengerRate * passengerCount());
}

function canAddPassenger() {
  return passengerCount() < 3;
}

function formSaveDisabled() {
  if (!state.storageOk) return true;
  if (formState.adhoc.some(n => n.trim() === '')) return true;
  return false;
}

function renderLogForm() {
  const el = document.getElementById('log-form');
  const regulars = state.settings.regulars;
  const isToday = formState.date === todayLocalISO();

  const pills = regulars.map(name => `
    <button type="button" class="pill ${formState.regulars.has(name) ? 'on' : ''}" data-regular="${escapeAttr(name)}">
      ${escapeAttr(name)}
    </button>
  `).join('');

  const adhocRows = formState.adhoc.map((val, i) => `
    <div class="adhoc-row" data-adhoc-index="${i}">
      <input type="text" value="${escapeAttr(val)}" placeholder="Name" aria-label="Ad-hoc passenger ${i + 1}">
      <button type="button" data-action="remove-adhoc" aria-label="Remove passenger ${i + 1}">×</button>
    </div>
  `).join('');

  el.innerHTML = `
    <div class="form-grid">
      <div>
        <span class="label">Date</span>
        <div class="date-row">
          <button type="button" class="today-btn ${isToday ? 'pressed' : ''}" data-action="today">Today</button>
          <input type="date" id="trip-date" value="${formState.date}">
        </div>
      </div>
      <div>
        <span class="label">Passengers (${passengerCount()}/3)</span>
        <div class="pill-row">${pills}</div>
        <div class="adhoc-list" style="margin-top: 8px; display: flex; flex-direction: column; gap: 8px;">
          ${adhocRows}
        </div>
        <button type="button" data-action="add-adhoc" ${canAddPassenger() ? '' : 'disabled'} style="margin-top: 8px;">+ Add passenger</button>
      </div>
      <div class="total-line">
        <span>Total</span>
        <span>£${formTotal().toFixed(2)}</span>
      </div>
      <button type="button" class="primary" id="save-trip" ${formSaveDisabled() ? 'disabled' : ''}>Save trip</button>
    </div>
  `;

  wireLogForm(el);
}

function wireLogForm(el) {
  el.addEventListener('click', e => {
    const action = e.target.dataset.action;
    const regular = e.target.dataset.regular;
    if (regular) {
      if (formState.regulars.has(regular)) formState.regulars.delete(regular);
      else if (canAddPassenger()) formState.regulars.add(regular);
      renderLogForm();
      return;
    }
    if (action === 'today') {
      formState.date = todayLocalISO();
      renderLogForm();
      return;
    }
    if (action === 'add-adhoc' && canAddPassenger()) {
      formState.adhoc.push('');
      renderLogForm();
      // Focus the new row
      const rows = el.querySelectorAll('.adhoc-row input');
      if (rows.length) rows[rows.length - 1].focus();
      return;
    }
    if (action === 'remove-adhoc') {
      const row = e.target.closest('[data-adhoc-index]');
      const idx = Number(row.dataset.adhocIndex);
      formState.adhoc.splice(idx, 1);
      renderLogForm();
      return;
    }
  });
  el.addEventListener('input', e => {
    if (e.target.id === 'trip-date') {
      formState.date = e.target.value;
      renderLogForm();
      return;
    }
    const row = e.target.closest('[data-adhoc-index]');
    if (row) {
      const idx = Number(row.dataset.adhocIndex);
      formState.adhoc[idx] = e.target.value;
      // Re-render only if save-disabled state changes — cheap: just re-render.
      renderLogForm();
      // Restore focus to the edited input (it was re-created).
      const newRow = el.querySelector(`[data-adhoc-index="${idx}"] input`);
      if (newRow) {
        newRow.focus();
        const len = newRow.value.length;
        newRow.setSelectionRange(len, len);
      }
    }
  });
}
```

- [ ] **Step 3: Verify in Safari**

Reload.

Expected:
- Log-form card shows today's date, "Today" button visually pressed, three regular pills (CE, DDR, HD), "+ Add passenger", Total £27.45, Save trip button.
- Tap CE — pill turns accent-coloured, count shows (1/3), Total £32.93.
- Tap CE again — deselects, count (0/3), total £27.45.
- Tap all three regulars — count (3/3), total £43.89, "+ Add passenger" disabled.
- Tap CE to free a slot, tap + Add passenger — empty row appears with focus.
- Type "JP", total now £43.89 again (2 regulars + 1 adhoc).
- Save trip — nothing happens yet (next task).
- Clear the ad-hoc input (backspace to empty) — Save button disables (but button label text is the same; check `disabled` attr in DevTools).
- Type a name into ad-hoc — Save re-enables.
- Tap × on ad-hoc row — row removed, total updates.
- Change date to a past date — "Today" button loses its pressed look. Tap Today — date jumps back to today, pressed look returns.

- [ ] **Step 4: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: log-a-trip form — date, regulars, ad-hoc, live total"
```

---

## Task 6: Save trip

Wire the Save button to persist a trip, reset the form, and mark state dirty. End state: tapping Save adds the trip to `mileage.trips` in localStorage and the form resets.

**Files:**
- Modify: `index.html` (`<script>` block)

- [ ] **Step 1: Add the save function**

Add near the other form functions (before `renderLogForm`):

```js
function saveTrip() {
  if (formSaveDisabled()) return;
  const passengers = [
    ...state.settings.regulars.filter(n => formState.regulars.has(n)),
    ...formState.adhoc.map(s => s.trim()).filter(Boolean)
  ];
  const trip = {
    id: genId(),
    date: formState.date,
    passengers,
    tripRate: state.settings.tripRate,
    passengerRate: state.settings.passengerRate
  };
  state.trips.push(trip);
  saveJSON(K_TRIPS, state.trips);
  setDirty(true);

  // Reset form
  formState.date = todayLocalISO();
  formState.regulars = new Set();
  formState.adhoc = [];

  render();
}
```

- [ ] **Step 2: Wire the Save button**

Inside `wireLogForm`, at the end of the `click` listener's handler chain (before the closing `});` of the `addEventListener('click', …)`), add:

```js
    if (e.target.id === 'save-trip') {
      saveTrip();
      return;
    }
```

- [ ] **Step 3: Verify in Safari**

Reload. Ensure `mileage.trips` is empty first:
```js
localStorage.removeItem('mileage.trips');
localStorage.removeItem('mileage.dirty');
```
Reload.

Expected:
- Tick CE. Total £32.93. Save trip.
- Form resets: today's date, no pills on, total £27.45.
- DevTools → localStorage: `mileage.trips` now contains an array with one trip: correct `date`, `passengers: ["CE"]`, rates frozen at 27.45/5.48.
- `mileage.dirty` = `true`.
- Save another trip with 2 passengers — new trip appended.
- Reload. Trips persist in localStorage (UI hasn't rendered them yet — that's Task 7).

- [ ] **Step 4: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: save trip — persist and reset form"
```

---

## Task 7: Month list

Render trips grouped by month, newest first, with expand/collapse. Current month auto-expanded. Per-month subtotal in header.

**Files:**
- Modify: `index.html` (`<style>` and `<script>` blocks)

- [ ] **Step 1: Append month-list CSS**

```css
.month {
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  overflow: hidden;
}
.month-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 16px;
  cursor: pointer;
  min-height: var(--tap);
  background: var(--card);
  border: none;
  width: 100%;
  font: inherit;
  color: var(--text);
  text-align: left;
}
.month-header .chev { color: var(--text-muted); margin-left: 8px; }
.month-header .total { font-weight: 600; }
.month-trips { display: flex; flex-direction: column; }
.trip-row {
  display: grid;
  grid-template-columns: 90px 1fr auto auto;
  gap: 8px;
  align-items: center;
  padding: 10px 16px;
  border-top: 1px solid var(--border);
  min-height: var(--tap);
}
.trip-row .date { color: var(--text-muted); font-size: 14px; }
.trip-row .names { font-size: 14px; overflow: hidden; text-overflow: ellipsis; }
.trip-row .amount { font-weight: 600; }
.trip-row button {
  min-width: 36px;
  min-height: 36px;
  background: transparent;
  border: none;
  color: var(--text-muted);
  font-size: 18px;
}
.empty-state { text-align: center; padding: 32px 16px; color: var(--text-muted); }
```

- [ ] **Step 2: Track expanded months in a module-level Set**

Near the top of the script, after the `state` object definition, add:

```js
const expandedMonths = new Set(); // e.g. "2026-04"

function currentMonthKey() {
  const d = new Date();
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}`;
}
```

- [ ] **Step 3: Implement helpers + `renderMonths`**

Replace the empty `function renderMonths() { /* filled in Task 7 */ }` body with:

```js
function tripAmount(trip) {
  return round2(trip.tripRate + trip.passengerRate * trip.passengers.length);
}

function monthKey(dateStr) {
  return dateStr.slice(0, 7); // "YYYY-MM"
}

function groupTripsByMonth(trips) {
  const map = new Map();
  for (const t of trips) {
    const k = monthKey(t.date);
    if (!map.has(k)) map.set(k, []);
    map.get(k).push(t);
  }
  // Sort trips within month newest-first.
  for (const arr of map.values()) {
    arr.sort((a, b) => b.date.localeCompare(a.date));
  }
  // Return as [[key, trips]] sorted newest-month first.
  return [...map.entries()].sort((a, b) => b[0].localeCompare(a[0]));
}

function monthLabel(key) {
  const [y, m] = key.split('-').map(Number);
  const d = new Date(y, m - 1, 1);
  return d.toLocaleDateString('en-GB', { month: 'long', year: 'numeric' });
}

function dayLabel(dateStr) {
  const [y, m, d] = dateStr.split('-').map(Number);
  const dt = new Date(y, m - 1, d);
  return dt.toLocaleDateString('en-GB', { weekday: 'short', day: 'numeric', month: 'short' });
}

function renderMonths() {
  const el = document.getElementById('months');
  if (state.trips.length === 0) {
    el.innerHTML = `<div class="empty-state">No trips recorded yet.</div>`;
    return;
  }
  // Ensure current month defaults to expanded on first render after load.
  if (!renderMonths._initialised) {
    expandedMonths.add(currentMonthKey());
    renderMonths._initialised = true;
  }
  const grouped = groupTripsByMonth(state.trips);
  el.innerHTML = grouped.map(([key, trips]) => {
    const isOpen = expandedMonths.has(key);
    const total = round2(trips.reduce((s, t) => s + tripAmount(t), 0));
    const rows = isOpen ? trips.map(t => `
      <div class="trip-row" data-trip-id="${escapeAttr(t.id)}">
        <span class="date">${dayLabel(t.date)}</span>
        <span class="names">${escapeAttr(t.passengers.join(', '))}</span>
        <span class="amount">£${tripAmount(t).toFixed(2)}</span>
        <button type="button" data-action="delete-trip" aria-label="Delete trip">×</button>
      </div>
    `).join('') : '';
    return `
      <div class="month">
        <button type="button" class="month-header" data-month="${key}">
          <span>${monthLabel(key)}</span>
          <span><span class="total">£${total.toFixed(2)}</span><span class="chev"> ${isOpen ? '▾' : '▸'}</span></span>
        </button>
        <div class="month-trips">${rows}</div>
      </div>
    `;
  }).join('');
  wireMonths(el);
}

function wireMonths(el) {
  el.addEventListener('click', e => {
    const monthBtn = e.target.closest('[data-month]');
    if (monthBtn) {
      const key = monthBtn.dataset.month;
      if (expandedMonths.has(key)) expandedMonths.delete(key);
      else expandedMonths.add(key);
      renderMonths();
      return;
    }
    // delete-trip handled in Task 8
  });
}
```

- [ ] **Step 4: Verify in Safari**

Reload with at least 2 trips in localStorage (save a couple first if needed — one with today's date, one with a past date you pick manually).

Expected:
- Current month section shows expanded by default with each trip row.
- Trip row shows e.g. "Tue 23 Apr", passenger names comma-separated, amount, ×.
- Past month section shows collapsed, tap header to expand.
- Totals shown in each month header.
- With zero trips: "No trips recorded yet." text appears.
- Tap month header toggles the chevron and the content.

- [ ] **Step 5: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: month list — grouped, expand/collapse, subtotals"
```

---

## Task 8: Delete trip

Wire the × on each trip row to remove the trip after confirmation.

**Files:**
- Modify: `index.html` (`<script>` block)

- [ ] **Step 1: Add the delete function**

Near `saveTrip`:

```js
function deleteTrip(id) {
  if (!confirm('Delete this trip?')) return;
  state.trips = state.trips.filter(t => t.id !== id);
  saveJSON(K_TRIPS, state.trips);
  setDirty(true);
  render();
}
```

- [ ] **Step 2: Wire the × button**

In `wireMonths`, inside the existing click listener, after the month-header handling, add:

```js
    const deleteBtn = e.target.closest('[data-action="delete-trip"]');
    if (deleteBtn) {
      const row = deleteBtn.closest('[data-trip-id]');
      deleteTrip(row.dataset.tripId);
      return;
    }
```

- [ ] **Step 3: Verify in Safari**

Reload with several trips.

Expected:
- Tap × on a trip — native confirm shows "Delete this trip?". Cancel does nothing. OK removes the row.
- Month subtotal updates.
- If the only trip in a month is deleted, that month section disappears.
- If all trips deleted, empty-state text replaces the list.
- DevTools → `mileage.trips` reflects the deletion. `mileage.dirty` = true.

- [ ] **Step 4: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: delete trip with confirm"
```

---

## Task 9: Grand total and Claim-complete button

Populate the actions zone with grand total, Export and Share placeholders, and the Claim-complete button wired with the dirty gate. Export and Share are stubbed as buttons that just flip dirty to false for now; they get real bodies in Tasks 10 and 11.

**Files:**
- Modify: `index.html` (`<style>` and `<script>` blocks)

- [ ] **Step 1: Append actions CSS**

```css
.grand {
  display: flex;
  justify-content: space-between;
  font-size: 18px;
  font-weight: 600;
  padding: 0 8px;
}
.export-row { display: flex; gap: 12px; }
.export-row button { flex: 1; }
.claim-hint { text-align: center; color: var(--text-muted); font-size: 13px; margin-top: -4px; }
```

- [ ] **Step 2: Implement `renderActions`**

Replace the empty `function renderActions() { /* filled in Task 9 */ }` body with:

```js
function grandTotal() {
  return round2(state.trips.reduce((s, t) => s + tripAmount(t), 0));
}

function renderActions() {
  const el = document.getElementById('actions');
  if (state.trips.length === 0) {
    el.innerHTML = '';
    return;
  }
  const claimDisabled = state.dirty;
  const hasShare = typeof navigator.share === 'function';
  el.innerHTML = `
    <div class="grand"><span>Grand total</span><span>£${grandTotal().toFixed(2)}</span></div>
    <div class="export-row">
      <button type="button" id="export-csv">Export CSV</button>
      ${hasShare ? '<button type="button" id="share-trips">Share</button>' : ''}
    </div>
    <button type="button" class="danger" id="claim-complete" ${claimDisabled ? 'disabled' : ''}>
      Claim complete — delete all
    </button>
    ${claimDisabled ? '<div class="claim-hint">Export or share first.</div>' : ''}
  `;
  wireActions(el);
}

function wireActions(el) {
  const exportBtn = el.querySelector('#export-csv');
  if (exportBtn) exportBtn.addEventListener('click', exportCSV);
  const shareBtn = el.querySelector('#share-trips');
  if (shareBtn) shareBtn.addEventListener('click', shareTrips);
  const claimBtn = el.querySelector('#claim-complete');
  if (claimBtn) claimBtn.addEventListener('click', claimComplete);
}

function exportCSV() {
  // real body in Task 10
  setDirty(false);
  render();
}

function shareTrips() {
  // real body in Task 11
  setDirty(false);
  render();
}

function claimComplete() {
  if (state.dirty) return;
  if (!confirm('Delete all trips? This cannot be undone.')) return;
  state.trips = [];
  saveJSON(K_TRIPS, state.trips);
  setDirty(false);
  render();
}
```

- [ ] **Step 3: Verify in Safari**

Reload with trips present and `mileage.dirty = true`.

Expected:
- Grand total shows, Export CSV and Share buttons visible (Share only if your Safari supports `navigator.share`; desktop Safari on recent macOS does).
- Claim complete button is disabled with grey caption "Export or share first."
- Tap Export CSV — button is wired, dirty flips to false, Claim complete enables and caption disappears.
- Tap Claim complete — confirm appears. Cancel → no change. OK → all trips deleted, empty state, actions area empty.
- With zero trips, actions section is empty.
- Add a trip — Claim-complete disabled again.

- [ ] **Step 4: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: grand total, claim-complete with dirty gate"
```

---

## Task 10: CSV export (real)

Generate a proper CSV string and trigger a download. Filename stamped with today's date.

**Files:**
- Modify: `index.html` (`<script>` block)

- [ ] **Step 1: Replace the stub `exportCSV`**

Find the stub `exportCSV` function and replace it with:

```js
function csvEscape(val) {
  const s = String(val);
  if (/[",\r\n]/.test(s)) return `"${s.replace(/"/g, '""')}"`;
  return s;
}

function buildCSV(trips) {
  const sorted = [...trips].sort((a, b) => b.date.localeCompare(a.date));
  const header = ['Date', 'Passengers', 'Passenger count', 'Trip rate', 'Passenger rate', 'Amount'];
  const rows = sorted.map(t => [
    t.date,
    t.passengers.join(', '),
    t.passengers.length,
    t.tripRate.toFixed(2),
    t.passengerRate.toFixed(2),
    tripAmount(t).toFixed(2)
  ]);
  const total = round2(sorted.reduce((s, t) => s + tripAmount(t), 0));
  rows.push(['', '', '', '', 'Total', total.toFixed(2)]);
  return [header, ...rows].map(r => r.map(csvEscape).join(',')).join('\r\n') + '\r\n';
}

function exportCSV() {
  if (state.trips.length === 0) return;
  const csv = buildCSV(state.trips);
  const blob = new Blob([csv], { type: 'text/csv;charset=utf-8' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `mileage-${todayLocalISO()}.csv`;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  setTimeout(() => URL.revokeObjectURL(url), 0);
  setDirty(false);
  render();
}
```

- [ ] **Step 2: Verify in Safari**

Reload. Ensure trips with 0, 1, and 3 passengers exist across two different months.

Expected:
- Tap Export CSV. File `mileage-YYYY-MM-DD.csv` downloads (Safari: Downloads folder or Files app on iOS).
- Open in Numbers or a text editor. Contents:
  - Header row: `Date,Passengers,Passenger count,Trip rate,Passenger rate,Amount`
  - Rows sorted newest first.
  - Multi-name passenger cell properly quoted, e.g. `"CE, DDR, JP"`.
  - Empty passengers cell is empty (not `""`).
  - Total row has leading blanks and grand total in the last column.
- Test with a trip where a passenger name contains a comma (type `"A, B"` into Settings regulars or ad-hoc) — cell is correctly quoted with inner `""` doubling.
- `mileage.dirty` flipped to false. Claim-complete button enabled.

- [ ] **Step 3: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: CSV export with proper quoting + total row"
```

---

## Task 11: Share sheet (real)

Replace the stub `shareTrips` with a real `navigator.share` call carrying a plain-text payload.

**Files:**
- Modify: `index.html` (`<script>` block)

- [ ] **Step 1: Replace the stub `shareTrips`**

```js
function buildShareText(trips) {
  const sorted = [...trips].sort((a, b) => b.date.localeCompare(a.date));
  const total = round2(sorted.reduce((s, t) => s + tripAmount(t), 0));
  const today = todayLocalISO();
  const header = `Mileage claim — ${today}\nTrips: ${sorted.length}\nTotal: £${total.toFixed(2)}\n`;
  const body = sorted.map(t =>
    `${dayLabel(t.date)} — ${t.passengers.join(', ')} — £${tripAmount(t).toFixed(2)}`
  ).join('\n');
  return `${header}\n${body}`;
}

async function shareTrips() {
  if (state.trips.length === 0) return;
  const text = buildShareText(state.trips);
  const today = todayLocalISO();
  try {
    await navigator.share({ title: `Mileage claim — ${today}`, text });
    setDirty(false);
    render();
  } catch (err) {
    // User cancelled or share failed — leave dirty as it was.
    if (err && err.name !== 'AbortError') console.error('Share failed:', err);
  }
}
```

- [ ] **Step 2: Verify in Safari**

On mobile Safari (iOS) ideally, or recent macOS Safari:

Expected:
- Tap Share — iOS/macOS share sheet appears with title "Mileage claim — 2026-04-23" and the multi-line text body visible in the preview.
- Send to yourself via Mail or Messages — body arrives intact.
- Cancel the share sheet — `mileage.dirty` does NOT flip (Claim-complete still disabled).
- Complete a share — `mileage.dirty` flips to false.
- In a browser without `navigator.share` (e.g. desktop Chrome), the Share button should not render at all. Verify via `navigator.share = undefined; render();` in the console.

- [ ] **Step 3: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add index.html
git commit -m "feat: share sheet — plain-text payload, cancel preserves dirty"
```

---

## Task 12: README hand-test checklist + polish

Fill the README with the full spec checklist. Do a pass of visual polish in Safari mobile (notch padding, tap-target spot-checks). Commit and call v1 done.

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Replace `README.md`**

```markdown
# Mileage Log

Single-page web app for logging recurring business-travel mileage claims.
Data lives in `localStorage`. Exports to CSV and iOS share sheet.

## Install

1. Put `index.html`, `manifest.webmanifest`, and `icon.svg` together in iCloud Drive.
2. Open `index.html` in Safari on iPhone.
3. Share → Add to Home Screen.

## Defaults

- Trip rate: £27.45
- Passenger rate: £5.48 per additional passenger
- Regulars: CE, DDR, HD (editable in Settings)
- Max passengers per trip: 3

## Hand-test checklist

Run through this after any non-trivial edit.

1. Fresh load (clear `mileage.*` keys in DevTools) — empty state, "No trips recorded yet."
2. Log a trip today with 0 / 1 / 2 / 3 passengers — total matches `27.45 + 5.48 × n`.
3. Log a past-dated trip — appears under the correct month section, current month still auto-expanded.
4. Add a free-text passenger, × to remove — total updates, 3-passenger cap disables "+ Add passenger".
5. Edit rates in Settings to 30.00 / 6.00 — a new trip uses the new rate, older trips keep their old rate (check via Export CSV — per-row rate columns differ).
6. Export CSV — file downloads, opens in Numbers/Excel, totals match on-screen grand total.
7. Share — iOS share sheet appears, text payload arrives intact when sent to Mail.
8. Claim complete — disabled before export, enabled after, wipes all trips on confirm, re-disables itself.
9. Reload the page after each of the above — state survives.
10. Safari Private Browsing — storage-unavailable banner appears, Save disabled.

## Files

- `index.html` — the app (HTML + inlined CSS + inlined JS)
- `manifest.webmanifest` — PWA manifest
- `icon.svg` — app icon
- `docs/superpowers/specs/` — design doc
- `docs/superpowers/plans/` — implementation plan
```

- [ ] **Step 2: Polish pass in mobile Safari (or DevTools mobile emulation)**

Open DevTools, switch to iPhone 14 preset (390×844). Verify:
- No horizontal scroll.
- ⚙ button easy to tap.
- Pills don't overflow; wrap sanely.
- Save trip button full-width card-edge-to-card-edge-ish.
- Safe-area inset padding leaves room above/below (test by rotating to landscape).
- Buttons and inputs meet the ≥44px tap target height (inspect computed height).

If anything's obviously cramped, tweak the CSS in `index.html`'s `<style>` block and note the tweak in the commit. Otherwise, skip.

- [ ] **Step 3: Commit**

```bash
cd ~/Documents/ClaudeCode/expenselogger
git add README.md index.html
git commit -m "docs: README hand-test checklist; mobile polish pass"
```

---

## Self-review (cross-check against spec)

- [x] Single self-contained `index.html` — Task 1 scaffolds; all feature tasks modify only `index.html`.
- [x] `manifest.webmanifest` + `icon.svg` for Add-to-Home-Screen — Task 1.
- [x] `mileage.settings`, `mileage.trips`, `mileage.dirty` in localStorage — Task 3.
- [x] Settings seeded with defaults on first run — Task 3.
- [x] Rates frozen per trip at entry time — Task 6 (`tripRate`/`passengerRate` copied from `state.settings`).
- [x] 6-char random `id` — Task 3 `genId`.
- [x] `date` = `YYYY-MM-DD` local — Task 3 `todayLocalISO`, Task 5 date input.
- [x] Passengers flat list of strings, regular+adhoc stored uniformly — Task 6.
- [x] Today button + pressed state — Task 5.
- [x] 3-passenger cap — Task 5 `canAddPassenger`.
- [x] Regulars as pill toggles driven by settings — Task 5.
- [x] Ad-hoc with + Add / × remove — Task 5.
- [x] Empty ad-hoc row disables Save — Task 5 `formSaveDisabled`, Task 6 `saveTrip` early-return.
- [x] Trim ad-hoc names on save — Task 6.
- [x] Month grouping, newest first, current month expanded — Task 7.
- [x] Per-month subtotals — Task 7.
- [x] Native `confirm()` for delete and claim-complete — Tasks 8, 9.
- [x] Grand total — Task 9.
- [x] Empty state hides grand total + claim-complete — Task 9.
- [x] Claim-complete disabled while dirty, caption "Export or share first" — Task 9.
- [x] CSV with per-row rates, total row, proper quoting — Task 10.
- [x] Filename `mileage-YYYY-MM-DD.csv` — Task 10.
- [x] Share payload plain-text, title+text, cancel preserves dirty — Task 11.
- [x] `navigator.share` feature-detect, hide Share button if missing — Task 9 (initial render gate), Task 11 (noop if somehow reached).
- [x] Storage-unavailable banner — Task 3.
- [x] Corrupt-JSON banner — Task 3.
- [x] Invalid rates validation in Settings — Task 4.
- [x] Hand-test checklist in README — Task 12.

No placeholders remain. Types stay consistent across tasks (`trip.id`, `trip.passengers`, `state.dirty`, `setDirty`).

---

## Execution handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-23-mileage-log.md`.

Two execution options:

1. **Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
