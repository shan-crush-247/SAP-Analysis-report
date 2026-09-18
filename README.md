# SAP Analysis Report

An offline dashboard for the SAP replication server's storage and BI-usage analysis.

The source of truth is a consolidated Excel workbook covering 18 replicated databases —
40,388 tables, ~1.46M columns, ~6.96 billion rows, ~5.58 TB of replicated storage — and the
question it answers is which of that is actually used by BI, and what the rest is costing.
The headline finding: **474 tables (1.17%) are BI-used; 39,914 are not**, accounting for
roughly **4.28 TB of potentially reclaimable storage**.

> **Internal data.** This repository contains database, table, column and stored-procedure
> inventories for production systems, plus the replication server address. Keep the
> repository private.

## Opening the report

Double-click **`Report/SAP_Replication_Analysis_Report.html`**.

That's the whole setup — no install, no build step, no local server, and no internet
connection. The analysed data is embedded in the page (pre-parsed and gzipped, 2.3 MB), so
it loads itself on open in well under a second. Nothing is ever uploaded anywhere.

Keep `Report/lib/` next to the HTML file. The whole `Report` folder is portable — copy it to
any machine and it behaves identically.

## What's in it

Twelve sections, reachable from the sidebar:

| Section | What it covers |
|---|---|
| Overview & Databases | Executive KPIs grouped by theme, distribution charts, then per-database storage, object counts and BI usage coverage |
| Tables | All 40,388 tables with usage classification and size |
| Columns | Column catalogue with data types, keys, and BI usage counts |
| Stored Procedures | The 597 BI procedures and their dependency complexity |
| SP → Table Lineage | Which SAP tables each procedure depends on, directly or via views |
| View Lineage | BI views traced back to the SAP objects they read |
| Unused Tables | Tables replicated but never referenced by BI |
| Top Insights | The 50 largest tables, storage against BI usage |
| Insertion Trends | Approximate daily insert/update volume, per table and per database |
| Daily Growth of BI Reports | All 474 BI-used tables with their linear growth per day, month and year, and projected size in 1 year |
| Daily Growth of Replication Server | Database-wise growth on the linear average — each database's current size (from `analysis_2.xlsx`) divided by the time since go-live — reported with and without transaction logs, alongside the replica's data/log split |

Every section supports Excel-style per-column dropdown filters, click-to-sort on any column,
a section-wide search, a database filter, frozen leading columns, and a TOTAL row summing
each numeric column across all filtered rows. Column headings match the workbook exactly.
**⤓ Export Excel** and **⤓ Export HTML** save exactly what's currently filtered and sorted,
totals included — the HTML export embeds the charts as images so it stands alone.

Growth everywhere is a **straight-line average**, not a measurement: the size workbook is a
single snapshot, so growth per period = current size ÷ time since go-live (1 Jan 2016 to the
17 Sep 2026 snapshot — 3,912 days; both set in the report source). Real growth is usually
front-loaded or accelerating, so the recent rate is likely higher than this average for busy
databases and lower for stable ones, and log size reflects log management rather than data
volume — which makes the without-LDF figures the better indicator. The Methodology panel on
the growth pages spells this out.

## Repository layout

| Path | Purpose |
|---|---|
| `Core File/` | The source workbooks, the data of record: the replication analysis and the all-tables growth scan |
| `Report/` | The generated dashboard — open `SAP_Replication_Analysis_Report.html` |
| `Report/lib/embedded-data.js` | The analysed data, pre-parsed and gzipped (~2.3 MB); this is what auto-loads |
| `Report/lib/growth-data.js` | The all-tables growth scan, pre-parsed and gzipped (~100 KB); supplies row-activity figures |
| `Report/lib/replica-size-data.js` | The replica's own size per database from `analysis_2.xlsx` (~1 KB); every size in the report is rescaled to this footprint |
| `Report/lib/*.min.js` | SheetJS and Chart.js, vendored so the report works offline |
| `Reference Report/` | An unrelated SCM dashboard, kept only as a design reference |
| `.claude/skills/report-generator/` | Skill used to regenerate or extend the report |

## Refreshing the data

The embedded copy is a snapshot. After updating the workbook in `Core File/`, re-embed it —
see the PowerShell snippet in [`Report/README.md`](Report/README.md). If the new workbook
also changes shape (sheets added, header rows moved), follow the `report-generator` skill,
which documents the parsing rules and the verification procedure.
