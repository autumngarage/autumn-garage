# 0001 — Autumn Garage is a coordination repo for the trio, not a monorepo

> Autumn Garage is the coordination layer for three independently-releasable CLIs — Touchstone (the ground), Cortex (the spine), Sentinel (the hands). It holds the shared plan, cross-tool decisions, and the journal of coordination work. The tools themselves stay in their own repos with independent release cadences.

**Status:** Accepted
**Date:** 2026-04-18
**Promoted-from:** — (direct authoring at repo init)
**Load-priority:** always

## Context

Three tools with overlapping concerns invite two failure modes: (a) a monorepo that couples release cadence and forces one big-bang integration, or (b) no shared planning surface, so decisions drift and the integration contract ossifies only after real divergence has occurred.

Cortex's own Doctrine 0002 already rules out code-level coupling — the three tools compose through file contracts, never through a shared library. But file-contract composition still needs a place to answer: what's the integration contract, who decides when it bumps, where do cross-tool decisions live, what's the plan?

A fourth repo — this one — holds that coordination without reintroducing code-level coupling.

## Decision

Autumn Garage exists as a fourth, separate repo alongside Touchstone, Cortex, and Sentinel. It is:

- **A meta-note system.** Doctrine holds cross-tool decisions (e.g., the boundary between Touchstone principles and Cortex Doctrine). Plans track coordinated workstreams (e.g., operationalizing Protocol T1.6 in Sentinel). Journal records the decisions, critiques, and dogfood findings that shaped the plan.
- **Not a monorepo.** Each tool ships on its own cadence from its own repo. Autumn Garage has no code dependencies on any of them.
- **A dogfood target.** Using Cortex to coordinate Cortex's own ecosystem is the strongest possible dogfood — every gap we hit writing `.cortex/` by hand here is a gap Sentinel and Touchstone would hit later. Manual authoring now; `cortex journal append` / authoring helpers land in Cortex Phase D.
- **Separate from the dogfood *application*.** The sibling repo `autumn-mail` is the real software testbed (a SwiftUI Gmail client). This repo tracks the plan; `autumn-mail` is where the tools run.

Inside falls: cross-tool doctrine, coordinated plans, journal of coordination events, state summary.
Outside falls: any single-tool decision (lives in that tool's `.cortex/`), code, templates, shared libraries.

## Consequences

- **What becomes easier:** future-session continuity for the coordination work (the journal is the memory); deciding where a given decision belongs (tool-scoped vs cross-tool); surfacing the integration contract as a first-class artifact rather than an implicit assumption.
- **What becomes harder:** four repos to update when a CIC-wide concept shifts; discipline required to avoid duplicating decisions between a tool's `.cortex/` and this one.
- **What this forecloses:** a monorepo rollup of the three tools; a shared Python library across Touchstone/Cortex/Sentinel; any coordination via chat messages, untracked docs, or ad-hoc wiki pages.

---

Falsification condition: if the coordination work fits comfortably inside Cortex's own `.cortex/` (because cross-tool decisions are rare enough), this repo becomes overhead and should be folded back into `cortex/`.
