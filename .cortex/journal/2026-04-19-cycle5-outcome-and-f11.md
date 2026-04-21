# Cycle 5 outcome — 1 reviewer-approved, verifier-blocked on swiftlint scaffold gap (F11)

**Date:** 2026-04-19
**Type:** plan-transition
**Trigger:** T1.6 (sentinel cycle ended), T2.2 (failed-approach), T2.3 (investigation)
**Cites:** journal/2026-04-19-cycle5-runtime-findings, journal/2026-04-19-cycle4-planner-grounding-findings, autumn-mail/.sentinel/runs/2026-04-19-133705.md

> Cycle 5 ran 2642s, $2.96/$5. Two work items executed: the approved gws-wrapper proposal (rejected by reviewer at 3/3 iterations) and the planner's follow-up refinement "Unblock and Land the Core gws CLI Wrapper" (reviewer-approved on iteration 3, verifier-blocked). Salvage tag `salvage/cycle-5-gws-wrapper-reviewer-approved` preserves the approved branch. Verifier failure root-caused to a touchstone scaffold gap — F11.

## Cycle 5 numbers

| Metric | Value |
|---|---|
| Items executed | 2 (1 approved, 1 rejected, 0 failed-mid-flight) |
| Time | 2642s (~44 min) |
| Cycle spend | $2.96 / $5 |
| Daily spend | $3.45 / $15 |
| Coder iterations max | 3 (hit by both items) |
| Reviewer changes-requested | 5 across both items |
| Reviewer approvals | 1 (on iteration 3 of item 2) |
| Verifier passes | 1 of 2 checks |

## What worked (vs cycle 4)

1. **Approved jumps queue (Fix 1)** — sentinel picked the existing approved gws-wrapper proposal first, not a regenerated refinement. Cycle 4's exact failure mode is fixed.
2. **Privacy-compliance lens 50→95** — scope qualifiers in CLAUDE.md/AGENTS.md/state.md correctly de-escalated codex-as-toolchain-reviewer. The lens reads scope qualifier text directly; sentinel's lens-`scope:` field (Fix 4) wasn't even needed for this case.
3. **Reviewer caught real CLI-shape bugs** — codex dry-ran `gws gmail +read` and `+reply` to find argument-format mismatches the coder produced. This is the kind of catch the cross-provider Doctrine-0002 split was designed for.
4. **Sentinel didn't hallucinate file state on iteration 2** — Fix 2's `git ls-files` pre-check would have rejected a refinement citing files that didn't exist, but the planner correctly cited extant files this time, so the check didn't have to fire. (Negative-evidence-only: the fix would have caught a regression but no regression occurred.)

## What didn't work — Finding F11

### Verifier blocked on swiftlint linting build artifacts

`swiftlint --strict` returned 24 violations on the reviewer-approved branch. **22 of 24 are in `.build/arm64-apple-macosx/debug/AutumnMailPackageTests.derived/runner.swift`** — a SwiftPM-generated file, not source code. Only 2 violations are in actual project source:

- `Sources/AutumnMail/GWSWrapper.swift:26` — `to` < 3 chars (identifier_name)
- `Tests/AutumnMailTests/GWSWrapperTests.swift:104` — line length 178 (>120 limit)

**Root cause:** autumn-mail has no `.swiftlint.yml`. SwiftLint does NOT exclude `.build/` by default; the tool requires explicit configuration to skip build artifacts. So every `swift test` run produces files swiftlint then complains about.

### Why this matters

This same pattern blocked cycle 3 (different work item, same kind of swiftlint noise). Sentinel iterated correctly to fix the 2 real violations (cycle 5's coder rename `d` → `decoder` and broke the long JSON line into multiline), but the dominant 22 violations stayed because no source code change can remove them.

### Fix surface (touchstone)

**F11 — touchstone's swift profile should ship a `.swiftlint.yml` with sensible excludes** (or update the existing `.touchstone-config` to point at one in `templates/swift/`). At minimum:

```yaml
excluded:
  - .build
  - .swiftpm
  - DerivedData
  - Pods
  - vendor
```

Optionally tune severity rules to match a sane default for new Swift projects. Add a regression test in touchstone that scaffolds a swift project, runs `swift test` (which generates `.build/`), then runs `swiftlint --strict` and expects zero violations on a clean scaffold.

This is template-level — every new touchstone-bootstrapped swift project would inherit it. Existing projects (just autumn-mail today) get the file by re-running `touchstone update` after the template ships, or hand-author it.

## Salvage state

- **`salvage/cycle-5-gws-wrapper-reviewer-approved`** — tag pointing at the cycle-2 branch's HEAD (reviewer-approved gws-wrapper refinement). Branch deleted; tag preserves the diff.
- The 2 real swiftlint violations on this branch are 5-minute fixes; the dominant `.build/` noise needs F11 to disappear.
- After F11 ships, salvage tag could be checked out, the 2 source fixes applied, and a normal PR opened — that's the salvage path to land autumn-mail's first real feature without waiting for another sentinel cycle.

## Updated priority order (after cycle 5)

The user redirected after cycle 5 mid-run: *"the first major goal is can we get codex to a point where it can be installed into our other repos."* So:

1. **Codex installability** (new P0 — TBD what specifically blocks it; awaiting user clarification).
2. **F11** (touchstone swift `.swiftlint.yml` template) — small, high-leverage fix. Unblocks autumn-mail salvage path.
3. **F8** (sentinel coder iteration limit + post-mortem) — workflow-quality.
4. **F7** (sentinel coder CLI-surface awareness) — prevents the entire class of arg-shape bugs codex caught.
5. **F6** (touchstone pre-commit excludes for `.cortex/journal/` + `.cortex/doctrine/`).
6. **F9, F10** (sentinel work-item splitting, failure-history tracking) — bigger architectural changes.
