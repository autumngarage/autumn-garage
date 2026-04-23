# Touchstone × Conductor UX-improvement pass

**Date:** 2026-04-23
**Type:** decision
**Trigger:** — (human-authored, multi-PR session summary)
**Cites:** plans/touchstone-conductor-integration, journal/2026-04-21-stage-1-2-delivered

> One session of dogfood-driven UX improvements on the open Touchstone × Conductor PRs (touchstone#54, conductor#5) plus a process-discipline change recorded as a touchstone principle. Seven commits to touchstone, one to conductor, all green tests, both PRs in sync with origin.

## Context

PRs from 2026-04-21 sat overnight without review activity. Rather than waiting on auto-review, walked the user-state space ("S1..S24") to find UX gaps the implementation missed. The exercise produced an actionable list ranked by how many personas hit each gap; this session worked through A → G of that list.

Mid-session the user requested a workflow change: stop batching commits, push at every clear stopping point. That became a memory entry plus a new section in `principles/git-workflow.md` (which propagates to every touchstone-bootstrapped project on init/update).

## What we decided

**Closed UX gaps (touchstone PR #54):**

- **A.** `touchstone init` writes 2.0-shape `.codex-review.toml` directly. Previously every new install fired migration warnings on first push because `write_review_onboarding_config` produced 1.x `[review].reviewers = [...]` shape. Adapter learned to actually parse `[review.conductor]` + 2.0 `[review.routing]` knobs (they were cosmetic before).
- **B.** Conductor route-log surfaces in the touchstone transcript. Provider, cost, tokens, duration are now visible per push (the observability promise of the migration).
- **C.** Cold-onboard error messages name `brew install autumngarage/conductor/conductor` and `conductor init` per failure case (CLI missing vs no provider configured).
- **D.** New `touchstone migrate-review-config` command rewrites legacy 1.x configs in-place. Backs up to `.bak`, idempotent, `--dry-run` available.
- **E.** `TOUCHSTONE_REVIEWER=local` no longer translates to a broken `--with local` pin (no such conductor provider). Now warns and pins to `--with ollama` as the closest 2.0 analog.
- **F.** Cache key includes the conductor knobs (`with`/`prefer`/`effort`/`tags`/`exclude`) and `REVIEW_MODE`, so a shallow review can't satisfy a later push expecting a deep one. Surfaced and fixed an underlying long-standing bug along the way (see below).
- **G.** New `touchstone review --dry-run` previews which provider would review the next push, what it'd cost, how hard it'd think — without spending tokens.

**Process-discipline change:**

- Memory entry "Commit and push at every clear stopping point" + new "Commit and push frequency" section in `principles/git-workflow.md`. Cites Robertson's "Commit Often, Perfect Later" essay and trunk-based-development as background. Codifies the rhythm: ~1 commit per 30–60 min, push after every commit when on an open PR branch.

**Closed conductor follow-ups (PR #5, earlier this round):**

`config show` displays all 5 knobs (was missing `tags`/`with`); wizard adds `[b]ack` and `[h]elp` per-provider menu options; `doctor` warns when ollama daemon is up but the declared default model isn't pulled; init summary rephrased so callers' overrides aren't misrepresented.

## Surprise finding

While implementing F, debug instrumentation revealed `review_cache_key` had been silently truncating its input for an unknown amount of time. The function referenced `$LOCAL_REVIEWER_COMMAND` (a 1.x variable that's never assigned in 2.0). With `set -u` active, the unset reference aborted the cache-key subshell partway through — discarding everything after it (assist fields, prompt content, file contents, branch diff). The hash effectively covered only the first ~5 fields by accident.

The fix was to use `${VAR:-}` default-to-empty for every variable in the key block, and bump the cache version v2 → v3 to invalidate the legacy half-broken entries on first push after upgrade. This is the kind of bug that stays latent because the system *appears* to work — caches hit, pushes pass — but the cache was a much weaker correctness invariant than anyone thought.

Worth a doctrine entry candidate: "When cache keys are composed in shell, every variable reference must be `${VAR:-}` — `set -u` will silently truncate otherwise."

## Consequences / action items

- [x] touchstone#54: 7 polish commits, all pushed to origin, tests 12/12
- [x] conductor#5: polish commits pushed, tests 147/147
- [x] `principles/git-workflow.md` updated with commit/push frequency section — propagates downstream on `touchstone update`
- [x] `feedback_frequent_commits.md` added to memory; MEMORY.md index updated
- [ ] Both PRs still awaiting reviewer activity — no auto-review fired in the 24h prior, may need to invoke codex review manually or just merge
- [ ] After merge: tag `touchstone v2.0.0` + `conductor v0.2.0`; bump brew formulas
- [ ] Run `touchstone migrate-review-config` against autumn-mail's `.codex-review.toml` (canonical user of touchstone, will validate the migration command on a real legacy config)
- [ ] H/I (retired `[review.local]`/`[review.assist]` user paths) blocked on conductor v0.3 (custom providers) and touchstone 2.1 (peer review return) — file as separate plans when those workstreams kick off
- [ ] Consider a doctrine entry on shell cache-key composition with `${VAR:-}` discipline

## What's *not* shipped

- Live end-to-end push of any of these changes through autumn-mail or sigint. Confidence is from mocked integration tests + scratch-repo dogfooding only. The route-log transcript change in particular will look different in real-world output (real conductor stderr formatting may surprise the awk filter); plan to validate on the next autumn-mail push.
- Conductor side: tags/with display in `config show` is text-only; no JSON consumer changed. Should be invisible to anyone, but worth a note.
