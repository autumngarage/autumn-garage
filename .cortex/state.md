---
Generated: 2026-04-18T13:30:00-07:00
Generator: hand-authored (regeneration infrastructure ships in Cortex Phase C)
Sources:
  - doctrine/0001-why-autumn-garage-exists (accepted 2026-04-18)
  - plans/autumn-mail-dogfood (active, created 2026-04-18)
  - journal/2026-04-18-kickoff
  - ../autumn-garage-plan.md (v2 at /Users/henry.modisett/Repos; to be migrated and bumped to v3)
  - github.com/autumngarage/{touchstone@1.1.0, cortex@0.1.0, sentinel@0.2.0}
  - github.com/googleworkspace/cli (gws, experimental)
Corpus: 1 Doctrine, 1 active Plan, 1 Journal entry
Omitted: []
Incomplete:
  - Principles-vs-Doctrine boundary doc (D1 from plan v2) — not yet written
  - Cortex Integration Contract (CIC) v0 spec — Cortex Protocol v0.2.0 already defines the read side; the write side (consumer → Cortex) operationalizes in Cortex Phase E
Conflicts-preserved: []
Spec: 0.3.1
---

# Project State

> Autumn Garage is the coordination repo for the Touchstone/Cortex/Sentinel trio. Autumn Mail is the dogfood target — a SwiftUI macOS Gmail client using `gws` + MLX Swift. Kickoff day. Cortex v0.1.0 shipped on Homebrew earlier today, which means the integration contract operationalization can begin immediately instead of waiting on Phase B.

## P0 — Operationalize Cortex Phase E integrations

The three tools already compose at the file-contract level (Cortex Doctrine 0002). Cortex Protocol v0.2.0 already specifies the write triggers consumers should honor (T1.6 sentinel-cycle, T1.7 touchstone-arch-diff, T1.9 pr-merged). Operationalization is: Sentinel and Touchstone begin writing those entries into `.cortex/journal/` in projects that have `.cortex/` present.

Operationalization gates on Cortex Phase D authoring helpers (`cortex journal draft`, `cortex plan spawn`). Until those ship, consumers drop well-formed markdown files with the right frontmatter directly — same contract, manual write.

Tracked in: `plans/autumn-mail-dogfood` as the first real test bed.

## P1 — Autumn Mail MVP

A buildable SwiftUI mail client reading Gmail via `gws` and drafting replies with MLX Swift. Full plan at `plans/autumn-mail-dogfood`. Success = user can triage, read, reply-via-local-LLM, and send on a clean macOS install with the garage installed.

## P2 — Principles vs Doctrine boundary (D1)

Not yet written. Short doc clarifying that Touchstone `principles/*.md` are universal/cross-repo (synced) and Cortex `doctrine/*.md` are per-project (versioned in that repo). Doctrine entries may `Grounds-in: touchstone-principle-<slug>`. Will ship as Cortex SPEC appendix + Touchstone principle.

## P3 — CIC v0 spec document

Cortex Protocol v0.2.0 covers the read side. The remaining work is formalizing the write side (consumer → Cortex) — ideally as an appendix to Cortex's own SPEC. Will happen alongside Cortex Phase D. No work in Autumn Garage required beyond capturing the decision trail here.

---

## Installed tool versions

- Touchstone 1.1.0 (9 unreleased PRs since tag; v1.2.0 cut overdue independently of this work)
- Cortex 0.1.0 (shipped 2026-04-18 on Homebrew)
- Sentinel 0.2.0

## Open decisions

D1 Principles/Doctrine boundary · D2 Write contract (settled: CLI-primary via Phase D `journal draft` helpers) · D3 Umbrella branding (settled: `autumngarage` org, pinned README) · D4 Tap structure (settled: three separate taps) · D5 Compat surfacing · D6 Review-hook → journal opt-in default · D7 Nested `.cortex/` in monorepos (deferred) · D8 Deferred-write longevity (settled: CLI-only, no file-drop fallback in v0)
