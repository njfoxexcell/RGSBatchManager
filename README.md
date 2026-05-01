# RGS Batch Manager

A portable Windows desktop tool for splitting pending `RGS*` Sales Entry batches in Dynamics GP (`EXCEL.dbo.SY00500`) into smaller chunks via the `dbo.usp_SplitSOPBatchIntoChunks` stored procedure.

Forked from [DvsBatchManager](https://github.com/njfoxexcell/DvsBatchManager); only the source-batch filter, chunk-prefix format, and hold handling differ.

## What It Does

1. Connects to `ExcellSQL\ERP` / `EXCEL` (Windows Auth).
2. Lists every pending `Sales Entry` batch with `BACHNUMB LIKE 'RGS%'` (batch number, trx count, decoded status, created/modified, comment).
3. User picks one or more batches via checkboxes.
4. **Mark Completed** runs `EXEC dbo.usp_SplitSOPBatchIntoChunks` once per selected batch with:
   - `@SourceBatchNumber` = the selected batch
   - `@ChunkBatchPrefix` = `RGIS` + `DDMMYY` (10 chars)
   - `@ChunkSize` = `100`
   - `@HoldToRemove` = `''` (empty — proc skips SOP10104 hold removal)
5. Errors surface in a message box; success refreshes the list.

## Build

Requires .NET 9 SDK on Windows.

```bash
# Debug build
dotnet build

# Portable single-file self-contained exe (~51 MB, no .NET runtime needed)
dotnet publish -c Release -r win-x64 --self-contained true \
  -p:PublishSingleFile=true -p:EnableCompressionInSingleFile=true \
  -o dist
```

The published exe lives at `dist/RGSBatchManager.exe` and runs on any Windows 10/11 x64 machine without installation.

## Configuration

Server, database, source-batch prefix, chunk-prefix root, chunk size, and the hold ID to strip are constants near the top of [`MainForm.cs`](MainForm.cs). Adjust and rebuild to change.

## Caveats

- The chunk prefix is the same for every selected batch in a run (`RGIS` + today's date), with no source-batch suffix. The proc adds a 3-digit chunk number, so the namespace per day is `001`–`999`. If you ever split more than ~999 docs total in one day across all RGS batches, the `chunk batch already exists` guard will fire — split the rest tomorrow, or add a source-batch suffix in `BuildChunkPrefix`.
- Hold removal is disabled (`@HoldToRemove = ''`). If RGS batches use a process hold that should be cleared during the split, set `HoldToRemove` in `MainForm.cs` to that hold ID and rebuild.
