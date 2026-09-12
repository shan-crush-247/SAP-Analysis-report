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

Eleven sections, reachable from the sidebar:

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
| Daily Growth (Estimated) | Estimated data written per day per table — cells written, and what it costs in storage per day, month and year |

Every section supports Excel-style per-column dropdown filters, click-to-sort on any column,
a section-wide search, a database filter, frozen leading columns, and a TOTAL row summing
each numeric column across all filtered rows. Column headings match the workbook exactly.
**⤓ Export Excel** and **⤓ Export HTML** save exactly what's currently filtered and sorted,
totals included — the HTML export embeds the charts as images so it stands alone.

The Daily Growth figures are **derived, not measured**: each table's average row size comes
from its own catalogue entry (`Data Size (MB) ÷ Row Count`) multiplied by its approximate
rows written per day. It covers only the sampled tables, so treat it as a lower bound. The
method is spelled out in the Methodology panel on that page.

## Repository layout

| Path | Purpose |
|---|---|
| `Core File/` | The source workbook. The data of record. |
| `Report/` | The generated dashboard — open `SAP_Replication_Analysis_Report.html` |
| `Report/lib/embedded-data.js` | The analysed data, pre-parsed and gzipped (~2.3 MB); this is what auto-loads |
| `Report/lib/*.min.js` | SheetJS and Chart.js, vendored so the report works offline |
| `Reference Report/` | An unrelated SCM dashboard, kept only as a design reference |
| `.claude/skills/report-generator/` | Skill used to regenerate or extend the report |

## Refreshing the data

The embedded copy is a snapshot. After updating the workbook in `Core File/`, re-embed it —
see the PowerShell snippet in [`Report/README.md`](Report/README.md). If the new workbook
also changes shape (sheets added, header rows moved), follow the `report-generator` skill,
which documents the parsing rules and the verification procedure.
