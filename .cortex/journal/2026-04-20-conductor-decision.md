# Conductor extracted as a fourth garage peer; Kimi becomes its v0.1 integration test

**Date:** 2026-04-20
**Type:** decision
**Trigger:** T1.1
**Cites:** doctrine/0004-conductor-as-fourth-peer, plans/conductor-bootstrap, journal/2026-04-20-litellm-evaluated-rejected, journal/2026-04-20-llm-provider-contract, journal/2026-04-20-side-quest-plans-extracted, doctrine/0001-why-autumn-garage-exists, doctrine/0003-llm-providers-compose-by-contract, plans/llm-provider-additions, plans/sentinel-codex-identifier-rename, plans/local-llm-provider-alignment

> The shape of the work clarified through three iterations today: (1) doctrine for shared provider contract (0003) addressed naming/UX drift but not implementation duplication; (2) extracting two pre-existing drifts as their own coordination plans surfaced that they all want the same fix; (3) the LiteLLM investigation ruled out adopting an existing OSS abstraction. Conductor — a small fourth garage peer that owns provider adapters and the manual/auto routing surface — is the right answer. Kimi becomes its v0.1 integration test because it forces the hardest case (HTTP API, real key handling, OpenAI-compat quirks) up front.

## Context

The day's progression:

1. **Morning:** user asked where Kimi should land in the garage. Survey found Touchstone's reviewer cascade and Sentinel's provider router both want it; planned to ship two parallel PRs.
2. **Doctrine 0003:** before writing PRs, we established the cross-tool contract for adding LLM providers (env var, base URL, default model, smoke test, setup UX). Wrote `integration/providers.md` as the mutable companion table.
3. **Two side-quest plans extracted:** writing the providers table surfaced pre-existing drift — Sentinel labels Codex as `openai`, and `local` means different things in Sentinel vs Touchstone. Promoted both to first-class coordination plans (`sentinel-codex-identifier-rename`, `local-llm-provider-alignment`).
4. **User raised the option of building our own LiteLLM-equivalent.** I pushed back; the conversation resolved to a deep investigation of LiteLLM first.
5. **LiteLLM investigation came back: don't adopt.** Supply-chain compromise (March 2026), API-key-first design that breaks Sentinel's "shell out to CLI" invariant, specific Moonshot bugs that would bite Sentinel on day one, and a 16 MB transitive-dep tree.
6. **User reframed the goal:** "a single way for the user to pick an LLM and assign it a job that is reused in Touchstone and Sentinel ... Sentinel has to do that no matter what. So naturally, we have to make that into a simple service." That reframe — Sentinel needs auto-routing regardless, so build it once as a peer rather than twice — is what justified extracting Conductor.

By that point, three plans (`llm-provider-additions`, `sentinel-codex-identifier-rename`, `local-llm-provider-alignment`) were all proposing implementations that would land in two repos each. Conductor collapses all three into a single implementation in one new repo, with the consuming tools migrating later.

## What we decided

1. **Build Conductor as a fourth garage peer.** Doctrine 0004 establishes it. Independent repo (`autumngarage/conductor`), brew-installable, same cadence-independence rules as the trio (Doctrine 0001).
2. **Conductor owns LLM provider adapters and the routing surface.** Five adapters at v0.1: `claude`, `codex`, `gemini`, `ollama`, `kimi`. Two modes: manual (`--with <id>`) and auto (`--auto`).
3. **Sentinel defaults to auto; Touchstone defaults to user's chosen reviewer.** Both can override.
4. **Composition stays shell-out.** Both consumers shell to `conductor call`; neither imports Conductor as a library. Preserves Sentinel's "no API keys in our process" invariant for every provider *except* Conductor (where the auth doctrine bends with documented exception, just for the Kimi-style API providers).
5. **Kimi is the v0.1 integration test case.** Build the harder shape (HTTP, key, OpenAI-compat quirks) first. claude/codex/gemini/ollama (easier shapes — shell-out and HTTP-without-auth) backfill after Kimi proves out.
6. **Three sibling plans collapse into Conductor's v0.1 scope.** They're not abandoned — they're absorbed. The codex/openai rename and the local/ollama alignment happen by virtue of Conductor establishing canonical identifiers; the Kimi rollout happens by Kimi being in Conductor.

Considered and rejected:

- **Adopt LiteLLM** — see `journal/2026-04-20-litellm-evaluated-rejected`. Supply-chain risk, auth doctrine break, specific Moonshot bugs, dependency weight.
- **Just build moonshot.py per tool** (the original plan) — works but locks in the per-tool-implementation pattern that the three side quests proved is bad for the trio. Adding the fourth provider is the right time to extract.
- **Make Sentinel's router public via a `sentinel route` subcommand** (Shape 2 from earlier in the conversation) — blurs Sentinel's identity, creates Touchstone→Sentinel directional dependency that Doctrine 0001 was designed to prevent.
- **Defer Conductor; ship Kimi the simple way first, refactor later** — tempting, but writing two parallel `moonshot.py` implementations now and throwing them away in a few weeks is wasteful; the user's reframe makes the extraction obvious.

## Consequences / action items

- [x] Doctrine 0004 written.
- [x] `plans/conductor-bootstrap.md` written.
- [ ] Three sibling plans annotated as "Superseded by Conductor v0.1" — pending.
- [ ] Create GitHub repo `autumngarage/conductor` — pending user confirmation. Public or private?
- [ ] Bootstrap repo via `touchstone new conductor --type python`.
- [ ] Build phase 1 of Conductor v0.1 (scaffolding + Kimi adapter + `conductor call --with kimi`).
- [ ] Update `integration/providers.md` to note Conductor as the canonical source of identifiers once v0.1 ships.
- [ ] Update `state.md` to reflect Conductor as the active workstream.
- [ ] Long-term: write Sentinel migration plan and Touchstone migration plan as separate coordination plans, after Conductor v0.1 ships.

## Reflection — what surprised me

The reframe from "where should Kimi go in the existing tools" to "Sentinel needs auto-routing anyway, so build it once" took five iterations to surface. Each iteration was useful — Doctrine 0003 (shared contract) is still load-bearing even with Conductor; the side-quest plans surfaced real drift that the new architecture addresses by construction; the LiteLLM investigation removed the easy-but-wrong path. But the journey from "add a config row" to "extract a fourth tool" is a real example of how the right scope emerges through dialogue, not specification.

The Cortex Protocol's bias toward append-only journaling captured the journey well: future readers can trace the reasoning by reading the four journal entries from today in order, and the doctrine + plan are the durable artifacts. None of the intermediate work was wasted.
