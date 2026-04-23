---
type: decision
Date: 2026-04-23T22:15
Author: claude-code (Henry Modisett)
Cites: plans/conductor-http-tool-use, journal/2026-04-23-touchstone-conductor-shipped, doctrine/0003-llm-providers-compose-by-contract
---

# Conductor v0.3 shipped — HTTP tool-use loop end-to-end, Slices A/B/C in one day

## Context

Stage 3 of the Touchstone × Conductor integration plan was the HTTP tool-use loop. Before today, `kimi.exec(tools=...)` raised `UnsupportedCapability`, so the router filtered kimi and ollama out of any exec request with a non-empty tool set. Three downstream consumers blocked on this:

1. Sentinel migration (Stage 5) — coder role is multi-turn with tool use
2. Touchstone cheap-path reviews — `[review.routing].small_with = "ollama"` with read-only tools
3. Direct `conductor exec --with kimi --tools Read,Grep ...` calls

## What shipped

All three planned slices + the ASCII branding task merged today, each as its own release.

| Slice | PR | Release | Scope |
|---|---|---|---|
| A   | [#7](https://github.com/autumngarage/conductor/pull/7)  | v0.3.0 | `conductor.tools` module (ReadTool/GrepTool/GlobTool + ToolExecutor + strict path validation); kimi.exec drives the full OpenAI-style tool-use loop with 10-iteration cap; router unblocked for read-only kimi tool requests |
| B   | [#8](https://github.com/autumngarage/conductor/pull/8)  | v0.3.1 | EditTool, WriteTool, BashTool added; workspace-write sandbox enforcement; ollama gets the same tool-use loop against `/api/chat`; kimi + ollama both declare the full six-tool set |
| Banner | [#9](https://github.com/autumngarage/conductor/pull/9) | (rolls into v0.3.2) | Pre-rendered CONDUCTOR ASCII glyph art, deep-purple colored, shown in `conductor init` and `conductor doctor`. Visual parity with touchstone's figlet hero but without the figlet runtime dep |
| C   | [#10](https://github.com/autumngarage/conductor/pull/10) | v0.3.2 | `max_context_tokens` declarations on every provider; HTTP loops halt before exceeding model context; per-iteration cost log in `usage["iterations"]`; new `--sandbox strict` adds POSIX rlimits + tighter timeouts on BashTool |

## Process notes worth preserving

**Commit-push cadence**: consistently committed and pushed per slice, with a separate PR per release. Post-merge I tagged each version on the squash-merge SHA and bumped the brew formula in the same workflow. No stacked PRs, no waiting-on-CI idle time (I opened the next slice's branch while the previous slice's CI ran).

**Test growth**: 203 → 241 (Slice A) → 272 (Slice B) → 282 (Slice C). The stable top-of-file state of the repo's test suite is now a decent regression net for the whole HTTP provider surface.

**Provider-filter tests under capability churn**: when I bumped kimi/ollama's supported_tools/sandboxes in Slice B, router tests that relied on their *narrow* sets broke. Fixed by monkeypatching the capability sets to the pre-v0.3.0 values in the test — the filter logic itself was always correct; the test was checking "what did kimi decline" and the answer moved. Comment explaining the intent is in tests/test_router.py.

**Strict sandbox scoping**: the original plan flirted with "spawn a subprocess sandbox with chroot" for `strict`. In practice I shipped POSIX rlimits + timeout clamps instead. True process isolation is a v0.4 topic — rlimits are best-effort (macOS ignores some) and primary defence remains path validation + cwd pinning. Calling out the limitation explicitly in the PR body prevents anyone from treating strict as a security boundary it's not.

## What this unblocks

- **Sentinel migration (Stage 5)** — sentinel's coder can now use any conductor provider including kimi/ollama, for multi-turn write tasks under workspace-write. The migration PR on sentinel's side is its own session; this slice ships the capability.
- **Touchstone cheap-path reviews** — `[review.routing].small_with = "ollama"` with `mode = "review-only"` now actually routes to ollama instead of falling through to hosted. No touchstone change needed.
- **Direct conductor callers** — `conductor exec --with kimi --tools Read,Grep,Edit --sandbox workspace-write ...` now works.

## What's explicitly NOT done

- Streaming tool-use (Stage 4 — orthogonal)
- Parallel tool execution (stage 4+ optimization; serial is fine for N<5)
- True process-isolation sandbox (v0.4 — needs a real threat model first)
- Sentinel's actual migration (its own session — conductor has shipped what it needs)

## Relationship to autumn-garage state

Update `.cortex/state.md` — quartet remains (Touchstone 2.1 · Cortex 0.2.3 · Sentinel 0.3.4 · **Conductor 0.3.2**). The "HTTP tool-use loop ships in Stage 3" bullet from previous state turns into an accomplishment line.
