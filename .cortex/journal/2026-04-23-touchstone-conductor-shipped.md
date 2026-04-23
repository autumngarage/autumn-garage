# Touchstone × Conductor SHIPPED — both PRs merged, both releases tagged

**Date:** 2026-04-23
**Type:** plan-transition
**Trigger:** T1.3 (Plan status change: active → shipped) + T1.9 (PR merged to default branch) ×3
**Cites:** plans/touchstone-conductor-integration, journal/2026-04-21-stage-1-2-delivered, journal/2026-04-23-touchstone-ux-pass

> Wrap-up of the multi-day Touchstone × Conductor integration. Both PRs (touchstone#54 squash-merged as 2ab5600, conductor#5 squash-merged as 0200332) landed within minutes of each other; both releases tagged + GitHub releases created; touchstone brew formula bumped; autumn-mail migration PR opened (autumn-mail#7).

## Context

PRs had been open since 2026-04-21 evening. 2026-04-22/23 was spent on dogfood-driven UX hardening (gaps A–G from a user-state scenario walk) plus codex-JSON / session-id research-driven follow-ups on conductor. Today's session added the merge + release sequence.

## What we decided

- **Squash-merge both PRs.** Single commit per PR on main keeps history scannable; the rich PR-description bodies (auto-included as squash-commit bodies) preserve the per-feature breakdown.
- **Tag conductor as v0.2.1, not v0.2.0.** The v0.2.1 work (session_id + `--resume`, codex JSONL session.created event capture, ollama model-pulled doctor warning) materially expands what v0.2 promised; calling it 0.2.0 would understate the surface.
- **Skip conductor brew tap creation.** `autumngarage/homebrew-conductor` doesn't exist yet; install via `uv tool install --from git+https://github.com/autumngarage/conductor@v0.2.1 conductor` until the tap is created (separate task). The conductor README and touchstone error messages already reference the eventual tap path; downstream users will get a friendly install hint pointing them to it.
- **Apply migration to autumn-mail.** Ran `touchstone migrate-review-config` against autumn-mail's `.codex-review.toml`; opened PR autumn-mail#7 with the result. This is the canonical real-world validation of the migration command (autumn-mail had `[review].reviewers = ["codex"]` + `[review.assist]` — exactly the legacy markers the migration is built for).

## Consequences / action items

- [x] touchstone v2.0.0 tagged, GitHub release created, brew formula updated, `brew upgrade` validated locally
- [x] conductor v0.2.1 tagged, GitHub release created, install via uv validated locally (`conductor --version` reports 0.2.1)
- [x] autumn-mail#7 opened with the migrated `.codex-review.toml`
- [x] `state.md` updated to reflect shipped state and current installed versions
- [x] Memory entry added: "AI reviewer is the gate, not human review" (workflow norm)
- [ ] **Create `autumngarage/homebrew-conductor` tap** with the v0.2.1 formula. Until then, all install instructions in conductor's README that say `brew install autumngarage/conductor/conductor` are aspirational. Reasonable next-session task.
- [ ] **Sentinel migration (Stage 5)** is the only Touchstone × Conductor plan piece still open. Still blocked on conductor v0.3 (HTTP tool-use loop for kimi/ollama). Separate plan — file when v0.3 work kicks off.
- [ ] Run touchstone update on autumn-mail (and other registered projects) to pick up the new principles/git-workflow.md and the synced scripts/codex-review.sh. Deferred for autumn-mail because it has uncommitted .sentinel/ work that touchstone update refuses to step over; user can run it after committing or stashing those.

## Surface change downstream users will notice on first push after upgrade

- Migration warnings WERE firing on every push if config was 1.x shape. Two options to silence: (a) `touchstone migrate-review-config` (recommended; clean rewrite), (b) ignore the warnings or set `CODEX_REVIEW_SUPPRESS_LEGACY_WARNINGS=1` (the auto-translation works either way).
- Cache version bumped v2 → v3. First push after upgrade will miss the cache (one full review re-runs); subsequent pushes hit the new cache as normal.
- Pre-push transcript now includes a Conductor route-log line (provider, cost, tokens, duration) inline. Users see *which* provider answered every push.

## What's *not* covered by this wrap-up

- Live end-to-end push of any of these changes through autumn-mail or sigint with a real conductor call. Mocks and scratch-repo dogfooding only. The next autumn-mail push after merging autumn-mail#7 will be the first real-world validation.
- Conductor brew tap as noted above.
- A doctrine entry on shell cache-key composition with `${VAR:-}` discipline (the latent bug surfaced during the F task work). Worth writing; deferred.
