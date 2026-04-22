# Stage 1 + Stage 2 delivered — Conductor v0.2 + Touchstone v2.0 PRs open

**Date:** 2026-04-21
**Type:** decision
**Trigger:** T1.9 (pull request opened on default branch) ×2 + T1.3 (plan Stage 1+2 → shipped-pending-review)
**Cites:** plans/touchstone-conductor-integration, journal/2026-04-21-touchstone-conductor-build-ratified, journal/2026-04-21-conductor-v0.1-shipped

> Same-session follow-through on the "let it rip" directive: Conductor v0.2 (Stage 1 — shell-out `exec` + capability declarations + prefer/effort axes + concierge init + graceful fallback) shipped to PR #5. Touchstone v2.0 (Stage 2 — delete 4 bash adapters, collapse to single `conductor` adapter, legacy-config auto-migration) shipped to PR #54. Net -448 lines in Touchstone; 4 per-provider adapter trios replaced by 1.

## Context

Earlier journal entry this session ratified the plan and committed to building. Stage 1 completed as 4 commits in Conductor (89 → 137 tests, all green). Stage 2 executed immediately after Stage 1 landed.

Stage 2 scope delivery:

- **Touchstone surface reduction:** 4 reviewer trios (codex/claude/gemini/local) + `build_local_reviewer_prompt` + `append_local_context_file` — ~400 lines deleted from `hooks/codex-review.sh` and `scripts/codex-review.sh`.
- **New single adapter:** `reviewer_conductor_{available,auth_ok,exec}` — ~60 lines. Translates REVIEW_MODE (diff-only/review-only/no-tests/fix) to Conductor's `--tools` + `--sandbox` flags. Uses `conductor exec` for tool-using modes, `conductor call` for diff-only. Prompt passed via stdin.
- **Config migration:** new `[review.conductor]` block with `prefer` / `effort` / `tags` / `with` / `exclude`. Legacy `[review].reviewers = [...]` auto-detected and translated with a one-time migration hint — users don't experience a hard break.
- **Env-var parity:** `TOUCHSTONE_CONDUCTOR_WITH`, `TOUCHSTONE_CONDUCTOR_PREFER`, `TOUCHSTONE_CONDUCTOR_EFFORT`, `TOUCHSTONE_CONDUCTOR_TAGS`, `TOUCHSTONE_CONDUCTOR_EXCLUDE`. `TOUCHSTONE_REVIEWER=<provider>` deprecated with auto-translation.
- **Retired** the legacy `[review.local]` section (with warning) and the `[review.assist]` peer-review flow (disabled in 2.0, returns in 2.1 via `conductor call --exclude`).
- **Tests:** 475 lines of v1.x cascade / per-provider-adapter-flag-translation tests retired (inherently obsolete — those behaviors moved into Conductor). Remaining tests migrated from `$FAKE_BIN/codex` mock (argv-based prompt + `login status` auth) to `$FAKE_BIN/conductor` mock (stdin-based prompt + `doctor --json` auth). All 11 test scripts pass; shellcheck clean.

## What we decided

- **Ship Stage 2 in the same session as Stage 1** rather than wait for Stage 1 to merge first. The contract between Touchstone and Conductor is well-defined (CLI shell-out with stdin prompt + sentinel output), tests mock the binary regardless of whether Conductor is locally installed, and both PRs now review each other's assumptions.
- **Peer review ([review.assist]) deferred to v2.1.** Needs route-log capture for `--exclude <primary_provider>` to work — a separate ~1-day task. Disabling cleanly in 2.0 with a deprecation warning is the honest move.
- **[review.local] retired, not migrated.** The custom-command escape hatch gets re-implemented as a Conductor custom-provider (roadmap v0.3). Users with [review.local] configs get a one-time warning hint.
- **No silent hard break.** Legacy `reviewers = [...]` configs auto-translate + warn; `TOUCHSTONE_REVIEWER` auto-translates to `TOUCHSTONE_CONDUCTOR_WITH`. Users can run 2.0 without editing configs — just with a migration hint on each push until they update.

## Consequences / action items

- [x] Conductor v0.2 PR: [#5](https://github.com/autumngarage/conductor/pull/5) — 137 tests green, clean commit history (4 commits)
- [x] Touchstone v2.0 PR: [#54](https://github.com/autumngarage/touchstone/pull/54) — 11 test scripts green, shellcheck clean, -448 net lines
- [ ] Await codex auto-review on both PRs; address findings as they surface
- [ ] After Conductor #5 lands: tag `v0.2.0`, update brew formula in `autumngarage/homebrew-conductor`
- [ ] After Touchstone #54 lands: tag `v2.0.0`, update brew formula in `autumngarage/homebrew-touchstone`
- [ ] Update `.cortex/state.md` in this repo to reflect the quartet's new versions
- [ ] File separate plan for Stage 3 (Conductor HTTP tool-use loop for kimi/ollama) — the remaining big-engineering item, ~4 weeks
- [ ] File separate plan for Sentinel migration (Stage 5) — blocked on Stage 3
- [ ] Peer-review-via-exclusion implementation → Touchstone 2.1

## What's out of scope for this session's delivery

- **Stage 3 — HTTP tool-use loop** for kimi/ollama. Explicitly deferred; needs its own plan and ~4 weeks of focused work. Today's ship has kimi/ollama cleanly raising `UnsupportedCapability` when tools are requested and being filtered from routing before invocation.
- **Sentinel migration** (replaces `src/sentinel/providers/*.py` with `conductor exec` subprocess calls). Tracked as Stage 5 in the plan; blocked on Stage 3 because Sentinel's coder role needs HTTP tool-use.
- **Brew tap publication** — both releases require the formula PRs separately.
- **Real-world end-to-end dogfood** — run the new setup on autumn-mail's next feature cycle; verify the route log, costs, fallback behavior on live traffic.
