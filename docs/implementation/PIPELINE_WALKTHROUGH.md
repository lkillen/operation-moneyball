# Pipeline Implementation Walkthrough

This document explains how the pipeline works end-to-end, from trigger to output artifacts. It's designed for developers who need to understand, extend, or debug the pipeline.

## High-Level Flow

```
GitHub repository_dispatch
        ↓
    dispatch.py (entry point)
        ↓ reads manifest for each file
    For each file (e.g., team_elos.csv):
        ↓
    ingress.load() → reads CSV, creates DatasetPacket
        ↓ packet carries (sport, league, season, dataset_name, dataframe, run_id)
    validate.run() → runs DQ checks, produces dq_report
        ↓ includes (dq_score, passed, failed_checks)
    health.record() → writes health record to health/{sport}/{league}/{season}/{dataset_name}/
        ↓
    transform.run() → converts to ParquetPacket + JsonPacket
        ↓ both include run_id for traceability
    publish.run() → writes parquet to Silver, JSON to Gold staging
        ↓
    After all files complete:
    observability.collect() → aggregates results into run record
        ↓
    observability.monitor() → checks for anomalies
        ↓
    observability.reporter() → sends Discord alert
```

## Key Concepts

### The Packet Architecture

Packets are the currency of the pipeline — immutable dataclasses that carry data + metadata through stages.

```
┌─ Packet (base)
│  ├─ run_id: str
│  ├─ sport: str
│  ├─ league: str
│  ├─ season: str
│  └─ dataset_name: str
│
├─ DatasetPacket (input)
│  ├─ dataframe: DataFrame
│  ├─ source_path: Path
│  ├─ row_count: int
│  └─ checksum: str
│
└─ TransformPacket (output)
   ├─ ParquetPacket
   │  ├─ parquet_bytes: bytes
   │  ├─ row_count: int
   │  ├─ dq_score: float
   │  ├─ write_mode: WriteMode
   │  ├─ primary_keys: list[str]
   │  └─ schema_version: str
   │
   └─ JsonPacket
      ├─ payload: dict
      ├─ row_count: int
      └─ dq_score: float
```

**Rule**: Every packet constructor requires `run_id` as the first parameter. Run_id is generated once per `dispatch.run()` and is the join key across all artifacts.

Example:
```python
packet = DatasetPacket(
    run_id="20250326T143000",  # Always first, cannot be omitted
    sport="basketball",
    league="ncaab",
    season="2025",
    dataset_name="team_elos",
    dataframe=df,
    source_path=Path("..."),
    row_count=len(df),
    checksum=checksum,
)
```

### The Manifest Format

Each CSV has a sidecar manifest (e.g., `team_elos.manifest.json`):

```json
{
  "sport": "basketball",
  "league": "ncaab",
  "season": "2025",
  "filename": "team_elos.csv",
  "checksum": "sha256_hex_string",
  "row_count": 12
}
```

- **Filename** — basename of the CSV (ingress uses this to locate the file)
- **Checksum** — SHA256 of file contents (ingress validates for data integrity)
- **Row count** — expected rows (ingress validates, emit warning if diverges)
- **Sport/league/season** — context (redundant with path, but explicit for clarity)

Manifests are created by `moneyball-analytics/generate_manifest.yml` and committed to the repo.

### The Run ID

Generated once per pipeline invocation: `strftime("%Y%m%dT%H%M%S")`

Example: `20250326T143000` = 2025-03-26 at 14:30:00

**Purpose**: Join key across all artifacts of a single pipeline run. Enables queries like "show me all artifacts from run X" or "which datasets were processed in run X?"

Used by:
- Health records: `health/basketball/ncaab/2025/team_elos/20250326T143000.json`
- Run records: `runs/basketball/ncaab/2025/20250326T143000.json`
- Parquet files: `parquet/basketball/ncaab/team_elos/2025/{run_id}.parquet` (future)
- JSON files: `gold_staging/{run_id}/team_elos.json` (future)

### Health Records vs Run Records

**Health Record** (per dataset, per run):
- Written by `health.recorder.record()` immediately after validate
- Path: `health/sport/league/season/dataset_name/{run_id}.json`
- Contains: dq_score, row_count, failed_checks, check_results
- Used for: trend analysis, anomaly detection per dataset

**Run Record** (aggregate, per run):
- Written by `observability.collector.collect()` after all files complete
- Path: `runs/sport/league/season/{run_id}.json`
- Contains: overall_success, passed/failed counts, per-file results
- Used for: timeline analysis, silence detection

## Adding a New Dataset

### 1. Create Silver schema contract
File: `schema/basketball/ncaab/silver/new_dataset.toml`

```toml
[columns]
column_name = { type = "string", nullable = true }

[write_config]
write_mode = "batch"
primary_keys = []
schema_version = "2025-v1"
null_threshold = 1.0
```

Start permissive (all strings, nullable). Tighten constraints as data validation rules become clear.

### 2. Create Gold schema contract
File: `schema/basketball/ncaab/gold/new_dataset.toml`

```toml
[structure]
required_keys = ["records"]
record_floor = 0
```

Placeholder initially. Refine as web layer defines JSON structure.

### 3. Register JSON shaper
File: `transform/json_shaper.py`

```python
@register("new_dataset")
def shape_new_dataset(df: pd.DataFrame, packet: DatasetPacket) -> dict:
    return {"records": df.to_dict(orient="records")}
```

### 4. Add test CSV + manifest
Files:
- `temp/basketball/ncaab/2025/new_dataset.csv`
- `temp/basketball/ncaab/2025/new_dataset.manifest.json`

Manifest:
```json
{
  "sport": "basketball",
  "league": "ncaab",
  "season": "2025",
  "filename": "new_dataset.csv",
  "checksum": "sha256sum new_dataset.csv | awk '{print $1}'",
  "row_count": 123
}
```

### 5. Update tests
File: `tests/test_schemas.py`

Add dataset to parametrize lists in `TestSilverSchemas` and `TestGoldSchemas`:
```python
@pytest.mark.parametrize(
    "dataset",
    [
        "days_games",
        "defense_position_elo",
        "lineups",
        "player_elo",
        "team_elos",
        "new_dataset",  # Add here
    ],
)
```

### 6. Run tests
```bash
pytest tests/test_schemas.py -v
pytest tests/test_pipeline.py::TestPipelineIntegration::test_all_datasets_process
```

All tests should pass. Run `dry_run.py` to verify end-to-end.

## Adding a New Sport/League

### 1. Create schema directories
```bash
mkdir -p schema/baseball/mlb/silver
mkdir -p schema/baseball/mlb/gold
mkdir -p temp/baseball/mlb/2025
```

### 2. Create schema contracts for each dataset
Same process as "Adding a New Dataset" above.

### 3. Create test CSVs and manifests
Same process as "Adding a New Dataset" above.

### 4. Verify paths work
Test that `core.paths` functions generate correct paths:
```python
from core.paths import silver_dir, health_dir, runs_dir

assert silver_dir("baseball", "mlb") == Path("storage/parquet/baseball/mlb")
assert health_dir("baseball", "mlb", "2025", "team_stats") == Path("storage/health/baseball/mlb/2025/team_stats")
```

## Modifying Existing Stages

### Example: Change validation behavior

File: `validate/runner.py`

```python
def run(packet: DatasetPacket) -> dict:
    """Run DQ checks, return dq_report."""
    contract = schema_loader.get_dq_rules(packet.sport, packet.league, packet.dataset_name)

    # Run checks
    check_results = checker.run_checks(packet.dataframe, contract)

    # Compute score
    passed_rows = len(packet.dataframe) - sum(r["rows_failed"] for r in check_results)
    dq_score = passed_rows / len(packet.dataframe) if len(packet.dataframe) > 0 else 1.0

    return {
        "dq_score": dq_score,
        "passed": dq_score == 1.0,
        "failed_checks": sum(1 for r in check_results if not r["passed"]),
        "row_count": len(packet.dataframe),
        "check_results": check_results,
    }
```

**Important**: Always return a dict with the same keys (dq_score, passed, failed_checks, row_count, check_results). The dict is consumed by `health.recorder.record()`, so changes here require coordination.

## Debugging

### Check what was processed
```bash
ls -la runs/basketball/ncaab/2025/
cat runs/basketball/ncaab/2025/20250326T143000.json | jq .
```

### Check data quality history for a dataset
```bash
ls -la health/basketball/ncaab/2025/team_elos/
cat health/basketball/ncaab/2025/team_elos/20250326T143000.json | jq .
```

### Run pipeline locally
```bash
cd moneyball-pipeline
python3 dry_run.py --keep-dirs
# Inspect temp directories before cleanup
```

### Run specific test
```bash
pytest tests/test_pipeline.py::TestPipelineIntegration::test_silver_parquet_created -v -s
```

## Common Patterns

### Accessing packet data in a stage
```python
def transform(dataset_packet: DatasetPacket) -> TransformPacket:
    run_id = dataset_packet.run_id
    sport = dataset_packet.sport
    league = dataset_packet.league
    season = dataset_packet.season
    dataset_name = dataset_packet.dataset_name
    df = dataset_packet.dataframe

    # Do work...

    # Create downstream packet with same run_id
    return ParquetPacket(
        run_id=run_id,  # Propagate
        sport=sport,
        league=league,
        season=season,
        dataset_name=dataset_name,
        parquet_bytes=parquet_bytes,
        # ...
    )
```

### Writing artifacts with run_id in filename
```python
# Good: run_id in filename enables correlation
path = silver_dir(sport, league, season, dataset_name) / f"{run_id}.parquet"

# Also good: run_id in JSON metadata
payload = {
    "run_id": run_id,
    "records": [...],
}
```

### Querying health history
```python
from health import store

# Get all records for a dataset
records = store.get_records(sport, league, season, dataset_name)

# Compute rolling average DQ score
dq_scores = [r["dq_score"] for r in records[-30:]]  # Last 30 runs
avg_dq = sum(dq_scores) / len(dq_scores)

# Check for drift
current_dq = records[-1]["dq_score"]
if current_dq < avg_dq * 0.95:
    print(f"DQ drift detected: {current_dq} < {avg_dq * 0.95}")
```

## Testing Strategy

Write tests for:
1. **Packet construction** — run_id propagates, no fields missing
2. **Schema loading** — schemas exist, have correct structure
3. **Stage behavior** — given input, produce expected output
4. **Integration** — dispatch.py + all stages together
5. **Artifacts** — files are created, have correct format

See `tests/README.md` for detailed testing guide.

## References

- `ADR-002` — Why run_id is first in Packet
- `ADR-003` — Why health + run records
- `ADR-004` — Why permissive schemas initially
- `ADR-005` — Why JSON shaper registry
- `core/CHANGES.md` — What changed in this session
- `health/README.md` — Health module details
- `observability/README.md` — Observability module details
- `tests/README.md` — Testing guide

