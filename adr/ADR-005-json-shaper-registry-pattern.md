---
id: ADR-005
title: JSON shaper registry pattern — pluggable dataset-specific transforms
status: Accepted
date: 2026-03-26
---

## Context
Transform stage converts Silver parquet (uniform, validated) into Gold JSON (dataset-shaped, Flask-friendly). Five datasets require different JSON structures — a days_games JSON looks nothing like a player_elo JSON.

The transform stage needs to:
1. Discover which shaper to use (lookup by dataset_name)
2. Apply dataset-specific logic (different JSON schemas per dataset)
3. Remain testable without committing to final Gold schemas (web layer still being designed)

## Decision
Implement a registry-based shaper system:

```python
# In transform/json_shaper.py
_SHAPERS = {}

def register(dataset_name: str):
    """Decorator to register a shaper function."""
    def decorator(func):
        _SHAPERS[dataset_name] = func
        return func
    return decorator

@register("days_games")
def shape_days_games(df: pd.DataFrame, packet: DatasetPacket) -> dict:
    return {"records": df.to_dict(orient="records")}

# ... 4 more @register decorators ...

def shape(df: pd.DataFrame, packet: DatasetPacket) -> dict:
    """Lookup and apply shaper for dataset."""
    shaper = _SHAPERS.get(packet.dataset_name)
    if not shaper:
        raise ValueError(f"No shaper registered for {packet.dataset_name}")
    return shaper(df, packet)
```

Shapers are registered at import time. Transform stage calls `shape()` which dispatches to the correct function. Each shaper is independent and testable.

## Alternatives considered
- Single monolithic transform function (unmaintainable, wrong JSON per dataset)
- Strategy pattern with classes (overkill for simple functions)
- Configuration-driven transforms (requires JSON/YAML schema definitions, defers decision)
- Separate transform scripts per dataset (fragmented, hard to test end-to-end)

## Consequences
- Adding a new dataset requires one new @register function
- Shapers are registered at module import → must be defined before `shape()` is called
- Tests verify all 5 shapers are registered (prevents forgotten datasets)
- Placeholder shapers return flat `{"records": [...]}` structure (sufficient until Gold schemas designed)
- Shapers can be iteratively refined as Gold JSON structure crystallizes
- If web layer requires cross-dataset JSON (e.g., composite API), transform stage must coordinate
- No shaper = explicit error (cannot accidentally skip a dataset)

