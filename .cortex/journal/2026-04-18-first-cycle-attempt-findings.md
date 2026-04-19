# First sentinel cycle on autumn-mail — T1.6 wins, planner→coder bug blocks execution

**Date:** 2026-04-18
**Type:** incident
**Trigger:** T1.2
**Cites:** plans/sentinel-cortex-t16-integration, plans/autumn-mail-dogfood, https://github.com/autumngarage/autumn-mail, journal/2026-04-18-r5-findings-from-fresh-scaffold

> Attempted the first real `sentinel work --auto --budget $5` cycle on autumn-mail to build the gws CLI wrapper. Cycle didn't execute the approved expansion because a planner→coder contract crash (TypeError on WorkItem.files) blocked the refinement queued ahead of it. T1.6 journal writes worked flawlessly across four failed attempts — strong evidence the file-contract composition is sound. Hotfix dispatched.

## What worked (big wins)

### 1. T1.6 lands on shipped binaries ✅

Every `sentinel work` invocation — dry run, failed runs, the moment-of-abort — produced a conformant `.cortex/journal/<date>-sentinel-cycle-<id>.md` entry. All validated clean under `cortex doctor`. Plan `plans/sentinel-cortex-t16-integration.md` success criterion #1 ("produces both `.sentinel/runs/` AND `.cortex/journal/` entries at cycle end") is verified on shipped brew binaries. This is the first real proof that cortex Doctrine 0002's file-contract composition works across independently-released tools.

### 2. Lens generation reads project memory ✅

Sentinel v0.3.0's scan of autumn-mail produced six lenses that directly reflected the hand-authored doctrine + plan + CLAUDE.md content:

- **privacy-compliance** 80/100 ← derived from CLAUDE.md's "no cloud LLMs ever"
- **gws-robustness** 0/100 ← derived from doctrine/0001 naming gws as the sole IO surface
- **local-llm-perf** 20/100 ← derived from plans/mvp's MLX Swift constraint
- **toolchain-dogfood** 45/100 ← derived from doctrine/0001's stated meta-purpose
- **swiftui-ux** 0/100 ← derived from the SwiftUI + macOS scope
- **project-craft** 65/100 ← derived from the principles/ files

The planner is actually reading the Cortex layers. That's the whole point, and it works.

### 3. Sibling detection works on shipped binaries ✅

`cortex doctor` on autumn-mail: `✓ touchstone 1.2.0 (installed)`, `✓ sentinel 0.3.0 (installed)` — detected correctly via the R3 file-contract pattern without any Python imports. Brew-installed versions see each other.

## What broke (the real blocker)

### Finding C1 — Planner→Coder WorkItem.files contract mismatch (crash)

```
File "sentinel/roles/coder.py", line 486, in execute
    files = ", ".join(work_item.files) or "(let coder determine)"
TypeError: sequence item 0: expected str instance, dict found
```

The planner parses `Files:` bullets in backlog entries as dicts like `{"path": "scripts/touchstone-run.sh", "note": "Entrypoint for running Sentinel..."}` — the `path — description` bullet format. The coder does `", ".join(work_item.files)` and assumes strings. Every refinement that has descriptions in its Files section will crash the cycle before the coder starts.

Impact: **sentinel v0.3.0 cannot execute any non-trivial work item**. The crash happens before the coder gets invoked, so the reviewer never runs either. Pure blocker.

Fixing: hotfix agent dispatched to cut sentinel v0.3.1 with the consumer side made tolerant of both list[str] and list[dict] shapes.

### Finding C2 — Planner doesn't know sentinel's own changelog (tautology loop)

Sentinel's toolchain-dogfood lens scanned autumn-mail and flagged: *"Sentinel runs are occurring but not being journaled, indicating a silent failure in the toolchain integration."* This produced a refinement — "Automate Sentinel Cycle Journaling" — that proposed implementing T1.6 via a project-local `scripts/touchstone-run.sh` hook and a new `.cortex/procedures/record-sentinel-cycle.sh` script.

The catch: T1.6 is already shipping in sentinel v0.3.0 itself. The planner doesn't know what sentinel ships. It's seeing the *lack* of project-local trigger scripts (which nobody needs) and proposing to create them. Classic case of an agent hallucinating a problem because it doesn't have ground-truth on the tool it's using.

Worse, `sentinel work` does scan → plan → execute in one atom. The plan step regenerates `backlog.md` from the scan. So **editing the backlog between runs doesn't stick** — the tautology refinement comes back on each work invocation.

Fix options:
- **Sentinel plan step should deduplicate proposals against sentinel's own version changelog.** If the proposal is "automate X" and X is sentinel's own v0.3.0 T1.6 feature, skip. Requires sentinel to know what it ships, which is non-trivial.
- **Heuristic: any scan-flagged "missing integration" whose "fix" requires a script in `scripts/` that duplicates a sentinel CLI command should be auto-rejected.** Cheaper; narrower.
- **User override: a `--skip-refinement <id>` flag on `sentinel work` to skip specific items without editing backlog.**

Not in scope for the hotfix; logged as a separate sentinel TODO.

### Finding C3 — `sentinel work` always replans; backlog edits are ephemeral

Documented behavior, but worth codifying. The `work` command's design is "scan → plan → execute" in one atom. This means the user cannot hand-tune the backlog between runs — each `work` starts from scratch. Good for autonomy (no stale state), bad for iterative guidance.

Request: consider a `sentinel work --no-replan` flag that executes the existing backlog without re-running scan + plan.

## Side finding — T1.6 entries pile up

Four `work` invocations produced four `sentinel-cycle-*.md` entries, even though the later three were failed attempts. This is correct per T1.6 design (write at cycle end regardless of outcome), but in a tight-iteration scenario like this, the journal fills up fast. Cortex has monthly/quarterly digest support in its SPEC — would want to activate that here eventually.

## State of autumn-mail

- `autumngarage/autumn-mail` live on GitHub.
- Touchstone scaffold + `.cortex/` + `.sentinel/` committed.
- Four failed T1.6 entries in `.cortex/journal/` — noisy but not harmful.
- `.sentinel/backlog.md` trimmed (tautology refinement deleted — will come back next scan).
- `.sentinel/proposals/implement-gws-cli-wrapper-for-gmail-i-o.md` marked approved, waiting for sentinel v0.3.1.

## Consequences / action items

- [ ] Wait on sentinel v0.3.1 hotfix (dispatched to agent). Then retry `sentinel work --auto --budget $5`.
- [ ] Journal v0.3.1 release confirmation here once landed.
- [ ] Re-run cycle, observe whether gws-wrapper actually gets coded + reviewed + PR'd.
- [ ] File a separate sentinel TODO for Finding C2 (planner changelog-awareness / tautology detection).
- [ ] File a separate sentinel TODO for Finding C3 (`--no-replan` flag).

## Meta — the dogfood thesis is working

This exact sequence — "new user opens the box, hits a crash, journals it, the loop fixes it" — is the point of autumn-mail existing. Sentinel v0.3.0 had two real bugs that weren't findable without a real Swift project with restored content driving the planner into unfamiliar shape. The bugs get fixed, sentinel v0.3.1 ships, cycle retries, next round of findings emerges.

What I was wrong about earlier: I said "the tool-side build-out is complete after R5." Completion is a fiction. The tools are never done; they're at the latest round of "what this scaffold surfaced." The loop is the product.
