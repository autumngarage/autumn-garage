# Setup reflection — new-user experience friction across the trio

**Date:** 2026-04-18
**Type:** decision
**Trigger:** T2.1
**Cites:** doctrine/0002-interactive-by-default, plans/autumn-mail-dogfood, journal/2026-04-18-scaffold-friction-findings, ../TODOS.md

> Reflection on the full setup process from empty directory to live autumn-mail project with first Sentinel dry-cycle complete. Five mechanics-level findings were already journaled earlier; this entry captures the broader NUX/flow observations that led to Doctrine 0002 (interactive-by-default) and the per-tool TODOs list.

## Context

Did the full setup by hand in one sitting — no shell scripts, no copy-paste macros. The point was to feel every step a new user would feel. This entry is the field notes.

Sequence covered:
- Created `autumn-garage/` coordination repo via `cortex init` + hand-authored doctrine/plans/journal
- `touchstone new autumn-mail --type swift --reviewer codex --no-register`
- `cortex init` inside autumn-mail
- Hand-authored `.cortex/doctrine/0001`, `plans/mvp.md`, `journal/2026-04-18-vision.md`
- Hand-wrote `Package.swift`, `Sources/`, `Tests/` because `--type swift` doesn't scaffold them
- `swift build` green
- First commit blocked by pre-commit hook; rerouted through feature branch
- `gh repo create --private --push` for both repos
- `brew install googleworkspace-cli` (gws 0.22.5)
- `sentinel work --dry-run` — auto-inited config, generated 6 custom lenses, $0.00 spend

Already-journaled mechanics findings (see `journal/2026-04-18-scaffold-friction-findings`):
1. `--type swift` doesn't scaffold a Swift package
2. `.gitignore` default missed Swift entries
3. Bootstrap paradox with `no-commit-to-branch` hook
4. Cortex Goal-hash round-trip awkward
5. No `plans/` template in `cortex init`

## New findings from this reflection (beyond the five)

### Cross-tool consistency

- **Three `doctor` idioms with different output shapes.** `touchstone doctor --project`, `cortex doctor`, `sentinel status`. Not wrong, just three vocabularies to learn. Each should surface sibling presence to give the user a "garage-aware" view.
- **Three `init` models.** `touchstone new <dir>` / `touchstone init`, `cortex init`, `sentinel work` (implicit auto-init). The inconsistency means users can't predict whether running a command will spend money, modify files, or just print help.
- **Default branch mismatch.** Touchstone scaffolded `master`; the `autumngarage` org uses `main`. One manual `git branch -m`.
- **Placeholder fill-in.** `{{PROJECT_NAME}}` and `{{PROJECT_DESCRIPTION}}` in `CLAUDE.md` / `AGENTS.md` — the tool knows the project name on `new`, should substitute.

### Cortex-specific

- **"Pending Phase C synthesis" stubs** are confusing when the user doesn't know what Phase C is. A user-facing "safe to hand-edit until synthesis ships" hint would help.
- **No `.cortex/README.md`** to orient a human arriving via file browser.
- **Three version numbers** (SPEC, Protocol, CLI) on day one are opaque to new users.

### Sentinel-specific

- **Config defaults pair coder + reviewer on the same provider** (claude-sonnet + claude-opus). Violates Sentinel's own doctrine that reviewer should be a different *provider*, not just a different model.
- **`.sentinel/` appears silently** on first cycle — no "here's what got created" summary.
- **No `.gitignore` guidance for `.sentinel/`.** Which files are ephemeral, which are durable?

### Touchstone-specific (beyond the already-journaled)

- **Codex review fires on the initial scaffold push**, reviewing its own template files.
- **`--no-register` is an opt-out.** Registry write to `~/.touchstone-projects` is a silent side effect — should be opt-in or confirmed.

## What we decided

Extracted the load-bearing rule as **`doctrine/0002-interactive-by-default`**: first-run commands prompt on TTY, flags override, `--yes` accepts defaults, non-TTY falls back gracefully, every wizard prints its equivalent flag-form at the end.

Per-tool actionables captured in `../TODOS.md` as a living checklist. No GitHub issues — the TODOs file is a single source of truth, cross-referenced from journal entries.

## Consequences / action items

- [x] Write `doctrine/0002-interactive-by-default`.
- [x] Write `TODOS.md` with per-tool action items.
- [x] This journal entry capturing the reflection.
- [ ] Next step: pick a tool to start with (leaning Touchstone — `touchstone new` is the front door and touches everything else). Scope the wizard as a plan in either this repo or directly in `touchstone/.cortex/plans/`.
- [ ] Then flip the gws-integration proposal and run the first real Sentinel cycle on autumn-mail.

## What we'd do differently

The value of doing this all by hand was exactly that every friction was visible. If I'd scripted it or let Sentinel autonomously scaffold, I would have seen maybe 2 of the ~15 findings now captured. Rule for future bootstraps: *always do the first run by hand, journal everything surprising, then automate.*

## Open questions

- Is a **meta-CLI** (`garage <cmd>` that dispatches to the three tools) worth revisiting, or is the shared-doctrine approach enough to hold consistency? My lean is still no meta-CLI — the file-contract composition is the whole point — but the three-doctor / three-init surface argues for *naming* consistency more than it argues for unification.
- Should **`~/.autumngarage-projects`** exist as a shared registry, or should each tool keep its own? Touchstone's registry exists today. Cortex and Sentinel don't have one. If we add one, it should be additive — not a replacement for per-tool state.
