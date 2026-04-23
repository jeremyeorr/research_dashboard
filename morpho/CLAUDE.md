# MORPHO / MOSA Recruitment Dashboard

Single-file HTML dashboard tracking enrollment, eligibility, randomization, safety,
and participant characteristics for the MORPHO/MOSA study — a randomized crossover
trial of acetazolamide vs placebo in patients with opioid-induced sleep apnea.

- **Arms**: `AZM-PLA` (rand_arm=1) and `PLA-AZM` (rand_arm=2)
- **Targets**: 50 screened+enrolled, 38 randomized by study start + 18 months
- **Source**: REDCap flat export → Excel (via Power Query) → `mosa_dashboard.html`
- **Owner**: Jeremy Orr (jeremyeorr@gmail.com), UCSD

## Repo layout

```
mosa_dashboard.html         # the dashboard — self-contained, ~1600 lines
morpho_export.xlsx          # REDCap export refreshed via Power Query (gitignored)
MORPHO recruitment.Rmd      # reference R report the dashboard mirrors (optional)
```

The HTML file bundles Chart.js 4.5, chartjs-adapter-date-fns, and SheetJS from
CDNs. No build step. Open it in a browser, point it at the xlsx, done.

## Development workflow

```bash
# Serve locally so File System Access API works on Chrome/Edge
python -m http.server 8000
# → http://localhost:8000/mosa_dashboard.html

# When iterating on the code, prefer Edit over Write — the file is large and
# most changes are localized to a single render function.
```

No test suite. Verify by reloading in the browser and clicking through each
filter/tab; the sample-data generator (Shift+click the refresh button, or the
"Use sample data" fallback) lets you exercise every code path without real
REDCap data.

## Data pipeline

1. REDCap flat export (one row per `record_id` × `redcap_event_name`). Event
   names: `screening_arm_1`, `baseline_arm_1`, `randomization_arm_1`,
   `overnight_1_arm_1`, `overnight_2_arm_1`, `study_completion_arm_1`,
   `log_forms_arm_1` (non-repeating + repeating log forms).
2. Power Query in `morpho_export.xlsx` pulls the CSV, normalizes, and writes
   a flat table to a known sheet name.
3. Dashboard reads the xlsx via SheetJS, then:
   - `normalizeRecords()` — coerces event names, strips whitespace, maps
     haven-style factor labels back to numeric keys using `LABEL_TO_KEY`.
   - `buildSubjects()` — collapses events into one row per subject with all
     derived fields (ages, BMI, AHI, MEQD, rand arm, completion status).
   - `buildAEs()` / `buildPDs()` — flattens repeating log-form rows, tagging
     each AE with `resolution_status` ("Resolved" / "Ongoing" / "Unknown")
     derived from `ae_outcome` + `ae_ongoing` + `ae_date_stop`.

`State.records` holds the raw rows; `State.subjects` / `State.aes` / `State.pds`
hold the derived views; `State.filtered.*` holds the currently-filtered slice
(driven by the top-of-page date range + rand-arm checkboxes).

## Rendering architecture

Every chart has a `renderXxx()` function that:
1. Calls `destroyChart(key)` to tear down any prior Chart.js instance
2. Reads from `State.filtered.*`
3. Creates a new Chart on a `<canvas>` inside a **positioned wrapper**
4. Stores the instance on `State.charts[key]` so the next render can destroy it

### The wrapper pattern (do not break this)

Chart.js with `maintainAspectRatio:false` measures its canvas's parent. If the
parent has no fixed height, Chart.js + the browser's layout feedback loop can
grow the canvas indefinitely on every resize/render — this is the bug that hit
the KPI tiles. Every chart canvas lives inside a wrapper like:

```html
<div class="kpi-spark"><canvas></canvas></div>
```

```css
.kpi-spark { position: relative; height: 32px; }
.kpi-spark canvas {
  position: absolute; inset: 0;
  width: 100% !important; height: 100% !important;
  display: block;
}
```

If you add a new chart, give its wrapper an explicit height (or flex it into a
sized parent). Do not set `maintainAspectRatio:true` as a workaround — it
breaks responsive layouts.

## Filters and state

```
State = {
  records, subjects, aes, pds,
  filtered: { subjects, aes, pds },   // post global filter
  tableFilters: { timeline: {...}, ae: {...}, pd: {...} },  // per-column
  charts: { [key]: ChartInstance }
}
```

- Global filters live in the header strip: start date, rand-arm checkboxes.
- Table filters are per-column, per-tab, preserved across re-renders with
  focus/caret restoration. Enum columns use a `<select>` driven by
  `STATUS_OPTIONS` or the `enumMap`/`enumOptions` on the column def.
- Study-start date (`cfg-start`): defaults to the earliest `date_screening`
  on load. If the user edits it, `dataset.userSet = '1'` locks that override
  so new data loads don't clobber it.

## Participant Characteristics section — the current design

Everything in this section (Summary Statistics grid AND the demographic charts)
is filtered to **enrolled-only** subjects (`s.consent_date != null`). Stacking
is by randomization status:

- `Randomized` → green (`#16a34a`)
- `Not randomized` → amber (`#ca8a04`)

Charts:
- Sex / Race / Ethnicity — horizontal stacked bars (`renderStackedCatBar`)
- Enrollment/randomization — 3-slice doughnut (pre-rand / AZM-PLA / PLA-AZM)
- AHI histogram — vertical stacked bars, bin edges `[0,5,15,30,60,∞]`
- MEQD histogram — vertical stacked bars, bin edges `[0,20,50,90,200,∞]`

Histogram bins are **right-open** (`[lo, hi)`) with the last bin catching
everything `>= 60` / `>= 200`. If you add bins, update both `edges` and
`labels` in the call site.

## REDCap field conventions

Fields referenced by key in `buildSubjects`:

```
Demographics:  age_screening, sex, race, ethnicity,
               height_subject_report, weight_subject_report, bmi_subject_report
Clinical:      meqd_screening, baseline_screen_ahi, screening_psg_ahi
Dates:         date_screening, consent_date, randomization_date,
               overnight_1_date, overnight_2_date, study_complete_date
Status:        eligibility_assessment, eligibility_pi_review, rand_arm,
               early_dc_reason, study_complete
AE fields:     ae_date_start, ae_date_stop, ae_description, ae_severity,
               ae_related, ae_outcome, ae_ongoing
```

BMI is computed as `(weight_lb / height_in^2) * 703` if `bmi_subject_report`
is missing.

AE resolution (`resolutionLabel`):
- `ae_outcome ∈ {1,2}` → Resolved
- `ae_outcome ∈ {3,4,5}` → Ongoing
- `ae_outcome == 6` → Unknown
- Else fall back to `ae_ongoing` and `ae_date_stop`

## Things to be careful about

- **Don't regenerate the whole file.** It's large and there's no test harness;
  a full rewrite will silently break edge cases. Use targeted `Edit` calls.
- **Don't remove `destroyChart()` calls** — Chart.js leaks canvases otherwise.
- **Don't hard-code bin counts in histograms.** The fixed-edge bins (0-5, 5-15,
  etc.) are clinically meaningful; the user asked for them explicitly.
- **Don't regress the global-filter scope.** Summary Stats + char charts are
  enrolled-only; everything else (screenings, eligibility, target lines) runs
  against the full filtered subject pool.
- **Excel serial dates** arrive as numbers from SheetJS. `toISO()` handles
  both serials and string dates — use it for any new date field.
- **Power Query refresh cadence**: the xlsx is refreshed manually by the user;
  the dashboard polls `file.lastModified` when the File System Access API is
  available, otherwise falls back to a manual "Refresh" button.

## Deferred / future

- REDCap direct API (bypass the Excel bridge). Would need a REDCap MCP server
  or a small CORS proxy; API token would live outside the repo.
- Per-site breakdown if the study opens additional sites.
- Export-to-PDF for DSMB reports (currently: browser print to PDF works fine).
