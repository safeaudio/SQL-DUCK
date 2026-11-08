# SQLDUCK

Browser-based inspector for SQL Server storage files — `.mdf`, `.ldf`, `.bak`.
Single HTML file, vanilla JS, no build step, no dependencies, no network.
Everything is parsed locally; nothing is uploaded.

## Quick start

```bash
python3 -m http.server 8766
```

Open <http://localhost:8766/sqlduck.html> and drop a file in.

Chrome treats every `file://` page as a unique opaque origin, which blocks
downloads and clipboard access — so serve it over HTTP rather than
double-clicking the file. The tool detects this and shows a banner.

## What it does

* **Object Explorer** — SSMS-style tree of tables, views, procedures, functions,
  keys and constraints, recovered from the system base tables
* **Table content** — rows decoded into readable values using the recovered
  column types (int, money, datetime, decimal, bit, nchar/nvarchar, guid…).
  Drag a column edge to resize, double-click it to fit; widths persist per table
* **Procedure source** — definitions reassembled from `sysobjvalues` chunks,
  syntax highlighted, copy and download as `.sql`
* **Storage** — page map, allocation units, page-by-page and record-by-record view
* **Text extraction** — every readable string in the file, with regex filter and
  CSV export; the fastest way to confirm a value made it into a backup
* **Diagnostics** — what was recovered, from which allocation unit, and a manual
  override when the automatic pick is wrong

Nothing about the on-disk layout of the system catalogue is hardcoded. The tool
locates `sysschobjs` by the fact that its rows name the base tables, then derives
every column offset by testing candidates against the data, and verifies the
recovered columns by checking they predict row shapes that actually exist on disk.

## Files

| File | Purpose |
|---|---|
| `sqlduck.html` | the tool — this is the only file you need to use it |
| `STP-SQLDUCK.md` | self-test procedure: scope, fixture, results, manual GUI checks, known limits |
| `northwind_sim.bak` | test fixture — generated MTF backup with the Northwind schema |
| `make_bak.js` | regenerates the fixture |
| `fixture.json` | expected values, written by `make_bak.js`, consumed by `stp.js` |
| `stp.js` | STP harness — 101 checks against the fixture |
| `test.js` | catalogue layout independence — 21 checks, run at 4 different layouts |
| `test2.js` | synthetic pipeline including the decoy allocation unit — 29 checks |
| `test3.js` | page index scale, 1.4 M pages past the 4 GB offset — 11 checks |
| `test4.js` | resizable grid columns — 33 checks |
| `run-all.js` | runs every suite and prints one summary |

## Testing

```bash
node run-all.js      # everything, 258 assertions
node stp.js          # STP only, human readable
node stp.js --md     # STP results as a markdown table
node make_bak.js     # rebuild the fixture from scratch
```

The suites drive the real application code extracted from `sqlduck.html` in a
headless VM context, so they test what ships rather than a copy.

## Against a real database

The fixture is generated, not a Microsoft artefact — see section 2 of the STP for
exactly where it is and is not faithful. For a real run:

```bash
curl -L -o AdventureWorks2019.bak \
  https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorks2019.bak
```

Compare the recovered tree against `sys.tables` and `sys.columns` on a restored
copy before trusting it on anything that matters.

## Known limitations

* Off-row LOB chunks are marked, not followed
* Object to allocation-unit mapping is by row shape, so two tables with an
  identical fixed length and column count can collide
* Null-bitmap positions assume no dropped columns
* Nullability is not reported — the `syscolpars.status` bit cannot be located
  reliably, so it is left blank rather than guessed
* Compressed or encrypted backups yield no readable pages
* `.ldf` support is limited to the file header, text extraction and hex
