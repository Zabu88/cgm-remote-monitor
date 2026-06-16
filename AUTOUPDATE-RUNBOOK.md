# Nightscout on Render — Auto-Update Runbook

_Last updated: 2026-06-16. Lives in the fork on the `automation` branch and locally at `C:\Projects\Cursor\nightscout-render-runbook.md`._

## What this is

Your Nightscout (CGM monitor) runs on **Render**, fed by **xDrip → MongoDB**. It now
**auto-updates** itself from the official Nightscout project, **auto-deploys**, and
**emails you** when something updates or fails.

```
upstream  nightscout/cgm-remote-monitor
   │  (daily 05:00 UTC) GitHub Action fast-forwards master to upstream
   ▼
fork  Zabu88/cgm-remote-monitor  (branch: master = exact mirror of upstream)
   │  push webhook
   ▼
Render web service  ──▶  https://nightscout-vzae.onrender.com
```

## Key facts / IDs

| Thing | Value |
|---|---|
| Live site | https://nightscout-vzae.onrender.com |
| Version check | https://nightscout-vzae.onrender.com/api/v1/status.json (`version` field) |
| Render service ID | `srv-d5ucrcp4tr6s73crkvc0` |
| Render dashboard | https://dashboard.render.com/web/srv-d5ucrcp4tr6s73crkvc0 |
| Render runtime | Node — build `npm install`, start `npm start` |
| GitHub fork | https://github.com/Zabu88/cgm-remote-monitor |
| GitHub account | `Zabu88` (email ondrej.petr88@gmail.com) |
| Data flow | xDrip writes glucose data **directly to MongoDB**; Nightscout only reads/displays it |

## How it works

1. **Fork** `Zabu88/cgm-remote-monitor`.
   - `master` = byte-for-byte mirror of upstream (this is what Render builds).
   - `automation` = the **default branch**; holds the sync workflow (+ this runbook).
     It is intentionally "1 commit ahead" of upstream — that's just our files.
2. **Daily sync** — `.github/workflows/sync-upstream.yml` runs at 05:00 UTC (and on demand).
   It calls GitHub's `merge-upstream` API to fast-forward `master` to upstream's latest.
   No tokens/secrets needed (uses the built-in `GITHUB_TOKEN`).
3. **Render auto-deploys** — Render watches `master` with Auto-Deploy = On Commit. When the
   sync advances `master`, GitHub pings Render and it rebuilds + redeploys automatically.

## Notifications (how you find out something happened)

- **Update succeeded** → auto-closed GitHub issue "Nightscout updated to vX.Y.Z" → email.
  (History under the fork's **Closed issues**.)
- **Sync failed** → open GitHub issue "Nightscout upstream sync failed" → email.
- **Deploy failed** → Render emails you (Render → Notifications → "Only failure notifications", Email).

## Common tasks

**Check current vs latest version**
- Current: open `https://nightscout-vzae.onrender.com/api/v1/status.json`, read `version`.
- Latest: open `https://raw.githubusercontent.com/nightscout/cgm-remote-monitor/master/package.json`, read `version`.

**Force an update right now**
- GitHub: fork → **Actions** tab → "Sync master with upstream Nightscout" → **Run workflow**.
  If a new version exists, this advances `master` and Render auto-deploys.

**Force a Render redeploy (without a code change)**
- Render dashboard → the service → **Manual Deploy** → **Clear build cache & deploy**.

**Roll back a bad deploy**
- Render dashboard → the service → **Events**/**Deploys** → pick the last good deploy → **Rollback**.

## Troubleshooting

**Site is down / "Application failed to respond"**
- It's the Render free plan, so the instance **spins down when idle** and takes ~30–60s to wake
  on the first request. Reload after a minute. If still down, check Render → Logs.

**Site isn't getting new versions**
1. Check fork **Actions** — is the daily sync running and green? If it's disabled (GitHub
   auto-disables scheduled workflows after 60 days of repo inactivity), open Actions and re-enable it.
2. Check Render → the service → **Settings → Build & Deploy**: Source must be
   `Zabu88/cgm-remote-monitor`, Branch `master`, **Auto-Deploy = On Commit**.
3. Manually run the sync (above) and watch whether Render kicks off a deploy.

**Got a "sync failed" email / issue**
- Open the run log linked in the issue. Most likely cause: `master` diverged from upstream so it
  can't fast-forward. Fix by resetting `master` to match upstream (see "Reset master" below),
  then re-run the sync. Close the issue when resolved.

**Got a Render "deploy failed" email**
- Render → the service → **Logs** to see the build error. The previous good version keeps serving
  in the meantime, so you're not down. If it's an upstream bug, wait for the next upstream fix or
  roll back (above).

**Reset `master` so the mirror is clean again** (only if sync complains about divergence)
- In the fork, `master` must contain **only upstream commits** (no extra files). If something got
  committed to `master` by mistake, hard-reset it to upstream:
  - Easiest via GitHub UI is tricky; quickest is locally:
    `git clone https://github.com/Zabu88/cgm-remote-monitor && cd cgm-remote-monitor`
    `git remote add upstream https://github.com/nightscout/cgm-remote-monitor`
    `git fetch upstream && git checkout master && git reset --hard upstream/master && git push --force origin master`
  - Then re-run the sync workflow.

## Design notes / gotchas (don't "fix" these — they're intentional)

- **`automation` is the default branch on purpose.** GitHub only runs *scheduled* Actions from the
  default branch, so the workflow has to live there. `master` is kept clean for exact builds.
- **`master` must stay an exact mirror of upstream** — never commit to it, or `merge-upstream`
  fast-forwards will break.
- **Upstream's own workflows (CodeQL, "CI test and publish Docker image") are disabled** on the fork.
  They're meant for the main project and would fail + email noise on every sync. Keep them disabled.
- **Issues are enabled** on the fork (off by default for forks) so notifications can post.
- **Workflow YAML:** in `run: |` blocks, keep multi-line shell strings indented inside the block —
  a continuation line at column 0 breaks YAML and silently drops the `workflow_dispatch` trigger.

## If you ever need to rebuild this from scratch

1. Fork `nightscout/cgm-remote-monitor` to your account.
2. Create branch `automation`; add `.github/workflows/sync-upstream.yml` (the sync + notify workflow).
3. Set `automation` as the default branch; enable Actions on the fork; enable Issues on the fork.
4. Disable the upstream CodeQL and CI/Docker workflows.
5. In Render, point the service Source at your fork, Branch `master`, Runtime Node, Auto-Deploy On.
6. Render → Notifications → Email, "Only failure notifications".
