---
# 0005 — Credentials live by reference, not by value

> Live credentials — database passwords, API tokens, deploy keys — never appear as literal strings in `.cortex/` (or anywhere else in the repo). They live in the secrets store of the service that owns them (Railway service variables, macOS Keychain, GitHub Actions secrets). Plans, runbooks, and journal entries reference them by service+variable name and resolve them at run-time. Consumer services reference upstream secrets via Railway template refs (`${{Service.VAR}}`) so a single rotation propagates atomically.

**Status:** Accepted
**Date:** 2026-04-26
**Promoted-from:** journal/2026-04-26-postgres-uri-leak-and-rotation
**Load-priority:** always

## Context

On 2026-04-26 GitGuardian flagged commit 2fbf3dd (PR #19) for exposing two Railway Postgres URIs with plaintext credentials inside `.cortex/plans/cutover-runbook.md`. The leak was real: both passwords appeared verbatim in a public-repo runbook documenting a database cutover. The credentials were rotated within minutes, but the incident exposed three structural weaknesses:

1. **Plans documented operations using literal credentials.** The runbook treated the connection strings as data the reader could copy-paste — convenient for the author, catastrophic in a public repo. Future runbooks would do the same unless the rule is explicit.
2. **Consumer services duplicated the credentials.** Both `outrider.DATABASE_URL` and `vanguard.DATABASE_URL_VANGUARD` stored the full connection string verbatim rather than referencing the Postgres service's variable. Rotation required updating the password in N+1 places (the DB itself + every consumer that hardcoded the URL), inviting drift and missed updates.
3. **The pre-commit gitleaks hook didn't catch it.** Gitleaks' default ruleset targets cloud-provider keys (AWS, GitHub, Slack tokens) and does not include a generic database-URI-with-password pattern. The hook was installed but blind to this class of secret.

The rotation containment was straightforward (`ALTER USER postgres WITH PASSWORD '<new>'`, update Railway service vars), but the structural fix — preventing recurrence — required a doctrine.

## Decision

**1. Live credentials never appear as literal strings in this repo.** Plans, runbooks, journals, and any other tracked file refer to credentials by their address in the owning secrets store. Concretely:

- **Railway-managed secrets** are referenced by service+variable name. Example, in a runbook:
  ```bash
  SHARED_URL=$(railway variables --service Postgres --kv | grep '^DATABASE_PUBLIC_URL=' | cut -d= -f2-)
  psql "$SHARED_URL" -c "..."
  ```
  Never: `psql 'postgresql://postgres:<LITERAL_PASSWORD>@host:5432/railway'`.
- **Keychain / 1Password / direnv-managed local secrets** are referenced by service+account+key triple, never resolved into the file.
- **GitHub Actions secrets** appear as `${{ secrets.NAME }}` references in workflow YAML, never resolved into commits.

**2. Consumer services reference upstream secrets via the owning service's variable.** On Railway, this means template-ref syntax:

```
outrider.DATABASE_URL          = ${{Postgres.DATABASE_URL}}
vanguard.DATABASE_URL_VANGUARD = ${{Postgres-B4xF.DATABASE_URL}}
```

Rotating the Postgres service variable propagates atomically; no consumer service variable holds a value that could go stale.

**3. The repo's `.gitleaks.toml` includes a custom `db-uri-with-password` rule** in addition to the gitleaks defaults. The rule fires on `<scheme>://<user>:<password>@<host>` for `postgres`, `postgresql`, `mysql`, `mongodb`, `mongodb+srv`, and `redis` schemes. Documentation placeholders (Railway template-ref syntax, angle-bracketed names like `<NEW_PW>`, generic words like `:password@`) are allowlisted so docs explaining the right pattern don't false-positive. This config is active in pre-commit and should also gate any future CI workflow.

**4. Detection is treated as the second line of defense, not the first.** The first line is the rule above (don't paste). Gitleaks exists to catch slips, not to make pasting safe. A rotation triggered by gitleaks finding a credential in a staged commit is the success case; a rotation triggered by GitGuardian finding it in a pushed commit is the failure case. Each gitleaks-caught slip is still a journal entry — the pattern matters.

## Consequences

- Runbooks become slightly more verbose: the "fetch from Railway" preamble adds a few lines compared to pasting the URI inline. Acceptable cost.
- Future Railway services should be onboarded with template-ref consumers from the start. Pasting the resolved URI into a consumer's variables is a smell.
- This doctrine applies to every repo in the garage. Each tool's repo should adopt the same `.gitleaks.toml` rule (or extend its own equivalent). Touchstone is the natural place to scaffold this — a follow-up workstream.
- The historical leak in git history (commit 2fbf3dd) is not rewritten; the credentials it exposes are dead. Standard guidance for rotated public secrets is "rotate, don't rewrite" — force-pushing public main is disruptive for marginal benefit once the keys no longer authenticate.

## Relationship to other doctrine

- **Doctrine 0001** (autumn-garage's reason to exist) frames the four-tool composition. This doctrine is one of the cross-cutting rules that applies to every tool's repo, not just this coordination repo.
- **Doctrine 0003** (LLM providers compose by contract, not code import) and **Doctrine 0004** (Conductor as fourth peer) both rely on the assumption that secrets live in the host environment, not in the code. This doctrine makes that assumption explicit.

## Open follow-ups

- Roll the `.gitleaks.toml` rule (or its equivalent) into Touchstone's scaffolding so every new repo inherits it.
- Audit each tool repo (touchstone, cortex, sentinel, conductor) for any plan/journal entries that contain literal credentials — same scrub-and-rotate procedure if found.
- Decide whether to add gitleaks to a CI workflow on this repo (currently only pre-commit; CI gate would catch slips that bypassed local hooks).
