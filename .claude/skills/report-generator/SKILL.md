---
name: report-generator
description: Generate or refresh the local, offline HTML analytics dashboard for the SAP Replication analysis Excel workbook (or a similarly-shaped analysis workbook) in this project. Use when the user asks to build, regenerate, refresh, or update "the report" / "the dashboard" from the Excel file in Core File, or wants HTML/Excel export added to it.
---

# SAP Replication Report Generator

Builds a single, self-contained, **fully offline** HTML dashboard that loads its data
straight from an Excel workbook (client-side, in the browser — no server, no upload,
nothing leaves the machine). This is the standing tool for turning the analysis workbook
in `Core File\` into something a non-technical stakeholder can open and click through.

## Inputs

- **`Core File\`** — contains the source workbook (currently
  `SAP_Replication_Consolidated_Analysis_Report_V1.1.xlsx`). This is the data of record.
  Re-inspect it whenever the user says the workbook changed — sheet names, header rows,
  and column names are all hardcoded in the generated report (see "How parsing works"),
  so a structural change upstream requires updating the report's config, not just re-running
  something.
- **`Reference Report\`** — contains a reference app (`SCM_Analysis_Abhimanyu.html`), a
  *different* BI dashboard (SCM procurement) built by someone else. It exists to convey a
  **concept**, not a template to copy:
  - a single HTML file the user opens directly (`file://`), no install, no server
  - loads an Excel workbook client-side via SheetJS (`XLSX.read`)
  - sidebar navigation across analysis "modules"
  - KPI cards + Chart.js charts + sortable/filterable tables per module
  - CSV/Excel export of whatever is currently on screen
  - dark/light theme, drill-down between sections
  Never copy its specific fields, labels, colors, or module names — this project's data is
  SAP database/table/column/BI-usage analysis, not procurement, so the sections, KPIs, and
  columns must be derived from *this* workbook's actual sheets.

## Output

- **`Report\SAP_Replication_Analysis_Report.html`** — the app itself. Opens directly by
  double-click, no build step.
- **`Report\lib\embedded-data.js`** — the **pre-parsed** dataset as a JS global
  (`window.EMBEDDED_DATA = {name, generated, gz}`), where `gz` is base64 of gzipped JSON,
  ~2.3 MB. The report calls `bootFromEmbedded()` on startup and loads it via
  `loadEmbeddedData()`, so **the user never sees a file picker**.

  Two deliberate decisions here, don't undo either:
  - **Embedded rather than fetched.** A `file://` page cannot fetch a local `.xlsx` (Chrome
    blocks it without `--allow-file-access-from-files`) and a local server would mean a
    process to start, so embedding is what makes double-click-and-go work.
  - **Pre-parsed rather than the raw workbook.** Embedding the `.xlsx` itself was tried
    first: `XLSX.read` alone cost **~6.2 s on every open** (plus row hydration), which the
    user rejected as too slow. Pre-parsing cut it to **~0.35 s** and shrank the payload from
    13.2 MB to 2.3 MB. Never go back to shipping the raw workbook as the primary path.

  The JSON shape is `{name, generated, overview, sections:{<id>:{headers,rows,note}}}` with
  `rows` as **arrays aligned to `headers`** (not objects — that's most of the size saving);
  `loadEmbeddedData()` rehydrates them into objects so the rest of the report is identical
  for both load paths.

  Keep the raw-`.xlsx` path (`loadWorkbook`) and `showPicker()` intact — they back the
  "Reload / Change File" button and the fallback when `embedded-data.js` is missing or the
  browser lacks `DecompressionStream`. Don't inline the payload into the HTML; a separate
  file keeps the report editable.

### Second data file: `growth-data.js`

`Report\lib\growth-data.js` (`window.EMBEDDED_GROWTH = {name, generated, gz}`) packs
`Core File\SAP_Database_Growth_AllTables_Report.xlsx` the same way: `{name, generated, note,
summary:{headers,rows}, detail:{headers,rows}}`. Sheet `01_DB_Wise_Growth_Summary` has its note in
row 0 and **headers on row 2**; `02_Table_Wise_Growth_Detail` has headers on row 0. It drives
`buildDbGrowth()` (the Replication Server growth tile and page). Things to keep true when touching it:
- The workbook's "Approx Growth … (rows)" is **inserts + updates**. Storage growth must use
  inserts only (`Avg Insert / Day`, `Busiest Month/Year Rows (Insert)`).
- Rows are sized per table with that table's `Data Size (MB) ÷ Row Count` from the replication
  catalogue (`STATE.data.tables`), matched on database + table name without the `dbo.` prefix.
- Per Year is each table's busiest calendar year, not per-day × 365 — don't "fix" that.
- Regenerate it with the same headless-Chrome pack step as below, XHR-ing this workbook instead.

### Third data file: `checklist-data.js`

`Report\lib\checklist-data.js` (`window.EMBEDDED_CHECKLIST = {name, sheet, generated, gz}`) packs the
**`DB Size`** sheet of `Core File\DBA Daily Checklist.xlsx` as `{name, sheet, generated,
dbs:[{name, ip, obs:[[excelSerialDate, sizeMB], …]}]}`. It drives the Replication Server growth
tile and page, which trend each database's readings by least squares. Notes:
- That sheet's dates sit in row 3 (Excel serials); row 4 says what each column holds. **The
  layout changes partway through**: May's blocks are a single `DB Size` column per date, but
  from 1 June (column BK onward) each date is four columns — `MDF`, `LDF`, `Total`,
  `Growth Diff Previous day` — with the date on the **MDF** column. Read row 4 to decide:
  `MDF` means the date's total is at `col+2`, otherwise the total is the dated column itself.
  Taking the dated column blindly silently mixes data-file-only values into the totals (this
  bug understated log growth once already). Non-numeric cells (`Sunday`, `Holiday`, `Leave`)
  are skipped.
- The checklist tracks the **source** databases that feed replication, on their own servers, so
  their sizes differ from the replica's catalogue sizes — don't treat the two as interchangeable.
- Status rules: Growing / Shrinking / Low confidence (<20 readings or <30 days) / No readings.
  Totals count growing databases only, so a maintenance shrink can't cancel real growth.
- The file is usually **open in Excel**, which locks it. Copy it to a temp path first and pack
  from the copy, otherwise reading fails with a sharing violation.

### Regenerating `embedded-data.js` after the workbook changes

The embedded data is a snapshot, so it must be rebuilt whenever `Core File\*.xlsx` changes.
Reuse the report's own parser rather than writing a second one:

1. Re-embed the raw workbook temporarily so the generator has something to read:
   base64 the `.xlsx` into `lib/embedded-data.js` as
   `window.EMBEDDED_WORKBOOK = {name, generated, b64}` (PowerShell:
   `[Convert]::ToBase64String([IO.File]::ReadAllBytes($xlsx))`).
2. `cp SAP_Replication_Analysis_Report.html _gen.html` and append a `<script>` that:
   `XLSX.read`s `window.EMBEDDED_WORKBOOK.b64` → builds `{overview, sections}` using the
   page's own `findSheet`/`sheetToAoa`/`parseOverviewSheet` and each config's `headerRow`,
   `noteRow` and `dropRow` → `JSON.stringify` → `CompressionStream("gzip")` → base64 →
   writes it into a `<pre>` between the markers `GENDATA:` and `:ENDGEN`.
3. Run it headless and extract (the marker charset keeps it HTML-safe, and the strict
   base64 class avoids matching the script's own source text in the dump):
   ```
   chrome.exe --headless=new --disable-gpu --no-sandbox --user-data-dir=<tmp> \
     --virtual-time-budget=300000 --dump-dom "file:///.../_gen.html" > gen.html
   grep -o 'GENDATA:[A-Za-z0-9+/=]\{1000,\}:ENDGEN' gen.html \
     | sed 's/^GENDATA://; s/:ENDGEN$//' > payload.b64
   ```
4. Write `window.EMBEDDED_DATA = {name:"...", generated:"...", gz:"<payload>"};` to
   `lib/embedded-data.js`, delete `_gen.html`, and re-run the verification test below.
- **`Report\lib\xlsx.full.min.js`** and **`Report\lib\chart.umd.min.js`** — vendored
  (downloaded once, committed to disk) copies of SheetJS and Chart.js. The HTML references
  them by relative path so the report works with **no internet connection**. If these ever
  go missing, re-fetch them:
  ```
  curl -sL -o "Report/lib/xlsx.full.min.js" https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js
  curl -sL -o "Report/lib/chart.umd.min.js" https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js
  ```
  Do not switch these to CDN `<script src="https://...">` tags — the workbook contains
  internal server/DB/login details and the whole point is that this never phones home.

## How the report works (so you can extend it correctly)

Everything lives in one `<script>` block in the HTML file. Key pieces, in order:

1. **`SHEET_CONFIGS`** — one entry per report section (`databases`, `tables`, `columns`,
   `sps`, `splineage`, `viewlineage`, `unused`, `insights`, `insTables`, `insDb`). Each entry
   declares: which sheet it reads (`sheetCode`, matched by the sheet name's numeric prefix,
   e.g. `"03"` matches `03_Table_Wise_Summary`), which row the real header is on
   (`headerRow`, 0-indexed), the `columns` to render (with a `type`:
   `text`/`int`/`num1`/`pct`/`pctsmall`/`size`/`badge`), which column is `dbKey` (drives the
   global database filter — `null` if the sheet has no database column), `kpis()` and
   `charts()` builder functions, `searchable` columns, optional `localFilters` (dropdowns
   built from distinct column values), and optional `dropRow(row)` to discard rows.

   **Header rows are not uniform and must not be assumed.** In the current workbook:
   sheets 02/03/04/05/08 have headers on row 0; sheets 06/07/09 have a title in row 0 and
   headers on row 1; sheets **10/11 have a note in row 0, a blank row 1, and headers on
   row 2**. Verify per sheet before editing — this exact detail silently produced zero rows
   on two sections until the functional test caught it.

   `dropRow` currently strips the **`TOTAL` summary row** from sheet 02 — without it every
   database-level KPI is exactly double the true figure. Watch for similar total/footer rows
   if new sheets are added.
2. **`01_Dashboard`** (the Overview sheet) is *not* in `SHEET_CONFIGS` — it's a hand-laid-out
   executive summary: KPI label/value row pairs at the top, then several small two-column
   tables sitting side by side further down (and the sheet's real content starts at cell
   **B2**, not A1). `parseOverviewSheet()` therefore locates blocks by **anchor text**
   (`findAnchor` looks for a heading and its right-hand neighbour, e.g. `"Status"`+`"Tables"`,
   `"Database"`+`"Storage GB"`) rather than fixed coordinates, and detects KPI blocks as a
   row of text labels sitting directly above a row of values in the same columns. Prefer
   extending that anchor approach over hardcoding cell positions — the layout moved once
   already.
3. **`fixSheetRange()`** — **do not remove this.** Several sheets in this workbook declare a
   `!ref` range that doesn't cover their real used area (the Dashboard sheet declares
   `B2:S77`), and `sheet_to_json` trusts `!ref`, silently returning a truncated or empty
   grid. This rebuilds `!ref` from the actual cell addresses, anchored at A1 so row/column
   indices stay absolute. Every sheet is passed through it before conversion.
4. **`aoaToObjects()`** — generic sheet → array-of-objects parser used for every sheet except
   Overview, given a header row index.
5. **Database names are spelled inconsistently across sheets** (`dbo`/`DBO`,
   `RRLive`/`RRLIVE`). The global filter's list of databases comes from the sheet 02 summary
   (the authoritative list, 18 entries) and matching is case-insensitive. Building the list
   as a union of every sheet's database column instead yields ~139 bogus entries.
6. **`getFilteredSortedRows(id)`** — applies the global DB filter, any local filters, the
   search box, and the current sort, in that order. Every KPI, chart, table render, and
   export reads through this function so they always stay in sync.
7. **`refreshSection(id)` / `renderTable(id, rows)`** — generic renderer shared by all 10
   data sections (KPIs → charts → paginated/sortable table). Don't duplicate this per
   section; extend the shared renderer or the per-section config instead.
8. **`exportExcel(id)` / `exportHtml(id)`** — export the *currently filtered* rows (not just
   the visible page) of the active section. Excel export uses SheetJS `writeFile`; HTML
   export builds a standalone, styled HTML file (with the section's chart(s) baked in as PNG
   snapshots via `canvas.toDataURL`) and downloads it via a Blob — both are real browser
   downloads (this file is not a hosted Artifact, so `<a download>` works normally, unlike in
   the Artifacts sandbox).

## Inspecting the source workbook (no Python/Node available here)

This environment does not have Node or Python installed. `.xlsx` is a zip of XML, so
inspect it with pure PowerShell + `System.IO.Compression` instead of trying to install a
runtime:
- open the file as a `ZipArchive`
- read `xl/workbook.xml` for sheet names/order and `xl/_rels/workbook.xml.rels` to map
  sheet `r:id` → the actual `xl/worksheetN.xml` part
- read `xl/sharedStrings.xml` for the shared-string table (text cells reference it by index)
- for each sheet, read the first several `<row>` elements and resolve each `<c>` cell's
  value, so you can see real header rows and sample data without opening Excel

This is the only reliable way to know true header-row positions before editing
`SHEET_CONFIGS` — don't guess from row 0 alone, several sheets here have a title row first.

## Testing a change (this environment has no Node/Python either)

Use a **temporary self-reporting harness page + headless Chrome `--dump-dom`**. This is
simple and reliable; don't reach for the DevTools Protocol over a raw WebSocket (tried,
it deadlocks on blocking socket reads and gives no partial output).

1. Copy the report to a scratch file next to it (so relative `lib/` paths still resolve):
   `cp SAP_Replication_Analysis_Report.html _test_harness.html`
2. Append a `<script>` to the copy that: XHRs the workbook as an `arraybuffer` from
   `../Core%20File/<file>.xlsx`, calls the page's own `loadWorkbook(buf, name)` and
   `onWorkbookLoaded()`, then loops every section id calling `navigate(id)` and collecting
   assertions — KPI text, `#tableWrap tbody tr` count, `#charts canvas` count, count of
   `—` cells in the first row (a high count means a column-key mismatch), and any `NaN`/
   `undefined` in KPIs. Wrap each step in try/catch, push failures into an errors array,
   and write the whole result as JSON into a hidden `<pre>` prefixed with `TESTRESULT:`.
3. Run it and grep the dumped DOM for that marker:
   ```
   chrome.exe --headless=new --disable-gpu --no-sandbox --allow-file-access-from-files \
     --user-data-dir=<fresh temp dir> --virtual-time-budget=120000 --dump-dom \
     "file:///D:/VS_PROJECTS/SAP%20Analysis%20report/Report/_test_harness.html" > dump.html
   grep -o 'TESTRESULT:.*' dump.html
   ```
4. **Delete the harness copy afterwards** — it must not ship in `Report\`.

Notes:
- `--allow-file-access-from-files` is required, otherwise the XHR for the workbook is blocked.
- Percent-encode spaces in the `file:///` URL. A raw path with spaces gets split into extra
  argv tokens and Chrome dies with `Multiple targets are not supported in headless mode`.
- A full run takes **several minutes** (10 MB workbook, ~150k rows) — run it in the
  background and wait, rather than assuming it hung.
- Cross-check a couple of numbers against the workbook itself rather than just "it rendered":
  e.g. filtering to `RHLLive` should leave 3,709 table rows, matching that database's
  `Table Count` on sheet 02.

Always run this full load-and-click-through test after changing `SHEET_CONFIGS`,
`parseOverviewSheet`, or any renderer — the large sheets (Tables: ~40k rows, Columns:
~104k rows) are exactly where silent header-offset or key-mismatch bugs show up (a column
renders as "—" everywhere, or a KPI is `NaN`), and they're easy to miss by eyeballing the
code.

## When asked to change scope

- **New section from a new sheet**: add one entry to `SHEET_CONFIGS` (columns, kpis,
  charts, filters) — don't invent a new rendering path, reuse `refreshSection`/`renderTable`.
- **A different source workbook entirely** (not just an updated version of this one): re-run
  the inspection step from scratch, don't assume this workbook's sheet layout carries over.
- **Publishing/sharing**: this report is intentionally local-only because the workbook
  contains internal server/DB/account details. Don't add artifact publishing, telemetry, or
  any network call unless the user explicitly asks and understands that tradeoff.
