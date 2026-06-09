---
id: ADR-003
title: Two-layer observability architecture — health records + run records
status: Accepted
date: 2026-03-26
---

## Context
Observability needs span two distinct use cases:
1. **Dataset-level trends**: DQ drift per dataset over time (20 runs of team_elos, is DQ declining?)
2. **Run-level audit trail**: Complete execution summary (all 5 datasets processed, 4 passed, 1 failed, why?)

A single record type would either duplicate data or require expensive joins.

## Decision
Implement two append-only record stores, each optimized for its query pattern:

### Layer 1: Health Records (per dataset, per run)
```
health/basketball/ncaab/2025/team_elos/{run_id}.json
```
Captures DQ state of one dataset in one run:
```json
{
  "run_id": "20250326T143000",
  "dataset_name": "team_elos",
  "passed": true,
  "dq_score": 1.0,
  "row_count": 12,
  "failed_checks": 0,
  "check_results": [...]
}
```
**Query pattern**: `health/basketball/ncaab/2025/team_elos/` → all team_elos history → detect drift.

### Layer 2: Run Records (aggregate, one per pipeline invocation)
```
runs/basketball/ncaab/2025/{run_id}.json
```
Summarizes entire run:
```json
{
  "run_id": "20250326T143000",
  "overall_success": true,
  "passed": 5,
  "failed": 0,
  "file_results": [
    { "dataset_name": "team_elos", "success": true, "dq_score": 1.0 },
    ...
  ]
}
```
**Query pattern**: `runs/basketball/ncaab/2025/` → timeline of runs → detect execution anomalies.

## Alternatives considered
- Single unified record (bloated, requires joins for different queries)
- Health records only (cannot answer "was the entire run successful?")
- Run records only (lose granular DQ history per dataset)
- Database/DuckDB (adds infrastructure, defeats "append-only in git" design)
- Both records in same file (violates single-responsibility, makes updates awkward)

## Consequences
- Two writes per run (health records in validate stage, run record after all files complete)
- Health records are written immediately after validate, even on failure — enables "why did this dataset fail?"
- Run records written by dispatch.py after all file results collected — provides timeline view
- Health records live in dataset-specific folders → efficient ls to get history
- Run records live in run folders → efficient ls to get execution timeline
- Monitor.py can query both: health for drift detection, runs for silence detection
- Future analytics can join on (dataset_name, run_id) if needed
- Append-only design survives accidental reruns (new run_id = new record, no conflicts)

