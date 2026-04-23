# Research Dashboard

Self-contained HTML dashboards for tracking clinical trial recruitment, built on REDCap + Excel.

## Structure

```
dashboard_template.html     # generic starting point for any REDCap crossover RCT
morpho/
  morpho_dashboard.html     # MORPHO/MOSA study (UCSD — opioid-induced sleep apnea)
  CLAUDE.md                 # developer notes for the MORPHO dashboard
```

## How it works

Each dashboard is a single HTML file. No build step, no server required.

1. Export REDCap data to Excel via Power Query (one row per `record_id` × `redcap_event_name`)
2. Open the dashboard in Chrome or Edge
3. Click **Pick xlsx file** and select the export
4. Data loads instantly in the browser via [SheetJS](https://sheetjs.com/)

Auto-refresh is supported via the [File System Access API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_Access_API) (Chrome/Edge only). Set Power Query to refresh on a schedule and enable AutoSave in Excel — the dashboard will pick up changes automatically.

## Starting a new study

1. Copy `dashboard_template.html` and rename it for your study
2. Follow the five-step checklist at the top of the `<script>` block:
   - Set your REDCap event names in `EVENTS`
   - Update `MAPS` with your field codings
   - Rename `clinical_var1_*` / `clinical_var2_*` to your variable names
   - Update `buildSubjects()` to pull the fields your study uses
   - Adjust the sample data generator ranges
3. Open in a browser with **Load sample data** to verify the layout before connecting real data

## Features

- KPI cards with sparklines (screened → eligible → enrolled → randomized → completed)
- Recruitment vs target trajectory charts
- Participant timeline (per-subject swimlane)
- Demographics & clinical variable histograms (stacked by randomization status)
- Screen fail and early discontinuation reason charts
- Adverse events and protocol deviation tables with per-column filtering
- Global date-range and arm filters
- Works offline — all dependencies loaded from CDN on first open, then cached by the browser

## Local development

```bash
python -m http.server 8000
# open http://localhost:8000/dashboard_template.html
```

Serving locally is required for the File System Access API to work. Shift-click the **Refresh** button to load sample data without a real xlsx file.
