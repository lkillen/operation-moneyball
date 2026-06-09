# Moneyball — Folder Structure Reference

Monolithic layout for local reference. Will be split into individual repos later.
See `operation_moneyball_architecture.html` for the full architecture diagram.

## Path convention

Follows the lakehouse pattern: `catalog / schema / table / partition`
Mapped to this project: `sport / league / table / season`

```
moneyball/
├── operation-moneyball/          # Overview repo — README, ADRs, architecture docs
│   ├── adr/                      # Architecture Decision Records
│   └── docs/architecture/        # Diagrams, structure reference (this file)
│
├── moneyball-analytics/          # All analytical intelligence lives here
│   ├── .github/
│   │   ├── workflows/            # generate_manifest.yml — triggers on push to egress/, commits manifest.json
│   │   └── scripts/              # generate_manifest.py — CI tooling, not analytical work
│   ├── data/raw/                 # Raw inputs (sports module data, source CSVs)
│   ├── transforms/               # Collaborator certified data transformation scripts
│   │   └── {sport}/{league}/
│   ├── models/
│   │   ├── random_forest/        # Random Forest prediction models
│   │   ├── monte_carlo/          # Monte Carlo simulations
│   │   └── elo/                  # Elo ratings
│   ├── registry/
│   │   └── lookup/               # Ex: Canonical ID lookup table (~50mb CSV)
│   └── egress/                   # BRONZE handoff zone → pipeline
│       ├── {sport}/              # Catalog
│       │   └── {league}/         # Schema
│       │       └── {season}/     # Season (Partition)
│       └── basketball/ncaab/2025/ # ex
│
├── moneyball-pipeline/           # Purely mechanical — Bronze in, Silver parquet + Gold JSON out
│   ├── .github/workflows/        # pipeline.yml · push_to_storage.yml
│   ├── dispatch.py               # Entry point — reads manifest, loops over declared files, fires independent run per file
│   ├── core/                     # Shared packet contracts — DatasetPacket, TransformPacket
│   ├── temp/                     # BRONZE transient scratch — receives from egress/, never committed
│   │   ├── {sport}/{league}/{season}/
│   │   └── basketball/ncaab/2025/ # ex
│   ├── ingress/                  # Reads manifest · verifies files · loads CSVs · wraps as DatasetPacket
│   ├── validate/                 # Bronze → Silver boundary · schema contracts · abort if fail
│   ├── schema/                   # Schema contracts per dataset — columns, types, keys, null thresholds
│   │   └── {sport}/{league}/
│   ├── transform/                # DatasetPacket → parquet (Silver) + JSON (Gold) → TransformPacket
│   ├── publish/                  # Atomic write of TransformPacket artifacts to moneyball-storage
│   ├── observability/            # Run logs · data freshness · DQ scores per dataset
│   ├── manifests/                # Run manifests · row counts · DQ score · input fingerprint
│   ├── health/                   # DQ reports · per-file pass/fail · thresholds · audit trail
│   └── tests/                    # pytest + DQ tests · failure aborts run, storage untouched
│
├── moneyball-storage/            # Shared GitHub repo — Silver recovery layer + run manifests
│   ├── parquet/                  # SILVER
│   │   ├── {sport}/{league}/{table}/{season}/
│   │   └── basketball/ncaab/team_elos/2025/ # ex
│   └── runs/                     # Manifests · DQ scores · versioning · APPEND ONLY
│       ├── {sport}/{league}/{season}/
│       └── basketball/ncaab/2025/ # ex
│
│   # GOLD — entity-shaped JSON lives in Cloudflare R2, NOT in this repo
│   # Uploaded by pipeline.yml Job 4 (deploy_gold) after each run
│   # Path on R2: {sport}/{league}/{entity}.json
│
├── moneyball-notifications/      # Standalone notification module · triggered by any repo's workflows
│   └── discord/                  # formatter.py · sender.py · POSTs to Discord #pipeline-alerts
│
└── moneyball-web/                # Flask + Gunicorn · Render free tier · auto-deploys on storage push
    └── app/
        ├── static/
        │   ├── css/
        │   └── js/
        ├── routes/               # Mirrors Gold layer namespace
        │   └── {sport}/{league}/
        └── templates/            # Jinja templates
            └── {sport}/{league}/
```

## Medallion tiers

| Tier   | Location                              | Description                                       |
|--------|---------------------------------------|---------------------------------------------------|
| Bronze | `egress/` → `temp/`                   | CSVs as received · unvalidated by pipeline        |
| Silver | `moneyball-storage/parquet/`          | Post-DQ validation · parquet · recovery layer     |
| Gold   | Cloudflare R2                         | Entity-shaped JSON · uploaded by pipeline · Flask fetches on demand |

## Path convention

| Layer      | Pattern                              | Location |
|------------|--------------------------------------|----------|
| Silver     | `parquet/sport/league/table/season/` | moneyball-storage (GitHub) |
| Gold       | `sport/league/entity.json`           | Cloudflare R2 |
| Runs       | `runs/sport/league/season/`          | moneyball-storage (GitHub) |
| Temp       | `temp/sport/league/season/`          | moneyball-pipeline (runner only, never committed) |
| Egress     | `egress/sport/league/season/`        | moneyball-analytics (GitHub) |
| Schema     | `schema/sport/league/`               | moneyball-pipeline (GitHub) |
| Transforms | `transforms/sport/league/`           | moneyball-analytics (GitHub) |

## Data journey

```
egress/sport/league/season/  →  temp/sport/league/season/  →  Ingress (reads manifest)
                                                                      ↓
                                                              Validate + DQ
                                                              ↓ abort if fail
                                                         Silver parquet (sport/league/table/season/)
                                                                      ↓
                                                              JSON transform
                                                                      ↓
                                                         Gold JSON (sport/league/entity/)  →  Flask
```
