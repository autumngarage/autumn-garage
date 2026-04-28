# Sentinel-as-drop-in-engineer — strategic direction + meta-repo discipline tightened

**Date:** 2026-04-28
**Type:** decision
**Trigger:** T1.1 (touches `.cortex/plans/`) + meta-repo discipline clarification
**Cites:** sentinel/.cortex/plans/sentinel-autonomous-engineer (lives in sentinel), sentinel/.cortex/plans/sentinel-conductor-migration (lives in sentinel), doctrine/0003-llm-providers-compose-by-contract, doctrine/0006-autumn-garage-is-meta-context (incoming)

> Today's Hermes-comparison conversation produced (1) a product direction for Sentinel as drop-in autonomous engineer with cortex/conductor/touchstone as load-bearing dependencies, (2) four cross-cutting decisions resolved through deep research, (3) a tightened meta-repo discipline: autumn-garage holds doctrine + state + journal + templates + shared infra only. All plans — even cross-cutting ones — live in the primary owner's repo. Cross-cutting decisions become *doctrine entries* here; cross-cutting *work* gets broken into per-tool plans/issues.

## Context

Today's conversation started as a question about Nous Research's Hermes Agent ("what if we installed this guy") and unfolded into a deeper audit of Sentinel's actual architecture. Three findings reshape Sentinel's product story:

1. **Sentinel is a one-way Cortex contributor.** Audit confirmed every `.cortex/` reference in `~/repos/sentinel/` is detection, config, or write — zero reads. `cortex manifest --budget N` ships in v0.2.3 and Sentinel calls it zero times. Each cycle starts amnesiac.
2. **Parallel mini-Cortex inside `.sentinel/state/rejections.jsonl`.** Sentinel reinvented append-only memory next door to the journal it's already writing to.
3. **`import conductor.providers` + git-ref pin.** Violates Doctrine 0003 in spirit; the 2026-04-24 `sentinel-conductor-migration` plan chose import deliberately (line 81), which creates the bundling pressure we'd been feeling.

Net: Sentinel can be the only autonomous engineer that ships with project memory, engineering values, and an audit trail by default — and drops into any repo with one command.

## What we decided

- Wrote `plans/sentinel-autonomous-engineer.md` as the product-shaped umbrella.
- Three pillars (memory / engineering values / agent delegation) → three peer subprocess contracts (Cortex / Cortex Doctrine / Conductor).
- Touchstone retains PR-shape ownership; new small seam: `open-pr.sh` reads `.sentinel/runs/<latest>.md` for sentinel-authored branches.
- Default Doctrine pack ships with `sentinel init` (or cortex; open question §1) — most product-distinctive move available because Hermes ships skills (capabilities), not values.
- 8 Sentinel workstreams, 3 Cortex, 2 Conductor, 2 Touchstone, sequenced in 4 waves.
- Existing `sentinel-conductor-migration` plan inherited as Wave 1 prerequisite — deliberately leaves the import-vs-subprocess tension unresolved (Open Question §4) for a short design spike.
- Renamed proposed `sentinel watch` → "scheduled `sentinel work`" per user note: stay on the existing command surface, don't fork a new subcommand.

## Discipline tightening (later in the same session)

Three principle clarifications, applied progressively:

1. **Per-tool work doesn't belong in autumn-garage.** Plans for tool-specific change live in the tool's own repo, or as GitHub issues against that tool.
2. **Four open questions resolved through deep research** (parallel Explore-agent runs on 2026-04-28, citing ESLint/TS/Ruff for config patterns, Homebrew docs + pre-commit/terraform precedents for packaging, GitHub Issue Forms + OpenAPI for schema versioning, LSP/MCP/dmypy/eslint_d/git for subprocess-vs-daemon trade-offs). Decisions: (a) Sentinel ships default Doctrine; `sentinel init` seeds Cortex; (b) two taps — `tools/` à la carte + `garage/` meta-formula; (c) frontmatter version + immutable HTML anchors for cycle-artifact schema; (d) subprocess for Sentinel→Conductor seam, revising the 2026-04-24 migration plan's import choice.
3. **Even cross-cutting plans live in the primary owner's repo.** The Sentinel-as-drop-in plan I wrote in autumn-garage was reverted; the plan lives in sentinel instead. autumn-garage's role is meta-context only: doctrine, state, journal, templates, shared infra. The pattern is "reads upward from tool .cortex/, writes downward as PRs/issues to tool repos." Doctrine 0006 codifies this (incoming).

## Consequences / action items — completed 2026-04-28

- [x] Wave 1 GitHub issues filed across tool repos: **sentinel#89** (subprocess migration), **sentinel#90** (default Doctrine pack), **cortex#60** (grep filter audit), **cortex#61** (`--seed-from` flag), **conductor#93** (CLI contract docs + regression test).
- [x] `sentinel-conductor-migration.md` moved from autumn-garage to sentinel/.cortex/plans/ (Slice B + Key design decisions revised for subprocess seam). PR: autumngarage/sentinel#91.
- [x] `sentinel-autonomous-engineer.md` moved from autumn-garage to sentinel/.cortex/plans/ (cross-cutting product plan; primary owner is sentinel). Same PR: autumngarage/sentinel#91.

## Pending — meta-repo cleanup sweep

- [ ] Doctrine 0006 codifying the meta-repo discipline (autumn-garage holds only meta-context; plans live in tool repos).
- [ ] Move 14 other tool/vanguard-specific plans out of autumn-garage to their owning repos: 4 → conductor, 2 → sentinel, 1 → touchstone, 1 → autumn-mail, 7 → vanguard (creates `.cortex/plans/`).
- [ ] End state: autumn-garage's `.cortex/plans/` empty.
- [ ] Wave 2–4 issues: filed when Wave 1 lands so they're written against current code.
