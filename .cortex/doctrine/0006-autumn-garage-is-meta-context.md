---
ID: 0006
Title: Autumn-garage is meta-context, not workspace
Date: 2026-04-28
Status: Accepted
Load-priority: always
---

# Doctrine 0006 — Autumn-garage is meta-context, not workspace

Autumn-garage exists to hold *understanding of the four quartet tools* (Touchstone, Cortex, Sentinel, Conductor) and *coordinate cross-cutting decisions*. It is not a workspace where tool-specific work accumulates.

## What lives here

- **Doctrine** — cross-cutting principles that bind the four-tool composition.
- **State** — the current awareness layer of all four tools.
- **Journal** — append-only record of coordination decisions and dogfood findings.
- **Templates** — Cortex protocol artifacts every tool can adopt.
- **Shared infra** — `.github/workflows/homebrew-bump.yml`, `.claude/skills/deploy/`, the eventual `autumngarage/garage/` meta-formula tap.

## What does NOT live here

- **Tool-specific plans** — live in the tool's own `.cortex/plans/`.
- **Tool-specific journal entries** — live in the tool's own `.cortex/journal/`.
- **Cross-cutting plans** — even when work spans multiple tools, the plan lives in the *primary owner's* repo. Cross-cutting decisions become *doctrine entries* here; cross-cutting *work* gets broken into per-tool plans/issues.
- **Customer or external-project artifacts** — outrider intel (vanguard, outrider) is a customer of autumn-garage; their planning lives in their repos and uses their own conventions. Autumn-mail is the user's own dogfood and uses its own `.cortex/`.
- **Tool source code, configuration, or release machinery.**

## How the read/write flow works

**Reads upward.** Autumn-garage's state and journal pull from each tool's `.cortex/state.md`, recent commits, and GH activity. Updates here describe what's happening; they don't drive it.

**Writes downward.** Cross-cutting decisions land as PRs to tool repos (with new tool-specific plans, code changes, or doctrine entries) or as GitHub issues. Autumn-garage holds the journal entry recording the decision; the implementation lives in the target repo.

For small/scoped delegation: GitHub issues against the tool repo. For larger work: PRs that add a plan to the tool's `.cortex/plans/` or change its code.

## Why

Without this discipline, autumn-garage accumulates work-tracking that should be in the tool repos. Tools become harder to use standalone — their own `.cortex/` is incomplete because half the relevant context lives in autumn-garage. The "compose by file contract" invariant (Doctrine 0003) breaks because consumers of one tool can't see plan-level context unless they also clone autumn-garage.

Mature ecosystems get this right structurally: ESLint shareable configs live in their own packages; TypeScript tsconfig bases live in `@tsconfig/...` packages; pre-commit hooks live in their own repos. The opinions ship with their owners. Autumn-garage's role is the *protocol* and the *system map*, not the *opinions*.

## Practical test

If you find yourself wanting to write a plan in autumn-garage, ask:

1. Is this for one specific tool? → It belongs in that tool's `.cortex/plans/`.
2. Is it cross-cutting? → It has a primary owner — write it there. The shared decisions become a doctrine entry here.
3. Is it for a customer's project (outrider intel) or a different ecosystem (autumn-mail)? → Belongs in that project's repo.

If the answer to all three is no, the plan probably doesn't need to exist.

## History

Codifies the principle articulated 2026-04-28 during the Hermes-comparison conversation: "Autumn-garage reads upwards and writes downwards. We don't need to store anything in Autumn Garage proper, aside from an understanding of each tool." Cleaned up 14 plans (8 moved to owning quartet/autumn-mail repos; 7 outrider-intel/customer plans deleted) in the same wave. See journal/2026-04-28-meta-repo-cleanup.md.
