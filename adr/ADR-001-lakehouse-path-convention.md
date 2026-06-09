---
id: ADR-001
title: Lakehouse path convention — sport/league/table/season/
status: Accepted
date: 2026-03-18
---

## Context
Data spans multiple sports, leagues, seasons, and datasets. Storage needed a consistent namespace that scales cleanly as new sports and leagues are added without code changes.

## Decision
Adopt the lakehouse pattern: `catalog / schema / table / partition`
Mapped to this project: `sport / league / table / season`

```
parquet/basketball/ncaab/team_elos/2025/
json/basketball/ncaab/team_elos/
runs/basketball/ncaab/2025/
```

Sport is the catalog boundary — datasets across sports share nothing analytically.
League is the schema boundary — CFL, NFL, XFL are all football but meaningfully different.
Table is the dataset — one folder per CSV type.
Season is the partition — surgical recovery and reprocessing without touching other seasons.

## Alternatives considered
- `sport/season/` — too flat, no table-level isolation
- `league/season/` — loses sport grouping, harder to reason about cross-league queries
- Hive-style `season=2025/` — compatible with Athena/Spark auto-discovery but ugly and unnecessary for a GitHub-based store

## Consequences
- Adding a new sport or league is a new folder, not a code change
- Season partitioning limits blast radius — reprocessing one season never touches another
- Structure is directly compatible with DuckDB, Spark, and Athena if query engine is added later
- Gold layer (web/) intentionally breaks the pattern — entity-based, not season-partitioned, because Flask pages need cross-season data
