# Rust Sidecar Development Docs

This folder defines the development plan for replacing the current Python XML
processing core with a Rust sidecar engine while keeping the PyQt UI intact.

## Documents

- [PRD.md](./PRD.md): product requirements, goals, non-goals, user flows, acceptance criteria.
- [ARCHITECTURE.md](./ARCHITECTURE.md): process boundary, module layout, data flow, packaging strategy.
- [COMMANDS_AND_SCHEMA.md](./COMMANDS_AND_SCHEMA.md): CLI commands, stdout JSONL protocol, config schemas, output contracts.
- [ROADMAP.md](./ROADMAP.md): staged implementation plan, milestones, risks, validation checklist.

## Core Decision

The PyQt application remains the desktop shell. Rust owns all heavy data work:

```text
PyQt UI
  -> healthpro-engine scan-sources
  -> healthpro-engine export
  -> generated CSV / optional Parquet / optional SQLite
```

This avoids a full GUI rewrite and removes the current memory bottleneck caused
by building a full Python list and pandas DataFrame before export.
