# LLM providers — reference table

> The shared, mutable companion to [Doctrine 0003 — LLM providers compose by shared contract, not shared code](../.cortex/doctrine/0003-llm-providers-compose-by-contract.md). This document is the **single source of truth** for the cross-tool contract: identifier, env var, endpoint shape, base URL, default model, and smoke test for every LLM provider any Autumn Garage tool calls.
>
> Per-tool implementations (Touchstone reviewer cascade, Sentinel `providers/<id>.py`, future Cortex synthesis backends) MUST satisfy these rows verbatim. PRs that add or modify provider support cite this file in their description.

**Owner:** autumn-garage (this repo).
**Updated by:** human edit + journal entry. Mutable — providers and defaults change as the menu grows or upstream APIs shift. Not under doctrine immutability.
**Last update:** 2026-04-21 — `kimi` row updated to route via Cloudflare Workers AI (Day 0 support for Kimi K2.6 landed on Cloudflare 2026-04-20). Previous: 2026-04-20 — initial table seeded.

---

## Table

| Identifier | Env var | Endpoint shape | Base URL / CLI | Default model | Smoke test |
|---|---|---|---|---|---|
| `kimi` | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` | OpenAI-compatible HTTP (Cloudflare Workers AI) | `https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/ai/v1` | `@cf/moonshotai/kimi-k2.6` | `curl -sf -X POST -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" -H "Content-Type: application/json" -d '{"model":"@cf/moonshotai/kimi-k2.6","messages":[{"role":"user","content":"ping"}],"max_tokens":1}' "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/ai/v1/chat/completions" \| head -c 200` |
| `claude` | `ANTHROPIC_API_KEY` (managed by `claude` CLI's own auth) | shell out to `claude -p` | CLI: `claude` | (per Claude defaults at call time) | `claude -p "ping" --output-format=text` |
| `codex` | `OPENAI_API_KEY` (managed by `codex login`) | shell out to `codex exec` | CLI: `codex` | `gpt-5.4` | `codex exec "ping" --json` |
| `gemini` | `GEMINI_API_KEY` | shell out to `gemini -p` | CLI: `gemini` | (per Gemini CLI defaults) | `gemini -p "ping"` |
| `local` | `LOCAL_LLM_BASE_URL`, `LOCAL_LLM_MODEL` (override; defaults `http://localhost:11434` + `qwen2.5-coder:14b`) | OpenAI-compatible HTTP (Ollama / LM Studio / llama.cpp server) | `$LOCAL_LLM_BASE_URL` | `$LOCAL_LLM_MODEL` | `curl -sf $LOCAL_LLM_BASE_URL/api/tags \| head -c 200` |

---

## Notes per provider

### `kimi` — Moonshot AI K2.6 via Cloudflare Workers AI

- Garage tools call Kimi through Cloudflare's OpenAI-compatible endpoint, not directly at Moonshot. Cloudflare added Day 0 Kimi K2.6 hosting on 2026-04-20; routing through Cloudflare gives us one credential surface, edge latency, and unified billing across any future CF-hosted models. See journal `2026-04-21-kimi-via-cloudflare.md` for the decision rationale.
- OpenAI SDK drop-in. Python: `OpenAI(api_key=os.environ["CLOUDFLARE_API_TOKEN"], base_url=f"https://api.cloudflare.com/client/v4/accounts/{os.environ['CLOUDFLARE_ACCOUNT_ID']}/ai/v1")`.
- Models: `@cf/moonshotai/kimi-k2.6` (default — 256k context, multimodal, tool calling), `@cf/moonshotai/kimi-k2.5` (256k, multimodal, predecessor). Cloudflare's model catalog is the source of truth for what's available at any moment — see https://developers.cloudflare.com/workers-ai/models/ .
- Pricing: Cloudflare's Workers AI metered billing; see https://developers.cloudflare.com/workers-ai/platform/pricing/ for per-neuron / per-token rates.
- **Direct Moonshot backend is a deferred option.** A consumer who wants to call `api.moonshot.ai` directly (own `MOONSHOT_API_KEY`, own billing, access to models not yet on Cloudflare like `kimi-k2-thinking` or `kimi-k2-turbo-preview`) can add a `--backend moonshot-direct` config knob to the `kimi` provider in a future Conductor release. That stays on the `kimi` identifier; the backend is a detail of how Conductor reaches the model family.
- **Moonshot's own `kimi` CLI** (`uv tool install --python 3.13 kimi-cli`) is interactive-only and not suitable for scripted dispatch by any garage tool.
- Sources: Cloudflare Kimi K2.6 changelog (https://developers.cloudflare.com/changelog/post/2026-04-20-kimi-k2-6-workers-ai/), Cloudflare OpenAI compatibility (https://developers.cloudflare.com/workers-ai/configuration/open-ai-compatibility/), Moonshot migration notes (https://platform.kimi.ai/docs/guide/migrating-from-openai-to-kimi).

### `claude` — Anthropic via Claude Code CLI

- Garage tools shell out to the `claude` CLI (typically `claude -p "<prompt>" --output-format=text` or `--output-format=stream-json`).
- Auth handled by the CLI itself (`claude` reads `ANTHROPIC_API_KEY` or uses the keychain login session). Garage tools do not read the env var directly.
- No HTTP path used by garage tools today. If a tool ever needs HTTP (e.g., Cortex Phase C wants raw API for cost control), that becomes a new provider row (`claude-http`?) or this row gains a second endpoint shape — not silent divergence.

### `codex` — OpenAI via Codex CLI

- Garage tools shell out to `codex exec`. Sentinel currently labels this provider `openai` in `src/sentinel/providers/openai.py` (class `OpenAIProvider`, enum `ProviderName.OPENAI`). **This is pre-existing drift** from the Doctrine 0003 rule that identifiers match across tools. Touchstone uses `codex`.
- **Follow-up tracked:** rename Sentinel's identifier `openai` → `codex` (or accept the alias and document it explicitly here). Either way, drift becomes intentional rather than accidental. See `plans/llm-provider-additions.md` § "Follow-ups (deferred)" → "Reconcile codex/openai identifier drift in Sentinel."
- Default model `gpt-5.4` is what Sentinel currently sets in `OpenAIProvider.__init__`.

### `gemini` — Google via Gemini CLI

- Garage tools shell out to `gemini -p`. Auth via `GEMINI_API_KEY` (read by the CLI, not by garage code).
- Default model not pinned at the garage level today; the CLI's own default applies. If this becomes a source of drift, pin a specific model ID here.

### `local` — Local LLM via OpenAI-compatible HTTP

- Covers Ollama (`http://localhost:11434/api/chat` or its OpenAI-compatible path), LM Studio, llama.cpp server, vLLM, and similar.
- Sentinel's `local.py` uses `httpx` against the Ollama-native endpoint with default model `qwen2.5-coder:14b`. Touchstone's reviewer cascade `local` entry expects the user-configured `command` (e.g., `ollama run YOUR_MODEL`) per `.codex-review.toml` — slight shape mismatch with Sentinel today.
- **Follow-up tracked:** align `local` invocation shape across tools. Either both call HTTP, or both shell out to a configurable command. Currently both work but a user moving between tools sees different config knobs. See `plans/llm-provider-additions.md` § "Follow-ups (deferred)."

---

## Adding a new provider

When a new provider is added (e.g., DeepSeek, Grok, Mistral):

1. **Open a coordination plan** in `.cortex/plans/` describing which tools will adopt it and in what order.
2. **Add a row to this table** with all six fields populated. If any field is unknown or undecided, the provider isn't ready to add — settle it first.
3. **Per-tool PRs reference this table** for env var name, default model, and smoke test. Tools may not invent values that aren't in the table.
4. **Bump the "Last update" date** at the top of this file.
5. **Journal the addition** (T1.1 fires because this file isn't in `.cortex/`, but it changes a load-bearing cross-tool surface — a journal entry is good citizenship per T2.1/T2.4).

When upstream changes a default or an env var convention:

1. **Update the row.** Bump the "Last update" date.
2. **Open a journal entry** describing the change and any per-tool work needed to absorb it.
3. **If the change is breaking** (env var rename, base URL change), file a per-tool follow-up to ship the migration.

---

## Why this lives outside `.cortex/`

The doctrine entry (0003) is immutable-with-supersede; the providers table changes whenever a model ID bumps or a new provider lands. Putting the mutable table in doctrine would force a supersede chain for every routine update. Putting it in `.cortex/state.md` would mix it with other cross-tool state and make it harder to reference from per-tool PRs.

`autumn-garage/integration/providers.md` is a stable URL the per-tool PRs can link to, sits at a top-level path that signals its purpose, and is editable without doctrine ceremony.
