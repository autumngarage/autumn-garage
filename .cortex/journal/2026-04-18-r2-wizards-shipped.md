# R2 shipped — interactive-by-default wizards across the trio

**Date:** 2026-04-18
**Type:** decision
**Trigger:** T2.1
**Cites:** doctrine/0002-interactive-by-default, journal/2026-04-18-r1-regression-pass, https://github.com/autumngarage/touchstone/pull/47, https://github.com/autumngarage/sentinel/pull/72, https://github.com/autumngarage/cortex/pull/17

> Three R2 PRs landed in parallel via background agents, implementing Doctrine 0002 across all three tools. Each prompts on TTY, respects flags, supports `--yes`, falls back gracefully non-TTY, and prints an "Equivalent to rerun" block teaching the flag form. Three latent issues surfaced and were fixed in-PR.

## What shipped

- **Touchstone** — `touchstone new` gets 7 prompts (language scaffold, reviewer, cortex init, sentinel init, registry, initial commit, GitHub repo) and 8 new flags. Trap-based cleanup on Ctrl-C. 391 additions.
- **Sentinel** — explicit `sentinel init` is now the canonical first-run; `sentinel work` still auto-inits but prints a visible warning. Reviewer default flipped from gemini to codex (orthogonal provider family). 11 new tests.
- **Cortex** — `cortex init` prompts for CLAUDE.md / AGENTS.md imports and `.gitignore` entries. Idempotent appends, placement after last existing `@<path>` line. 19 new tests.

## Three latent issues surfaced by R2

### 1. `scripts/open-pr.sh` hardcodes `--base $DEFAULT_BRANCH`

The script can't open stacked PRs. Cortex's R2 agent had to bypass with `gh pr create --base feat/plans-template-and-readme` manually. This is now a Touchstone TODO — `open-pr.sh` should accept `--base <branch>` or auto-detect from the parent tracking branch.

The irony: we want tooling that encourages stacked-PR workflows (it's the clean pattern for dependent changes), and our own tooling didn't support it until we tried.

### 2. Touchstone's review-config block had bare `read` calls

Lurking in `new-project.sh`: a block that called `read -r -p "..."` without TTY-gating. Non-TTY invocations would hang waiting for stdin. Nobody had hit this before because `touchstone new` in practice is run interactively — but the wizard now deliberately exercises both paths, and the non-TTY test caught it. Added a `YES_MODE` branch that applies hosted/auto defaults instead of hanging.

Generalizable lesson: **any `read` call in a CLI must have a non-TTY fallback.** Candidate for a lint rule in Touchstone's own shellcheck setup: grep for `read -r -p` without nearby `[ -t 0 ]` or `YES_MODE` guard.

### 3. `--yes` non-determinism under conditional host state

Touchstone R2's parity test (`--yes should equal flag-driven defaults`) was non-deterministic because on a host where `cortex` and `sentinel` are installed, the wizard defaults to `--with-cortex --with-sentinel`, but a flag-driven invocation without those flags produces `--no-with-cortex --no-with-sentinel`. The test now forces explicit `--no-with-*` on both sides.

Generalizable lesson: **defaults that depend on host state make "accept defaults" non-reproducible across machines.** The printed flag-form block helps (the user sees what was chosen), but test harnesses need to pin host-state-dependent defaults explicitly.

## What worked about parallel R2 dispatch

Same pattern as R1 worked again:
- Detailed briefs reading CLAUDE.md + Doctrine 0002 + original journal context.
- Explicit scope as numbered items.
- "Stacked on PR #N" explicit so agents branched from the right place.
- Report-back format capped at ~200 words.
- `bash scripts/open-pr.sh` (NOT `--auto-merge`) — which is how we discovered issue #1 above.

Total wall time for R2: ~16 min (Cortex) + ~9 min (Sentinel) + ~16 min (Touchstone) running in parallel. Would have been ~1.5 hours sequential.

## Consequences / action items

- [x] All R2 items in TODOs.md crossed off with PR links.
- [x] New TODO added: `open-pr.sh --base` support.
- [ ] User reviews and merges the six open PRs (three R1 + three R2 stacked). Merge order: R1 first, R2 second per tool.
- [ ] After merges + `brew upgrade`, re-run regression against the brew-installed binaries. Confirm the Homebrew path works.
- [ ] Then dispatch R3 (cross-tool sibling detection in doctors) and R4 (remaining heavier items).
- [ ] Then return to autumn-mail for the first real Sentinel cycle.

## Six PRs currently open

The user has ~30 min of review surface across the trio:

| Tool | R1 | R2 (stacked on R1) |
|---|---|---|
| Touchstone | [#46](https://github.com/autumngarage/touchstone/pull/46) — swift scaffold + gitignore + initial commit + var subst + default branch | [#47](https://github.com/autumngarage/touchstone/pull/47) — `new` wizard |
| Cortex | [#16](https://github.com/autumngarage/cortex/pull/16) — plans template + `.cortex/README.md` | [#17](https://github.com/autumngarage/cortex/pull/17) — `init` interactive |
| Sentinel | [#71](https://github.com/autumngarage/sentinel/pull/71) — `.sentinel/.gitignore` | [#72](https://github.com/autumngarage/sentinel/pull/72) — `init` wizard + reviewer provider fix |
