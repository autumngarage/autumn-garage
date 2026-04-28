# Sentinel as drop-in autonomous engineer — plan written

**Date:** 2026-04-28
**Type:** decision
**Trigger:** T1.1 (touches `.cortex/plans/`)
**Cites:** plans/sentinel-autonomous-engineer, plans/sentinel-conductor-migration, plans/sentinel-cortex-t16-integration, doctrine/0003-llm-providers-compose-by-contract

> Wrote `plans/sentinel-autonomous-engineer.md` to capture the product framing surfaced in today's Hermes-comparison conversation: Sentinel as drop-in autonomous engineer with Cortex/Conductor/Touchstone as load-bearing dependencies via subprocess + file contract.

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

## Updates 2026-04-28 evening

Two principle clarifications applied later in the same session, both reflected in plan updates:

1. **Per-tool work doesn't belong in autumn-garage.** Plans for tool-specific change live in the tool's own repo, or as GitHub issues against that tool. autumn-garage's `.cortex/plans/` is reserved for cross-tool coordination thinking. The plan was slimmed accordingly: per-tool task lists removed; replaced with a brief pointer to issue families per repo. Existing `sentinel-conductor-migration.md` flagged as a candidate to move into sentinel's own repo.
2. **Four open questions resolved through deep research** (parallel Explore-agent runs on 2026-04-28, citing ESLint/TS/Ruff for config patterns, Homebrew docs + pre-commit/terraform precedents for packaging, GitHub Issue Forms + OpenAPI for schema versioning, LSP/MCP/dmypy/eslint_d/git for subprocess-vs-daemon trade-offs). Decisions baked into the plan: (1) Sentinel ships defaults; `sentinel init` seeds Cortex; (2) two taps — `tools/` à la carte + `garage/` meta-formula; (3) frontmatter version + immutable HTML anchors for cycle-artifact schema; (4) subprocess for Sentinel→Conductor seam, revising the 2026-04-24 migration plan's import choice.

## Consequences / action items — completed 2026-04-28

- [x] Wave 1 issues filed: **sentinel#89** (subprocess migration), **sentinel#90** (default Doctrine pack), **cortex#60** (grep filter audit), **cortex#61** (`--seed-from` flag), **conductor#93** (CLI contract docs + regression test).
- [x] Migration plan moved: `sentinel-conductor-migration.md` from `autumn-garage/.cortex/plans/` to `~/repos/sentinel/.cortex/plans/` (Slice B + Key design decisions revised for subprocess seam, header banner records the move).
- [x] Autumn-garage plan body updated to reference issues and the migration plan's new home.

## Pending

- [ ] Wave 2–4 issues: filed when Wave 1 lands so they're written against current code (read-side cortex consumption, rejection fold, file-state isolation, cycle-artifact schema, init bootstrap, trust controls, scheduled work, Touchstone PR-body anchor consumer, reviewer journal awareness).
- [ ] Track the plan via Cortex once Phase D (`cortex journal append`) ships; until then plan updates go through manual file edits.
