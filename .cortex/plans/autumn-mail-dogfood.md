---
Status: active
Written: 2026-04-18
Author: human
Goal-hash: 177fab6a
Updated-by:
  - 2026-04-18T13:30 human (created; kickoff plan for the Autumn Mail dogfood target)
Cites: doctrine/0001-why-autumn-garage-exists, https://github.com/autumngarage/autumn-mail, https://github.com/googleworkspace/cli
---

# Autumn Mail — the real dogfood target

> Run the full Autumn Garage stack (Touchstone + Cortex + Sentinel) against a real SwiftUI mail client that reads and replies to Gmail using a local LLM. Measure where the tools help, where they get in the way, and where the integration contract is missing pieces. Every finding becomes a journal entry here.

## Why (grounding)

Dogfooding Cortex on its own repo is necessary but circular — Cortex is its own domain. The tools are meant to be useful *for other projects*. A real third-party-shaped project is the honest test: does `touchstone new` scaffold Swift usefully? Does Cortex's Protocol match what a SwiftUI/local-LLM project actually needs? Can Sentinel produce valid cycles against a codebase whose stack it has never seen before?

Autumn Mail is deliberately chosen to stress the tools along several axes:

- **Non-Python stack (Swift / SwiftUI).** Forces Touchstone's swift profile out of theory into practice. Forces Sentinel's providers to evaluate Swift diffs.
- **External CLI dependency (`gws`, Google Workspace CLI).** Dogfoods the "wrap a CLI" pattern. Real OAuth, real rate limits, real "experimental, expect breaking changes" churn.
- **Local-LLM-only (MLX Swift).** Exercises Sentinel's Ollama/local-provider plumbing against a non-Python integration.
- **Small enough to ship real features weekly.** Real PRs for Touchstone's review gate. Real cycles for Sentinel to assess/plan/execute. Real journal entries for Cortex to ingest.

Grounds-in: `autumn-garage/.cortex/doctrine/0001-why-autumn-garage-exists`.

## Approach

- Scaffold with `touchstone new autumn-mail --type swift --reviewer codex`.
- `cortex init` inside the scaffold.
- Drop a `plans/mvp.md` into `autumn-mail/.cortex/plans/` describing the MVP (see Success Criteria below).
- Write an opening journal entry capturing the vision and the open questions.
- Kick off `sentinel work` and observe. Capture every surprise as a journal entry in *this* repo (autumn-garage) — the coordination record.

## Success Criteria

1. `brew install autumngarage/touchstone/touchstone autumngarage/cortex/cortex autumngarage/sentinel/sentinel` on a clean machine produces a working garage.
2. `touchstone new autumn-mail --type swift` scaffolds a buildable SwiftUI app (`xcodebuild` succeeds) with `.touchstone-config`, `.pre-commit-config.yaml`, `CLAUDE.md`, `AGENTS.md`, principles, and scripts in place.
3. `cortex init` runs clean in the scaffolded project; `cortex doctor` exits 0.
4. The app reads at least the 10 most recent unread messages via `gws +triage` and displays them in a SwiftUI list.
5. The app drafts a reply using MLX Swift with a user-specified local model (`llama-3.1-8b-instruct` or similar); the draft opens in a SwiftUI composer pre-filled.
6. The user can send the draft via `gws +reply`.
7. `sentinel work` runs at least one clean cycle end-to-end: assesses the repo via its Swift/SwiftUI lenses, plans, executes against a feature branch, and gets reviewer sign-off.
8. At least one cycle produces a valid Cortex journal entry (Type: `cycle-complete` or `decision`) *by hand* (since `cortex journal append` ships in Phase D). Entry validates in `cortex doctor`.
9. At least three coordination-journal entries in *this* repo (`autumn-garage`) capture dogfood findings — gaps, surprises, missing contract pieces.

## Work items

- [ ] Create `autumngarage/autumn-mail` on GitHub (private).
- [ ] `touchstone new autumn-mail --type swift --reviewer codex` — scaffold the app.
- [ ] `cortex init` in the scaffold; edit CLAUDE.md to add `@.cortex/protocol.md` + `@.cortex/state.md` imports.
- [ ] Seed `autumn-mail/.cortex/plans/mvp.md` with the starter prompt (the "what to build") shaped as Success Criteria 4–6 above.
- [ ] Seed `autumn-mail/.cortex/journal/2026-04-18-vision.md` with an opening decision entry capturing direction + open questions.
- [ ] Install `gws` via `brew install googleworkspace/tap/cli` (or fallback to npm if tap not present); verify on PATH. OAuth setup deferred to first real run.
- [ ] Add MLX Swift as a Swift Package dependency; wire a minimal "summarize inbox" prompt as proof-of-life.
- [ ] First `sentinel work` cycle against autumn-mail; capture findings as a journal entry here.

## Follow-ups (deferred)

- Full Gmail threading, attachments, search, multi-account — deferred until MVP ships and works for one account.
- iOS / iPadOS targets — macOS only for MVP.
- Model management UI (swap MLX models at runtime) — later.

## Known limitations at exit

- MVP will likely hit gaps in Cortex's `cortex journal append` (not yet shipped — Phase D). Hand-authored journal entries are the documented workaround.
- Sentinel's Swift lenses may be thin compared to Python; this is itself a finding and a journal entry.
- `gws` is explicitly experimental; expect breaking changes. The Swift shell-out layer must tolerate a rough CLI surface.
