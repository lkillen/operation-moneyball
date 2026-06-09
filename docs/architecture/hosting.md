# Infrastructure & Hosting Architecture

## Guiding principle

One vendor, one job. No vendor does more than one thing. Keeps the system portable and easy to reason about.

---

## Stack

| Vendor | Role | Cost |
|---|---|---|
| GitHub | Source code, Silver parquet, run manifests, collaboration | Free |
| Cloudflare R2 | Gold JSON storage — served to Flask at startup | Free tier (10GB storage, no egress fees) |
| Cloudflare CDN | Static assets — CSS, JS, fonts | Free |
| Oracle Cloud Compute | Flask app host — nginx + gunicorn | Free tier (under consideration — see below) |

---

## Data layer by tier

```
Bronze  → egress/ and temp/ in moneyball-pipeline (GitHub Actions runner, ephemeral)
Silver  → moneyball-storage on GitHub (parquet, version-controlled, analyst-accessible)
Gold    → Cloudflare R2 (JSON, CDN-served, Flask-facing)
```

Silver stays on GitHub so the collaborator can access it without leaving their environment.
Gold moves to R2 because it is web-facing, not analytics-facing — the collaborator has no need for it.

---

## Request flow

```
User hits domain
        ↓ Cloudflare (proxy, SSL, DDoS protection)
        ↓
Oracle Compute — nginx → gunicorn → Flask
        ↓ on first request per entity
Flask fetches Gold JSON from Cloudflare R2 (@lru_cache — cached for life of process)
        ↓
Jinja2 renders HTML with injected data
        ↓
Browser fetches CSS/JS from Cloudflare CDN
```

---

## Deployment flow

```
Code change to moneyball-web on GitHub
        ↓ GitHub Actions
SSH into Oracle instance → git pull → restart gunicorn
```

Data changes do not trigger a Flask restart — Gold JSON is fetched fresh from R2 on the next
cold start or process restart. Flask does not redeploy when the pipeline runs new data.

---

## Oracle Cloud (under consideration)

Oracle Cloud Always Free tier offers ARM-based Ampere A1 instances — up to 4 OCPUs and 24GB RAM, always free.

**Advantages over Render free tier:**
- No spin-down on inactivity
- Significantly more compute resources
- Full server control (nginx, systemd, firewall)
- Real production DevOps experience

**Risk:**
- Oracle has a documented history of terminating free tier accounts suspected of misuse
- Mitigation: keep setup scripted so the server can be rebuilt quickly

**Status:** Under consideration. Render remains the fallback if Oracle proves unreliable.
If Oracle is used, Cloudflare proxies all traffic — Oracle's IP is never exposed publicly.

---

## Collaborator access

The collaborator's workflow does not change. They interact only with GitHub:
- Push to `egress/` in moneyball-analytics
- Read Silver parquet from moneyball-storage if needed for analytics

R2, Oracle, and Cloudflare are invisible to them.

---

## Secrets by repo

| Secret | Repo | Purpose |
|---|---|---|
| `PIPELINE_REPO_TOKEN` | moneyball-analytics | Trigger repository_dispatch to pipeline |
| `STORAGE_REPO_TOKEN` | moneyball-pipeline | Read/write moneyball-storage |
| `CF_R2_TOKEN` | moneyball-pipeline | Upload Gold JSON to Cloudflare R2 |
| `CF_R2_BUCKET_NAME` | moneyball-pipeline | R2 bucket target |
| `CF_R2_BASE_URL` | moneyball-web | Base URL for fetching Gold JSON from R2 |
| `DISCORD_WEBHOOK_URL` | moneyball-pipeline | Pipeline run notifications |
| `CACHE_REFRESH_TOKEN` | moneyball-pipeline + moneyball-web | Pipeline POSTs to Flask to clear Gold JSON cache after deploy |
| `FLASK_BASE_URL` | moneyball-pipeline | Flask app URL — used by pipeline to call /cache/refresh |
