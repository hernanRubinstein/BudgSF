# BudgSF

A single-page monthly budget calculator, Hebrew/RTL UI, used by two people (household) to plan a
USD monthly budget. It's a PWA with no build tooling — the deployed artifact is the source file.

## File map

- `index.html` — the entire app: markup, CSS, and JS all inline. This is the only file to edit.
- `manifest.json` — PWA manifest (name, icons, standalone display).
- `apple-touch-icon.png`, `icon-192.png`, `icon-512.png` — PWA icons.

There is no `package.json`, no source directory, no bundler. Nothing gets compiled — whatever is
in `index.html` is what runs in the browser and what gets deployed.

## Conventions to preserve

- **Single file, vanilla JS.** Keep markup, styles, and script inline in `index.html`. Do not split
  into modules, add a framework, or introduce a bundler/npm dependency — the single-file simplicity
  is intentional for a small personal tool, not a gap to fix.
- Small helper functions near the top of the `<script>` block do the formatting/DOM-lookup work:
  `$` (getElementById), `num` (parse a field as float), `usd`/`eur` (currency formatting) —
  `index.html:246-252`. Reuse these rather than re-deriving formatting logic.
- UI strings are Hebrew, page direction is RTL (`dir="rtl"` on `<html>`). Keep new UI text
  consistent with that (Hebrew labels, RTL-aware layout).

## Data model / core calculation

`compute()` (`index.html:341-351`) is the single source of truth for all derived numbers:

- **Income** = EMBO stipend (`embo`, annual EUR ÷ 12) converted to USD via `eurusd` rate, minus a
  conversion `fee`%, plus optional spouse income (`spouseIncome`, only if `spouseWorks` is checked).
- **Expenses** = `rent` + sum of `livingCats` (a user-editable array of `{id, name, value}` living
  expense categories, rendered/edited via `renderCats()`/`addCat()`/`deleteCat()`,
  `index.html:392-437`) + Israel debt repayment.
- **Israel debt repayment** = (`ilsCard` + `ilsLoan`, both in ILS) ÷ `usdils` rate, plus the same
  conversion `fee`%.
- **Surplus** = income − expenses. Negative surplus is styled/labeled as a deficit throughout.

Everything downstream (the summary card, the live donut chart via `drawLivePie()`, the exported PNG
report) is derived from this one function — don't duplicate the calculation elsewhere.

## Persistence & sync

- `localStorage` (`sfBudgetSettings`, `sfBudgetHistory`) is the local source of truth and works
  fully offline.
- Firebase Realtime Database mirrors the same state at the `budget/shared` path (project
  `sf-budget-6d4e4`) so both household members see each other's edits, via `schedulePush()` /
  `applyRemote()` (`index.html:296-327`). Writes are debounced (500ms) and tagged with a random
  per-tab `MY_ID` so a device ignores its own echo.
- The Firebase web config embedded in the script (apiKey, etc.) is a public client key — this is
  normal for Firebase and not a secret; access control is enforced by RTDB security rules, not by
  hiding this config.
- If the Firebase scripts fail to load or `initFirebase()` throws, the app degrades to local-only
  (`setSync('local')`) — sync is additive, never a hard dependency.

## Monthly history & image export

`summarizeMonth()` (`index.html:525-548`) snapshots the current `compute()` result into `hist[]`,
keyed by year-month (re-saving the same month overwrites its entry rather than duplicating it).
`reportDataURL()` (`index.html:553-625`) renders a shareable PNG summary via `<canvas>`, and
`generateImage()` triggers `navigator.share()` (falling back to a download link) so either partner
can save/share the month's summary as an image.

## Deployment

Hosted on **Firebase Hosting** under the same project as the Realtime Database (`sf-budget-6d4e4`,
https://console.firebase.google.com/u/0/project/sf-budget-6d4e4/overview). Deploying is
`firebase deploy` from the repo root — there is no CI pipeline; changes to `index.html` take effect
on next deploy.

## Known hardcoded values

Default exchange rates (`eurusd` at `index.html:188`, `usdils` at `index.html:207`) and the rates
quoted in the on-page disclaimer (`index.html:213`) reflect market rates at a point in time. Nudge
these when they drift noticeably from current rates — they're planning defaults, not a live feed.

## Testing

No test suite, no linter. Verify changes by opening `index.html` in a browser (or the Firebase
Hosting URL) and exercising both tabs — the calculator (adjust rent/income/categories, check the
totals and pie chart update) and the tracking view (summarize a month, delete a month, undo).
