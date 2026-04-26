---
Status: proposed
Written: 2026-04-26
Author: claude-code (drafted at human direction during P3.2 cleanup of the 2026-04-26 all-nighter)
Goal-hash: (recompute with cortex doctor)
Updated-by:
  - 2026-04-26T05:45 claude-code (initial draft — captures the workstream that gates dropping the outrider Python dep from vanguard)
Cites: plans/full-vanguard-outrider-separation, journal/2026-04-26-decoupling-foundation-shipped, vanguard#21 (in-process flywheel deletion), vanguard#22 (AST allow-list test)
---

# Vanguard execution flywheel — Stage 2: vanguard owns its own DB schema

> Migrate the SQLAlchemy models for tables vanguard owns out of `outrider._platform.db` / `outrider._platform.db_queries` into `vanguard/_platform/db.py`. This is the workstream that closes the last ~60 allow-listed `outrider.*` imports in vanguard and unblocks Stage C.3 of the parent plan (drop the `outrider @ git+https` dep, remove `GITHUB_PAT` from `railway.json`, remove `OUTRIDER_PAT` from CI).

## Why

Per `journal/2026-04-26-decoupling-foundation-shipped.md`, the all-nighter landed the decoupling foundation at the intelligence layer. The proposal-as-product, contract mirrors, in-process flywheel deletion, reference-data-over-HTTP, and AST guards are all in. The symbolic ship — dropping the dep — did not happen because vanguard still imports outrider's Postgres ORM machinery. Specifically:

- ~42 imports of `outrider._platform.db_queries`
- ~18 imports of `outrider._platform.db`

These are the SQLAlchemy models for tables vanguard already owns at the Postgres level (`vanguard-db` is a separate database; the *models* are what's still upstreamed from outrider's package). Vanguard cannot drop the dep until these move.

This stage was scoped in the parent plan as B.2 ("Vanguard ports its DB models"). It was deferred from the all-nighter because it requires a multi-deploy migration with dual-write semantics — too risky to land in one overnight session. It needs its own plan.

## Scope

In:

- Port `events_outbox`, `outcomes_inbox`, `vanguard_proposal_cursors`, `vanguard_risk_state` SQLAlchemy models to `vanguard/_platform/db.py`.
- Port the query helpers (`db_queries.*`) vanguard actually uses — audit first; some may be dead post-tonight's flywheel deletion (#21).
- Dual-write deploy: vanguard writes to both old (outrider-defined) and new (vanguard-defined) ORMs for one deploy window; verify row equivalence; cutover reads; remove old paths.
- Drop the `outrider @ git+https://...` line from `vanguard/pyproject.toml`.
- Remove the `GITHUB_PAT` plumbing from `vanguard/railway.json` buildCommand.
- Remove the sibling-checkout step + `OUTRIDER_PAT` reference from `vanguard/.github/workflows/test.yml`.
- Verify Railway deploy + CI both green from a fresh clone.
- Tighten the AST allow-list in `vanguard/tests/aegis/unit/test_no_outrider_imports.py` to zero (or near-zero — Stage C.2 test residuals close in parallel).

Out:

- Vanguard's own *learning loop* (per-broker fill quality, per-strategy realization, paper/live drift, exec-side autolab). That is the much larger `vanguard-execution-flywheel-stage-3-learning-loop` workstream. This Stage 2 is *just DB ownership* — the precondition for the dep drop. It does NOT ship vanguard's own multi-axis monitoring or healing.
- The remaining C.2 test rewrites (`outrider.api.*` test imports, calibration carve-outs, reasoning carve-outs). Tracked separately under the parent plan's Stage C.

## Stages

### S1 — schema audit + migration prep

- Audit which `outrider._platform.db.*` tables vanguard actually reads/writes today. Some may have been orphaned by tonight's flywheel deletion (#21) and can simply be dropped from the import surface without porting.
- Draft `vanguard/_platform/db.py` with SQLAlchemy models matching the live Postgres schema for vanguard-owned tables.
- Add an alembic migration (or whatever vanguard uses today) that creates the new tables under their new model names *without dropping the old ones yet*. This is the dual-write precondition.
- One PR.

### S2 — dual-write deploy

- Vanguard writes to both old (via `outrider._platform.db_queries`) and new (via `vanguard._platform.db`) ORMs for every relevant transaction. Reads still go through the old path.
- Deploy. Verify row-by-row equivalence over a 24–48h window. Add a small parity-check job that flags mismatches.
- One PR; one deploy window.

### S3 — cutover reads

- Vanguard reads switch from `outrider._platform.db_queries` to `vanguard._platform.db` query helpers. Writes stay dual until S4.
- Verify read paths match expected behavior (test suite + smoke against staging).
- One PR; one deploy.

### S4 — drop the old path + drop the dep

- Remove `outrider._platform.db_queries` and `outrider._platform.db` import sites from vanguard. Allow-list shrinks by ~60.
- Once the AST allow-list is empty (this stage + Stage C.2 finish, depending on which lands last):
  - Drop `outrider @ git+https://...` from `vanguard/pyproject.toml`.
  - Remove `GITHUB_PAT` from `vanguard/railway.json` buildCommand.
  - Remove the sibling-checkout step + `OUTRIDER_PAT` from `vanguard/.github/workflows/test.yml`.
- One PR. **This is Stage C.3 of the parent plan — the symbolic ship.**

### S5 — Stage D.2 final verification

- Fresh-clone bootstrap on a clean machine: `git clone vanguard && cd vanguard && uv venv && uv pip install -r requirements.txt && bash scripts/test-trade-critical.sh`. Exits 0 with no PATs configured.
- Write the closing journal entry (`journal/2026-XX-XX-vanguard-outrider-fully-separated.md`).
- Flip `plans/full-vanguard-outrider-separation.md` `Status:` to `shipped`.
- Flip this plan `Status:` to `shipped`.

## Success criteria

1. `grep -E "outrider\._platform\.(db|db_queries)" vanguard/ --include="*.py" -rn` returns empty.
2. AST allow-list `outrider._platform.db` and `outrider._platform.db_queries` entries removed.
3. (At S4 close) `grep -i outrider vanguard/pyproject.toml` returns empty.
4. (At S4 close) `grep GITHUB_PAT vanguard/railway.json` returns empty.
5. (At S4 close) `grep -i 'outrider\|OUTRIDER_PAT' vanguard/.github/workflows/test.yml` returns empty.
6. Fresh-clone bootstrap green on a machine with no other repos cloned.
7. Vanguard core aegis suite stays green throughout.
8. No row-mismatch alerts during S2 dual-write window.

## Risks

- **Schema drift between outrider's models and the live Postgres schema.** Vanguard has been running against the live schema for months; if outrider's ORM definitions have drifted from live (additive columns, indexes added in production but not in code), the new vanguard models must match *live*, not the outdated outrider definition. Mitigation: introspect the live `vanguard-db` schema before drafting the new models.
- **Dual-write performance.** Writing to both ORMs doubles the per-transaction work. May be material on the hot paths (outbox relay, outcomes ingest). Mitigation: dual-write window kept short (24–48h); if measurable slowdown, async one of the two writes.
- **Race during cutover reads (S3).** If a read happens after the write switch but before all in-flight transactions complete, the new ORM might miss in-flight rows. Mitigation: standard "wait for in-flight to drain" deploy procedure; the dual-write window in S2 means *all* recent rows are in both.

## Known limitations at exit

- This plan does NOT build vanguard's own learning loop. That is a separate, larger workstream (`vanguard-execution-flywheel-stage-3-learning-loop`, TBC). Closing this plan unblocks that; it does not require it.
- The `outrider.api.*` test imports (~23 sites) are NOT in scope here. They close under Stage C.2 of the parent plan. Both must reach the AST allow-list's empty state before the dep drop in S4 can land.
