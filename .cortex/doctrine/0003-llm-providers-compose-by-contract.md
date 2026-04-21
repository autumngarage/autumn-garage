# 0003 — LLM providers compose across the trio by shared contract, not shared code

> Each Autumn Garage tool that calls an LLM (today: Touchstone's review hook, Sentinel's role router; tomorrow: Cortex's Phase C synthesis commands) implements its own provider adapters. Tools do not share a provider library. Instead, they share a documented **provider contract** — env var names, base URL conventions, model ID conventions, smoke-test conventions — that all garage tools satisfy identically. Adding a new provider means writing a thin per-tool adapter against the same contract.

**Status:** Accepted
**Date:** 2026-04-20
**Promoted-from:** journal/2026-04-20-llm-provider-contract
**Load-priority:** always

## Context

The trio is about to add Kimi (Moonshot AI) as a callable LLM. Touchstone wants it as a reviewer-cascade entry alongside `codex`/`claude`/`gemini`. Sentinel wants it as a `providers/moonshot.py` peer alongside `claude.py`/`openai.py`/`gemini.py`/`local.py`. Cortex's Phase C synthesis will eventually want it too.

The naive shape would be a shared `garage-providers` library that both tools import. That is exactly what Cortex Doctrine 0002 forbids ("the three tools compose through file contracts, never through a shared library") and what Autumn Garage Doctrine 0001 reinforces ("no code dependencies between tools"). The trio's release-cadence-independence depends on this rule.

But the symptom is real: without coordination, two tools will drift on env var names, default model IDs, base URLs, and setup UX. A user who configures Kimi for Touchstone and then for Sentinel should not be re-learning the conventions. The integration contract is the surface that prevents that drift without reintroducing code coupling.

The Kimi specifics that surfaced this question:

- API base URL: `https://api.moonshot.ai/v1`
- Auth: `Authorization: Bearer $MOONSHOT_API_KEY` (Moonshot's documented env var)
- OpenAI SDK drop-in (use `openai` client with `base_url` override)
- Default model: `kimi-k2.6` (256k context, multimodal, tool calling)
- Official `kimi` CLI exists but is interactive-only — not suitable for non-interactive scripting (no `-p` flag, no stdin, no JSON output). Treat as not-present for tool integration; call the OpenAI-compatible endpoint instead.

These specifics belong in the rollout plan, not the doctrine. The doctrine names the *rule*; the plan applies it to a specific provider.

## Decision

**The contract.** Every LLM provider added to a garage tool MUST satisfy the following shared contract:

1. **Provider identity.** A short, lowercase, single-word identifier (`kimi`, `claude`, `codex`, `gemini`, `local`). The same identifier is used in every tool's config (Touchstone's reviewer cascade, Sentinel's `providers/<id>.py` filename and `ProviderName` enum, Cortex's synthesis routing).
2. **Auth.** A single environment variable using the provider's own documented convention when one exists (`MOONSHOT_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`). Tools do not invent new env var names; they read what the provider's own SDK reads. For providers without a canonical env var (local LLMs), tools agree on a single garage-wide name (`LOCAL_LLM_*`).
3. **Endpoint shape.** Whether a provider is invoked via (a) OpenAI-compatible HTTP, (b) provider-native SDK, or (c) shelling out to a CLI is captured per-provider in the contract. Both consuming tools (Touchstone, Sentinel) implement the SAME shape for the SAME provider. Mixing — e.g., Touchstone shells out to `kimi` CLI while Sentinel uses HTTP — is forbidden because the user-visible failure modes diverge.
4. **Default model.** A single recommended model ID per provider, agreed across tools. Users may override per-tool. Tools document the default in their own README; the contract pins which ID is "default for the garage" so they don't drift.
5. **Smoke test.** A single documented one-liner that proves auth + endpoint work (e.g., a `curl` against `/v1/models` for OpenAI-compatible endpoints). Each tool's setup flow runs the same smoke test on first configure. This is the cheapest gate against "I configured it but it doesn't work."
6. **Setup UX.** Per Doctrine 0002, first-time provider setup is interactive on a TTY. The wizard prompts for the API key, writes it to the user's shell rc / config in a documented way, runs the smoke test, and prints the equivalent flag-form. Each tool implements its own wizard; they share the prompts, defaults, and smoke-test command via the contract.

**Where the contract lives.** This file (the doctrine) names the rule. The *current contents* of the contract — the table of providers and their concrete values for fields 1–5 — live in `autumn-garage/integration/providers.md` (a non-doctrine reference document). Doctrine entries are immutable-with-supersede; the providers table changes whenever a new provider is added or a default model is bumped, and we don't want a supersede chain for every model bump.

**Where implementation lives.** Per-tool, in the tool's own repo. Touchstone gets a new entry in its reviewer-cascade table and the corresponding shell wrapper in `hooks/`. Sentinel gets a new `providers/<id>.py` and an enum entry. Cortex (Phase C) gets a new synthesis backend. Each is a self-contained PR in its own repo. Coordination across PRs happens via a single `plans/` entry here in the garage repo (e.g., `plans/llm-provider-additions.md`).

**Inside scope:** every LLM call any garage tool makes, including review, planning, scanning, synthesis, summarization, classification.
**Outside scope:** non-LLM third-party APIs (gh, git, gws), local subprocess tools (swiftlint, ruff). Those have their own integration patterns and don't need this contract.

## Consequences

- **What becomes easier:** adding a new provider to all three tools — the contract names the five things to nail down once; each tool implements against them in parallel. Users see consistent env var names, prompts, and defaults across tools.
- **What becomes harder:** the providers reference document (`integration/providers.md`) becomes a coordination chokepoint — every new provider needs an entry there before per-tool PRs can land cleanly. Acceptable because providers are added rarely (every few months at most).
- **What this forecloses:** a shared `garage-providers` Python/Bash library; per-tool divergence on env var names; per-tool divergence on default model IDs; per-tool reinvention of "how do I configure provider X" wizards.

---

Falsification condition: if two tools that satisfy this contract still produce divergent user experiences for the same provider — different env var names accepted, different smoke-test outcomes, different "configured but doesn't work" failure modes — then the contract is too thin and either expands or is replaced by a shared adapter library (relaxing Cortex Doctrine 0002 for this concern).
