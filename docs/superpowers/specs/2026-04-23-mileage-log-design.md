# Mileage Log — Design

**Date:** 2026-04-23
**Status:** Approved, ready for implementation planning

## Purpose

A single-screen, mobile-optimised web app for logging recurring business-travel
mileage claims. One user, one device, one repetitive round trip. Each trip is a
fixed round-trip claim plus a per-passenger uplift. Output is a CSV or share-
sheet payload that the user hands to whoever processes the claim, after which
the local log is wiped.

Default values (editable via Settings):

- Trip rate: **£27.45** per round trip
- Passenger rate: **£5.48** per additional passenger
- Regular passengers: **CE**, **DDR**, **HD**

## Non-goals

- Multi-device sync, accounts, or any server.
- Undo for delete or claim-complete actions (native `confirm()` is the guard).
- Editing a saved trip (delete + re-add).
- Accessibility work beyond semantic HTML, good contrast, and ≥44px tap
  targets. No ARIA live regions or keyboard-only flows — this is a personal
  mobile tool.
- Multi-user use.

## Shape and delivery

A single self-contained `index.html` file (CSS and JS inlined). No build step,
no runtime network calls, no fonts from CDNs. The file lives in iCloud Drive;
the user opens it in Safari and uses "Add to Home Screen" for an app-like
launcher.

Repo layout:

```
expenselogger/
  index.html
  manifest.webmanifest
  icon.svg
  README.md
  docs/superpowers/specs/2026-04-23-mileage-log-design.md
```

`manifest.webmanifest` + `icon.svg` make the home-screen install produce a
proper icon and standalone (chrome-less) launch. No hosting, no deploy
pipeline. Git repo exists purely for version history and as a recovery path if
iCloud loses the file.

## Data model

Three keys in `localStorage`, all JSON.

### `mileage.settings`

```json
{
  "tripRate": 27.45,
  "passengerRate": 5.48,
  "regulars": ["CE", "DDR", "HD"]
}
```

Seeded with defaults on first run. Editable via a Settings sheet. `regulars`
is editable (add/remove) so staff changes don't require a code edit.

### `mileage.trips`

```json
[
  {
    "id": "k3f9a2",
    "date": "2026-04-23",
    "passengers": ["CE", "DDR", "JP"],
    "tripRate": 27.45,
    "passengerRate": 5.48
  }
]
```

- `id`: 6-char random string (e.g. `Math.random().toString(36).slice(2, 8)`).
  Internal use only — identifies a row for deletion. Never shown.
- `date`: `YYYY-MM-DD`. The only time-ish field, and the displayed one.
- `passengers`: flat list of strings. No distinction between "regular" and
  "ad-hoc" at storage time — a name is a name. A typed name that happens to
  match a regular (e.g. user types "CE") is stored verbatim, not deduplicated.
- `tripRate` / `passengerRate`: **frozen at entry time**. Historical trips
  never re-price when Settings are edited. This is a deliberate audit-trail
  choice.

Per-trip amount (derived, not stored):
`tripRate + passengerRate × passengers.length`.

### `mileage.dirty`

```json
true
```

Boolean. Set to `true` whenever a trip is added or deleted. Set to `false`
whenever Export CSV or Share runs successfully. Gates the "Claim complete"
button (see UI below).

### Schema versioning

No `version` field. If the shape ever changes, a migration will be added at
load time. YAGNI until it happens.

## UI — single scrolling page

Mobile-first, three zones stacked vertically. Chunky tap targets throughout
(≥44px).

### Zone 1 — "Log a trip" card (top)

```
┌─────────────────────────────────────┐
│  Date                               │
│  [ Today ]   [ 2026-04-23  ▼ ]      │
│                                     │
│  Passengers (0/3)                   │
│  ( CE )  ( DDR )  ( HD )            │
│  [ + Add passenger ]                │
│                                     │
│  Total: £27.45                      │
│  [      Save trip      ]            │
└─────────────────────────────────────┘
```

- **Today** button fills the date input with today's **local** date (user's
  device timezone, not UTC — derive from local `Date` components, never from
  `toISOString()`). Appears visually "pressed" when the selected date matches
  today.
- Date input is native `<input type="date">` — Safari presents its wheel
  picker.
- **Regulars** (`CE`, `DDR`, `HD`) render as pill-shaped toggles driven by
  `settings.regulars`. Tap to toggle selection.
- **+ Add passenger** inserts a row with a text input and a × remove button.
  The button disables when the total passenger count hits 3 (regulars
  selected + ad-hoc rows combined).
- **Total** recalculates live as selections change.
- **Save trip** disabled if any ad-hoc row has empty/whitespace-only text.
  On save: trim ad-hoc names, persist trip with current `tripRate` and
  `passengerRate` from settings, set `dirty = true`, reset the form to
  defaults (today, no passengers).

### Zone 2 — Month list (middle)

```
April 2026                  £247.05  ▾
March 2026                  £164.70  ▸
```

Sorted newest month first. Current month auto-expanded on load; others
collapsed. Tap header to expand/collapse.

Expanded rows:

```
Tue 23 Apr   CE, DDR, JP        £43.89   ⓧ
```

- Sorted newest date first within a month.
- Empty passenger list shows as a blank space (no placeholder text).
- × opens a native `confirm("Delete this trip?")`; on OK, removes the trip
  and sets `dirty = true`.

Empty-state copy (no trips at all): "No trips recorded yet." Grand total row
and Claim-complete button hidden in this state.

### Zone 3 — Actions (bottom)

```
Grand total: £411.75

[  Export CSV  ]   [  Share  ]

[  Claim complete — delete all  ]
```

- **Export CSV** downloads `mileage-YYYY-MM-DD.csv` (today's date). Sets
  `dirty = false`.
- **Share** calls `navigator.share({ title, text })` with the plain-text
  payload (see "CSV + Share formats"). Sets `dirty = false` on successful
  share. If `navigator.share` is not available, the button hides itself at
  page load.
- **Claim complete — delete all** is disabled while `dirty === true`, with
  caption "Export or share first". When enabled: native `confirm("Delete all
  trips? This cannot be undone.")`; on OK, wipes `mileage.trips` and resets
  `dirty = false`.

### Settings sheet

A cog icon in the top-right opens a modal/sheet:

- **Trip rate** — numeric input, ≥0, ≤2 decimals.
- **Passenger rate** — numeric input, ≥0, ≤2 decimals.
- **Regulars** — list with add/remove. Each entry is a short string
  (uppercase by convention but not enforced). Removing a regular does not
  rewrite historical trips.
- **Save** validates inputs; disabled with inline error if invalid. Changes
  apply to future trips only.

## CSV + Share formats

### CSV (`mileage-YYYY-MM-DD.csv`)

```
Date,Passengers,Passenger count,Trip rate,Passenger rate,Amount
2026-04-23,"CE, DDR, JP",3,27.45,5.48,43.89
2026-04-17,,0,27.45,5.48,27.45
2026-03-28,"HD",1,27.45,5.48,32.93
,,,,Total,104.27
```

- Rows sorted newest-first across all months (flat list).
- Passenger names comma-separated inside a single cell, whole cell quoted to
  preserve the embedded comma.
- `Trip rate` and `Passenger rate` columns are per-row because they are
  frozen at entry time and may differ across rows after a rate change.
- Final row is a grand total with leading empty cells.
- File encoded as UTF-8, `\r\n` line endings, downloaded via a Blob + object
  URL.

### Share payload (plain text)

```
Mileage claim — 2026-04-23
Trips: 7
Total: £247.05

Tue 23 Apr — CE, DDR, JP — £43.89
Wed 17 Apr — — £27.45
Tue 28 Mar — HD — £32.93
...
```

Passed as the `text` field of `navigator.share({ title, text })`. `title` is
`Mileage claim — YYYY-MM-DD`.

## Edge cases and error handling

Handled explicitly:

- **No trips yet** — empty-state text, grand total and Claim-complete hidden.
- **Empty ad-hoc passenger row** — Save disabled until text is entered or the
  row is removed.
- **Whitespace in ad-hoc names** — trimmed on save.
- **Ad-hoc duplicates a regular** (e.g. user types "CE") — allowed, stored
  as typed. Not worth UI complexity to prevent.
- **Multiple trips on the same date** — allowed. Each gets its own row.
- **`localStorage` unavailable** (e.g. Safari Private Browsing) — on load, a
  write+read probe runs. On failure, a top-of-page banner appears: "Storage
  unavailable — trips won't be saved. Turn off Private Browsing." Save button
  disabled while this state holds.
- **Corrupt JSON in `localStorage`** — every parse wrapped in try/catch. On
  failure, log to console, treat that key as empty, and show a banner:
  "Could not load saved trips." Do not auto-delete the corrupt value — the
  user can inspect in DevTools.
- **Invalid rates in Settings** — numeric validation (≥0, ≤2 decimals). Save
  disabled with inline error if invalid.
- **`navigator.share` unavailable** — Share button hides itself at page load;
  CSV remains as the export path.

Not handled (YAGNI):

- Multi-device sync.
- Undo.
- Trip editing.
- Keyboard-only navigation, ARIA live regions, screen-reader polish beyond
  semantic HTML.

## Testing

No test framework. A hand-test checklist in the README:

1. Fresh load (empty storage) — shows empty state.
2. Log a trip today with 0 / 1 / 2 / 3 passengers — total matches expected.
3. Log a past-dated trip — appears under the correct month section.
4. Add free-text passenger, × to remove — total updates, 3-passenger cap
   enforced.
5. Edit rates in Settings — new trip uses new rate, old trips keep old rate.
6. Export CSV — file downloads; opens correctly in Numbers/Excel; totals
   match on-screen values.
7. Share — iOS share sheet appears with the plain-text payload.
8. Claim complete — disabled before export; after export, wipes all trips
   and re-disables itself.
9. Reload the page after each of the above — state survives.
10. Safari Private Browsing — storage-unavailable banner appears, Save
    disabled.
