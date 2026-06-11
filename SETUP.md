# Operation Moneyball — Development Setup

## GitHub Actions Secrets

The pipeline relies on GitHub Actions to trigger across repos. Both the data team (moneyball-analytics) and pipeline team (moneyball-pipeline) need to set up personal access tokens (PATs).

### 1. Create a Personal Access Token

**Each person does this once:**

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
2. Click "Generate new token"
3. Name: `moneyball-automation` (or similar)
4. Expiration: 90 days (or your preference)
5. Repository access: Select moneyball-* repos
6. Permissions:
   - Repository → Contents: Read and write
   - Repository → Actions: Read and write
7. Click "Generate token"
8. **Copy the token immediately** — you won't see it again

### 2. Add Your Token to Both Repos

Each person adds their PAT to **both** repos (so either person can trigger the pipeline):

**In moneyball-pipeline:**
- Settings → Secrets and variables → Actions → New secret
- Name: `STORAGE_REPO_TOKEN` → Paste your token → Save

**In moneyball-analytics:**
- Settings → Secrets and variables → Actions → New secret
- Name: `PIPELINE_REPO_TOKEN` → Paste your token → Save

**Repeat for Logan and Collaborator**

### 3. Test the Flow

Trigger manually:
1. Push CSV + manifest to `moneyball-analytics/egress/sport/league/season/`
2. This triggers `generate_manifest.yml`
3. Which triggers `pipeline.yml` in moneyball-pipeline via `repository_dispatch`
4. Pipeline processes data and writes to moneyball-storage

Or manually in moneyball-pipeline:
- Actions → Pipeline → Run workflow → Enter sport/league/season

## Secrets Reference

| Secret | Repo | Purpose |
|---|---|---|
| `STORAGE_REPO_TOKEN` | moneyball-pipeline | Clone/push to moneyball-storage (both have PAT) |
| `PIPELINE_REPO_TOKEN` | moneyball-analytics | Trigger moneyball-pipeline (both have PAT) |
| `DISCORD_WEBHOOK_URL` | moneyball-pipeline | (optional) Discord notifications |
| `CACHE_REFRESH_TOKEN` | moneyball-pipeline | (optional) Flask cache refresh |
| `FLASK_BASE_URL` | moneyball-pipeline | (optional) Web app URL |

## Troubleshooting

**Pipeline doesn't trigger:**
- Check moneyball-analytics Actions tab — did `generate_manifest.yml` run?
- Verify `PIPELINE_REPO_TOKEN` is set in moneyball-analytics
- Check the dispatch job output for API errors

**Storage writes fail:**
- Verify `STORAGE_REPO_TOKEN` is set in moneyball-pipeline
- Token needs `Contents: Read and write` on moneyball-storage

**Manifests not generating:**
- Check `.github/scripts/generate_manifest.py` runs successfully
- Verify CSV is in `egress/sport/league/season/`
