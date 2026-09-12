# SAP Replication Analytics — Local Report

Double-click **`SAP_Replication_Analysis_Report.html`**. That's it — the workbook is
embedded in the report, so it loads its own data on open. No file picker, no server, no
internet connection, and nothing is ever uploaded anywhere.

There's a short loading spinner on open while it indexes ~150,000 rows.

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
| `lib\embedded-data.js` | The workbook, embedded (~13 MB). This is what loads automatically. |
| `lib\xlsx.full.min.js`, `lib\chart.umd.min.js` | Excel parsing + charting, vendored for offline use. |

Keep the `lib\` folder beside the HTML file. You can copy the whole `Report` folder to any
machine and it works the same, with no setup.

## Refreshing the data

The embedded copy is a snapshot. When the workbook in `..\Core File\` is updated, re-embed it:

```powershell
$xlsx  = "D:\VS_PROJECTS\SAP Analysis report\Core File\SAP_Replication_Consolidated_Analysis_Report_V1.1.xlsx"
$outJs = "D:\VS_PROJECTS\SAP Analysis report\Report\lib\embedded-data.js"
$b64   = [System.Convert]::ToBase64String([System.IO.File]::ReadAllBytes($xlsx))
$js    = "window.EMBEDDED_WORKBOOK = {name:""" + (Split-Path $xlsx -Leaf) + """, generated:""" + (Get-Date -Format "yyyy-MM-dd HH:mm") + """, b64:""" + $b64 + """};"
[System.IO.File]::WriteAllText($outJs, $js, [System.Text.Encoding]::UTF8)
```

If the new workbook also changed shape (sheets added, headers moved), use the
`report-generator` skill in `.claude\skills\` — it documents the parsing rules and the
test procedure.
