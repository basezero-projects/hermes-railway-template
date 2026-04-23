---
name: incident-alert
description: Poll critical SYVR services for health; post Discord alert the moment something breaks; post resolution message when it recovers. Dedupe so active incidents don't spam.
tags: [ops, alerting, monitoring, discord]
version: "1.0.0"
created: "2026-04-23"
---

# Incident Alert — Real-Time Health Monitoring

## When to use

Cron-invoked at multiple intervals:
- **Every 5 minutes** — critical-path health (user-facing: simsweep-auth, SYVRFinance prod, Cloudflare workers)
- **Every 15 minutes** — secondary checks (Vercel deploys, GitHub Actions CI)
- **Every hour** — slow checks (Railway service health, domain/SSL expirations — but only alert on 7-day windows)

Each cron invocation passes a `tier` prompt arg (`critical`, `secondary`, `slow`) and the skill runs the matching check set.

## Alert rules

For each check:
1. Run the check (HTTP GET, CLI command, or API call).
2. Evaluate pass/fail.
3. Look up the check in `/data/.hermes/state/incident-alerts.json`:
   - **Not currently alerting + check fails** → post alert, record alert in state.
   - **Currently alerting + check fails** → no-op (debounce). Update `last_seen_failing` timestamp.
   - **Currently alerting + check passes** → post resolution message, mark resolved in state (or remove entry).
   - **Not currently alerting + check passes** → no-op.

State file schema:
```json
{
  "simsweep-auth-health": {
    "status": "alerting",
    "first_failed": "2026-04-23T18:00:00Z",
    "last_seen_failing": "2026-04-23T18:20:00Z",
    "alert_message_id": "discord-msg-id",
    "failure_reason": "HTTP 503"
  }
}
```

## Checks — critical tier (every 5 min)

| Check ID | Command / URL | Pass criteria |
|---|---|---|
| `simsweep-auth-root` | `curl -m 10 https://simsweep-auth.syvr.dev/` | HTTP 200 AND response body JSON contains `"service":"simsweep-auth"` |
| `syvrfinance-prod-root` | `curl -m 10 https://scintillating-dinosaur-695.convex.cloud/` | HTTP 200 |
| `syvrfinance-dev-root` | `curl -m 10 https://fine-bat-215.convex.cloud/` | HTTP 200 |

**Deferred (need endpoint confirmation before wiring):** `simsweep-cdn` (need real hostname — `cdn.simsweep.com` does not resolve; ask Wes), Hetzner-Postgres-specific liveness (the Worker doesn't currently expose a DB-status endpoint — relying on `simsweep-auth-root` as a rough proxy means we'd miss DB-only outages; log as a TODO).

## Checks — secondary tier (every 15 min)

| Check ID | Source | Alert when |
|---|---|---|
| `vercel-deploys` | `vercel list --limit 5` per project (syvr-site, syvr-admin, RaysAutoService, proglass-site, syvr-gen-studio) | Latest deploy state = `ERROR` or `CANCELED` |
| `github-ci-simsweep-beta` | `gh run list -R basezero-projects/SimSweep-Beta --limit 1 --branch dev` | `conclusion = failure` |
| `github-ci-simsweep-auth` | `gh run list -R basezero-projects/simsweep-auth --limit 1 --branch main` | `conclusion = failure` |
| `github-ci-syvrfinance` | `gh run list -R basezero-projects/SYVRFinance --limit 1 --branch master` | `conclusion = failure` |
| `convex-crons-syvrfinance` | via admin endpoint listing recent cron runs | any cron's last-run status = fail |

## Checks — slow tier (hourly)

| Check ID | Source | Alert when |
|---|---|---|
| `railway-ghostface` | `railway status` for this service | Status != ACTIVE |
| `railway-simsweepbot` | `railway status -p proactive-illumination -s <id>` | Status != ACTIVE |
| `ssl-cert-syvr-dev` | `openssl s_client -connect syvr.dev:443 -servername syvr.dev < /dev/null` + parse | Days-until-expiry <= 7 |
| `ssl-cert-simsweep-com` | same pattern | Days-until-expiry <= 7 |
| `ssl-cert-simsweep-auth-syvr-dev` | same pattern | Days-until-expiry <= 7 |
| `domain-expiries` | GoDaddy CLI | Any domain expiring in <= 30 days |

## Alert message format

New incident:
```
🚨 **Incident — <check-id>**

**What:** <human description of the failing check>
**Started:** <time> (<X minutes ago>)
**Error:** `<short error from check output>`

**Likely cause:** <heuristic based on error pattern — e.g., "Cloudflare Worker redeployed recently; check deploy logs" if a Worker started 5xx'ing within 10 min of a new deploy>
**Suggested fix:** <actionable next step — e.g., "Check `wrangler deployments list --name simsweep-auth`. Roll back if the current version is bad.">

**Dashboards:**
- <link to the relevant dashboard>
```

Resolution:
```
✅ **Resolved — <check-id>**

Incident lasted <X min>. Service is healthy again.
```

## Heuristics for "likely cause" + "suggested fix"

- **HTTP 5xx on a Cloudflare Worker within 10 min of a deploy** → "Recent deploy may have bad code. Check `wrangler deployments list --name <worker>`. Roll back with `wrangler rollback`."
- **HTTP 503 on simsweep-auth + `db` field missing** → "Hetzner Postgres may be down. Check Hetzner console; verify Hyperdrive config reachable."
- **Failed CI on a repo immediately after a push** → "Most likely cause is the push itself. Check `gh run view <run-id> --log-failed`."
- **Vercel deploy FAILED** → "Build error. Check `vercel logs <deploy-url>`. First thing to try: `npm install && npm run build` locally."
- **Convex cron failure** → "Check `npx convex logs --tail` against the deployment. Look for recent errors."
- **Railway service down** → "Check `railway logs --lines 50`. Most common cause: crashed process, OOM, or deploy still rolling out."
- **SSL cert < 7 days** → "Certbot/Cloudflare should auto-renew. Verify renewal process hasn't been broken. Check DNS/TXT record health."
- **Domain < 30 days** → "Renew at registrar. GoDaddy → domain settings → enable auto-renew if not already."

## Delivery

Post to `$DISCORD_HOME_CHANNEL`. Use an @everyone mention only for critical-tier alerts on user-facing services (simsweep-auth, SYVRFinance prod). Secondary/slow tier: no mentions.

Suppress alerts between 23:00–06:00 MT except for critical-tier user-facing services (so overnight CI failures wait until morning, but a downed Worker pings immediately).

## State file locking

The state file at `/data/.hermes/state/incident-alerts.json` can be read/written by multiple cron tiers racing. Use file locking (fcntl or a sentinel `.lock` file) to avoid corrupt writes. If lock is held, retry up to 3 times with 100ms sleeps.

## Initial run

On first-ever invocation, create the state file as `{}`. Assume all checks start clean; any failure this run becomes a new alert.
