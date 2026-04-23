# Roadmap: Rust Sidecar Migration

## Phase 0: Documentation and Test Fixtures

Deliverables:

- Development docs in `docs/rust-sidecar/`.
- Synthetic Apple Health XML fixture generator.
- Small sample ZIP/XML fixtures for scan/export tests.
- Baseline measurement script for current Python memory and elapsed time.

Acceptance:

- Documented command protocol is stable enough for implementation.
- At least one small fixture contains `Record`, `Workout`, multiple sources,
  and at least two matching categories.

## Phase 1: Rust CLI MVP

Deliverables:

- `engine/` Rust crate.
- `healthpro-engine scan-sources`.
- `healthpro-engine export`.
- ZIP/XML input support.
- `Record` and `Workout` extraction.
- CSV output matching current Python columns.
- JSONL progress events.
- Default category rules embedded in Rust or loaded from JSON.

Acceptance:

- CLI scans sample `export.xml`.
- CLI scans sample `export.zip`.
- CLI exports selected sources into current 15 category names.
- CLI memory remains bounded on generated large fixture.
- CLI can be run independently from terminal.

## Phase 2: PyQt Integration Behind Feature Flag

Deliverables:

- Python sidecar resolver.
- `QProcess`-based scan controller.
- `QProcess`-based export controller.
- JSONL parser and log renderer.
- Temporary selected source JSON generation.
- Environment flag fallback:
  - `HEALTHPRO_USE_RUST_ENGINE=1`
  - `HEALTHPRO_ENGINE_PATH=/path/to/healthpro-engine`

Acceptance:

- UI source scan works through Rust when flag is enabled.
- UI export works through Rust when flag is enabled.
- UI remains responsive during scan/export.
- Errors from Rust are shown in the existing log/error UI.

## Phase 3: Default Rust Path and Python Fallback

Deliverables:

- Rust engine enabled by default.
- Python implementation retained only as fallback or removed after confidence.
- Packaged builds include sidecar binary.
- README documents performance path and troubleshooting.

Acceptance:

- macOS packaged app finds and runs sidecar.
- Linux AppImage/deb finds and runs sidecar.
- Windows installer finds and runs sidecar.
- Small-file output parity with old Python path is verified.

## Phase 4: Performance and Output Enhancements

Deliverables:

- Benchmarks for scan/export throughput.
- Optional Parquet output.
- Optional SQLite ingest mode based on Apple-Health-Resonator-CLI.
- Better progress estimates using ZIP entry uncompressed size where possible.
- Cancel support from GUI.

Acceptance:

- Large-file export is measurably faster than Python path.
- Parquet output is validated with Polars or DuckDB.
- SQLite mode supports basic read-only queries if added.

## Phase 5: Cleanup

Deliverables:

- Remove pandas from the hot path.
- Remove old DataFrame-dependent export code if fallback is no longer needed.
- Normalize dependency declarations.
- Add CI jobs for Rust tests and Python syntax checks.

Acceptance:

- `requirements.txt`, `requirements-linux.txt`, and `pyproject.toml` no longer
  disagree about core runtime dependencies.
- CI builds Rust engine for target platforms.
- Documentation reflects the new default architecture.

## Implementation Order

1. Add Rust crate and compile a `healthpro-engine --help`.
2. Port input detection from current Python and Apple-Health-Resonator-CLI.
3. Implement XML stream reader using `quick-xml`.
4. Implement `scan-sources`.
5. Implement category config loading.
6. Implement direct CSV export.
7. Add CLI tests with synthetic fixtures.
8. Add Python `QProcess` integration behind feature flag.
9. Add packaging hooks.
10. Make Rust the default path.

## Risk Register

| Risk | Impact | Mitigation |
|---|---|---|
| CSV output differs from current Python output | Users may see changed files | Add fixture parity tests before switching default |
| XML edge cases fail in Rust parser | Some exports fail | Keep per-record error log and test real exports |
| Sidecar path resolution fails in packaged app | App cannot scan/export | Implement explicit resolver and package-specific tests |
| Windows path/encoding issues | Broken source matching or output paths | Use UTF-8 JSON files and Rust `PathBuf`; test Windows build |
| Category matching creates duplicate rows across categories | Output semantics may differ | Preserve current behavior first; document multi-category routing |
| Re-reading XML for export takes time | Scan then export reads twice | Accept for bounded memory; future SQLite/Parquet cache can avoid repeat reads |

## Validation Checklist

- `scan-sources` returns correct sources for XML.
- `scan-sources` returns correct sources for ZIP.
- `export` respects selected source filter.
- `export` writes UTF-8 BOM CSV by default.
- `export` splits files after configured chunk size.
- `export` preserves `Record` field mapping.
- `export` preserves `Workout` field mapping.
- `export` writes no empty category files.
- GUI remains responsive while sidecar runs.
- Cancel/terminate behavior does not leave corrupted partial files unnoticed.
- Packaged app can locate sidecar without environment variables.
- Large fixture peak RSS is bounded and documented.

## Benchmark Plan

Recommended benchmark scenarios:

```text
small:      10k records, 5 sources
medium:    1M records, 8 sources
large:     20M+ records, 10 sources
zip-large: same large XML compressed in ZIP
```

Metrics:

- scan elapsed time
- export elapsed time
- peak RSS
- rows written
- output size
- errors/skips

Compare:

- current Python path
- Rust sidecar CSV path
- future Rust Parquet path

## Definition of Done

The Rust sidecar migration is complete when:

- The default GUI path uses Rust for scan and export.
- 10GB+ XML exports no longer require full-record in-memory materialization.
- CSV output remains compatible with the current category contract.
- Packaged apps ship and locate the sidecar on supported platforms.
- The old pandas hot path is removed or clearly marked as fallback only.
