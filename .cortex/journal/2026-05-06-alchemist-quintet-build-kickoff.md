# Alchemist build kickoff — quartet becomes quintet

**Date:** 2026-05-06
**Type:** decision
**Trigger:** T1.1 (creates `.cortex/plans/alchemist.md`, will draft Doctrine 0008 on v0.1 ship)
**Cites:** plans/alchemist, doctrine/0001, doctrine/0003, doctrine/0004, doctrine/0007

> Alchemist — the issue-driven transmuter — is under construction. Repo + brew tap created, v0.0.x scan/doctor/banner shipped to Railway in dry-run, v0.1 transmute loop being built in parallel. The quartet is becoming a quintet.

## Context

The plan `.cortex/plans/alchemist.md` (drafted 2026-05-06) sat under "Draft (vision; not yet under construction)" for less than a day before the user said go. Pre-build pressure-test by the conductor council surfaced a handful of hardening items (subprocess timeouts, lockfile state machine, prompt-injection delimiters, dogfood gates). Those landed in the design before any code.

The Sentinel/Alchemist overlap question — initially flagged as "artificial cleavage" by the council — was resolved by the user pulling the hedge: Sentinel does not need to grow `--from-issues` after all. Sentinel owns the autonomous-backlog posture; Alchemist owns the issue-driven posture. They do not overlap.

User constraint added during build: a deliberate dogfood period must precede any live operation, "so it doesn't just turn on for the first time and start blasting issues." Three manual gates wired in (dry-run + test label → live + test label → live + real label), each operator-approved, no auto-graduation.

## What we decided

- **Build alchemist as the fifth tool.** Same release cadence shape as the existing four (own repo, own tap, own brew formula, hatch-vcs). Hosted on Railway as a cron service.
- **One deployment scopes to one GitHub org.** Operators who want alchemist on a different org deploy a second instance with their own config. Multi-org reusability is achieved by deployment, not by code.
- **Auth: PAT for v0.1, GitHub App for v0.2.** Code path is auth-agnostic (consumes `GITHUB_TOKEN` env var); the operational story documents the migration.
- **Provider: route via OpenRouter (kimi/deepseek) in the headless container.** Conductor's `claude` and `codex` providers use OAuth-via-CLI which doesn't survive in a Linux container; OpenRouter-keyed providers are the only path that works.
- **Quartet → quintet doctrine.** Doctrine 0008 to be drafted ratifying the five-tool framing when v0.1 ships. Doctrine 0007 hue table extended additively (ANSI 222 amber primary + 230 gold accent for alchemist).

## Consequences / action items

- [ ] Draft Doctrine 0008 (quintet framing) when alchemist v0.1 ships
- [ ] Amend Doctrine 0007 hue table with the alchemist amber/gold pair
- [ ] Update meta-repo `CLAUDE.md` and `.cortex/state.md` to add alchemist alongside the four existing tools (deferred until v0.1 stabilizes — state should match installed reality, not in-flight repos)
- [ ] Extend the `/deploy` skill to survey alchemist's release path
- [ ] Track "new operator setup wizard" via [autumngarage/alchemist#2](https://github.com/autumngarage/alchemist/issues/2) — out of scope for v0.1
- [ ] When alchemist's first labelled issue ships a merged PR, write a journal entry crediting the loop (T1.6 Cortex integration is itself a v0.2 follow-on)

## What's already shipped today

- `autumngarage/alchemist` repo created (public, MIT)
- `autumngarage/homebrew-alchemist` tap created with placeholder formula (the shared `homebrew-bump.yml` workflow will rewrite url+sha256 on first release)
- v0.0.x bootstrap PR merged: `alchemist scan` + `alchemist doctor` + `alchemist banner`, 18 tests, ruff clean, brand surface, CLAUDE.md, `.cortex/` skeleton
- Dockerfile bundling gh + git + conductor v0.10.1 + touchstone v2.7.0 + alchemist (503 MB, builds in ~5 s after first warm-up)
- `railway.json` configuring cron `*/5 * * * *` and Dockerfile builder
- Railway project `alchemist` provisioned (separate from `daring-strength` per memory note)
- One service `alchemist-cron`, environment variables seeded from local credentials (`gh auth token` + conductor's keychain `OPENROUTER_API_KEY`)
- First deploy in flight at the time of writing

## What's still in flight

- v0.1.0 transmute loop — delegated to conductor (claude provider, code-effort high) to build `runner.py`, `locks.py`, `briefs.py`, brief template, PR body template, plus the test suite
- Persistent volume on Railway (CLI panicked on pre-deploy attach; will retry post-deploy)
- Dogfood A: filing 2–3 trivial test issues across `touchstone`/`cortex`/`sentinel` READMEs labelled `alchemist-test`, watching ticks process them in dry-run
- GitHub App registration (deferred to v0.2 alongside the auth migration)
