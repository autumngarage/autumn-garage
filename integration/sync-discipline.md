# Scope-aware sync discipline for file-shipping tools

> Canonical cross-tool pattern for Autumn Garage CLIs that copy managed files into consuming projects and later update them in place.
>
> This is a **coordination convention**, not a shared runtime library. Each tool implements the same behavior in its own repo.

**Owner:** autumn-garage (this repo)  
**Applies to:** Touchstone, Cortex, and any future tool with managed-file sync behavior  
**Does not apply to:** tools that do not write managed files into consumer repos (or that only generate per-project state)

---

## Problem class

File-shipping tools share two recurring failure modes:

1. **Scope-blind dirty-tree refusal** — update blocks when *any* file is dirty, even if the tool will not touch those paths.
2. **Silent drift after skipped sync** — blocked updates are easy to miss, so projects drift further behind over time.

This discipline defines the safety, liveness, and observability rules that prevent both.

---

## Required invariants

Any `<tool> update` / `<tool> update-all` implementation that writes managed files MUST satisfy:

1. **Safety invariant:** by default, never write a path that overlaps uncommitted local changes.
2. **Liveness invariant:** unrelated local changes outside the tool's planned write set do not block update.
3. **Observability invariant:** every blocked sync leaves a durable local record and is surfaced to the user on a later run.

---

## Canonical behavior

### 1) Planned-write-set-first dirty check (scope-aware)

Before mutating files, compute the update's **planned write set** (all paths the command may create, overwrite, or delete in this run). Compare this set against dirty paths from `git status --porcelain`.

- If `dirty ∩ planned_write_set` is empty, proceed.
- If overlap is non-empty, refuse by default (unless force flag is set).
- Dirty files outside scope never block.

Planned write set should include:

- tool-owned managed directories/files (e.g., shipped templates/scripts/hooks), and
- project-owned files the tool may patch from templates in this run.

### 2) Skip logging + drift surfacing

When update is refused due to overlap, write a skip event to project-local audit storage:

- Suggested path: `.git/<tool>/sync-skips.jsonl`
- One JSON object per line, append-only.

Minimum event fields:

- `timestamp`
- `tool_version`
- `project_version` (if tracked)
- `reason` (e.g., `dirty_overlap`)
- `overlap_paths` (bounded list)
- `planned_write_count`

On subsequent tool runs, emit a visible notice summarizing recent skips, e.g.:

- how many updates were skipped,
- how long the blocking pattern has persisted,
- project version vs current tool-scaffold version,
- most recent blocking reason pattern.

### 3) Dry-run always allowed

`<tool> update --dry-run` MUST run even on dirty trees.

Dry-run should still compute and display overlap findings so users can resolve conflicts before rerunning without `--dry-run`.

### 4) Explicit overlap override

Provide `--force-overlap` (or equivalent explicit flag) to proceed despite overlap.

Requirements:

- loud warning before writes,
- include overlap summary in output,
- log a force event to local audit trail.

---

## Non-goals

- No shared code import across tools.
- No changes to doctrine immutability rules in `.cortex/`.
- No requirement that all tools use identical internal data structures; only behavior is standardized.

---

## Relationship to Doctrine 0008

Doctrine 0008 standardizes command names (`update`, `update-all`, `doctor`).
This document standardizes **how update decides to proceed or refuse** when local changes exist.

Both apply together:

- 0008: shared CLI surface semantics
- sync discipline: scoped safety + anti-drift behavior for managed-file syncing

---

## Adoption checklist (per tool repo)

- [ ] Compute planned write set before any writes.
- [ ] Replace global dirty-tree gate with scoped overlap check.
- [ ] Ensure dry-run path bypasses blocking gate while reporting overlaps.
- [ ] Add `--force-overlap` with warning + audit logging.
- [ ] Add local skip log (`.git/<tool>/sync-skips.jsonl`) and user-facing drift notice.
- [ ] Add regression tests for disjoint dirty paths vs overlapping dirty paths.
