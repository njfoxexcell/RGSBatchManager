# RGS Batch Manager — Claude Code Context

## Project
- **Name:** RGS Batch Manager
- **Repo:** TBD (will live alongside DvsBatchManager on github.com/njfoxexcell)
- **Local path:** C:/Users/njfox/RGSBatchManager/
- **Type:** Portable Windows desktop tool (WinForms, .NET 9, single-file self-contained)
- **Sister project:** [DvsBatchManager](https://github.com/njfoxexcell/DvsBatchManager) — same architecture, different filter/prefix/holds.

## What This App Does
Lists every pending `Sales Entry` batch in Dynamics GP whose `BACHNUMB LIKE 'RGS%'`, lets the user multi-select, and calls `EXCEL.dbo.usp_SplitSOPBatchIntoChunks` once per selected batch to split each into 100-doc chunks. The new chunk batches are named `RGIS` + `DDMMYY` + `NNN` (3-digit chunk suffix appended by the proc).

## Tech Stack
- .NET 9 Windows Forms (`net9.0-windows`), nullable + implicit usings on
- `Microsoft.Data.SqlClient` 5.2.x (Windows Integrated Auth, `TrustServerCertificate=true`)
- Distributed as a single self-contained exe (~51 MB) — no install, no runtime dependency

## DB Connection
- **Server:** `ExcellSQL\ERP`
- **Database:** `EXCEL` (the GP company DB)
- Auth: Windows Integrated. The user running the exe needs read on `SY00500` and execute on `dbo.usp_SplitSOPBatchIntoChunks`.

## Key Files
- [`Program.cs`](Program.cs) — WinForms entry point.
- [`MainForm.cs`](MainForm.cs) — single-form UI; holds the `Server` / `Database` / `SourcePrefix` / `ChunkPrefixRoot` / `ChunkSize` / `HoldToRemove` constants.
- [`BatchRepository.cs`](BatchRepository.cs) — `GetPendingRgsBatchesAsync` (filter `RGS%`, `NOLOCK`) and `SplitBatchAsync` (calls the proc, now also passes `@HoldToRemove`).
- [`app.manifest`](app.manifest) — long-path-aware; DPI awareness comes from `<ApplicationHighDpiMode>` in csproj.
- [`RGSBatchManager.csproj`](RGSBatchManager.csproj) — release config publishes single-file self-contained win-x64 with compression.

## Chunk Prefix Convention
`MainForm.BuildChunkPrefix(sourceBatch)` = `"RGIS"` + `DateTime.Now.ToString("ddMMyy")`.

The source-batch arg is intentionally **ignored** — every batch in a single run gets the same prefix. The proc appends a 3-digit chunk number to make BACHNUMB unique. Example for 2026-04-30:
- Any RGS source → prefix `RGIS300426` → chunks `RGIS300426001`, `RGIS300426002`, …

**Collision risk:** if more than ~999 docs are split across all RGS batches in one calendar day, or if you re-run a previous day's split and chunks already exist, the proc returns error 50005-equivalent (`Chunk batch name X already exists`). The DVS sister app sidesteps this by appending the last 2 chars of the source batch to the prefix; if RGS hits this in practice, copy that approach.

## Hold Handling
`@HoldToRemove` is passed as `''` (empty), which sets `@StripHolds = 0` inside the proc — no `SOP10104` deletes happen. This is the safe default since RGS batches don't use the DVS-specific `DVSPOST` hold.

If RGS batches use a hold that should be cleared during split, change the `HoldToRemove` constant in `MainForm.cs` to that hold ID. The hold ID must exist in `EXCEL.dbo.SOP00100` or the proc errors out before doing anything.

## Stored Procedure Notes
Same proc as DvsBatchManager — see that repo's `CLAUDE.md` for the full signature, return codes, and atomicity guarantees. Important bits relevant here:
- `@ChunkBatchPrefix VARCHAR(12)` — the 10-char `RGIS` + `DDMMYY` fits without truncation. (Originally `VARCHAR(10)`; widened during DvsBatchManager work.)
- Each chunk is its own atomic transaction. Loop breaks on first failure; source batch is preserved.
- Source batch is deleted only on full reconciliation (no errors, all docs moved, `SOP10100` empty for source). `@DeleteSourceWhenEmpty` defaults to `1` and we don't override it.

## Build / Run
```bash
# Debug
dotnet build

# Portable single-file exe (output: dist/RGSBatchManager.exe)
dotnet publish -c Release -r win-x64 --self-contained true \
  -p:PublishSingleFile=true -p:EnableCompressionInSingleFile=true \
  -o dist
```

## Working Conventions
- All read queries against the GP DB use `WITH (NOLOCK)`.
- Timestamps are server-local (`DateTime.Now` on the C# side; the proc uses `GETDATE()`).
- Migrations / DDL go in `sql/NNN_description.sql`, idempotent — **the user runs them manually**, not the agent.
- Commit and push after completing a prompt set.
