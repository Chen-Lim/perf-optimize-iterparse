# Architecture: Rust Sidecar Engine

## High-Level Shape

```text
health_app.py
  MainWindow
  ScanProcessController
  ExportProcessController
        |
        | QProcess + JSONL protocol
        v
healthpro-engine
  scan-sources
  export
```

The sidecar is an external executable, not a Python extension. This keeps the
initial integration simple and isolates failures: if Rust panics or exits with
an error, the GUI can show an error instead of crashing the whole process.

## Why Sidecar Instead of PyO3 First

- Lower packaging complexity for MVP.
- Easier debugging from terminal.
- CLI can be benchmarked independently.
- No Python ABI coupling.
- Works well with PyInstaller by copying one platform-native binary.
- Future PyO3 remains possible if direct in-process calls become necessary.

## Proposed Repository Layout

```text
health_app.py
categories/
  health_categories.json
engine/
  Cargo.toml
  src/
    main.rs
    cli.rs
    input.rs
    scanner.rs
    exporter.rs
    categories.rs
    csv_writer.rs
    progress.rs
    error.rs
docs/
  rust-sidecar/
```

The `engine` crate can reuse code and patterns from
`Apple-Health-Resonator-CLI`, especially:

- `quick-xml` streaming reader
- `zip` input handling
- `clap` command structure
- `serde` JSON output
- `rusqlite` batch writer patterns for the future SQLite mode
- JSONL error logging style

## Process Boundary

### Python Responsibilities

- File picker.
- Source checkbox UI.
- Theme and layout.
- Start/stop Rust process.
- Parse JSONL progress events.
- Render logs and final status.
- Locate sidecar executable in dev and packaged modes.

### Rust Responsibilities

- Open `.zip` or `.xml`.
- Locate the correct Apple Health XML entry.
- Stream XML with bounded memory.
- Sanitize known invalid bytes.
- Scan source names and counts.
- Filter selected sources.
- Route records to category writers.
- Split outputs by row threshold.
- Emit progress events.
- Write error logs.

## Data Flow: Scan

```text
Input ZIP/XML
  -> open input stream
  -> XML stream reader
  -> for each Record/Workout:
       sourceName -> source counter
       update totals
       emit periodic progress
  -> final JSON summary
```

Memory depends on:

- XML read buffer.
- Unique source map.
- Progress counters.

It does not depend on total record count.

## Data Flow: Export

```text
Input ZIP/XML
  -> open input stream
  -> XML stream reader
  -> for each Record/Workout:
       extract fields
       source filter
       category match
       write CSV row directly
       rotate part file if row limit reached
       emit periodic progress
  -> close writers
  -> final JSON summary
```

Memory depends on:

- XML read buffer.
- Selected source set.
- Category rule set.
- CSV writer buffers.
- Counters.

It does not depend on total record count.

## Category Matching

The current Python category map should move to a JSON config file. Matching
should be case-insensitive and based on whether normalized `type` contains any
configured keyword.

Example:

```json
{
  "id": "1_Heart_Cardio",
  "keywords": [
    "heartrate",
    "restingheartrate",
    "heartratevariability",
    "walkingheartrateaverage"
  ]
}
```

The Rust exporter should load this once and use normalized lowercase strings.
For MVP, a linear scan over 15 categories is acceptable. If categories grow,
replace with precompiled regex or Aho-Corasick.

## CSV Writer Strategy

Each active category owns a writer state:

```text
CategoryWriter
  base_name
  current_part_index
  current_row_count
  total_row_count
  csv::Writer<BufWriter<File>>
```

Writers should be opened lazily: only create a category file when the first row
for that category appears. This preserves the current "SKIPPED if no data"
behavior without empty CSV files.

## Output Splitting

Default row threshold should mirror current behavior:

```text
880000 rows per CSV part
```

For a small category:

```text
1_Heart_Cardio.csv
```

For a large category:

```text
1_Heart_Cardio_Part1.csv
1_Heart_Cardio_Part2.csv
```

## Progress Protocol

Use stdout JSONL. Every line must be valid JSON.

Examples:

```json
{"event":"started","command":"scan-sources","input":"/path/export.zip"}
{"event":"progress","processed":500000,"records":498000,"workouts":2000}
{"event":"done","records":12000000,"workouts":3200,"elapsed_ms":65000}
```

The GUI should treat unknown event types as loggable but non-fatal to allow
protocol evolution.

## Error Handling

Rust exits with:

- `0`: success
- `1`: user/input/config error
- `2`: parse error that prevents completion
- `3`: output write error
- `70`: internal unexpected error

Recoverable per-record errors should be counted and written to an error log.
Fatal errors should emit one final error JSON event before exit when possible.

## Packaging Strategy

Development mode:

```text
engine/target/release/healthpro-engine
```

Packaged mode:

```text
HealthPro.app/Contents/MacOS/healthpro-engine
dist/HealthPro/healthpro-engine.exe
dist/healthpro/healthpro-engine
```

Python should resolve the sidecar in this order:

1. Environment override: `HEALTHPRO_ENGINE_PATH`
2. Directory of `sys.executable`
3. Repository dev path: `engine/target/release/healthpro-engine`
4. PATH lookup

## Migration Strategy

The Rust engine should be added behind a small process controller. The old
Python implementation can remain as a temporary fallback until parity tests pass.

Recommended flags:

```text
HEALTHPRO_USE_RUST_ENGINE=1
HEALTHPRO_ENGINE_PATH=/custom/path/healthpro-engine
```

After validation, Rust becomes the default path and Python DataFrame processing
can be removed or kept only as a debug fallback.
