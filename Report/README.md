# SAP Replication Analytics — Local Report

Double-click **`SAP_Replication_Analysis_Report.html`**. That's it — the data is embedded in
the report, so it loads itself on open. No file picker, no server, no internet connection,
and nothing is ever uploaded anywhere.

It opens in well under a second. The data ships **pre-parsed** (gzipped JSON, 2.3 MB) rather
than as the raw workbook, because parsing the `.xlsx` in the browser cost about 6 seconds on
every single open.

## Using it

- Pick a section in the left sidebar: **Overview & Databases** (the landing page — summary
  KPIs grouped by theme and charts on top, the per-database table underneath), Tables,
  Columns, Stored Procedures, SP → Table Lineage, View Lineage, Unused Tables, Top Insights,
  the two Insertion Trend views, **Daily Growth of BI Reports**, and **Daily Growth of
  Replication Server** (measured database-wise growth per day, month and year, broken down by
  table type and by BI linkage).
- Every column heading has an **Excel-style dropdown filter**: tick the values you want, or
  type in "Text contains". Each entry shows how many rows carry that value. Headings also
  sort on click, and there's a section-wide search plus a database filter. **Clear filters**
  lights up whenever any filter is active and resets everything.
- The first columns (database and name/count) stay **frozen** while you scroll sideways.
- Each table ends with a **TOTAL row** summing every numeric column across all filtered
  rows — not just the page you're looking at.
- Column headings are exactly as they appear in the workbook, so they line up with the
  source data and with anything exported.
- **⤓ Export Excel** / **⤓ Export HTML** save exactly what's currently filtered and sorted,
  totals included (the HTML export embeds the charts as images, so it stands alone).
- **🔄 Refresh** re-reads the embedded dataset and redraws the current section, keeping your
  filters.

## Files

| File | Purpose |
|---|---|
| `SAP_Replication_Analysis_Report.html` | The report. Open this. |
| `lib\embedded-data.js` | The analysed data, pre-parsed and gzipped (~2.3 MB). This is what loads automatically. |
| `lib\xlsx.full.min.js`, `lib\chart.umd.min.js` | Excel parsing + charting, vendored for offline use. |

Keep the `lib\` folder beside the HTML file. You can copy the whole `Report` folder to any
machine and it works the same, with no setup.

## Refreshing the data

The embedded data is a snapshot taken from the workbook in `..\Core File\`. When that
workbook is updated, `lib\embedded-data.js` has to be regenerated — the report will keep
showing the old numbers until it is.

Regenerating means re-running the workbook through the report's own parser and re-packing
the result, so it isn't a one-line copy. The procedure is written up in the
**`report-generator` skill** (`.claude\skills\report-generator\SKILL.md`) — ask Claude Code
to refresh the report data and it will follow it, including the verification step that
checks the row counts still line up with the workbook.

Note that **🔄 Refresh** re-reads the embedded dataset — it does not go back to the workbook
in `Core File\`, so it won't pick up a newer workbook on its own. That needs the
regeneration step above.
