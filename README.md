# ninye-runner

A **lightweight GitHub Actions runner** that drives the Ninye YouTube Shorts `'P'` Words harvester on a 5-minute cron and pushes the freshly harvested `data/` back to the real project repository.

> **This repo is intentionally minimal.** It contains *only* this README and the workflow file at `.github/workflows/harvest.yml`.
>
> It does **not** contain:
> - the dashboard,
> - the Vercel app,
> - the comment database,
> - any Cloudflare code,
> - any duplicated project data,
> - any API keys or tokens.
>
> The actual project — harvester code, dashboard, serverless API, and `data/` — lives at [`Vicdara/YouTube-Shorts-P-Words-Intelligence`](https://github.com/Vicdara/YouTube-Shorts-P-Words-Intelligence). This runner only checks that repo out, runs the harvester, and pushes `data/` back.

---

## Why this exists

GitHub Actions on the `Vicdara` account is currently blocked by billing limits, so the in-repo workflow at `.github/workflows/update_comments.yml` can no longer fire. To keep the harvester running 24/7, this repo is designed to be **forked to a second GitHub account** where Actions billing still works. The fork runs the workflow on cron, checks out the real project repo, harvests new comments, and pushes the updated `data/` back.

Nothing about the main project repo changes — the runner just calls into it from outside.

---

## Architecture

```
        SecondAccount/ninye-runner   <-- this repo (forked to a second account)
                  |
                  |  GitHub Actions cron every 5 minutes
                  |  (workflow_dispatch also supported for manual "Run Now")
                  v
   Vicdara/YouTube-Shorts-P-Words-Intelligence
                  |
                  |  git push data/   (auth: MAIN_REPO_TOKEN)
                  |  commit by ninye-runner[bot]
                  v
        Vercel / ninye-tracker dashboard   (auto-redeploys on push)
```

| Component | Role |
|---|---|
| `SecondAccount/ninye-runner` | Schedule + checkout + harvest + push. Stores **no data**, has **no code** of its own. |
| `Vicdara/YouTube-Shorts-P-Words-Intelligence` | Source of truth. Holds harvester code (`scripts/unified_harvester.py`, `scripts/update_reports.py`), runtime config (`data/harvest_settings.json`), the live comment database (`data/all_raw_comments_stream.jsonl`), summary data (`data/summary_data.json`), and the dashboard + serverless API. |
| Vercel / ninye-tracker | Reads `data/` from the project repo and serves the public dashboard. Auto-redeploys whenever `data/` changes. |

### Why fork instead of just moving the workflow?

GitHub Actions billing is per-account, not per-repo. Moving the workflow to a new repo under the *same* account would not help. Forking to a *different* account (with its own Actions quota) sidesteps the billing block on `Vicdara` entirely, while still writing results back to the canonical project repo via a scoped token.

---

## What this repo contains

```
ninye-runner/
├── .github/
│   └── workflows/
│       └── harvest.yml    # cron + checkout + harvest + push pipeline
└── README.md              # this file
```

That is the entire repo. No source code, no data, no secrets.

---

## How the workflow works

1. **Trigger** — runs on a GitHub cron every 5 minutes, or manually via `workflow_dispatch`.
2. **Concurrency** — a single concurrency group `ninye-harvest` with `cancel-in-progress: false` ensures two harvests never run at the same time.
3. **Checkout** — clones `Vicdara/YouTube-Shorts-P-Words-Intelligence@main` using `secrets.MAIN_REPO_TOKEN`.
4. **Python 3.11** — set up via `actions/setup-python@v5`, with pip caching keyed on `scripts/requirements.txt` (read from the project repo).
5. **Install** — `pip install -r scripts/requirements.txt` (also from the project repo).
6. **Read settings** from `data/harvest_settings.json` (with safe defaults if the file is missing):
   - `enabled` — default `true`
   - `interval_minutes` — default `30`
   - `workers_count` — default `3`
   - `harvest_mode` — default `delta`
7. **Read last completed sync time** from `data/summary_data.json` → `metadata.last_updated`.
8. **Decide whether to run**:
   - `workflow_dispatch` → always run immediately, regardless of timer.
   - `schedule` → only run when `enabled=true` AND elapsed minutes since `last_updated` >= `interval_minutes`. Otherwise skip cleanly.
9. **Run the harvester**:
   ```bash
   python scripts/unified_harvester.py <workers> <ytdlp> <scrapers>
   ```
   The mode → engine mapping (mirrors the main project's existing `update_comments.yml`):

   | `harvest_mode` | `<workers>` (API) | `<ytdlp>` | `<scrapers>` |
   |---|---|---|---|
   | `delta` (default) | `workers_count` | `0` | `0` |
   | `reverse`        | `workers_count` | `0` | `2` |
   | `full`           | `workers_count` | `2` | `2` |
10. **Update reports** — `python scripts/update_reports.py` regenerates `data/summary_data.json` and `data/youtube_p_comments_ranked.md`.
11. **Commit and push** — stages only `data/`, commits with a `chore(data): ninye-runner harvest sync ...` message, and pushes back to `main` on `Vicdara/YouTube-Shorts-P-Words-Intelligence` using `MAIN_REPO_TOKEN`. If the push conflicts with another commit (rare, but possible if the dashboard also wrote), the workflow fetches, rebases, and retries up to 5 times. If a rebase has a conflict, the run aborts cleanly — the next run will catch up.

---

## Required secrets

> **GitHub repository secrets are NOT copied when a repo is forked.**
> You must add both secrets manually on the forked copy under the second account, under **Settings → Secrets and variables → Actions → New repository secret**.

### `YOUTUBE_API_KEYS`

The complete comma-separated YouTube Data API v3 key pool, e.g.:

```
AIzaSyAAA...,AIzaSyBBB...,AIzaSyCCC...
```

Passed to the harvester as the `YOUTUBE_API_KEYS` environment variable. The harvester reads this env var first, then falls back to the `api_keys.txt` file in the project repo.

### `MAIN_REPO_TOKEN`

A **fine-grained** GitHub Personal Access Token created on the `Vicdara` account, scoped to **only** `Vicdara/YouTube-Shorts-P-Words-Intelligence` with at least:

| Permission | Level |
|---|---|
| Contents | Read and write |
| Metadata | Read (auto-required) |

No other permissions, no other repositories.

Create at: https://github.com/settings/personal-access-tokens/new

This token is used for both the initial `actions/checkout` and the final `git push` back to `main`. It is never committed to the repo.

---

## Setup on the second account

1. **Fork the runner repo.**
   - Go to https://github.com/Vicdara/ninye-runner
   - Click **Fork** → choose the second account as the fork owner.

2. **Enable GitHub Actions on the fork.**
   - On the fork: **Settings → Actions → General**
   - Select **Allow all actions and reusable workflows**.
   - Under **Workflow permissions**, choose **Read and write permissions** (not strictly required since this workflow does not push to the fork itself, but harmless).
   - Click **Save**.

3. **Add `YOUTUBE_API_KEYS`.**
   - On the fork: **Settings → Secrets and variables → Actions → New repository secret**
   - Name: `YOUTUBE_API_KEYS`
   - Value: comma-separated list of YouTube API keys.

4. **Add `MAIN_REPO_TOKEN`.**
   - Create a fine-grained PAT on the `Vicdara` account scoped to `Vicdara/YouTube-Shorts-P-Words-Intelligence` with **Contents: Read and write**.
   - On the fork: **Settings → Secrets and variables → Actions → New repository secret**
   - Name: `MAIN_REPO_TOKEN`
   - Value: paste the token.

5. **Run the workflow manually once to verify.**
   - On the fork: **Actions** tab → select **Ninye Harvester** workflow → **Run workflow** → choose `main` branch → click **Run workflow**.
   - Open the run and confirm:
     - The "Check out the real project repo" step succeeds.
     - The "Read settings" step prints `Harvest DUE` or `Manual trigger`.
     - The "Run Ninye Harvester" step produces a `HARVEST RUN COMPLETED` summary.
     - The "Commit and push" step prints `Push succeeded on attempt 1`.
   - Verify the project repo's commit history shows a `chore(data): ninye-runner harvest sync ...` commit by `ninye-runner[bot]`.

After the manual run succeeds, the 5-minute cron takes over automatically.

---

## Notes and constraints

- This runner repo contains **no dashboard, no Vercel app, no Cloudflare code, no comment database, no duplicated project data**.
- It does **not modify** the main project repo's code or move/delete any API keys.
- It does **not touch Vercel**. The Vercel deployment continues to auto-redeploy whenever `data/` changes on the main project repo, exactly as before.
- Only `data/` is ever staged for commit. Any drift in code, configs, or other files from the harvest run is discarded.
- `harvest_settings.json` is read live on every run, so interval / workers / mode can be tuned from the dashboard without touching this runner.

---

## License

MIT — same as the parent project.
