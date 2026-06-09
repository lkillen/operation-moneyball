---
id: ADR-004
title: Permissive schemas for testing — all strings, nullable, zero thresholds
status: Accepted
date: 2026-03-26
---

## Context
Pipeline validation (validate stage) runs DQ checks against schema contracts. Initial schemas were undefined, blocking testing. Data types and acceptable null rates were unknown. Creating strict schemas requires domain expertise from the analytics team.

The product goal is to ship a working pipeline early with placeholder data, allow analytics team to define real constraints later, and avoid test/prod divergence.

## Decision
Create maximally permissive schema contracts for all 5 NCAAB datasets:
- **All columns**: type=string, nullable=true
- **Null thresholds**: null_threshold=1.0 (100% null allowed)
- **No min/max/allowed_values constraints**
- **Write mode**: batch (append adds complexity later)
- **Primary keys**: empty list (upsert dedup not needed for test data)
- **Schema version**: "2025-v1" (bumped if contracts change)

This ensures all test data passes validation (dq_score=1.0 guaranteed), unblocking transform/publish testing.

Contracts live at `schema/basketball/ncaab/silver/{dataset_name}.toml` and are owned by the analytics team — they will tighten constraints as data sources are validated.

## Alternatives considered
- Strict schemas from day one (blocks testing, requires domain knowledge up front)
- No schemas, skip validation (loses observability, defers tech debt)
- Type inference from test data (fragile, diverges from prod expectations)
- Schema comments with TODOs (unclear when/how to fill in, easy to ignore)

## Consequences
- All test data passes DQ checks → dq_score=1.0 for all test runs
- No real data quality signal yet (intentional, expected during bootstrap phase)
- Test suite cannot verify DQ failure paths (will be added when strict schemas exist)
- Analytics team must update contracts before production deployments
- Schema version bumps signal contract changes (enables downstream coordination)
- Permissive baseline makes validation code testable without domain expertise
- Future contract tightening is backward-compatible (only makes checks stricter)

