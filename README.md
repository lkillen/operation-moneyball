# Operation Moneyball

<!-- TODO: 1-2 sentence description of what this project is and why you built it -->

A data engineering portfolio project. The pipeline is the product.

## Repos

| Repo | Purpose |
|---|---|
| [moneyball-analytics](#) | Analytical workspace — models, transforms, egress |
| [moneyball-pipeline](#) | Bronze in, Silver parquet + Gold JSON out |
| [moneyball-storage](#) | Shared artifact store — Silver + Gold layer |
| [moneyball-notifications](#) | Standalone Discord notification module |
| [moneyball-web](#) | Flask app — visual proof the pipeline works |

<!-- TODO: Replace # with actual GitHub repo URLs when split -->

## Documentation

**Start here**: [docs/INDEX.md](docs/INDEX.md) — Navigation hub for all documentation

### Quick Links
- **Architecture decisions**: See [adr/](adr/) for Architecture Decision Records
  - [ADR-001: Lakehouse path convention](adr/ADR-001-lakehouse-path-convention.md)
  - [ADR-002: Packet run_id propagation](adr/ADR-002-packet-run-id-propagation.md)
  - [ADR-003: Two-layer observability](adr/ADR-003-two-layer-observability.md)
  - [ADR-004: Permissive schemas for testing](adr/ADR-004-permissive-schemas-for-testing.md)
  - [ADR-005: JSON shaper registry](adr/ADR-005-json-shaper-registry-pattern.md)

- **Implementation guides**:
  - [PIPELINE_WALKTHROUGH.md](docs/implementation/PIPELINE_WALKTHROUGH.md) — How the pipeline works, how to add datasets
  - [QUICK_REFERENCE.md](docs/implementation/QUICK_REFERENCE.md) — Cheat sheet for common operations

- **Module documentation**:
  - [moneyball-pipeline/README.md](../moneyball-pipeline/README.md) — Pipeline overview and status
  - [moneyball-pipeline/health/README.md](../moneyball-pipeline/health/README.md) — Health module
  - [moneyball-pipeline/observability/README.md](../moneyball-pipeline/observability/README.md) — Observability module
  - [moneyball-pipeline/tests/README.md](../moneyball-pipeline/tests/README.md) — Testing guide

- **Session summary**: [docs/SESSION_SUMMARY_20250326.md](docs/SESSION_SUMMARY_20250326.md) — What was accomplished in this session

See [`docs/architecture/folder_structure.md`](docs/architecture/folder_structure.md) for the full path convention and data journey.

See [`operation_moneyball_architecture.html`](operation_moneyball_architecture.html) for the system diagram.

## Current sports

- `basketball/ncaab`
- `football/nfl` — placeholder

## Status

<!-- TODO: Brief note on where the project is at — planning, in progress, deployed, etc. -->
