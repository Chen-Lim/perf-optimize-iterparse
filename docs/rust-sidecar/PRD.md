# PRD: Rust Sidecar Engine for Apple Health Pro

## Background

The current application uses PyQt for UI and Python for data processing. The
processing flow is already split into scan and export stages:

- `ParseThread` scans the ZIP/XML, discovers `sourceName` values, and builds a
  full pandas DataFrame.
- `ExportThread` filters that DataFrame by selected sources and writes category
  CSV files.

This works for smaller exports, but breaks down when `export.xml` is 10GB+.
The root cause is not `iterparse` itself. The issue is the full in-memory
materialization of every Apple Health `Record` and `Workout` as Python objects,
then pandas columns and filtered copies.

## Problem Statement

For very large Apple Health exports, users need the app to:

- Avoid memory growth proportional to XML size.
- Export selected sources faster than the current Python/pandas path.
- Preserve the existing GUI workflow.
- Keep generated CSV outputs compatible with the current 15-category behavior.
- Remain packageable as a no-Python-runtime desktop app on macOS, Windows, and Linux.

## Goals

- Replace the heavy Python scan/export core with a Rust sidecar binary.
- Keep the PyQt GUI as the user-facing shell.
- Implement streaming scan and streaming export with bounded memory.
- Preserve current source selection and category export UX.
- Emit structured JSONL progress events that the UI can consume.
- Support direct reading from `export.zip` and `export.xml`.
- Create a foundation for future Parquet and SQLite outputs.

## Non-Goals

- Do not rewrite the GUI in Rust.
- Do not introduce PyO3/maturin in the first implementation.
- Do not require users to install Rust or Python.
- Do not change the default CSV category names in the first release.
- Do not implement advanced data analysis in the sidecar.
- Do not parse nested workout routes or full metadata in the MVP unless needed
  for current CSV compatibility.

## Target Users

- Users exporting multi-GB or 10GB+ Apple Health archives.
- Users who want one-click CSV generation from a desktop app.
- Advanced users who may later use a standalone CLI for automation.

## User Flow

### Flow 1: Scan Sources

1. User selects an Apple Health `export.zip`.
2. PyQt starts `healthpro-engine scan-sources <input>`.
3. Rust streams the XML and counts sources without storing all records.
4. Rust emits progress JSONL and a final source summary.
5. PyQt renders source checkboxes.

### Flow 2: Export Selected Sources

1. User selects one or more sources.
2. PyQt writes selected sources to a temporary JSON file.
3. PyQt starts `healthpro-engine export <input> --sources <file> --out <dir>`.
4. Rust streams the XML again.
5. For each matching record, Rust routes it directly to the matching category writer.
6. Rust emits progress JSONL.
7. PyQt displays completion and output directory.

## Functional Requirements

### Input

- Accept `.zip` and `.xml`.
- For `.zip`, locate Apple Health XML entries by known names and fallback size
  heuristics.
- Ignore `export_cda.xml`.
- Support at least `export.xml`, `导出.xml`, and `輸出.xml`.
- Handle invalid vertical tab byte `0x0B` consistently with the current
  `CleanStream` behavior.

### Scan

- Extract `sourceName` from `Record` and `Workout`.
- Count total records and workouts.
- Count records per source.
- Return stable sorted source list.
- Keep memory bounded by number of unique sources, not by number of records.

### Export

- Filter by selected source names.
- Parse both `Record` and `Workout`.
- Preserve current output columns for CSV:
  - `type`
  - `value`
  - `unit`
  - `startdate`
  - `sourcename`
- Preserve current 15 category file base names.
- Split large CSV outputs using the configured row threshold.
- Use UTF-8 with BOM for CSV compatibility with Excel unless explicitly disabled.
- Apply the current reproductive value cleanup for
  `HKCategoryValueVaginalBleeding`.

### Progress and Errors

- Emit machine-readable JSONL to stdout.
- Emit human-readable diagnostics to stderr only.
- Include periodic processed counts.
- Include records written per category.
- Include skipped/error counts.
- Write detailed parse/export errors to a JSONL error log when possible.

### Packaging

- Rust binary must be included in PyInstaller output.
- App runtime must locate the binary relative to the Python executable or app bundle.
- macOS, Windows, and Linux builds must each use native Rust targets.

## Performance Requirements

MVP target requirements:

- Memory should remain effectively constant during scan/export.
- For 10GB+ XML, peak RSS should be bounded primarily by writer buffers and XML buffers.
- Export must not build a full in-memory table.
- Scan/export should provide progress at least every 500,000 parsed entities.

Stretch requirements:

- Export throughput should be at least 2x faster than current Python/pandas path
  on large files.
- CSV writer should avoid per-record heap churn where practical.
- Future Parquet output should be faster and smaller than CSV for large exports.

## Compatibility Requirements

- Existing UI flow remains recognizable.
- Existing generated CSV category names remain stable.
- Existing default chunk size behavior remains approximately stable.
- The Python fallback can remain temporarily during migration, but Rust should be
  the default once verified.

## Acceptance Criteria

- `healthpro-engine scan-sources export.zip` returns a valid source summary.
- `healthpro-engine export export.zip --sources selected.json --out output`
  writes expected category CSV files.
- PyQt can consume JSONL progress without blocking the UI.
- Running scan/export on synthetic large XML demonstrates bounded memory.
- Existing small sample exports produce category files compatible with the
  current Python implementation.
- PyInstaller bundles contain the Rust binary and run without requiring Rust.
