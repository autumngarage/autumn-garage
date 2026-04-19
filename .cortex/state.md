---
Generated: 2026-04-18T22:00:00-07:00
Generator: hand-authored (regeneration infrastructure ships in Cortex Phase C)
Sources:
  - doctrine/0001-why-autumn-garage-exists, doctrine/0002-interactive-by-default
  - plans/autumn-mail-dogfood (active), plans/sentinel-cortex-t16-integration (active)
  - journal/2026-04-18-{kickoff, plan-authoring-cortex-gaps, scaffold-friction-findings, stacked-merge-recovery, r1-regression-pass, r2-wizards-shipped, setup-reflection, r5-findings-from-fresh-scaffold, t16-shipped-and-audit-gap, first-cycle-attempt-findings}
  - autumn-garage-plan.md (v3)
  - TODOs.md
  - github.com/autumngarage/{touchstone@1.2.2, cortex@0.2.0, sentinel@0.3.4}
  - github.com/autumngarage/autumn-mail (live, commit 8d771f0 plus unpushed cycle artifacts)
Corpus: 2 Doctrine, 2 active Plans, 10 Journal entries
Omitted: []
Incomplete:
  - autumn-mail MVP features (0 of 14 Success Criteria met — container + tools wired, no user-facing features yet)
  - Sentinel→Cortex T1.7 (touchstone pre-merge → doctrine/candidate) and T1.9 (PR merged → journal/pr-merged) not yet operationalized. T1.6 shipped in sentinel v0.3.0+; T1.7/T1.9 follow the same pattern in future work.
  - Cortex `doctor --audit` T1.6 classification — Cortex Phase B first-slice deferred this; without it, T1.6 entries validate but aren't enforced in the audit layer
Conflicts-preserved: []
Spec: 0.3.1
---

# Project State — Autumn Garage

> Coordination repo for the Touchstone/Cortex/Sentinel trio + autumn-mail dogfood project. 2026-04-18 was the build-out day: scaffolded everything, shipped 7 tool releases across the trio, landed 4 rounds of improvements (R1–R4 plus T1.6 + R5 + V1/V2), attempted 3 real cycles on autumn-mail. Tools are measurably sharper. Autumn-mail has a scaffold and wired integrations but 0 user-facing features. Next session picks up at inspecting the verifier failure blocking the reviewer-approved gws-wrapper branch.

## P0 — Autumn-mail first cycle completion

Cycle 3 (2026-04-18 20:20–21:00) on sentinel v0.3.3 with swiftlint installed: reviewer PASSED the gws-wrapper proposal (first time); verifier still blocked at 1/2 checks. Branch left at `sentinel/wi-cycle-N-implement-core-gws-cli-wrapper-for-gmail-i-o` locally in the autumn-mail repo. $1.53/$5 spent.

Next step: `brew upgrade sentinel` to 0.3.4 (graceful missing-tool handling + better verifier math), inspect `.sentinel/verifications.jsonl` + `.sentinel/executions/` to see the actual failing check, decide between:
- **Salvage path:** check out the branch locally, apply codex's earlier findings (pipe-buffering deadlock, wire GmailClient into AutumnMailApp), open PR manually.
- **Retry path:** with v0.3.4 verifier logic, re-run `sentinel work --auto --budget $5 --coder-timeout 1200`. Should now verify correctly.

Tracked as task #21 in the task tracker. Full cycle log at `/tmp/sentinel-cycle3.log` + branches in autumn-mail local repo.

## P1 — Autumn-mail MVP remaining features

See `plans/autumn-mail-dogfood.md` for full plan. 14 Success Criteria. 0 met. After gws-wrapper lands:
- MLX Swift local-LLM wiring (model download + inference)
- SwiftUI views: Inbox (triage), Message (read + Draft reply), Composer (editable + Send)
- End-to-end flow: triage → read → draft-via-local-LLM → send
- OAuth setup docs for `gws auth setup`

Each can be a separate `sentinel work` cycle or combined into a larger one.

## P2 — Cortex Phase C gap blocks T1.6 full enforcement

Cortex v0.2.0's `doctor --audit` classifies T1.1/T1.5/T1.8/T1.9 but NOT T1.6. This means autumn-mail accumulates valid T1.6 journal entries but `cortex doctor --audit` doesn't yet confirm they match fires. Filed as a cortex TODO in `TODOs.md` under the Cortex section. Can ship independently as a Cortex Phase C extension.

## P3 — Tool-side improvements surfaced today, not yet addressed

Findings from 2026-04-18 cycles that became new TODOs:
- **Sentinel:** role-timeout generalization (Monitor/Researcher/Planner/Reviewer still use legacy `scan.provider_timeout_sec`; only Coder got per-role in C5). Parallel-agent coordination hazard (agents on same repo step on each other). C6 lens determinism drift.
- **Touchstone:** `scripts/open-pr.sh --base <branch>` for stacked PRs (found during R2/R3/R4 coordination).
- **Coordination playbook:** prefer bundled rounds over stacked PRs from the start.

All in `TODOs.md`.

---

## Installed tool versions (2026-04-18 end-of-day)

- **Touchstone 1.2.2** — [swift scaffold + interactive wizard + sibling detection + R5 ordering + R5.3 shellcheck CI + setup.sh per-profile dev-tools]
- **Cortex 0.2.0** — [plans template + .cortex/README.md + init interactive + sibling detection + hand-authored-placeholder stubs]
- **Sentinel 0.3.4** — [.sentinel/.gitignore + init wizard + reviewer=codex default + sibling detection + T1.6 Cortex journal writes + coder timeout config + built-in registry + rejection memory + R5.2 gitignore fix + graceful missing-tool verifier + verifications.jsonl audit]

## Shipped releases today

touchstone: 1.1.0 → 1.2.0 → 1.2.1 → 1.2.2 · cortex: 0.1.0 → 0.2.0 · sentinel: 0.2.0 → 0.3.0 → 0.3.1 → 0.3.2 → 0.3.3 → 0.3.4. ~20 PRs total across the three repos, all via parallel agent dispatch + Codex auto-merge-review.

## Open decisions

D1 Principles/Doctrine boundary (deferred) · D2 Write contract (settled: CLI-primary via Phase D) · D3 Umbrella branding (settled) · D4 Tap structure (settled) · D5 Compat surfacing (settled: `<tool> version --verbose` prints CIC range) · D6 Review-hook → journal opt-in default (deferred; R4 registry visibility partially addresses) · D7 Nested `.cortex/` in monorepos (deferred) · D8 Deferred-write longevity (settled).
