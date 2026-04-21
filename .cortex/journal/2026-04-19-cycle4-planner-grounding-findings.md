# Cycle 4 (paused) — three planner-grounding findings

**Date:** 2026-04-19
**Type:** decision
**Trigger:** T2.2 (failed-approach), T2.3 (investigation)
**Cites:** journal/2026-04-18-session-wrap, .sentinel/runs/2026-04-19-095130.md, .sentinel/worktrees/wi-cycle-1/Sources/AutumnMail/GmailClient.swift

> Resumed dogfood with `sentinel work --auto --budget 5 --coder-timeout 1200` against autumn-mail (sentinel 0.3.4). Stopped mid-review after surfacing three planner findings that matter more than the cycle's output. The salvage tag `salvage/cycle-3-gws-wrapper-pre-retry` preserves the prior reviewer-approved diff in case any of these fixes need a regression baseline.

## Finding F1 — Planner hallucinates file state

The planner emitted a refinement titled "Harden `GmailClient.swift` for Robust `gws` CLI Interaction" and the executor ran it. **`GmailClient.swift` does not exist on `main`** — only `AutumnMailApp.swift` does. The cycle-3 work that originally created the file lived on the deleted branch. The planner had no apparent ground truth for the file's existence; it carried the reference forward from prior scan/proposal context.

The coder silently absorbed the contradiction by **creating the file from scratch** with reasonable hardening shape (retry policy, ProcessRunner abstraction). Tests passed. This is the most insidious failure mode: the planner asked for "improve existing code" and the coder delivered "create new code" — same diff, wrong category.

**Fix surface (sentinel):**
- Pre-execution `git ls-files` check on every refinement's cited files. If any are absent on HEAD, fail loudly or auto-promote to expansion semantics.
- Refinement vs. expansion semantic guard: a refinement whose coder net-creates files is a category error.

## Finding F2 — Planner conflates app-runtime and dev-toolchain constraints

Refinement #2 in the new backlog: *"Remove Cloud LLM from Development and Review Toolchain"*. It flags `.sentinel/config.toml` and `setup.sh` for using `openai`/`codex`. But the project's "no cloud LLM" mandate (CLAUDE.md) explicitly applies to **runtime drafting via MLX Swift**, not the dev-toolchain reviewer where Garage Doctrine 0002 *requires* reviewer ≠ coder (codex is the cross-provider reviewer).

Sentinel's privacy-compliance lens has no scope. It applies the constraint globally and proposes work that would dismantle the Doctrine-0002-mandated cross-provider reviewer.

**Fix surface (sentinel + cortex):**
- Sentinel: lens definitions get a `scope:` field — e.g., `scope: ["Sources/**", "Tests/**"]` for privacy-compliance. Toolchain dirs (`.sentinel/`, `.cortex/`, `scripts/`, `hooks/`) are out of scope for app-runtime constraints.
- Cortex: `cortex doctor` warns when CLAUDE.md asserts unscoped constraints (no `(applies to: runtime|toolchain|both)` qualifier). Prevents the conflation at the source.

## Finding F3 — `Status: approved` is silently bypassed by auto-mode

The cycle-3 reviewer-approved proposal `2026-04-18-implement-core-gws-cli-wrapper-for-gmail.md` sat in `.sentinel/proposals/` with `**Status:** approved` markdown. Sentinel's planner regenerated refinements + expansions and the auto-mode order ran the new refinement first. The user-marked "do this next" signal was bypassed without warning.

**Fix surface (sentinel):**
- Auto-mode order should be: **approved-proposals → refinements → expansions**. Or at minimum, warn when approved proposals exist but the planner picked something else.
- Bonus: when approved proposals exist, suppress regeneration of overlapping new proposals to avoid drift.

## Finding F4 (carryover) — Planner generates semantic dups

12 proposals were queued at the start of cycle 4 — three of them flavors of "gws CLI wrapper for Gmail I/O". Rejection memory only catches exact-title matches. Already noted in passing at cycle-2 wrap; restating here because it surfaced cleanly today.

**Fix surface (sentinel):**
- Planner pre-flight: simple title+rationale keyword-overlap check against existing `proposals/` and `backlog.md` before emitting new ones. ~30 lines.

## Tooling-adjacent finding F5 — Vesper CLI hangs

Outside the trio but bit us today: `vesper split` and `vesper state` hung silently (no timeout, no error) because two stale `vesper state` PIDs from earlier today held the daemon socket. Killed them and the daemon recovered — but `vesper split` continued to hang on subsequent attempts, so we fell back to running `sentinel work` via `nohup` in the parent shell. Not in the trio's scope; logged here so it doesn't get lost.

## What I'd do next

In priority order:

1. **Sentinel: `Status: approved` jumps the queue.** Highest workflow value, smallest diff.
2. **Sentinel: planner pre-flight file-existence check.** Prevents F1 silent miscategorization.
3. **Sentinel: lens `scope:` field.** Bigger refactor; needs schema change. Pairs with cortex doctor warning.
4. **Sentinel: planner pre-flight dedupe.** Cheap heuristic.
5. **Cortex: doctor warning on unscoped CLAUDE.md constraints.** Prevents F2 at the source.

Salvage tag `salvage/cycle-3-gws-wrapper-pre-retry` and the cycle-4 worktree at `.sentinel/worktrees/wi-cycle-1/` both preserved as reference. The cycle-4 worktree's `GmailClient.swift` is decent code that could be salvaged into a normal PR if we want to unblock P1 before fixing the planner.
