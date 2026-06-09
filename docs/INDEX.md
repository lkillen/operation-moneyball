# Documentation Index

Navigate the Operation Moneyball documentation by topic.

## Architecture & Design Decisions

**Where to start**: Read these to understand why the pipeline works the way it does.

- [ADR-001: Lakehouse path convention](../adr/ADR-001-lakehouse-path-convention.md) — Why we organize data as `sport/league/table/season`
- [ADR-002: Packet run_id propagation](../adr/ADR-002-packet-run-id-propagation.md) — Why run_id flows through all pipeline stages
- [ADR-003: Two-layer observability](../adr/ADR-003-two-layer-observability.md) — Why we have health records + run records
- [ADR-004: Permissive schemas for testing](../adr/ADR-004-permissive-schemas-for-testing.md) — Why initial schemas are maximally permissive
- [ADR-005: JSON shaper registry](../adr/ADR-005-json-shaper-registry-pattern.md) — Why we use a decorator pattern for dataset transforms

## Implementation Guides

**Where to go when you're building or modifying code**.

- [Pipeline Walkthrough](implementation/PIPELINE_WALKTHROUGH.md) — End-to-end flow from trigger to output. How to add datasets, modify stages, debug.
- [Quick Reference](implementation/QUICK_REFERENCE.md) — Cheat sheet for common operations (testing, debugging, schema management)

## Module Documentation

**Where to go for specific modules**.

### Pipeline Infrastructure
- [moneyball-pipeline/core/CHANGES.md](../../moneyball-pipeline/core/CHANGES.md) — What changed in this session (run_id addition, etc.)

### Pipeline Stages
- `moneyball-pipeline/ingress/` — Load CSVs, validate checksums
- `moneyball-pipeline/validate/` — Run DQ checks, produce health records
- `moneyball-pipeline/transform/` — Convert to parquet + JSON
- `moneyball-pipeline/publish/` — Write artifacts to storage

### Observability
- [moneyball-pipeline/health/README.md](../../moneyball-pipeline/health/README.md) — Dataset quality tracking per run
- [moneyball-pipeline/observability/README.md](../../moneyball-pipeline/observability/README.md) — Anomaly detection and Discord alerts

### Testing
- [moneyball-pipeline/tests/README.md](../../moneyball-pipeline/tests/README.md) — How tests are organized, how to run them, fixtures

## Quick Links

### I want to...

**Understand the overall architecture**
→ Read [ADR-001](../adr/ADR-001-lakehouse-path-convention.md), then [Pipeline Walkthrough](implementation/PIPELINE_WALKTHROUGH.md)

**Add a new dataset**
→ Follow [Adding a New Dataset](implementation/PIPELINE_WALKTHROUGH.md#adding-a-new-dataset) in the Walkthrough

**Add a new sport/league**
→ Follow [Adding a New Sport/League](implementation/PIPELINE_WALKTHROUGH.md#adding-a-new-sportleague) in the Walkthrough

**Debug a failed run**
→ Use [Debugging](implementation/PIPELINE_WALKTHROUGH.md#debugging) section and [Quick Reference](implementation/QUICK_REFERENCE.md#debugging)

**Run tests locally**
→ See [tests/README.md](../../moneyball-pipeline/tests/README.md)

**Understand data quality tracking**
→ Read [ADR-004](../adr/ADR-004-permissive-schemas-for-testing.md), then [health/README.md](../../moneyball-pipeline/health/README.md)

**Understand observability/alerting**
→ Read [ADR-003](../adr/ADR-003-two-layer-observability.md), then [observability/README.md](../../moneyball-pipeline/observability/README.md)

**See what changed in this session**
→ Check [core/CHANGES.md](../../moneyball-pipeline/core/CHANGES.md)

**Find common operations (tests, debugging, etc.)**
→ Use [Quick Reference](implementation/QUICK_REFERENCE.md)

## Documentation Structure

```
operation-moneyball/
├── adr/                    ← Architecture Decision Records
│   ├── ADR-001
│   ├── ADR-002
│   ├── ADR-003
│   ├── ADR-004
│   └── ADR-005
├── docs/
│   ├── INDEX.md           ← You are here
│   ├── architecture/       ← General architecture docs
│   ├── implementation/
│   │   ├── PIPELINE_WALKTHROUGH.md    ← How to build on the pipeline
│   │   └── QUICK_REFERENCE.md         ← Cheat sheet
│   ├── onboarding/        ← Getting started guides
│   └── ...

moneyball-pipeline/
├── core/
│   ├── CHANGES.md         ← What changed (run_id, health_dir)
│   ├── packets.py
│   ├── paths.py
│   └── enums.py
├── health/
│   ├── README.md          ← Health module docs
│   ├── recorder.py
│   └── store.py
├── observability/
│   ├── README.md          ← Observability module docs
│   ├── collector.py
│   ├── monitor.py
│   └── reporter.py
├── tests/
│   ├── README.md          ← Testing guide
│   ├── conftest.py
│   ├── test_stages.py
│   ├── test_pipeline.py
│   └── test_schemas.py
└── README.md              ← Pipeline overview
```

## For Future LLMs

This documentation was created to help you (and other LLMs) understand the pipeline architecture, navigate the codebase, and make informed decisions about changes or extensions.

**Start here**:
1. Read [ADR-001](../adr/ADR-001-lakehouse-path-convention.md) to understand the storage model
2. Skim [Pipeline Walkthrough](implementation/PIPELINE_WALKTHROUGH.md) to understand the flow
3. Check [core/CHANGES.md](../../moneyball-pipeline/core/CHANGES.md) to see what's been done recently
4. Use [Quick Reference](implementation/QUICK_REFERENCE.md) for specific operations

**Key concepts to understand**:
- **Packets**: Immutable dataclasses carrying data + metadata through stages (see ADR-002)
- **Run ID**: Unique identifier per pipeline invocation, join key across all artifacts
- **Health Records**: Per-dataset DQ tracking (see ADR-003)
- **Run Records**: Aggregate pipeline execution summary
- **Schemas**: Input (Silver) and output (Gold) contracts (see ADR-004)
- **Shapers**: Dataset-specific JSON transforms (see ADR-005)

**When adding features**:
1. Check if there's already an ADR for it
2. Read the relevant module README
3. Follow the implementation pattern in similar code
4. Update tests alongside your changes
5. Consider creating a new ADR if you're making architectural decisions

