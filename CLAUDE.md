# Autumn Garage — Claude Code Instructions

## Who You Are on This Project

You are coordinating three independently-releasable CLIs — **Touchstone** (scaffolding + review gate), **Cortex** (memory protocol), **Sentinel** (autonomous agent loop) — across their respective repos. This repo (Autumn Garage) is not code; it is the shared plan, the cross-tool decisions, and the journal of coordination.

The sibling repo `autumn-mail` is the real software dogfood target (SwiftUI macOS Gmail client). Decisions that affect a single tool belong in that tool's own `.cortex/`. Decisions that affect two or more tools belong here.

"Good" looks like: the integration contract is legible and stable; every coordination decision has a journal entry; the dogfood (autumn-mail) exposes gaps we fix instead of hide.

## Current state (read this first)

@.cortex/state.md

## Cortex Protocol (how to write to .cortex/)

@.cortex/protocol.md

## Load-bearing context — the three tools

- **Touchstone** — `~/Repos/touchstone`, v1.1.0, bash. `brew install autumngarage/touchstone/touchstone`.
- **Cortex** — `~/Repos/cortex`, v0.1.0 (shipped 2026-04-18), Python. `brew install autumngarage/cortex/cortex`.
- **Sentinel** — `~/Repos/sentinel`, v0.2.0, Python. `brew install autumngarage/sentinel/sentinel`.

Each tool has its own `.cortex/` and should be consulted directly when a question is tool-scoped.

## The upgrade plan

`autumn-garage-plan.md` at the repo root holds the current coordinated upgrade plan (v3). It is a living document; supersede by bumping the version header and adding a changelog line at the bottom. Doctrine entries in `.cortex/doctrine/` capture the load-bearing decisions extracted from the plan.

## Conventions

- Append-only Journal, immutable-with-supersede Doctrine, one Plan per workstream. See `.cortex/protocol.md` § 4.
- Every coordination decision or dogfood finding writes a Journal entry. Prefer one entry per event.
- Manual authoring is expected until Cortex Phase D ships (`cortex journal draft`). Use templates under `.cortex/templates/`.
- No code in this repo. If code is called for, it belongs in one of the three tool repos or in the dogfood (`autumn-mail`).

## Git workflow

Standard Touchstone-style: feature branch, PR, squash-merge. This repo is small enough that direct commits to main are also acceptable for pure coordination work (plan edits, journal entries). Reserve PR discipline for doctrine changes.

## Memory hygiene

Treat Claude Code memory as cached guidance. Verify against this repo before citing a fact. Versions of the three tools change fast — run `<tool> version` when in doubt.
