# Repository traffic metrics

Public, cumulative traffic for this GitHub repository (official GitHub Traffic API).

| File | Purpose |
|------|---------|
| [`traffic.json`](traffic.json) | Machine-readable totals + per-day history (used by README badges) |
| [`traffic-history.md`](traffic-history.md) | Human-readable daily table |

## How it works

1. [`.github/workflows/repo-traffic-badges.yml`](../../.github/workflows/repo-traffic-badges.yml) runs daily (and on manual dispatch).
2. [`scripts/update_repo_traffic.py`](../../scripts/update_repo_traffic.py) fetches the last **14 days** from GitHub.
3. Daily rows merge into `view_days` / `clone_days` — **older days already stored are kept**, so totals accumulate.

## README badges

- **Repo Views** → `views_total` (repository page views)
- **Git Clones** → `clones_total` (`git clone` events)

## Optional baseline (pre-tracking history)

If you tracked views elsewhere before this workflow existed, set once in `traffic.json`:

```json
{
  "baseline_views": 120000,
  "baseline_clones": 8500
}
```

The next workflow run adds API-measured days on top (does not overwrite baselines).

## Run manually

GitHub → **Actions** → **Update repo traffic badges** → **Run workflow**

### Why badges stay at 0

The workflow may **commit** `traffic.json` but leave totals at **0** when the GitHub Traffic API returns **403**. The default `GITHUB_TOKEN` in Actions often **cannot** read `/repos/{owner}/{repo}/traffic/*`.

**Fix (one-time):**

1. Create a **classic PAT** with **`repo`** scope (or fine-grained: **Administration: Read** + **Metadata: Read** on this repo).
2. Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
3. Name: **`GH_TRAFFIC_TOKEN`**, value: your PAT.
4. Re-run **Update repo traffic badges** and confirm the job log shows non-zero `days=…` (not HTTP 403).

Or locally:

```bash
export GITHUB_REPOSITORY=brokermr810/QuantDinger
export GITHUB_TOKEN=ghp_...   # PAT with repo scope, not the Actions GITHUB_TOKEN
python scripts/update_repo_traffic.py
```
