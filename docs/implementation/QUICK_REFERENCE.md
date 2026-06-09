# Quick Reference — Common Tasks

A cheat sheet for frequent operations on the moneyball pipeline.

## Testing

### Run all tests
```bash
cd moneyball-pipeline
pytest
```

### Run specific test category
```bash
pytest tests/test_stages.py        # Unit tests
pytest tests/test_pipeline.py      # Integration tests
pytest tests/test_schemas.py       # Schema validation
```

### Run tests with verbose output
```bash
pytest -v tests/test_pipeline.py::TestPipelineIntegration
```

### Run tests and see print statements
```bash
pytest -s tests/test_stages.py
```

### Simulate entire pipeline locally
```bash
cd moneyball-pipeline
python3 dry_run.py
python3 dry_run.py --keep-dirs  # Keep temp dirs for inspection
```

## Debugging

### Check recent runs
```bash
ls -la moneyball-pipeline/storage/runs/basketball/ncaab/2025/
```

### Inspect run record
```bash
cat moneyball-pipeline/storage/runs/basketball/ncaab/2025/20250326T143000.json | jq .
```

### Check health history for a dataset
```bash
ls -la moneyball-pipeline/storage/health/basketball/ncaab/2025/team_elos/
cat moneyball-pipeline/storage/health/basketball/ncaab/2025/team_elos/20250326T143000.json | jq .
```

### List parquet files
```bash
ls -la moneyball-pipeline/storage/parquet/basketball/ncaab/*/2025/
```

### Check gold JSON
```bash
cat moneyball-pipeline/gold_staging/20250326T143000/team_elos.json | jq .
```

## Schema Management

### Add schema for new dataset
```bash
# Create silver (input) schema
cat > moneyball-pipeline/schema/basketball/ncaab/silver/new_dataset.toml << 'EOF'
[columns]
column_name = { type = "string", nullable = true }

[write_config]
write_mode = "batch"
primary_keys = []
schema_version = "2025-v1"
null_threshold = 1.0
EOF

# Create gold (output) schema
cat > moneyball-pipeline/schema/basketball/ncaab/gold/new_dataset.toml << 'EOF'
[structure]
required_keys = ["records"]
record_floor = 0
EOF
```

### View current schema
```bash
cat moneyball-pipeline/schema/basketball/ncaab/silver/team_elos.toml
```

### Update schema version
```bash
# Edit file and change schema_version
vim moneyball-pipeline/schema/basketball/ncaab/silver/team_elos.toml
# Change: schema_version = "2025-v1" → schema_version = "2025-v2"
```

## Test Data

### Add test CSV + manifest
```bash
# Create CSV (5 rows example)
cat > moneyball-pipeline/temp/basketball/ncaab/2025/new_dataset.csv << 'EOF'
col1,col2,col3
val1,val2,val3
val4,val5,val6
val7,val8,val9
val10,val11,val12
val13,val14,val15
EOF

# Create manifest with correct checksum
CHECKSUM=$(shasum -a 256 moneyball-pipeline/temp/basketball/ncaab/2025/new_dataset.csv | awk '{print $1}')
cat > moneyball-pipeline/temp/basketball/ncaab/2025/new_dataset.manifest.json << EOF
{
  "sport": "basketball",
  "league": "ncaab",
  "season": "2025",
  "filename": "new_dataset.csv",
  "checksum": "$CHECKSUM",
  "row_count": 5
}
EOF
```

### Regenerate manifest for existing file
```bash
cd moneyball-pipeline/temp/basketball/ncaab/2025
CHECKSUM=$(shasum -a 256 team_elos.csv | awk '{print $1}')
ROW_COUNT=$(tail -n +2 team_elos.csv | wc -l)
jq --arg checksum "$CHECKSUM" --arg row_count "$ROW_COUNT" \
  '.checksum = $checksum | .row_count = ($row_count | tonumber)' \
  team_elos.manifest.json > team_elos.manifest.json.tmp
mv team_elos.manifest.json.tmp team_elos.manifest.json
cat team_elos.manifest.json
```

## Packet Operations

### Create a packet manually (testing)
```python
from core.packets import DatasetPacket
from pathlib import Path
import pandas as pd

df = pd.DataFrame({"col1": [1, 2, 3]})
packet = DatasetPacket(
    run_id="20250326T143000",
    sport="basketball",
    league="ncaab",
    season="2025",
    dataset_name="test_data",
    dataframe=df,
    source_path=Path("/tmp/test.csv"),
    row_count=len(df),
    checksum="abc123def456",
)

print(packet.run_id)  # 20250326T143000
print(packet.dataset_name)  # test_data
```

## Health Record Queries

### Get all health records for a dataset
```python
from health import store

records = store.get_records(
    sport="basketball",
    league="ncaab",
    season="2025",
    dataset_name="team_elos",
)

for record in records:
    print(f"{record['run_id']}: DQ={record['dq_score']}, Rows={record['row_count']}")
```

### Compute average DQ score
```python
from health import store

records = store.get_records("basketball", "ncaab", "2025", "team_elos")
avg_dq = sum(r["dq_score"] for r in records) / len(records) if records else 0.0
print(f"Average DQ: {avg_dq:.2f}")
```

### Check for DQ drift
```python
from health import store

records = store.get_records("basketball", "ncaab", "2025", "team_elos")
if len(records) >= 2:
    recent_avg = sum(r["dq_score"] for r in records[-30:]) / min(30, len(records))
    current = records[-1]["dq_score"]
    drift = ((current - recent_avg) / recent_avg * 100) if recent_avg > 0 else 0
    print(f"DQ drift: {drift:.1f}% (current={current:.2f}, avg={recent_avg:.2f})")
```

## Observability

### Trigger anomaly detection
```python
from observability import monitor

signals = monitor.check_all()
for signal in signals:
    print(f"{signal.severity}: {signal.message}")
```

### Send Discord alert
```python
from observability import reporter

run_summary = {
    "run_id": "20250326T143000",
    "overall_success": True,
    "passed": 5,
    "failed": 0,
    "file_results": [...],
}

webhook_url = "https://discord.com/api/webhooks/..."
reporter.report_run(run_summary, webhook_url)
```

## GitHub Actions

### Trigger pipeline manually
Go to: https://github.com/logankillen/moneyball-pipeline/actions/workflows/pipeline.yml
Click "Run workflow", select "main" branch, click "Run workflow"

### View workflow logs
https://github.com/logankillen/moneyball-pipeline/actions

Click the run, then click a job to see full output.

### View run records in storage
```bash
git clone https://github.com/logankillen/moneyball-storage
cat moneyball-storage/runs/basketball/ncaab/2025/20250326T143000.json | jq .
```

## Documentation

### Key architecture decisions
- `operation-moneyball/adr/ADR-002` — run_id propagation
- `operation-moneyball/adr/ADR-003` — two-layer observability
- `operation-moneyball/adr/ADR-004` — permissive schemas
- `operation-moneyball/adr/ADR-005` — JSON shaper registry

### Implementation guides
- `operation-moneyball/docs/implementation/PIPELINE_WALKTHROUGH.md` — end-to-end flow
- `moneyball-pipeline/tests/README.md` — testing guide
- `moneyball-pipeline/health/README.md` — health module details
- `moneyball-pipeline/observability/README.md` — observability module details

## Virtual Environment

### Create (first time)
```bash
cd moneyball-pipeline
python3 -m venv venv
source venv/bin/activate
pip install pandas pyarrow pytest
```

### Activate
```bash
cd moneyball-pipeline
source venv/bin/activate
```

### Deactivate
```bash
deactivate
```

### Check what's installed
```bash
pip list
pip show pandas
```

## File Locations

| File | Purpose |
|------|---------|
| `dispatch.py` | Pipeline entry point |
| `core/packets.py` | Packet dataclasses |
| `core/paths.py` | Storage path helpers |
| `ingress/loader.py` | CSV loading |
| `validate/runner.py` | DQ checks |
| `transform/json_shaper.py` | Dataset JSON shapes |
| `publish/silver_writer.py` | Write parquet |
| `publish/gold_writer.py` | Write JSON |
| `health/recorder.py` | Write health records |
| `health/store.py` | Query health records |
| `observability/monitor.py` | Anomaly detection |
| `observability/reporter.py` | Discord alerts |
| `schema/*/silver/*.toml` | Input schema contracts |
| `schema/*/gold/*.toml` | Output schema contracts |
| `temp/*/` | Test CSVs + manifests |
| `tests/conftest.py` | Test fixtures |
| `.github/workflows/pipeline.yml` | GitHub Actions workflow |

## Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `STORAGE_PATH` | Local storage root | `/Users/logan/moneyball-tst/moneyball-pipeline/storage` |
| `DISCORD_WEBHOOK_URL` | Discord alert endpoint | `https://discord.com/api/webhooks/123/abc` |
| `STORAGE_REPO_TOKEN` | Git token for storage commits | (set in GitHub secrets) |

## Common Errors

### "No such file or directory: temp/basketball/ncaab/2025"
→ Test CSVs don't exist. Create them manually or check STORAGE_PATH.

### "NameError: name 'register' is not defined"
→ @register decorators used before register() function defined. Move register() before decorators in json_shaper.py.

### "dq_score not perfect: {}"
→ Test schemas are too strict. Check schema TOML files, ensure null_threshold=1.0 and all columns nullable=true.

### "Could not parse summary: {}"
→ dispatch.py didn't output JSON. Check dispatch.py has `print(json.dumps(run_summary))` at the end.

### "TypeError: DatasetPacket missing required positional argument: 'run_id'"
→ Packet constructed without run_id. Check all DatasetPacket() calls have run_id as first parameter.

