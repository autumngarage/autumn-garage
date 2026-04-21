# Conductor v0.1.0 shipped — fourth garage peer is live

**Date:** 2026-04-21
**Type:** decision
**Trigger:** T1.3 (plan transition) + T1.9 (PRs merged)
**Cites:** doctrine/0004-conductor-as-fourth-peer, plans/conductor-bootstrap, journal/2026-04-20-conductor-decision, journal/2026-04-20-litellm-evaluated-rejected, journal/2026-04-21-kimi-via-cloudflare, integration/providers.md

> Tagged `v0.1.0` on `autumngarage/conductor`. The fourth garage peer is shipping. Five providers (kimi, claude, codex, gemini, ollama) behind one CLI: `conductor call --with <id> | --auto`, with `list` / `smoke` / `doctor` discovery and an `init` wizard that stores creds in macOS Keychain or direnv. 89 mocked tests. Four PRs merged across 2026-04-20/21. The three side-quest plans (`llm-provider-additions`, `sentinel-codex-identifier-rename`, `local-llm-provider-alignment`) are superseded by the Conductor bootstrap — they collapse into its v0.1 scope by construction.

## What shipped

- **Repo:** [autumngarage/conductor](https://github.com/autumngarage/conductor).
- **Release:** [v0.1.0](https://github.com/autumngarage/conductor/releases/tag/v0.1.0).
- **Four PRs merged:**
  - #1 scaffold + Kimi adapter via Cloudflare Workers AI
  - #2 claude/codex/gemini/ollama adapters + registry
  - #3 auto-mode router with rule-based tag scoring
  - #4 list/smoke/doctor commands + `conductor init` wizard + credentials resolver
- **Tests:** 89 passing (mocked httpx + mocked subprocess; no live network in CI by default). Live-smoke job gated on `workflow_dispatch` with the org-level Cloudflare secrets.
- **Org-level GitHub secrets:** `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID` set on `autumngarage` org with `visibility: all`, so any garage repo picks them up.

## What the day taught

1. **The LiteLLM investigation was worth the ~3 minutes it took.** The March 2026 PyPI supply-chain compromise, the API-key-first auth mismatch with Sentinel's "never touches keys" invariant, and the specific reasoning_content stripping bug that would've hit Kimi thinking models combined to make the "just adopt LiteLLM" path obviously wrong *before* we paid to find out. Journaled in `2026-04-20-litellm-evaluated-rejected.md`.
2. **Cloudflare's Day 0 Kimi hosting landed *one day* before we needed it.** The original plan routed through `api.moonshot.ai` directly; Cloudflare's 2026-04-20 changelog changed the right answer. The pivot cost one commit because the shape hadn't calcified. Journal: `2026-04-21-kimi-via-cloudflare.md`.
3. **Three converging plans collapse cleanly when you extract the real abstraction.** Adding Kimi, renaming `openai`→`codex` in Sentinel, and reconciling the `local` semantics across Touchstone/Sentinel all wanted the same thing: one place that owns provider adapters. That place is Conductor. All three plans are now `Status: superseded`.
4. **A parallel background process was editing the repo mid-session.** A half-built Mistral adapter (against the wrong `mistralai` SDK version) kept reappearing. Backed out the broken code; left `mistral` as a reserved slot in `DEFAULT_PRIORITY` so the real adapter can slot in later without a priority reshuffle. Surface-level unresolved: was this another Claude session, a Sentinel cycle, or a local linter? Worth investigating before the next build session.

## State after v0.1

- **Trio + Conductor** — all four garage tools have a release: Touchstone 1.2.3, Cortex 0.2.3, Sentinel 0.3.4, Conductor 0.1.0.
- **Provider identifiers are canonical.** `claude`, `codex`, `gemini`, `kimi`, `ollama`. Sentinel's internal `openai` label is now pure technical debt — cleared when Sentinel migrates to shell out to Conductor (future plan).
- **`integration/providers.md`** is the single source of truth for env var / base URL / default model / smoke test across all tools.

## Consequences / action items

- [x] Tag v0.1.0 + GitHub release.
- [x] Four bootstrap PRs merged.
- [ ] Commit and push this accumulated autumn-garage coordination work (doctrine 0003/0004, bootstrap plan, journals, providers.md, superseded plan annotations, state.md update).
- [ ] Future: write `plans/sentinel-conductor-migration.md` — replace Sentinel's `providers/{claude,openai,gemini,local}.py` with shell-outs to `conductor call`. This is where the codex/openai rename and the local/ollama alignment actually land (Sentinel stops implementing those adapters entirely).
- [ ] Future: write `plans/touchstone-conductor-migration.md` — extend the reviewer cascade to support `auto` as a valid entry resolving via `conductor call --auto`.
- [ ] Future: write `plans/cortex-phase-c-conductor-wiring.md` — Cortex synthesis backends call Conductor instead of shelling to `claude -p` directly.
- [ ] Future: brew tap (`autumngarage/homebrew-conductor`) + homebrew formula → `brew install` path.
- [ ] User rotation reminder — the Cloudflare token created mid-session is in the session log / screenshot / paste buffer. Rotate and re-run `gh secret set` when convenient.
- [ ] Investigate the parallel-editing process (Mistral files appearing). If it's a Sentinel cycle on the conductor repo, want it paused until Sentinel is Conductor-aware.

## Reflection

The v0.1 landed in a single session across 2026-04-20 → 2026-04-21, roughly 4 PRs and 1,500 lines of Python + tests. The interesting pattern: every pivot (Doctrine 0003, LiteLLM rejection, Cloudflare routing, Conductor extraction) emerged from explicit pauses to ask "is the plan still the right plan?" before committing. The runtime code is small precisely because that planning overhead was real. Future Conductor work should preserve the posture.
