# Scope-aware sync discipline for managed-file tools

> Canonical pattern for Autumn Garage tools that copy managed files into consuming projects and later update those files in place.
>
> Current adopters: **Touchstone** (`touchstone update`) and **Cortex** (template/doctrine sync flows).

## Why this exists

Tools that sync managed files into a project can fail in two predictable ways:

1. **Scope-blind dirty checks** block updates because *any* local dirt exists, even when none of those paths overlap what the tool would write.
2. **Silent drift** accumulates when skipped updates are only shown once and never surfaced again.

This discipline standardizes behavior so users get safe updates without unnecessary blocking, and drift stays visible.

## Normative behavior

Any Autumn Garage tool that writes managed files into an existing project SHOULD implement all of the following.

### 1) Planned-write-set overlap check (not global clean-tree check)

Before writing:

1. Compute the **planned write set** for this invocation.
   - Include every path the tool would write, replace, delete, or chmod.
   - Include tool-owned paths and project-owned files that are template-diffed/merged.
2. Read dirty paths from `git status --porcelain`.
3. Block only when `dirty_paths ∩ planned_write_set != ∅`.
4. Ignore dirty paths outside the planned write set.

If blocked, print overlapping paths and the reason code `dirty-overlap`.

### 2) Persistent skip logging + drift notice

When an update is skipped (for `dirty-overlap` or other policy reasons):

- Append one JSON line to `.git/<tool>/sync-skips.jsonl`.
- On the next interactive command run, show a notice summarizing recent skips and version drift.

Suggested JSONL fields:

- `ts` (ISO-8601 timestamp)
- `tool`
- `project_root`
- `project_version`
- `target_version`
- `reason`
- `overlap_paths` (array)
- `command` (invoked subcommand/flags)

Suggested notice shape:

- `Skipped N updates in the last D days (reason: dirty-overlap). Current: vX, project: vY.`

### 3) Dry-run is always allowed

`--dry-run` MUST bypass overlap blocking and still compute/report:

- planned write set
- overlap set
- version drift

No filesystem writes occur.

### 4) Explicit override for overlap conflicts

Provide `--force-overlap` to proceed despite overlap.

Requirements:

- loud warning before writes
- explicit mention of overlapping paths
- normal write/sync audit trail still recorded

## Shared reason codes

Use stable, machine-readable reason codes for skip logs and notices:

- `dirty-overlap`
- `merge-conflict-risk`
- `missing-prerequisite`
- `policy-block`
- `unknown-error`

## Invariants

Implementations should preserve these invariants:

1. **Safety invariant:** no write occurs to a path currently dirty unless `--force-overlap` is set.
2. **Liveness invariant:** unrelated dirty files do not block sync.
3. **Visibility invariant:** every skip is durably logged and later surfaced.
4. **Idempotence invariant:** rerunning with no source/version changes produces no writes.

## Adoption guidance

- Prefer **shared convention, independent implementation**.
  - This keeps tool repos decoupled (no cross-tool runtime imports) while enforcing one behavior contract.
- Keep per-tool implementation details local, but match this interface and reason-code contract.
- Add per-tool regression tests for:
  - dirty file outside scope (should proceed)
  - dirty file inside scope (should block unless forced)
  - dry-run on dirty tree (should always run)
  - skip log + follow-up notice emission

## Scope notes

This discipline is for tools that ship/update managed project files. It does not apply to:

- generated per-project runtime state stores
- stateless routers
- tools that never mutate project files

## History

Drafted for autumn-garage issue #29 (2026-05-14):
https://github.com/autumngarage/autumn-garage/issues/29
