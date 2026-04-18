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
- [x] **`--type swift` scaffolds a Swift Package.** Shipped in [touchstone#46](https://github.com/autumngarage/touchstone/pull/46) — `Package.swift` + `Sources/<PascalName>/<PascalName>App.swift` + `Tests/<PascalName>Tests/SmokeTests.swift`. Follows PR #44 pattern.
- [x] **Per-profile `.gitignore` entries.** Shipped in touchstone#46 — `--type swift` appends `.build/`, `.swiftpm/`, `*.xcodeproj/`, `DerivedData/`, `Package.resolved`.
- [x] **`touchstone new` creates initial commit.** Shipped in touchstone#46 — created before hooks install so `no-commit-to-branch` never fires. Bootstrap paradox solved.
- [x] **Default branch respects `git config init.defaultBranch`.** Shipped in touchstone#46 — `git init -b "$default_branch"` defaulting to `main`.
- [x] **Template `{{PROJECT_NAME}}` substitution works non-TTY.** Shipped in touchstone#46 — `INPUT_NAME` defaults to `basename(PROJECT_DIR)` on non-TTY fresh scaffolds, so CLAUDE.md/AGENTS.md get substituted in agent-driven flows.
- [ ] **Registry is opt-in (or confirmed).** Today `touchstone new` registers silently unless `--no-register` is passed. Flip default to opt-in or add a prompt.
- [ ] **First-push Codex review exempt.** First push on a fresh scaffold is reviewing AI-generated template files, wastes tokens/quota. Skip review for the initial push (identified by HEAD being one commit old).
- [ ] **`touchstone doctor --project` surfaces `.cortex/` + `.sentinel/` presence.** Cross-tool doctor integration per shared item above.

## Cortex

- [x] **`plans/template.md` shipped in templates/.** Shipped in [cortex#16](https://github.com/autumngarage/cortex/pull/16) — canonical Plan template with required frontmatter + exact section headings + Goal-hash hint that surfaces doctor's helpful recompute message.
- [x] **`.cortex/README.md` scaffolded by `cortex init`.** Shipped in cortex#16 — orientation doc naming all six layers, safe-to-hand-edit rules, pointers to the Protocol.
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
- [x] **`.sentinel/.gitignore` scaffolded by init.** Shipped in [sentinel#71](https://github.com/autumngarage/sentinel/pull/71) — `state/` gitignored; durable artifacts (config.toml, runs/, proposals/, scans/, backlog.md, lenses.md, domain_brief.md) kept trackable. Never overwrites.
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
- [x] **Round 1 shipped across all three tools** via three parallel background agents — [touchstone#46](https://github.com/autumngarage/touchstone/pull/46), [cortex#16](https://github.com/autumngarage/cortex/pull/16), [sentinel#71](https://github.com/autumngarage/sentinel/pull/71). Regression pass against `/tmp/dogfood-check` confirmed all five original frictions fixed; see [`journal/2026-04-18-r1-regression-pass`](.cortex/journal/2026-04-18-r1-regression-pass.md). 2026-04-18.
