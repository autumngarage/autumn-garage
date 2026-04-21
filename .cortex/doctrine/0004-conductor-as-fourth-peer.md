# 0004 — Conductor joins the garage as a fourth peer; LLM provider adapters and routing live there

> Autumn Garage grows from a trio to a quartet. **Conductor** is a small CLI that owns LLM provider adapters and the user-facing "pick an LLM, give it a job" surface. Both Sentinel and Touchstone call `conductor` as a peer (shell out, same as they call `claude` / `codex` / `gemini` / `gh` today) — they do not implement provider adapters themselves once Conductor v1 ships. Conductor exposes two modes: **manual** (`--with <provider>`) and **auto** (`--auto` — Conductor's own router picks). Sentinel defaults to `--auto`; Touchstone defaults to the user's chosen reviewer with `--auto` available as an override.

**Status:** Accepted
**Date:** 2026-04-20
**Promoted-from:** journal/2026-04-20-conductor-decision
**Load-priority:** always
**Supersedes (in part):** doctrine/0001 (extends the trio framing to a quartet — see § Relationship to 0001)

## Context

Three pressures converged on the same answer:

1. **Sentinel needs task-aware LLM routing regardless.** Picking the right provider per role (Monitor cheap, Coder strong, Researcher long-context) is part of Sentinel's product. The router exists in `src/sentinel/providers/router.py` today.

2. **Touchstone needs a reviewer cascade with the same conventions.** Today the cascade is hardcoded in bash. Adding Kimi means another branch in the case statement; adding "auto-pick-the-cheapest-that-can-handle-this-diff" means duplicating Sentinel's routing logic in bash. Wrong direction.

3. **Three open coordination plans were converging on the same solution from different angles.** `plans/llm-provider-additions.md` (Kimi rollout), `plans/sentinel-codex-identifier-rename.md` (cross-tool identifier consistency), and `plans/local-llm-provider-alignment.md` (semantic gap on `local`) all exist because *each tool implements provider adapters independently and they drift.* Doctrine 0003 (shared contract, not shared code) is the right rule for *contracts*; it doesn't fix the underlying duplication of *implementation*.

The investigation into LiteLLM (journal/2026-04-20-litellm-evaluated-rejected — being authored alongside this doctrine) ruled out adopting an existing OSS abstraction: supply-chain risk (March 2026 PyPI compromise), API-key-first design that breaks Sentinel's "user authenticates with their CLI" invariant for every provider, specific Moonshot bugs that would bite Sentinel on day one, and a 16 MB transitive-dep tree pinning `openai==2.24.0`. Building our own thin abstraction — narrowly scoped to what the garage actually needs — is the better trade.

The user explicitly framed it: *"a single way for the user to pick an LLM and assign it a job that is reused in Touchstone and Sentinel ... Sentinel has to do that no matter what. So naturally, we have to make that into a simple service."*

## Decision

**1. Conductor exists as a fourth peer in the garage.** Independent repo `autumngarage/conductor`. Independent release cadence. Brew-installable: `brew install autumngarage/conductor/conductor`. Same cadence-independence rules as the trio (Doctrine 0001).

**2. Conductor owns LLM provider adapters.** Five at v0.1: `claude` (shell out to Claude Code CLI), `codex` (shell out to Codex CLI), `gemini` (shell out to Gemini CLI), `ollama` (HTTP via httpx), `kimi` (HTTP via httpx — the first API-key-touching adapter, included by design as the v0.1 integration test case). Future providers add as new adapters in Conductor; consuming tools do not need PRs.

**3. Conductor's CLI surface is small.** v0.1 ships:

| Command | Purpose |
|---|---|
| `conductor call --task "..." --with <id>` | Manual mode — call a specific provider |
| `conductor call --task "..." --auto` | Auto mode — Conductor's router picks |
| `conductor list` | Show available providers + their detected status |
| `conductor smoke <id>` | Run the provider's smoke test |
| `conductor init` | Interactive setup (Doctrine 0002 compliance) |
| `conductor doctor` | Diagnostic: what's installed, what's configured, what env vars are set |

`conductor call` reads the task from `--task` or stdin. Returns the model's response on stdout. JSON output via `--json` for programmatic consumers (Sentinel especially).

**4. Two modes, not five.** Manual = pick one provider explicitly. Auto = let Conductor's router pick. No tier system, no "smart fallback chains" in v0.1 — just manual or auto.

**5. Auto-mode v0.1 is rule-based.** Tasks carry tags (`code-review`, `bulk-summarize`, `long-context`, `cheap`, `strong-reasoning`). Providers carry capability tags. The router matches. ~50 lines of code; no LLM-meta-routing yet (deferred to v0.2 or later if/when the rule-based version proves insufficient).

**6. Defaults differ per consuming tool:**

- **Sentinel** defaults to `--auto`. User can override per-role in `.sentinel/config.toml` with `provider = "kimi"` or similar. The override is passed to `conductor call --with kimi`.
- **Touchstone** defaults to the user's configured reviewer (today's behavior — `reviewers = ["codex", "claude", ...]` cascade). User can switch any cascade entry to `auto` to get Conductor's router pick.

**7. Conductor is the canonical owner of provider identifiers.** `claude`, `codex`, `gemini`, `ollama`, `local-command`, `kimi`. These names are defined in Conductor's source. `autumn-garage/integration/providers.md` continues to document them but Conductor is now the de-facto source of truth (bumps to providers.md follow Conductor releases).

**8. Composition by file/CLI contract, not shared code.** Same rule as Doctrine 0001: Sentinel and Touchstone do not import Conductor as a Python library. They shell out to `conductor` and parse its stdout/JSON. This preserves Sentinel's "shell out to CLIs" invariant (which the journal/2026-04-20-litellm-evaluated-rejected analysis identified as load-bearing) and keeps each tool's release independent of Conductor's.

Inside scope for Conductor: provider adapters, routing, manual/auto mode CLI, setup wizard, smoke tests, the providers config schema.

Outside scope (at least for v0.1): cost tracking (rely on each provider's own `usage` fields surfaced in JSON output), streaming (non-streaming requests only), caching (rely on upstream provider caching), gateway features (no proxy server, no virtual keys, no RBAC, no admin UI), LLM-based meta-routing for auto mode, multi-provider parallel calls, fallback chains, retry logic beyond the trivial.

## Consequences

- **What becomes easier:** adding a new provider — one PR to Conductor, both tools benefit. Cross-tool identifier drift becomes structurally impossible (one source of truth). The "auto-pick the right LLM for this job" capability becomes available to any garage tool with one shell-out. Doctrine 0003's contract is now satisfied by construction (consuming tools can't drift because they don't implement adapters).
- **What becomes harder:** four-tool coordination (was three). Brew tap, release flow, README, init wizard, tests for a brand-new tool. Per-tool migration: Sentinel and Touchstone both eventually replace their internal provider code with `conductor call` shell-outs (a real refactor, not an additive change).
- **What this forecloses:** per-tool provider implementations diverging again; in-process API SDK adoption in Sentinel or Touchstone (both stay shell-out-only); LiteLLM/OpenRouter as runtime dependencies anywhere in the garage.
- **Side quests resolved:** `plans/llm-provider-additions.md`, `plans/sentinel-codex-identifier-rename.md`, and `plans/local-llm-provider-alignment.md` all collapse into Conductor's v0.1 scope (where the canonical identifiers are established once, in one place, and the consuming tools just adopt them via the shell-out boundary). Each plan gets a "superseded by conductor-bootstrap" note.

## Relationship to Doctrine 0001

Doctrine 0001 ("Autumn Garage is a coordination repo for the trio, not a monorepo") named the trio: Touchstone (the ground), Cortex (the spine), Sentinel (the hands). Adding Conductor doesn't invalidate that framing — the *coordination repo's* purpose (cross-tool decisions, plans, journal) is unchanged. But the trio is now a quartet, and the metaphor extends: **Conductor is the voice** — the one that picks who plays and tells them what to do.

Doctrine 0001 is amended to read "trio or larger" wherever it says "trio," and the install block in the autumn-garage README adds the Conductor brew line. Doctrine 0001's falsification clause ("if coordination work fits comfortably inside Cortex") still applies symmetrically: if Conductor's work fits comfortably inside Sentinel (because Touchstone never actually adopts the auto mode), Conductor folds back into Sentinel as a `sentinel route` subcommand.

This is recorded as an amendment, not a supersede. Doctrine 0001 stays in force; Doctrine 0004 extends it.

---

Falsification condition for 0004: if six months after Conductor v0.1 ships, Touchstone is still calling `claude` / `codex` / `gemini` directly (not `conductor`), then Conductor failed to be useful as a peer to Touchstone and should fold into Sentinel as `sentinel route`. Conductor's value depends on at least two consumers; one consumer makes it overhead.
