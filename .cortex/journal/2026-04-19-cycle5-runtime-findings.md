# Cycle 5 (in progress) — runtime findings on improved sentinel 0.3.5

**Date:** 2026-04-19
**Type:** decision
**Trigger:** T2.2 (failed-approach), T2.3 (investigation)
**Cites:** journal/2026-04-19-cycle4-planner-grounding-findings, /tmp/sentinel-cycle5.log, autumngarage/sentinel#81, autumngarage/cortex#22, autumngarage/autumn-mail#1

> Resumed dogfood with sentinel 0.3.5 (post-PR #81) + cortex 0.2.1 (post-PR #22) + autumn-mail PR #1 (scope qualifiers added to CLAUDE.md/AGENTS.md/state.md). Cycle 5 immediately confirmed Fix 1 (approved jumps queue) and the privacy-compliance lens shift (50→95). New findings F6–F10 emerged. Append, don't replace, the cycle-4 findings.

## What confirmed working

- **Fix 1 (approved jumps queue) — confirmed.** Sentinel picked the proposal `2026-04-18-implement-core-gws-cli-wrapper-for-gmail.md` (Status: approved) ahead of the 1 newly-generated refinement and 3 expansion proposals. Cycle 4's exact failure mode is gone.
- **Privacy-compliance lens — 50→95.** With CLAUDE.md/AGENTS.md/state.md scope qualifiers in place, sentinel's lens stops scoring the toolchain reviewer (codex in `.sentinel/config.toml` and `setup.sh`) as a violation. Strong evidence the lens reads scope qualifier text directly from CLAUDE.md, even without sentinel's own lens-scope field being explicitly set in `.sentinel/lenses.md`.
- **Cortex doctor warning — caught the violation it warns about.** When PR #2 attempted whitespace cleanup on `.cortex/journal/` files, codex (the merge reviewer) cited cortex protocol §4.1 directly: "Cortex journal entries are append-only; this PR edits an existing journal entry in place for blank-line cleanup." The cortex doctor warning fires on the *write* side; codex's protocol-aware review fires on the *modification* side. Two layers, one invariant.

## New findings

### F6 — Pre-commit hooks violate cortex append-only invariant

Already documented in TODOS.md (Touchstone section). Pre-commit's `trailing-whitespace` and `end-of-file-fixer` hooks (and likely `mixed-line-ending`) auto-modify any tracked file with whitespace issues — including `.cortex/journal/` entries that are append-only per protocol §4.1. Touchstone's bundled `.pre-commit-config.yaml` template needs `exclude: ^\.cortex/(journal|doctrine)/` on those hooks.

**Reproduction:** rebase that brings back un-trimmed journal files → `git push` → hook tries to fix → conflict-loop → push fails. Or any direct edit to a journal file.

### F7 — Coder has no awareness of installed CLI surfaces it shells out to

When the gws-wrapper work item told the coder "shell out to `gws gmail +triage|+read|+send|+reply`", the coder produced calls like `gws gmail +read <id>` and `gws gmail +reply <id> --body ...`. Codex (the reviewer) caught these by *running* `gws gmail +read --dry-run` and `gws gmail +reply --dry-run` and observing argument-validation errors. The installed gws 0.22.5 actually requires `--id <ID>`, `--message-id <ID>`, `--format json` — surface the coder couldn't have known without inspecting the binary.

**Fix surface (sentinel):**
- Coder pre-flight: when a work item names CLI tools (e.g., `gws`, `swift`, `xcrun`) in `files`, `acceptance_criteria`, or `verification`, run `<tool> --help` (and optionally `<tool> <subcommand> --help`) and prepend the output to the coder's prompt. Bounded by length (truncate or summarize), gated by a per-tool allowlist.
- Alternative: a `cli_surfaces:` field on work items / proposals that lists tools whose help should be loaded. More opt-in but more deterministic.

This pairs with finding F1 from cycle 4 (file-existence grounding). Both are about the planner/coder lacking ground truth about the project's actual environment.

### F8 — Coder iteration limit is fixed at 3, with no escalation path

Cycle 5's gws-wrapper attempt hit `coder iterations: 3/3` with the reviewer still emitting `changes-requested`. Sentinel correctly stopped, but the failure mode is opaque: the user has to dig into `.sentinel/reviews/` to see what the reviewer was asking for. Three iterations also wasn't enough for this work item — the coder was making real progress (silent-failure → still raw API → wrong arg shape) but ran out of budget for the convergence step.

**Fix surface (sentinel):**
- Iteration limit configurable via `.sentinel/config.toml` (e.g., `coder.max_iterations = 5`).
- Print a clear post-mortem when iterations exhaust: which findings remained, what the reviewer's last words were, what next step the user could take (manual fix, scope-down work item, etc.). Currently it just moves on silently.
- Optional: detect "no progress" loops (reviewer findings unchanged across iterations) and bail earlier with a different message.

### F9 — Planner overscopes work items vs. coder/reviewer's per-iteration capacity

"Implement Core `gws` CLI Wrapper for Gmail I/O" was a single proposal with 4–5 sub-surfaces: `+triage`, `+read`, `+send`, `+reply`, plus the wrapper structure, plus integration into AutumnMailApp. Three coder iterations couldn't converge on all of them with each iteration fixing real but partial issues. The work item was correctly scoped at the *intent* level but exceeded what coder+reviewer can land in 3 turns.

**Fix surface (sentinel):**
- Planner heuristic: if a proposal/refinement has more than N (≈3) acceptance criteria OR cites more than M (≈3) files, suggest splitting it. Either auto-split into sub-proposals or flag for user review.
- Or: plan-time estimate of "iterations needed" based on AC count, surfaced as `estimated_iterations: 5` on the work item; sentinel sets coder.max_iterations to that.

### F10 — Refinement-after-failure can self-loop

After the gws-wrapper work item failed verifier, sentinel's planner generated a refinement titled "Unblock and Land the Core `gws` CLI Wrapper" — same target, slightly different framing. Sentinel then started executing this refinement in the same cycle. If the same problems recur, this could chase its tail across N cycles trying to land the same thing.

**Fix surface (sentinel):**
- Track per-target failure history. If a work item targeting the same files / acceptance criteria has failed in the last K cycles, the planner should *not* emit a near-identical refinement; it should either escalate to the user ("this target failed K times, manual intervention needed") or generate a *meaningfully different* refinement (e.g., split into sub-items, reduce scope).
- Pairs with F9 — splitting work items reduces this risk because each piece is smaller and more likely to converge.

## What I'd do next

In priority order, after cycle 5 finishes:

1. **Sentinel: F8 (iteration limit + post-mortem).** Highest user-facing value — the silent move-on is the most painful failure mode.
2. **Sentinel: F7 (CLI surface pre-load).** Biggest dogfood signal of *real bugs prevented* — codex shouldn't have to dry-run binaries to catch arg shapes.
3. **Touchstone: F6 (pre-commit hook excludes for `.cortex/journal/` + `.cortex/doctrine/`).** Template-level fix flows to all bootstrapped projects.
4. **Sentinel: F9 (work-item splitting heuristic).** Bigger architectural change; benefits compound with F8.
5. **Sentinel: F10 (failure-history tracking).** Largest scope; defer until F7–F9 land and we can see whether the self-loop materializes in practice.

Cycle 5 still running; final outcome (verifier pass/fail on the refinement) will append below or in a new entry.
