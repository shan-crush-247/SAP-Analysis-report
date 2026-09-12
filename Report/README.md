# SAP Replication Analytics — Local Report

Double-click **`SAP_Replication_Analysis_Report.html`**. That's it — the data is embedded in
the report, so it loads itself on open. No file picker, no server, no internet connection,
and nothing is ever uploaded anywhere.

It opens in well under a second. The data ships **pre-parsed** (gzipped JSON, 2.3 MB) rather
than as the raw workbook, because parsing the `.xlsx` in the browser cost about 6 seconds on
every single open.

## Using it

- Pick a section in the left sidebar: Overview, Databases, Tables, Columns, Stored
  Procedures, SP → Table Lineage, View Lineage, Unused Tables, Top Insights, and the two
  Insertion Trend views.
- Narrow things down with the search box, the column-header sorting, the dropdown filters,
  and the database filter at the top of each section.
- **⤓ Export Excel** / **⤓ Export HTML** save exactly what's currently filtered and sorted
  (the HTML export embeds the section's charts as images, so it can be shared on its own).
- **🔄 Reload / Change File** lets you point the report at a different workbook ad hoc,
  without changing the embedded copy.

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

**In the meantime**, the report can always read a workbook directly: click
**🔄 Reload / Change File** and pick any `.xlsx`. That path parses in-browser (slower, ~6s)
but needs no regeneration, so it's the quick way to look at an updated workbook.
