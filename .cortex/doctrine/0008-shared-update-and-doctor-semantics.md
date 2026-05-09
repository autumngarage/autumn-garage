---
ID: 0008
Title: Shared update / update-all / doctor semantics across the quartet
Date: 2026-05-08
Status: Active
Load-priority: always
---

# 0008 — Every tool exposes the same `update`, `update-all`, and `doctor` verbs

> The four Autumn Garage CLIs (Touchstone, Cortex, Sentinel, Conductor) share three commands with identical semantics: `<tool> update` brings the current repo's installed scaffolding up to date, `<tool> update-all` is the batch form across known projects, and `<tool> doctor` is a structural install/scaffolding check that exits non-zero on broken state. Self-update of the binary itself stays brew-only — no tool ships a `self-update`.

## Context

A 2026-05-08 audit of the four CLIs found three different verbs for the same conceptual operation — "bring this repo's scaffolding up to date with the installed tool version":

| Tool | Per-repo verb | Batch verb | Doctor scope |
|---|---|---|---|
| Touchstone | `update` | `sync` | review-fail-open trends |
| Cortex | `sync` (composes `refresh-state` + `refresh-index`) | — | structural validation |
| Sentinel | — (none) | — | — (no doctor) |
| Conductor | `refresh-on-commit` (hook-driven) | `refresh-consumers` | env + provider health |

A user shelling between `touchstone update`, `cortex sync`, and `conductor refresh-consumers` has to relearn the verb each time, even though the operation is the same shape. `doctor` is similarly fragmented: one tool reports review trends, one validates structure, one diagnoses the environment, and one has nothing.

This is the kind of drift Doctrine 0003 ("LLM providers compose by contract, not import") and 0004 (Conductor as fourth peer) cannot fix on their own — those govern the *integration contract between tools*, not the *user-facing CLI surface*. We need a separate rule covering the surface every user touches.

The user request was direct: *"can you make sure there's consistency across all the tools around updating. i want to have shared semantics on the commands"*.

## Decision

**1. Three shared verbs, identical contract across all four tools.**

| Verb | Contract |
|---|---|
| `<tool> update` | Bring the current repo's installed scaffolding up to date with the running tool version. Idempotent — second run with no version change is a no-op. Flags: `--dry-run`, `--check`. Operates on the current working directory only. |
| `<tool> update-all` | Walk the tool's registered/known projects and run `update` against each. Tools that don't track multiple projects (Sentinel) may omit this. Flags: `--check`, plus tool-specific filtering. |
| `<tool> doctor` | Verify the install + scaffolding state of the current repo. Exits non-zero on broken state. Reports: tool version, scaffolding version, drift, missing required files, schema-migration debt. **Does not** report ephemeral metrics (review fail-open counts, recent run history) — those belong in `<tool> status` or a tool-specific reporting command. |

**2. Self-update of the tool binary is brew-only.** No tool ships a `self-update` or `upgrade` command. `brew upgrade autumngarage/<tool>/<tool>` remains the single mechanism. This is consistent with how the four are already distributed and avoids each tool re-implementing self-replacement.

**3. Migration commands stay tool-specific.** One-shot schema bumps (e.g., `touchstone migrate-review-config`, `cortex migrate-state`) are not the recurring `update` verb. They live alongside `update` and surface from `doctor` when the project needs them. Migrations may be composed into `update` when safe and idempotent, but the dedicated subcommand remains for explicit operator-driven runs.

**4. Per-tool migration plan to land the shape:**

| Tool | Today | Target |
|---|---|---|
| **Touchstone** | `update`, `sync`, `doctor` (review trends) | Keep `update`. Rename `sync` → `update-all` with `sync` aliased for back-compat. Move review-trend report to `touchstone review-stats` (or fold into `touchstone status`). `doctor` becomes a structural install/scaffolding check. |
| **Cortex** | `sync`, `refresh-state`, `refresh-index`, `migrate-state`, `doctor` (structural ✅) | Add `cortex update` as the primary verb; `sync` aliased for back-compat. Keep `refresh-state` and `refresh-index` as named primitives that `update` composes. `doctor` already structural — no change. No `update-all` (cortex is per-repo today). |
| **Sentinel** | — | Add `sentinel update` (verify `.sentinel/` config schema, run any pending migrations). Add `sentinel doctor` (config schema + cortex/conductor reachability). No `update-all` — sentinel is intentionally per-repo. |
| **Conductor** | `refresh-on-commit` (hook), `refresh-consumers`, `doctor` (env+providers) | Add `conductor update` (the per-repo refresh — what `refresh-on-commit` does without the hook framing). Alias `refresh-consumers` → `conductor update-all`. `refresh-on-commit` stays as the hook entry point (it has different operational semantics — auto-stash, branch-tracked). `doctor` already in scope. |

**5. Aliases live for at least two minor versions.** `touchstone sync`, `cortex sync`, and `conductor refresh-consumers` continue to work but emit a deprecation note pointing to the canonical verb. They are removed no sooner than the second minor release after the alias lands.

## Consequences

- **What becomes easier:** A user dropping into any of the four tools knows the verb. Documentation collapses — one section in autumn-garage's README explains `update` / `update-all` / `doctor` once. Cross-tool scripting (the `/deploy` skill, future CI workflows) can assume a uniform interface. Onboarding to the quartet flattens: learn three verbs, get four tools.
- **What becomes harder:** Each tool needs a coordinated PR. Touchstone has the largest semantic move (its `doctor` changes shape and `sync` is renamed). Sentinel needs to grow surface area it doesn't have today. Aliases add code paths to maintain through the deprecation window.
- **What this forecloses:** Future tools joining the family with different verbs for the same operation (a fifth tool — Alchemist is in flight — must adopt `update` / `update-all` / `doctor`). Per-tool freedom to make `doctor` a metrics report; metrics reports must use a different name. A `<tool> self-update` command would now require superseding this doctrine.

## Relationship to other doctrine

- **Doctrine 0001** (coordination repo for the trio, not a monorepo) — unchanged. This entry adds a cross-tool surface contract, which is exactly what autumn-garage's coordination role is for.
- **Doctrine 0002** (interactive by default) — `update` and `doctor` honor it: prompts on ambiguity in TTY contexts, non-interactive flags (`--check`, `--dry-run`) for CI.
- **Doctrine 0003** (LLM providers compose by contract) — orthogonal. 0003 governs how tools talk to each other; 0008 governs the surface every tool exposes to the user.
- **Doctrine 0004** (Conductor as fourth peer) — Conductor is bound by 0008 the same as the other three. Conductor's existing `doctor` is closest to compliant; the new verb is `update`.
- **Doctrine 0006** (autumn-garage is meta-context) — this entry is exactly the kind of cross-cutting decision that belongs here, not in any single tool's `.cortex/`.

## Falsification condition

If six months after the per-tool issues land, the verbs have visibly drifted again (a tool ships `<tool> sync` as a non-alias new command, or `doctor` is overloaded with metrics in any tool), this doctrine failed and a successor must explain why a uniform CLI surface is not achievable for this family.

## History

Articulated 2026-05-08 in response to a direct request to unify update semantics across the quartet. Audit method: `Explore` agent surveyed each tool's CLI entry point and listed all subcommands; survey + analysis in `journal/2026-05-08-shared-update-semantics.md`. Per-tool implementation tracked as GitHub issues against each tool repo — see the journal entry for the four issue links.
