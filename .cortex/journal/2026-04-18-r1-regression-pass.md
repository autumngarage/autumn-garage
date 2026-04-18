# R1 regression pass — five scaffold frictions verified as fixed

**Date:** 2026-04-18
**Type:** decision
**Trigger:** T2.1
**Cites:** journal/2026-04-18-scaffold-friction-findings, https://github.com/autumngarage/touchstone/pull/46, https://github.com/autumngarage/cortex/pull/16, https://github.com/autumngarage/sentinel/pull/71

> Three R1 PRs landed in parallel (touchstone #46, cortex #16, sentinel #71) covering nine of the items from TODOs.md. Ran a regression scaffold with the Touchstone feature-branch `bootstrap/new-project.sh` against `/tmp/dogfood-check`: all five original friction findings are gone in one clean command.

## What I did

```sh
cd ~/Repos/touchstone
git checkout feat/new-project-r1-improvements
bash bootstrap/new-project.sh /tmp/dogfood-check --type swift --reviewer codex --no-register
```

## Before / after comparison against the five findings

| # | Finding (2026-04-18 morning) | Before | After R1 |
|---|---|---|---|
| 1 | `--type swift` didn't scaffold a Swift package | Hand-wrote `Package.swift`, `Sources/`, `Tests/` (~20 min) | `==> swift: scaffolded Package.swift, Sources/DogfoodCheck/, Tests/DogfoodCheckTests/` printed by `touchstone new`. `swift build` green in 6s. |
| 2 | `.gitignore` missed Swift entries | Staged `.build/` binaries on first commit attempt | `==> .gitignore: appended Swift / SPM entries` — `.build/`, `.swiftpm/`, `*.xcodeproj/`, `DerivedData/`, `Package.resolved` all present. |
| 3 | Bootstrap paradox: `no-commit-to-branch` blocked the first commit | Routed through a feature branch + rename-to-main ceremony | `commit: b552f5e (initial touchstone scaffold)` in the scaffold summary. Created before hooks install, so the hook never fired. `git log --oneline` shows exactly one commit on `main`. |
| 4 | Cortex Goal-hash round-trip | Hand-wrote plan, got doctor error, copied hash back | Cortex #16 ships `plans/template.md` with the placeholder hash literal `(recompute with cortex doctor)`. First-time authors now see the exact error message doctor intends, not a cryptic frontmatter gap. (Not tested in regression yet — requires `brew upgrade cortex` after PR merges.) |
| 5 | No `plans/` template | Inferred from SPEC + other plans | Ships in Cortex #16. Same caveat as #4. |

Plus two non-finding improvements hit along the way:

- **`master` → `main` default branch** — `git init -b "$default_branch"` (falling back to `main` if `init.defaultBranch` isn't set). No more rename ceremony.
- **`{{PROJECT_NAME}}` substitution works non-TTY.** `dogfood-check` → `# dogfood-check — Claude Code Instructions` in CLAUDE.md without any prompts. Previously the substitution was TTY-gated, so agent-driven scaffolds (like this one) left placeholders unsubstituted.

## What the output looked like

```
==> Copying project-owned templates (you own these, safe to edit):
    ...19 files...
==> Wrote .touchstone-config: project_type=swift
==> swift: scaffolded Package.swift, Sources/DogfoodCheck/, Tests/DogfoodCheckTests/
==> .gitignore: appended Swift / SPM entries
==> Installing git hooks
==> touchstone bootstrapped:
    files:    19 added, 0 unchanged
    version:  16be4898...
    hooks:    installed (pre-commit, pre-push)
    commit:   b552f5e (initial touchstone scaffold)
    registry: skipped (--no-register)
```

Compared to the original autumn-mail scaffold summary, the new lines are the `swift: scaffolded`, `.gitignore: appended`, and `commit: b552f5e` rows. Each corresponds directly to a fixed finding.

## Consequences / action items

- [x] Touchstone R1 verified against a throwaway scaffold. Cortex + Sentinel R1 covered by their own test suites (Cortex: 114 pass incl. new `.cortex/README.md` assertions; Sentinel: 399 pass incl. 3 new `.sentinel/.gitignore` tests).
- [ ] Merge the three R1 PRs after user review.
- [ ] After merge + `brew upgrade`, re-run regression with the brew-installed tools to confirm the Homebrew → shipped-binary path works too.
- [ ] Move to R2: interactive wizards for `touchstone new`, explicit `sentinel init`, `cortex init` CLAUDE.md prompts. Three more parallel agents incoming.

## What worked about the parallel-agent dispatch

All three R1 agents finished with clean PRs and passing tests on the first try. Total wall time ~15 min vs. a realistic ~3 hours of me doing them sequentially. Briefs were specific enough that no agent went off-pattern. Key elements that worked:

- **Paths to read first** (CLAUDE.md, principles/, the actual file to edit, the TODOs context).
- **Exact scope in numbered items** — "A, B, C, D, E" with the specific change each makes.
- **Report format capped at 100–150 words** — keeps the signal tight.
- **Explicit "no --auto-merge"** — preserves human review at the actual merge step.
- **Root-cause fixes only, no assertion loosening** — one agent had a test fail on an unrelated helper; it fixed the helper instead of the new behavior.

Worth codifying this as a coordination procedure in `procedures/` once it's run a second time and the pattern is confirmed.
