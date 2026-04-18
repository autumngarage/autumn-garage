# Five friction findings from scaffolding autumn-mail end-to-end

**Date:** 2026-04-18
**Type:** decision
**Trigger:** T2.2
**Cites:** plans/autumn-mail-dogfood, ../../autumn-mail/.cortex/journal/2026-04-18-vision, ../../autumn-mail, ../../touchstone, ../../cortex

> First end-to-end scaffold of `autumn-mail` (touchstone new + cortex init + git + gh + first push) surfaced five friction points that should each become an issue against the corresponding tool. Recording them here rather than in the tool repos because they're integration-shaped — each one lives at a seam between tools.

## Context

Ran the sequence a real user would run: `touchstone new autumn-mail --type swift --reviewer codex` → `cortex init` → hand-author doctrine/plan/journal → `swift build` → commit → `gh repo create --push`. Got to a live, private `autumngarage/autumn-mail` with 0 errors on `cortex doctor` and a passing `swift build` in ~40 minutes of active work. The friction points below were the things that slowed it down.

## Findings

### Finding 1: Touchstone's swift profile doesn't scaffold a Swift package

`touchstone new autumn-mail --type swift --reviewer codex` added 19 files — CLAUDE.md, AGENTS.md, principles/, scripts/, .touchstone-*, .pre-commit-config.yaml, setup.sh — but **no `Package.swift`, no `Sources/`, no `Tests/`**. The user has to hand-create the SwiftPM layout or run `swift package init` separately.

**Suggested fix:** `--type swift` should either run `swift package init --type executable` automatically OR scaffold a minimal `Package.swift` + `Sources/<Name>/main.swift` + `Tests/<Name>Tests/` similar to what the `--scaffold-tests` flag does for Python/Node/Go. Related touchstone PRs #43 and #44 establish the pattern; swift is the obvious next profile.

**Where to file:** `autumngarage/touchstone` issue.

### Finding 2: Touchstone's default `.gitignore` is language-agnostic and misses Swift

The scaffold's `.gitignore` covered Python, Node, secrets, and editors — but not `.build/`, `.swiftpm/`, `Package.resolved`, or `*.xcodeproj/`. First `swift build` produced `.build/` which was then staged by `git add -A`, including the compiled binary. Stopped by pre-commit's secret-detection / large-file checks, but the signal was noisy.

**Suggested fix:** `--type swift` should append Swift-specific `.gitignore` entries. Same pattern as Finding 1 — per-profile post-scaffold append.

**Where to file:** `autumngarage/touchstone` issue. Likely the same PR as Finding 1.

### Finding 3: Bootstrap paradox — `no-commit-to-branch` blocks the first commit

Touchstone installs a `no-commit-to-branch` pre-commit hook that forbids direct commits to `main`/`master`. Good discipline. But `touchstone new` doesn't create an initial commit itself, so the user's first attempt is on `main` with no commits yet, and the hook blocks them.

Workarounds:
- `git commit --no-verify` (violates touchstone's own principle against bypassing hooks).
- Route the first commit through a feature branch, then rename/merge to main. This is what I did: `git checkout -b chore/initial-scaffold`, commit, `git branch -m main`. Works but is a ceremony the user shouldn't have to perform.

**Suggested fix:** `touchstone new` should end by creating the initial commit itself (on `main`, with a known-good template message). This is how `cargo new`, `npm init`, `gh repo create --clone` all behave. The initial-commit event is bootstrap-phase and the feature-branch discipline kicks in from commit #2 onward.

**Where to file:** `autumngarage/touchstone` issue.

### Finding 4: Cortex requires pre-computed Goal-hash (awkward round-trip)

`cortex doctor` rejects plans with `Goal-hash: (computed by cortex doctor)` or any placeholder — it demands the exact SHA256[:8] of the normalized title per SPEC § 4.9. Workflow today: write the plan, run `cortex doctor`, copy the hash from the error message, paste back into frontmatter, re-run. Hit this twice today (once in autumn-garage, once in autumn-mail).

**Suggested fix:** ship `cortex plan spawn <slug>` in Phase D; it pre-fills `Goal-hash:` from the title. Already on the Cortex roadmap — this finding just confirms the friction is real.

**Where to file:** confirmed against existing Phase D roadmap; no new issue needed. Optional: add a `--fix-goal-hash` flag to `cortex doctor` that rewrites placeholders in-place for the mid-Phase-C window.

### Finding 5: No plans/ template shipped with `cortex init`

`.cortex/templates/` ships `doctrine/candidate.md`, `journal/*.md`, `digest/*.md` — but no `plans/template.md`. First-time authors have to infer the required shape (Goal-hash, Updated-by, Cites, ## Why (grounding), ## Approach, ## Success Criteria, ## Work items) from SPEC.md or by reading another project's plans/. Missed the `Updated-by:` field on the first try; doctor caught it.

**Suggested fix:** ship a canonical `plans/template.md` alongside the existing templates. Same release as the `plan spawn` Phase D work.

**Where to file:** `autumngarage/cortex` issue.

## Consequences / action items

- [ ] File `autumngarage/touchstone` issue: scaffold `Package.swift` + `Sources/` + `Tests/` for `--type swift` (Findings 1 + 2).
- [ ] File `autumngarage/touchstone` issue: `touchstone new` should create initial commit (Finding 3).
- [ ] File `autumngarage/cortex` issue: ship `plans/template.md` alongside existing templates (Finding 5).
- [ ] Confirm Finding 4 against the open Phase D plan; reuse that plan's work item.

## Meta

Four of the five findings map to tool issues. One (Finding 4) confirms an existing roadmap item. None of them blocked work; each cost 2–5 minutes to work around. Compound cost over many dogfood runs is what motivates fixing them rather than tolerating.

The value of doing the end-to-end scaffold manually — instead of letting Sentinel do it — is exactly that these frictions are visible now rather than hidden behind an autonomous agent's retries.
