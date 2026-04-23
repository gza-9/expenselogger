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
