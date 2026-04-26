# Postgres URI leak rotated; doctrine 0005 + gitleaks rule shipped

**Date:** 2026-04-26
**Type:** incident + decision
**Trigger:** T1.2 (incident: secret exposure) + T1.1 (doctrine added)
**Cites:** doctrine/0005-credentials-by-reference-not-value, plans/cutover-runbook.md

> GitGuardian flagged two literal Railway Postgres URIs in `cutover-runbook.md` (commit 2fbf3dd, PR #19). Both passwords rotated within minutes. Consumer services switched from hardcoded URLs to Railway template refs. Doctrine 0005 codifies the rule. A custom gitleaks `db-uri-with-password` rule fills a gap the default ruleset doesn't cover.

## Context

The runbook for the upcoming vanguard-DB cutover documented connection strings inline:

```
- Shared Postgres (source): `postgresql://postgres:<LITERAL_PASSWORD>@ballast.proxy.rlwy.net:51204/railway`
- Postgres-B4xF (target):   `postgresql://postgres:<LITERAL_PASSWORD>@yamabiko.proxy.rlwy.net:25354/railway`
```

PR #19 squash-merged at 17:15:39 UTC. GitGuardian's automated scan emailed an alert at 17:15:41 UTC — two-second turnaround on a public repo. The alert was indistinguishable from a phishing attempt at first glance, which prompted a verification step before action.

Three structural problems surfaced:
1. **Plans were written assuming a private repo.** Pasting the URL was the convenient form for someone holding the runbook open while running the steps. In a public repo it's a leak.
2. **Consumer services duplicated the credential.** `outrider.DATABASE_URL` and `vanguard.DATABASE_URL_VANGUARD` each held the full URL as a literal — rotation would have required updating N+1 places.
3. **Pre-commit gitleaks was installed but blind.** The default ruleset targets cloud-provider keys (AWS, GitHub, Slack) and does not include a generic `<scheme>://user:password@host` pattern. Pre-commit reported "Detect hardcoded secrets ... Passed" on the leak commit.

## What we did

**Containment (CLI, ~10 min):**
1. `ALTER USER postgres WITH PASSWORD '<new>'` on each Postgres via the proxy URL using the still-valid old password.
2. Verified new password authenticates: `psql` returns `now()`.
3. Updated four Railway service variables on each Postgres (`POSTGRES_PASSWORD`, `PGPASSWORD`, `DATABASE_URL`, `DATABASE_PUBLIC_URL`).
4. Switched consumer DATABASE_URLs to template refs:
   - `outrider.DATABASE_URL` → `${{Postgres.DATABASE_URL}}`
   - `vanguard.DATABASE_URL_VANGUARD` → `${{Postgres-B4xF.DATABASE_URL}}`
5. Confirmed Railway auto-redeployed both consumers (outrider logs: "Database tables initialized for cluster=outrider"; vanguard back in main loop within ~60s).

**Repo cleanup:**
- PR #20 — scrubbed the runbook, replaced literals with `railway variables` lookup preamble + `$SHARED_URL` / `$B4XF_URL` env vars in every subsequent psql/pg_dump invocation.
- Did **not** rewrite git history. The credentials are dead post-rotation; force-pushing public main would break every clone for marginal benefit. Standard "rotate, don't rewrite" guidance.

**Structural prevention (this PR):**
- `.gitleaks.toml` — extends gitleaks defaults with two custom rules:
  - `db-uri-with-password` matches `<scheme>://user:password@host` for postgres/postgresql/mysql/mongodb/mongodb+srv/redis schemes.
  - `pgpassword-env` matches `PGPASSWORD=<value>` inline on commands.
  - Allowlist covers Railway template refs, angle-bracketed placeholders (`<NEW_PW>`), and a stopword list for known-false generic-api-key matches in prose.
- Verified the rule fires on a file containing the original leaked URIs (2 findings) and that the working tree scans clean.
- `doctrine/0005-credentials-by-reference-not-value.md` — codifies the rule.

## What surprised me

- Gitleaks defaults don't cover DB URIs. The gap felt obvious in retrospect but was invisible until tested.
- Railway template refs (`${{Service.VAR}}`) are the right primitive but the CLI's `--kv` view resolves them, so verifying a ref took requires testing rotation propagation rather than inspection.
- GitGuardian emails landed two seconds after PR squash-merge. The detection-to-alert latency is effectively zero. This is the right model — the leak is public from the moment it's pushed.

## Consequences / action items

- [x] Both passwords rotated (Postgres, Postgres-B4xF).
- [x] Consumer services on template refs.
- [x] Runbook scrubbed (PR #20, merged).
- [x] `.gitleaks.toml` custom rule + doctrine 0005 (this PR).
- [ ] Roll the gitleaks rule into Touchstone's scaffolding so every garage repo inherits it.
- [ ] Audit each tool repo (touchstone, cortex, sentinel, conductor) for any literal credentials in `.cortex/` or elsewhere — scrub-and-rotate if found.
- [ ] Decide whether to add gitleaks to a CI workflow on this repo (currently only pre-commit; CI gate catches slips that bypass local hooks).
- [ ] Mark resolved in GitGuardian dashboard once this PR merges.
