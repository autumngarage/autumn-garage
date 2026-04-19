# Autumn Garage — TODOs

Living checklist of actionable improvements discovered through dogfood.
Grouped by tool. Check off as shipped. Items carry a short hint and a
journal-entry pointer for the full context.

---

## Cross-tool / shared

- [x] **Interactive-by-default across all first-run commands.** Doctrine 0002 implemented in [touchstone#47](https://github.com/autumngarage/touchstone/pull/47), [sentinel#72](https://github.com/autumngarage/sentinel/pull/72), [cortex#17](https://github.com/autumngarage/cortex/pull/17). Each tool has a TTY-gated wizard, `--yes` defaults, non-TTY fallback, and prints an "Equivalent to rerun" block.
- [x] **Each tool's `doctor` surfaces siblings' presence + version.** Shipped R3 across all three — [touchstone#48](https://github.com/autumngarage/touchstone/pull/48) stacked on #47, [cortex#18](https://github.com/autumngarage/cortex/pull/18) stacked on #17, [sentinel#73](https://github.com/autumngarage/sentinel/pull/73) stacked on #72. Consistent pattern: `shutil.which` / `command -v` CLI detection + marker-file check + 3s-timed version shell-out with `<tool> version` / `<tool> --version` fallback. Zero code imports (Doctrine 0002). ✓/—/! glyphs for full/absent/mixed state.
- [ ] **"Getting Started with the Full Garage" doc.** Single page, pinned from each tool's README, walks a brand-new user from `brew install` to first working cycle. Lives in `autumn-garage/` or as a pinned repo in the org.
- [ ] **Default-branch consistency.** All three tools respect `git config init.defaultBranch` (defaulting to `main`). Touchstone currently creates `master`.
- [ ] **Graceful "first commit ever" handling.** Bootstrap paradox — `no-commit-to-branch` pre-commit hook blocks the very first commit when there's nothing upstream to branch from. Either `touchstone new` makes the initial commit itself, or the hook exempts commit-#0.

## Touchstone

- [x] **`touchstone new <dir>` interactive wizard.** Shipped in [touchstone#47](https://github.com/autumngarage/touchstone/pull/47) (stacked on #46). 7 new prompts, 8 new flags, `--yes` support, Ctrl-C cleanup via trap, "Equivalent to rerun" teach-by-doing block. Fixed latent stdin-hang in review-config `read` calls as part of non-TTY hardening.
- [x] **`--type swift` scaffolds a Swift Package.** Shipped in [touchstone#46](https://github.com/autumngarage/touchstone/pull/46) — `Package.swift` + `Sources/<PascalName>/<PascalName>App.swift` + `Tests/<PascalName>Tests/SmokeTests.swift`. Follows PR #44 pattern.
- [x] **Per-profile `.gitignore` entries.** Shipped in touchstone#46 — `--type swift` appends `.build/`, `.swiftpm/`, `*.xcodeproj/`, `DerivedData/`, `Package.resolved`.
- [x] **`touchstone new` creates initial commit.** Shipped in touchstone#46 — created before hooks install so `no-commit-to-branch` never fires. Bootstrap paradox solved.
- [x] **Default branch respects `git config init.defaultBranch`.** Shipped in touchstone#46 — `git init -b "$default_branch"` defaulting to `main`.
- [x] **Template `{{PROJECT_NAME}}` substitution works non-TTY.** Shipped in touchstone#46 — `INPUT_NAME` defaults to `basename(PROJECT_DIR)` on non-TTY fresh scaffolds, so CLAUDE.md/AGENTS.md get substituted in agent-driven flows.
- [x] **Registry write is visible.** Shipped in [touchstone#49](https://github.com/autumngarage/touchstone/pull/49). Every registry outcome now prints a visible line ("Registered in …", "Already registered …", "Registry skipped (--no-register)"). Default stays opt-in for script compat.
- [x] **First-push Codex review exempt.** Shipped in touchstone#49. `hooks/codex-review.sh` + `scripts/codex-review.sh` skip review when `git rev-list --count HEAD == 1` on the default branch. Defensive fall-through if detection fails. `CODEX_REVIEW_FORCE=1` still bypasses.
- [ ] **`scripts/open-pr.sh` supports `--base <branch>` for stacked PRs.** Today it hardcodes `--base $DEFAULT_BRANCH`. The Cortex R2 agent had to bypass with `gh pr create --base <r1-branch>` to open a stacked PR. Accept an explicit base, or auto-detect from the parent tracking branch.
- [ ] **Document the "don't stack PRs with `gh pr merge --squash`" gotcha** in `principles/git-workflow.md` (touchstone). See [`journal/2026-04-18-stacked-merge-recovery`](.cortex/journal/2026-04-18-stacked-merge-recovery.md). Alternative: auto-detect stack context in `open-pr.sh` and warn.
- [ ] **Coordination playbook:** when the user says "ship it all," default to bundled rounds from the start rather than stacked PRs. Faster review, cleaner merge.
- [ ] **R5 scaffold hardening (touchstone):** (1) initial commit must run AFTER `cortex init` / `sentinel init` so their artifacts are captured; (2) `--with-sentinel` should not blanket-ignore `.sentinel/` at project root — Sentinel's own `.sentinel/.gitignore` controls what's ignored; (3) fix SC2034 unused-variable warnings in `scripts/codex-review.sh` (C_GREEN, C_CYAN). Surfaced in [`journal/2026-04-18-r5-findings-from-fresh-scaffold`](.cortex/journal/2026-04-18-r5-findings-from-fresh-scaffold.md). Bundle as touchstone v1.2.1.
- [x] **`touchstone doctor --project` surfaces `.cortex/` + `.sentinel/` presence.** Shipped in [touchstone#48](https://github.com/autumngarage/touchstone/pull/48).

## Cortex

- [ ] **Sibling-detection glyph set is underspecified.** Current R3 uses `✓` for "CLI installed" regardless of whether the project actually uses the tool — so `cortex doctor` in a repo without `.touchstone-config` still shows `✓ touchstone 1.2.2 (installed) — .touchstone-config absent`. Misleading. Propose four-state glyph set: `✓` marker present + CLI installed (fully wired), `○` CLI installed but marker absent (available but unused), `—` CLI not installed, `!` marker present but CLI missing. All three tools need updating (touchstone, cortex, sentinel) since each shipped R3 with the same two-state glyph bug. 2026-04-18.
- [ ] **`cortex doctor --audit` classifies T1.2/T1.3/T1.4/T1.6/T1.7 fires.** Phase B first-slice only shipped T1.1/T1.5/T1.8/T1.9 classification. T1.6 specifically blocks autumn-mail's full validation loop: Sentinel #74 ships T1.6 journal writes, but `doctor --audit` won't match them to cycle fires until this lands. Source: sentinel#74 agent report + existing Phase C deferrals in cortex's own plans.
- [x] **`plans/template.md` shipped in templates/.** Shipped in [cortex#16](https://github.com/autumngarage/cortex/pull/16) — canonical Plan template with required frontmatter + exact section headings + Goal-hash hint that surfaces doctor's helpful recompute message.
- [x] **`.cortex/README.md` scaffolded by `cortex init`.** Shipped in cortex#16 — orientation doc naming all six layers, safe-to-hand-edit rules, pointers to the Protocol.
- [x] **Stubs in `state.md` / `map.md` carry guidance.** Shipped in [cortex#19](https://github.com/autumngarage/cortex/pull/19). "Hand-authored placeholder" language + "hand-editable until `cortex refresh-{layer}` ships" frontmatter note. `.cortex/templates/README.md` updated to match.
- [ ] **`cortex doctor` errors point at templates.** "Plan missing required `Updated-by`" should include "(see `.cortex/templates/plans/template.md`)" once that template exists.
- [ ] **Version surfacing is clearer for new users.** Three version numbers (SPEC v0.3.1-dev, Protocol v0.2.0, CLI v0.1.0) in `cortex version` output. Either consolidate for user-facing display or group under headings (author vs. consumer).
- [ ] **`cortex init` prints what was created.** Today it prints "Scaffolded /path (spec v0.3.1-dev)" and next steps. A file count / structure summary would reassure the user.
- [ ] **Interactive mode for `cortex init`.** Per [`doctrine/0002`](.cortex/doctrine/0002-interactive-by-default.md). Prompt: add `@.cortex/protocol.md` + `@.cortex/state.md` imports to existing CLAUDE.md? add `.cortex/pending/` + `.cortex/rejected/` to `.gitignore`?
- [x] **`cortex doctor` surfaces Touchstone + Sentinel presence.** Shipped in [cortex#18](https://github.com/autumngarage/cortex/pull/18).

## Sentinel

- [x] **Explicit `sentinel init` as the first-run path.** Shipped in [sentinel#72](https://github.com/autumngarage/sentinel/pull/72) (stacked on #71). `sentinel init` unhidden and canonical; `sentinel work` still auto-inits but prints a visible warning pointing at `sentinel init`.
- [x] **Interactive `sentinel init` wizard.** Shipped in sentinel#72 — providers multi-select, coder, reviewer (default different provider), models, budget, optional scan. `--yes` defaults; non-TTY preserves implicit behavior. Prints equivalent flag form.
- [x] **Default config uses different providers for coder + reviewer.** Shipped in sentinel#72 — reviewer now defaults to `codex` (orthogonal provider family) instead of `gemini`. Fallback chain: codex→gemini→local→claude-with-warning. `apply_preset("recommended", ...)` now post-checks reviewer != coder.
- [x] **`.sentinel/.gitignore` scaffolded by init.** Shipped in [sentinel#71](https://github.com/autumngarage/sentinel/pull/71) — `state/` gitignored; durable artifacts (config.toml, runs/, proposals/, scans/, backlog.md, lenses.md, domain_brief.md) kept trackable. Never overwrites.
- [ ] **"What was created" summary after first `sentinel work`.** Today `.sentinel/config.toml`, `lenses.md`, `domain_brief.md`, etc. appear silently. List them at end of first cycle.
- [ ] **Budget prompt on first real cycle.** If `--budget` not passed on a TTY, prompt: "Set a budget for this cycle? (default: daily cap $15)". Avoid surprise runaways.
- [x] **Operationalize Cortex Protocol T1.6.** Shipped in [sentinel#74](https://github.com/autumngarage/sentinel/pull/74) stacked on #73, per the plan at [`plans/sentinel-cortex-t16-integration`](.cortex/plans/sentinel-cortex-t16-integration.md). 34 new tests, 8/12 success criteria automated, 1 integration-test-gated-on-real-cortex, 3 requiring manual verification or downstream Cortex work. Real dependency surfaced: criterion #3 (doctor --audit matching T1.6 fires) can't land until Cortex classifies T1.6 in its audit (Phase B first-slice deferred).
- [x] **`sentinel status` surfaces Cortex + Touchstone presence.** Shipped in [sentinel#73](https://github.com/autumngarage/sentinel/pull/73).
- [ ] **Pre-commit / branch-discipline composition.** `sentinel work` tripped `no-commit-to-branch` during its own work. Clarify which tool owns git discipline when multiple are co-installed (Touchstone's hook, presumably — but Sentinel's feature-branch behavior must agree).

---

## Done

- [x] **Doctrine 0002 — interactive-by-default.** 2026-04-18.
- [x] **Initial `autumngarage/autumn-garage` coordination repo** with doctrine/plans/journal + v3 plan. 2026-04-18.
- [x] **Initial `autumngarage/autumn-mail` dogfood project** with Swift Package, SwiftUI stub, `.cortex/`, pre-push review passing on first push. 2026-04-18.
- [x] **First Sentinel dry-cycle** against autumn-mail — 6 custom lenses, 31/100 health score, 3 expansion proposals queued, $0.00 spend. 2026-04-18.
- [x] **Five scaffold-friction findings** journaled: [`journal/2026-04-18-scaffold-friction-findings`](.cortex/journal/2026-04-18-scaffold-friction-findings.md). 2026-04-18.
- [x] **Round 1 shipped across all three tools** via three parallel background agents — [touchstone#46](https://github.com/autumngarage/touchstone/pull/46), [cortex#16](https://github.com/autumngarage/cortex/pull/16), [sentinel#71](https://github.com/autumngarage/sentinel/pull/71). Regression pass against `/tmp/dogfood-check` confirmed all five original frictions fixed; see [`journal/2026-04-18-r1-regression-pass`](.cortex/journal/2026-04-18-r1-regression-pass.md). 2026-04-18.
- [x] **Round 2 shipped across all three tools** — interactive-by-default wizards implementing Doctrine 0002. [touchstone#47](https://github.com/autumngarage/touchstone/pull/47) stacked on #46, [sentinel#72](https://github.com/autumngarage/sentinel/pull/72) stacked on #71, [cortex#17](https://github.com/autumngarage/cortex/pull/17) stacked on #16. Three gotchas surfaced and fixed in-PR: Touchstone `open-pr.sh` hardcoded `--base main` (workaround, tracked as new TODO), latent stdin-hang in review-config `read` calls (fixed), `--yes` non-determinism on machines with cortex+sentinel installed (test harness forces explicit `--no-with-*`). 2026-04-18.
- [x] **Round 3 shipped across all three tools** — each doctor/status surfaces sibling presence + version. [touchstone#48](https://github.com/autumngarage/touchstone/pull/48) stacked on #47, [cortex#18](https://github.com/autumngarage/cortex/pull/18) stacked on #17, [sentinel#73](https://github.com/autumngarage/sentinel/pull/73) stacked on #72. Consistent convergent detection pattern across all three tools without shared code (Doctrine 0002 honored). All three agents independently caught Sentinel's Click `--version` convention and added the fallback. 2026-04-18.
- [x] **Round 4 (small items) shipped on Touchstone + Cortex.** [touchstone#49](https://github.com/autumngarage/touchstone/pull/49) stacked on #48 — registry writes visible + first-push Codex review exempt. [cortex#19](https://github.com/autumngarage/cortex/pull/19) stacked on #18 — "Hand-authored placeholder" stub language replaces "pending Phase C synthesis." T1.6 Sentinel→Cortex journal write operationalization deliberately deferred (Cortex Phase E territory, deserves a plan entry rather than a one-shot agent brief). 2026-04-18.
