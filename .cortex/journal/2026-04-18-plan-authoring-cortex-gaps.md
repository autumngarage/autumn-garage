# Hand-authoring a Cortex Plan surfaced three gaps cortex v0.1.0 should fill

**Date:** 2026-04-18
**Type:** decision
**Trigger:** T2.2
**Cites:** plans/autumn-mail-dogfood, ../../cortex

> First manual pass at writing a Cortex Plan inside `autumn-garage/.cortex/` produced three `cortex doctor` errors on the first run. Each error maps to a known-deferred item in Cortex's own plans/phase-c-first-synthesis and plans/phase-d (authoring helpers). Logging as the first real dogfood finding.

## Context

Wrote `plans/autumn-mail-dogfood.md` by hand using the candidate.md template shape as a reference. Ran `cortex doctor`. Three errors came back:

1. **Missing `Updated-by:` writer history** (SPEC § 3.4). The candidate.md template lives under doctrine/, not plans/. No plans/ template was scaffolded by `cortex init`.
2. **Missing required `## Success Criteria` section.** My heading was `## Success Criteria (MVP)` — the check is exact-match, not prefix. I edited to pass; open question whether prefix-match is the spec's intent.
3. **`Goal-hash: (computed by cortex doctor)` placeholder rejected.** Expected: the author pre-computes the hash from the H1 title per SPEC § 4.9 normalization. Doctor printed the expected hash in the error, which I copy-pasted. Round-trip is awkward.

## What we decided

Fixed all three locally and moved on. The real decisions are for Cortex:

- **Ship a `plans/*.md` template** alongside `templates/doctrine/candidate.md`. Same structure, with `Updated-by:` and `## Success Criteria` pre-filled so first-run authors don't hit these errors.
- **`cortex plan spawn <slug>`** (Phase D) should pre-fill Goal-hash from the title, pre-fill `Updated-by:` with the current author+timestamp, and include the canonical section headers. This converts three errors into zero for the hand-authoring path.
- **Consider prefix-matching section headers** in the doctor — `## Success Criteria (MVP)` vs `## Success Criteria` is the kind of micro-friction that will get hit repeatedly by humans. Counter-argument: exact-match is simpler and the template already uses the exact form. Log the tension; don't decide today.

## Consequences / action items

- [ ] File an issue on `autumngarage/cortex`: ship a plans template + enhance `cortex init` to scaffold plan stubs.
- [ ] File an issue on `autumngarage/cortex`: decide prefix-match vs exact-match for section-header validation (lean: keep exact-match, fix templates to be canonical).
- [ ] When Phase D lands, migrate `plans/autumn-mail-dogfood.md` to a spawned-then-edited form as a regression check.

## Open question

Is `Goal-hash` something a human should ever write by hand, or is it strictly machine-computed? SPEC § 4.9 says normalization is deterministic from the title, so the field is redundant with the title — but the redundancy is the integrity check. Minor, but worth clarifying in docs.
