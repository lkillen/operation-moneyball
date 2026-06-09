---
id: ADR-002
title: Packet run_id propagation — immutable audit trail through pipeline stages
status: Accepted
date: 2026-03-26
---

## Context
The pipeline processes files independently through multiple stages (ingress → validate → transform → publish). Each file execution needs to be traceable end-to-end for:
- Observability reporting (correlate all artifacts from a single run)
- Health record tracking (link DQ scores to a specific execution)
- Debugging and incident response (find all files affected by a specific run)

Without run_id, artifacts (parquet files, JSON, health records) are orphaned from their execution context.

## Decision
Add `run_id: str` as the first field in the base `Packet` dataclass. Run_id is generated once per pipeline invocation (ISO 8601 timestamp: `%Y%m%dT%H%M%S`), passed to all packets, and propagates through inheritance to `DatasetPacket`, `ParquetPacket`, and `JsonPacket`.

```python
@dataclass
class Packet:
    run_id: str  # Generated once per dispatch.run() invocation
    sport: str
    league: str
    season: str
    dataset_name: str
```

Each stage receives packets with run_id intact, appends it to all output artifacts, and passes it downstream. Run_id becomes the immutable join key across all pipeline artifacts.

## Alternatives considered
- Generate run_id per file per stage (loses cross-stage correlation)
- Use Git commit SHA as run_id (decouples from human-readable timestamps)
- Store run_id separately in a mapping file (adds another lookup + synchronization risk)
- Skip run_id entirely, use timestamps on files (fragile, filesystem-dependent)

## Consequences
- All packets carry 7 required fields (run_id + 6 identity fields) — minor overhead
- Run_id must be passed to every packet constructor — enforced by dataclass, cannot be forgotten
- Health records, parquet files, and JSON outputs include run_id — enables exact correlation across all artifacts
- Run records aggregate all file results under a single run_id — enables timeline analysis
- Future monitoring queries can `GROUP BY run_id` to reconstruct complete execution context
- Enables idempotency check: if same run_id rerun, determine delta from previous records

