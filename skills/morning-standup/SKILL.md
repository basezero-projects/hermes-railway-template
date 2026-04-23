---
name: morning-standup
description: Daily 9 AM MT digest — what changed across every SYVR project in the last 24 hours, what needs attention, what's expiring soon. Post one concise message to the Ghostface home Discord channel.
tags: [ops, standup, daily, discord]
version: "1.0.0"
created: "2026-04-23"
---

# Morning Standup — Daily Digest

## When to use

Cron-invoked at **15:00 UTC** (9 AM MDT). One digest per run, posted to the Ghostface home Discord channel (env `DISCORD_HOME_CHANNEL`).

## Goal

Tell Wes what needs his attention today in ~60 seconds of reading. Skip sections that have nothing to report — empty sections are noise.

## Scope (what to check)

### Git activity — last 24h

For each repo in `basezero-projects`, fetch commits on the default branch + any active `dev`/`stable` branches since yesterday 15:00 UTC. If the repo has no commits in that window, skip it.

Commands:
```bash
# list active repos
gh repo list basezero-projects --limit 50 --json name,updatedAt,pushedAt \
  | jq -r '.[] | select((.pushedAt // .updatedAt) > (now - 86400 | todateiso8601)) | .name'

# per repo, commits since 24h ago on each interesting branch
for branch in main master dev stable beta; do
  gh api "repos/basezero-projects/$REPO/commits?sha=$branch&since=$(date -u -d '24 hours ago' -Iseconds)" \
    --jq '.[] | "\(.sha[0:7]) \(.commit.author.date[0:10]) \(.commit.message | split("\n")[0])"' 2>/dev/null
done
```

Format per repo: `**<repo>** (N commits on <branch>)`, then up to 3 most-recent commit subject lines, truncate 80 chars.

### CI / deploy status

- **GitHub Actions:** for each repo with commits in 24h, check latest run status: `gh run list -R basezero-projects/<repo> --limit 3 --json status,conclusion,name,headBranch`. Flag any `conclusion=failure` or `status=in_progress`>20min as a problem.
- **Vercel:** `vercel list --scope basezero` (or equivalent). Report any failed deploys in 24h. Skip successes.
- **Cloudflare Workers:** `wrangler deployments list --name simsweep-auth --limit 5` and same for `simsweep-cdn`. Report current deployed version + date.
- **Railway:** `railway status` for `outstanding-liberation` (Ghostface, this service) and `proactive-illumination` (SimSweepBot). Report if not `ACTIVE`/`SUCCESS`.

Format: one line per failed/stale thing; skip if everything is green.

### Health checks — core services

- `curl -s -o /dev/null -w "%{http_code}" https://simsweep-auth.syvr.dev/health` — expect 200
- `curl -s -o /dev/null -w "%{http_code}" https://fine-bat-215.convex.cloud/health` — SYVRFinance dev
- `curl -s -o /dev/null -w "%{http_code}" https://scintillating-dinosaur-695.convex.cloud/health` — SYVRFinance prod
- `curl -s -o /dev/null -w "%{http_code}" https://ghostface.syvr.dev/` — local Ollama tunnel (not in use yet but monitor)

Format: only report non-200 or timeouts. Skip if everything's 200.

### SYVRFinance — pending items

`curl -s "https://scintillating-dinosaur-695.convex.cloud/api/summary?year=$(date +%Y)" -H "X-API-Key: $SYVR_FINANCE_API_KEY"` — then per the response, note:
- Pending-review transaction count (if `pendingCount > 0`)
- Large unreviewed expenses (amount > $100 and status=pending)
- Quarterly tax deadline if within 14 days
- Net profit delta vs. same day last week (if available)

Format:
```
**SYVRFinance**
- 3 pending transactions for review ($47 total)
- Q2 federal tax due in 9 days: $1,240
```
Skip if nothing actionable.

### Patreon

Reuse the `patreon-sync-daily-silent` output if available (it already runs at 15:00 UTC, same time). New members since yesterday + canceled members. Skip if 0/0.

### Expirations — next 30 days

- **Domains:** `godaddy_cli.py list --expiring 30` (or equivalent). Flag any domain expiring <30 days.
- **SSL certs:** check `https://syvr.dev`, `https://simsweep.com`, `https://simsweep-auth.syvr.dev` via `curl --cert-status` or `openssl s_client`. Flag <30 days.
- **OAuth tokens:** Gmail refresh token health — last successful `gmail_scan_failed` ntfy? If recent failure, flag.

Format:
```
**Expiring soon**
- simsweep.com — 12 days
- SSL cert on syvr.dev — 8 days
```
Skip if nothing expiring <30 days.

### Anything on fire right now

Read `/data/.hermes/state/incident-alerts.json` — if any alert is currently active (unresolved), surface a one-line summary at the TOP of the digest in **bold**.

## Output format

```
🌅 **Morning standup — <date>**

🔥 **Active incident:** <if any, from incident-alerts.json>

**Shipped yesterday**
- **simsweep-auth** (2 commits on main)
  - `abc1234` feat: add rate limiting to /cc/browse
  - `def5678` chore: bump version to 0.11.1
- **SYVRFinance** (4 commits on master)
  - ...

**Needs attention**
- 3 pending transactions in SYVRFinance for review
- CI failing on syvr-site — deploy blocked
- Q2 federal tax due in 9 days

**Expiring soon**
- SSL cert on syvr.dev — 8 days

**Stable/green:** <count of repos with clean CI and no commits>
```

Omit any section that's empty. If literally everything is quiet, post: `🌅 Morning standup — <date>. Nothing to report. Everything green.`

## Delivery

Post to Discord channel ID from `$DISCORD_HOME_CHANNEL` env var. If the digest exceeds 2000 chars (Discord message limit), break by section and post as thread replies.

## Error handling

- If any check times out after 15s, note it as `⚠ <service> — check timed out` and continue. Don't block the whole digest on one slow service.
- If credentials for a check are missing, silently skip that check (don't surface as an error).
- If the whole skill fails, post a short error message to Discord so Wes knows the digest didn't run.

## State

No state persistence needed for standup itself. Runs stateless — each day's digest is fresh from current observations. Reads the incident state but doesn't write it.
