# Autumn Garage — TODOs

Living checklist of actionable improvements discovered through dogfood.
Grouped by tool. Check off as shipped. Items carry a short hint and a
journal-entry pointer for the full context.

---

## Cross-tool / shared

- [ ] **Interactive-by-default across all first-run commands.** See [`doctrine/0002`](.cortex/doctrine/0002-interactive-by-default.md). Each tool ships its wizard; flags remain as overrides.
- [ ] **Each tool's `doctor` surfaces siblings' presence + version.** `cortex doctor` says "sentinel detected at 0.2.0"; `touchstone doctor` says "cortex detected at 0.1.0"; etc. One-line detection, massive orientation win for new users.
- [ ] **"Getting Started with the Full Garage" doc.** Single page, pinned from each tool's README, walks a brand-new user from `brew install` to first working cycle. Lives in `autumn-garage/` or as a pinned repo in the org.
- [ ] **Default-branch consistency.** All three tools respect `git config init.defaultBranch` (defaulting to `main`). Touchstone currently creates `master`.
- [ ] **Graceful "first commit ever" handling.** Bootstrap paradox — `no-commit-to-branch` pre-commit hook blocks the very first commit when there's nothing upstream to branch from. Either `touchstone new` makes the initial commit itself, or the hook exempts commit-#0.

## Touchstone

- [ ] **`touchstone new <dir>` interactive wizard.** Flags stay; wizard runs when called on a TTY without conflicting flags. Prompts: dir, project type (auto-detected, confirm), language-scaffold? reviewer, register-in-touchstone-projects? initialize Cortex? initialize Sentinel? create initial commit? create GitHub repo? At end, print the equivalent flag form. Grounds-in [`doctrine/0002`](.cortex/doctrine/0002-interactive-by-default.md).
- [ ] **`--type swift` scaffolds a Swift Package.** Minimum: `Package.swift` + `Sources/<Name>/main.swift` (or `App.swift` for executable) + `Tests/<Name>Tests/SmokeTests.swift`. Follows pattern of the existing `--scaffold-tests` work for Python/Node/Go.
- [ ] **Per-profile `.gitignore` entries.** `--type swift` appends `.build/`, `.swiftpm/`, `Package.resolved`, `*.xcodeproj/`, `DerivedData/`. Other profiles get their idiomatic entries too.
- [ ] **`touchstone new` creates initial commit.** Solves the bootstrap paradox; matches `cargo new` / `npm init -y` behavior.
- [ ] **Registry is opt-in (or confirmed).** Today `touchstone new` registers silently unless `--no-register` is passed. Flip default to opt-in or add a prompt.
- [ ] **Templates substitute known variables automatically.** `{{PROJECT_NAME}}` in `CLAUDE.md` / `AGENTS.md` / `README.md` is replaced with the directory name on scaffold. No reason to make the user fill that in by hand.
- [ ] **First-push Codex review exempt.** First push on a fresh scaffold is reviewing AI-generated template files, wastes tokens/quota. Skip review for the initial push (identified by HEAD being one commit old).
- [ ] **`touchstone doctor --project` surfaces `.cortex/` + `.sentinel/` presence.** Cross-tool doctor integration per shared item above.

## Cortex

- [ ] **`plans/template.md` shipped in templates/.** Today only `doctrine/candidate.md` + `journal/*.md` + `digest/*.md` are shipped. A plan template (with `Updated-by`, `## Success Criteria`, etc. prefilled) would reduce first-run friction.
- [ ] **`.cortex/README.md` scaffolded by `cortex init`.** Short orientation: "these are the layers, here's what to edit, here's what's auto-generated." Currently a user arriving via file browser has to read SPEC.md to orient.
- [ ] **Stubs in `state.md` / `map.md` carry guidance.** Today they say "pending Phase C synthesis" — confusing for a user who doesn't know what Phase C is. Replace with "safe to hand-edit; `cortex refresh-state` will regenerate from primary sources when it ships."
- [ ] **`cortex doctor` errors point at templates.** "Plan missing required `Updated-by`" should include "(see `.cortex/templates/plans/template.md`)" once that template exists.
- [ ] **Version surfacing is clearer for new users.** Three version numbers (SPEC v0.3.1-dev, Protocol v0.2.0, CLI v0.1.0) in `cortex version` output. Either consolidate for user-facing display or group under headings (author vs. consumer).
- [ ] **`cortex init` prints what was created.** Today it prints "Scaffolded /path (spec v0.3.1-dev)" and next steps. A file count / structure summary would reassure the user.
- [ ] **Interactive mode for `cortex init`.** Per [`doctrine/0002`](.cortex/doctrine/0002-interactive-by-default.md). Prompt: add `@.cortex/protocol.md` + `@.cortex/state.md` imports to existing CLAUDE.md? add `.cortex/pending/` + `.cortex/rejected/` to `.gitignore`?
- [ ] **`cortex doctor` surfaces Touchstone + Sentinel presence.** Shared doctor integration.

## Sentinel

- [ ] **Explicit `sentinel init` as the first-run path.** Today `sentinel work` auto-inits config, which means first-time users can't preview config before spending. `sentinel init` becomes the explicit entry; `sentinel work` requires config to exist (gracefully suggesting `sentinel init` if missing).
- [ ] **Interactive `sentinel init` wizard.** Prompts: detected providers (multi-select), coder provider, reviewer provider (default to *different* from coder — enforce doctrine in defaults), daily budget cap, run a scan now? Print flag-form at end.
- [ ] **Default config uses different providers for coder + reviewer.** Currently both claude (different models). Default should be e.g., coder=claude-sonnet, reviewer=codex-gpt. This is Sentinel's own doctrine; defaults should reflect it.
- [ ] **`.sentinel/.gitignore` scaffolded by init.** Committed: `config.toml`, `backlog.md`, `lenses.md`, `domain_brief.md`, `runs/`, `proposals/`, `scans/`. Gitignored: `state/`.
- [ ] **"What was created" summary after first `sentinel work`.** Today `.sentinel/config.toml`, `lenses.md`, `domain_brief.md`, etc. appear silently. List them at end of first cycle.
- [ ] **Budget prompt on first real cycle.** If `--budget` not passed on a TTY, prompt: "Set a budget for this cycle? (default: daily cap $15)". Avoid surprise runaways.
- [ ] **Operationalize Cortex Protocol T1.6.** At cycle end, if `.cortex/` present, write `journal/sentinel-cycle.md` entry. Gated on Cortex Phase E integration; tracked here for visibility.
- [ ] **`sentinel status` surfaces Cortex + Touchstone presence.** Shared doctor integration.
- [ ] **Pre-commit / branch-discipline composition.** `sentinel work` tripped `no-commit-to-branch` during its own work. Clarify which tool owns git discipline when multiple are co-installed (Touchstone's hook, presumably — but Sentinel's feature-branch behavior must agree).

---

## Done

- [x] **Doctrine 0002 — interactive-by-default.** 2026-04-18.
- [x] **Initial `autumngarage/autumn-garage` coordination repo** with doctrine/plans/journal + v3 plan. 2026-04-18.
- [x] **Initial `autumngarage/autumn-mail` dogfood project** with Swift Package, SwiftUI stub, `.cortex/`, pre-push review passing on first push. 2026-04-18.
- [x] **First Sentinel dry-cycle** against autumn-mail — 6 custom lenses, 31/100 health score, 3 expansion proposals queued, $0.00 spend. 2026-04-18.
- [x] **Five scaffold-friction findings** journaled: [`journal/2026-04-18-scaffold-friction-findings`](.cortex/journal/2026-04-18-scaffold-friction-findings.md). 2026-04-18.
