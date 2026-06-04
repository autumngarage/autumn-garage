# Autumn Garage — Claude Code Instructions

## Who You Are on This Project

You are coordinating four independently-releasable CLIs across their respective repos:

- **Touchstone** — scaffolding + pre-push AI review gate. *The ground.*
- **Cortex** — portable file-format protocol for project memory. *The spine.*
- **Sentinel** — autonomous assess→plan→delegate→review loop. *The hands.*
- **Conductor** — capability-aware router across LLM providers. *The voice.*

This repo (Autumn Garage) is the shared plan, the cross-tool decisions, the journal of coordination, and a small amount of cross-cutting infrastructure (a reusable Homebrew tap-bump workflow and a `/deploy` skill). Decisions that affect a single tool belong in that tool's own `.cortex/`. Decisions that affect two or more tools, or the integration contract between them, belong here.

The sibling repo `autumn-mail` is the real software dogfood target (SwiftUI macOS Gmail client). Findings from it feed back here as journal entries.

"Good" looks like: the integration contract is legible and stable; every coordination decision has a journal entry; the dogfood exposes gaps we fix instead of hide.

## Current state (read this first)

@.cortex/state.md

## Cortex Protocol (how to write to .cortex/)

@.cortex/protocol.md

## Load-bearing context — the four tools

Each tool installs independently and ships through its own Homebrew tap (`autumngarage/homebrew-<tool>`). The tap formula is the canonical "what version is current" record — never pin versions in this doc; check the tap or run `<tool> version` when it matters.

- **Touchstone** — `~/Repos/touchstone`, bash. `brew install autumngarage/touchstone/touchstone`. Scaffolding, pre-push AI review gate, cross-repo sync. The ground every other tool sits on.
- **Cortex** — `~/Repos/cortex`, Python. `brew install autumngarage/cortex/cortex`. Portable `.cortex/` protocol — append-only Journal, immutable-with-supersede Doctrine, one Plan per workstream. The spine that holds project memory across tools and time.
- **Sentinel** — `~/Repos/sentinel`, Python. `brew install autumngarage/sentinel/sentinel`. Autonomous assess→plan→delegate→review loop. Does the work; cycles through tasks within a budget.
- **Conductor** — `~/Repos/conductor`, Python. `brew install autumngarage/conductor/conductor`. Capability-aware router across `claude`, `codex`, `gemini`, `kimi`, `ollama`. The voice every other tool that needs an LLM speaks through (always as a subprocess, never as a library import).

The four compose by file contract, not code import — the load-bearing invariant. See `.cortex/doctrine/0003-llm-providers-compose-by-contract.md` and `.cortex/doctrine/0004-conductor-as-fourth-peer.md`.

Each tool has its own `.cortex/` and should be consulted directly when a question is tool-scoped.

## Shared infrastructure in this repo

This repo hosts a small amount of cross-cutting infrastructure beyond the markdown coordination — both pieces touch every tool's release path, so changes to either warrant PR review.

- **`.github/workflows/homebrew-bump.yml`** — reusable workflow each tool's own `release.yml` calls (pinned `@v1`) when a GitHub Release is published. Computes the tarball SHA, rewrites `url` + `sha256` in the corresponding `homebrew-<tool>` Formula, and commits directly to the tap's `main`. One source of truth for the four tools' release-to-tap path.
- **`.claude/skills/deploy/SKILL.md`** — Claude Code skill that surveys the four tool repos for unpublished commits past the latest tag, suggests a version bump from conventional-commit prefixes, and cuts releases through each tool's flavor (hatch-vcs vs cortex's `__init__.py` + `pyproject.toml` bump vs touchstone's `bin/touchstone release` helper). Use it by working in this repo and asking Claude to "deploy pending work" or running `/deploy`.

## The upgrade plan

`autumn-garage-plan.md` at the repo root holds the current coordinated upgrade plan (versioned in-document). It is a living document; supersede by bumping the version header and adding a changelog line at the bottom. Doctrine entries in `.cortex/doctrine/` capture the load-bearing decisions extracted from the plan.

## Conventions

- Append-only Journal, immutable-with-supersede Doctrine, one Plan per workstream. See `.cortex/protocol.md` § 4.
- Every coordination decision or dogfood finding writes a Journal entry. Prefer one entry per event.
- Manual authoring is expected until Cortex Phase D ships (`cortex journal draft`). Use templates under `.cortex/templates/`.
- No application code in this repo. Cross-cutting CI workflows and shared agent skills are welcome — Doctrine 0001/0002 still hold (the tools compose through file contracts, never code imports).

## Git workflow

Standard Touchstone-style: feature branch, PR, squash-merge. This repo is small enough that direct commits to main are also acceptable for pure coordination work (plan edits, journal entries). Reserve PR discipline for: doctrine changes, the shared workflow under `.github/workflows/`, and the `/deploy` skill — those affect every tool's release path.

## Memory hygiene

Treat Claude Code memory as cached guidance. Verify against this repo before citing a fact.

- Things that change fast (memory rots): each tool's current version, current PR/commit refs, "today's" / "this week's" work. Check the tap formula or run `<tool> version`; check `.cortex/state.md` or the relevant tool's `.cortex/journal/`.
- Things that change slowly (memory tends to be reliable): each tool's role; the four-tool composition; the file-contract invariant; the release flow shape; `.cortex/` invariants.

<!-- conductor:begin v0.10.32 -->
@~/.conductor/delegation-guidance.md
<!-- conductor:end -->
