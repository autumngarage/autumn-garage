---
Status: active
Written: 2026-04-24
Author: claude-code (Henry Modisett, @henry@perplexity.ai)
Goal-hash: sen2cdr01
Updated-by:
  - 2026-04-24T01:30 claude-code (initial plan — Slice A brew hygiene shipped, Slices B-E scoped)
Cites: plans/touchstone-conductor-integration, plans/conductor-http-tool-use, journal/2026-04-24-local-llm-dogfood, doctrine/0003-llm-providers-compose-by-contract, doctrine/0004-conductor-as-fourth-peer
---

# Sentinel → Conductor migration

> Collapse sentinel's own `src/sentinel/providers/` onto conductor. Sentinel keeps its role loops and high-level contracts; every LLM call underneath becomes a conductor call. Completes the trio→quartet→collapse arc started with touchstone v2.0.

## Why

Three drivers:

1. **Doctrine 0003 / 0004**: "every tool in the garage composes over one provider contract." Touchstone collapsed onto conductor in v2.0 (-448 lines, one adapter). Sentinel is the last consumer maintaining a parallel provider layer. Every bug fix, every new provider, every capability axis (effort, sandbox, tool-use, context budget) has to be implemented twice today.
2. **Sentinel dogfood gains**: conductor's v0.3.3 shipped graceful fallback, capability filters, session resume, per-iteration cost tracking, context-budget halts, silent-fail diagnostics for local LLMs. Sentinel inherits none of these without the migration.
3. **Local LLM story**: sentinel's `LocalProvider` says `agentic_code=False` — sentinel's coder can't use ollama today. Conductor's v0.3.x HTTP tool-use loop gives sentinel's coder a local path for the first time.

## Current-state audit

Captured in full during 2026-04-24 reconnaissance (see journal for findings). Summary:

**Sentinel's provider layer** (5 files, ~2000 lines + ~2100 lines of tests):
- `interface.py`: `Provider` ABC with `chat` / `code` / `research` / `chat_json` / `detect` + `ChatResponse` dataclass + `ProviderCapabilities` + shared subprocess + budget helpers
- `claude.py` / `openai.py` / `gemini.py` / `local.py`: concrete impls
- `router.py`: picks provider for each role, applies `DEFAULT_RULES` task-aware overrides, scopes coder's timeout separately

**Features sentinel has that conductor doesn't** (need to port or keep sentinel-side):
- `chat_json(prompt, schema)` — structured-output with schema validation
- `DEFAULT_RULES` — per-task/prompt-size model overrides (e.g., "synthesize task on gemini → force gemini-2.5-pro regardless of config", "evaluate_lens with prompt > 60k tokens on gemini → force gemini-2.5-flash")
- `--disallowedTools` for read-only chat safety (claude only)
- `--approval-mode plan` for Gemini read-only (conductor's gemini adapter already does this)
- Per-role timeout isolation (coder's timeout is separate from monitor/planner/researcher)
- `max_turns` config (conductor uses hardcoded 10 for HTTP loops; shell-outs pass to CLI's native `--max-turns`)
- `stderr` + `raw_stdout` on every `ChatResponse` path, including errors — load-bearing for coder transcripts and budget tracking
- `session_id` captured from CLI output

**Features conductor has that sentinel doesn't** (inherited post-migration):
- `supported_tools` / `supported_sandboxes` capability declarations → router-enforced before call
- `--prefer best / cheapest / fastest / balanced`
- `--effort minimal..max` with per-provider token translation
- `--exclude <name>` to skip a provider
- `call.usage.iterations` cost-per-turn log
- `hit_context_budget` + `hit_iteration_cap` signals on HTTP loops
- Graceful one-hop fallback on 5xx / 429 / timeout
- Silent-fail stealth-tool-call guard (v0.3.3)
- `--resume <session_id>` for multi-turn sessions
- ASCII hero banner, concierge init

## Target-state design

**Layer the sentinel API onto conductor, don't rewrite the roles.** The role loops (`roles/monitor.py`, `roles/coder.py`, etc.) should not need to change. Keep sentinel's `Provider` interface as the contract the roles consume; implement that contract by delegating to conductor.

### Architecture after migration

```
sentinel.roles.{monitor,coder,reviewer,…}
         │
         │ (unchanged API)
         ▼
sentinel.providers.Provider   ◄── kept: the ABC + ChatResponse dataclass
         │
         │ (new single implementation)
         ▼
sentinel.providers.conductor_adapter.ConductorAdapter
         │
         │ (subprocess OR Python import)
         ▼
conductor.providers.{ClaudeProvider,CodexProvider,GeminiProvider,
                     KimiProvider,OllamaProvider}
```

The adapter is the only place that knows about conductor. The roles keep their current imports. Sentinel keeps its config schema (users don't retype anything). Router keeps its `DEFAULT_RULES` — it picks the model name, then the adapter routes through conductor with that model.

### Key design decisions

- **Python import, not subprocess.** Sentinel already depends on `httpx`, `pydantic`, `click` — adding `conductor` as a library dep is cheap. Subprocess per call has overhead we don't need when we own both sides.
- **`chat_json` stays in sentinel.** Conductor doesn't expose a schema-validating call. Sentinel's adapter can implement `chat_json` on top of conductor's `call()` with a prompt-engineering preamble + `json.loads` + pydantic/jsonschema validate. This keeps conductor's surface small.
- **`DEFAULT_RULES` stays in sentinel.** Task-aware model overrides are a sentinel concern (it's about which model sentinel's workload wants, not a router-universal rule). Sentinel's router pre-resolves the model, then calls `conductor.get_provider(name).call(model=...)`.
- **Per-role timeout stays in sentinel.** Sentinel's `Router` constructs per-role `ConductorAdapter` instances with role-specific timeouts. The adapter forwards timeout to conductor via the `timeout_sec` kwarg on the provider's `__init__`.
- **`code()` delegates to conductor's shell-out `exec()` for claude/codex** (sentinel's current coder path); for local/kimi it delegates to conductor's HTTP tool-use loop (new local-coder capability). Sentinel's `CoderConfig.max_turns` maps to conductor's `KIMI_MAX_TOOL_ITERATIONS` / equivalent (may need conductor to expose this as a kwarg — see Slice B).
- **`ChatResponse.stderr / raw_stdout / session_id` preserved.** Sentinel's adapter reads these from conductor's `CallResponse.raw` (which conductor already captures) and re-exposes them on the sentinel dataclass so roles' existing error handling still works.

## Slice plan

### Slice A — brew hygiene ✅ shipped 2026-04-24

- Sentinel brew formula already at v0.3.6 with matching sha256
- Description trimmed from 82 → 74 chars to pass `brew audit --strict`
- Verified `brew install autumngarage/sentinel/sentinel` gives a working sentinel; `sentinel providers` detects all four providers end-to-end
- No code change; one-line formula fix committed to homebrew-sentinel

### Slice B — `ConductorAdapter` shim (~1 session)

Write `src/sentinel/providers/conductor_adapter.py` implementing sentinel's `Provider` ABC:
- Construct with `(provider_name, model, timeout_sec, max_turns, ollama_endpoint)`
- `chat()` → `conductor.get_provider(provider_name).call(prompt, model=model, effort='medium')` with translation
- `chat_json()` → `chat()` with JSON-shaped prompt + parse + `jsonschema` validate
- `research()` → `chat()` (for providers with `web_search` capability; today only affects gemini's implicit grounding)
- `code()` → `conductor.get_provider(provider_name).exec(prompt, tools=…, sandbox='workspace-write', cwd=working_directory, timeout_sec=coder_timeout)`
- `detect()` → inspect `conductor.get_provider(provider_name).configured()`

Feature-flag at construction so the migration is reversible in prod (`SENTINEL_USE_CONDUCTOR=0` reverts). Tests: ~40 new tests in `tests/test_conductor_adapter.py` covering every method + error paths.

**Risks addressed in Slice B:**
- R6 (stderr/raw_stdout/cost on every path) — adapter populates these from conductor's `CallResponse.raw`
- R7 (cost on non-zero exit) — conductor already preserves usage on `ProviderHTTPError`; adapter maps that through
- R10 (tool-use loop ownership) — for claude/codex, conductor's shell-out adapters already pass `--max-turns` via the CLI; we pipe it through

**Risks deferred to Slice C/D:**
- R1 `--disallowedTools` for claude chat mode — needs a conductor change to accept a negative-tool-filter (new kwarg on `call()`)
- R9 ollama endpoint — need a way to pass `OLLAMA_BASE_URL` per-instance (currently env-only in conductor; may need kwarg)

### Slice C — router migration (~0.5 sessions)

- Update `src/sentinel/providers/router.py` to instantiate `ConductorAdapter` instead of `ClaudeProvider`/`OpenAIProvider`/etc.
- `DEFAULT_RULES` keep working — they pick the model, router passes it to the adapter
- Add `SENTINEL_USE_CONDUCTOR` env flag: when unset or falsy, fall back to native providers; when set, use adapter
- Feature-flag lets us merge without flipping default; flip default after Slice D validates end-to-end
- Tests: existing `test_router.py` stays mostly intact — now runs against the adapter with conductor mocked at the Python level

### Slice D — role dogfood (~1-2 sessions)

Run real sentinel cycles against autumn-mail or a scratch repo with `SENTINEL_USE_CONDUCTOR=1`:
1. **Monitor first** — simplest role, all `chat()` / `chat_json()`, no tool-use. If monitor works end-to-end, the common path is validated.
2. **Researcher** — chat-only + web_search implicit.
3. **Reviewer** — `chat_json()` with structured verdict schema. Stresses the JSON path hardest.
4. **Planner** — currently stubbed in sentinel, low risk.
5. **Coder** — agentic `code()`, highest risk. Validates `--max-turns`, `--dangerously-skip-permissions` flow through conductor's shell-out.

Each role is a separate commit; each is a real-workload validation, not a unit test. Expect to file small conductor PRs for gaps surfaced here (e.g., `--disallowedTools` kwarg from R1).

Flip the default (`SENTINEL_USE_CONDUCTOR=1` → default true) once all five roles pass a real cycle.

### Slice E — delete native providers (~0.5 sessions)

- Delete `src/sentinel/providers/claude.py`, `openai.py`, `gemini.py`, `local.py` (not `interface.py` — the ABC + `ChatResponse` stay as the contract the adapter satisfies).
- Router loses its `ProviderName → ProviderClass` map; constructs only `ConductorAdapter`.
- Delete `tests/test_providers.py` (242 lines) and `tests/test_openai_ndjson.py` (105 lines) — now conductor's responsibility.
- Trim `tests/test_router.py` to test adapter-instantiation + `DEFAULT_RULES`, not provider internals.
- Net: -1800 to -2200 lines depending on what gets trimmed from integration tests.

Bump sentinel to v0.4.0 for the version landmark ("sentinel now runs on conductor exclusively").

## Risks (from the recon, restated)

| # | Risk | Addressed in slice | Notes |
|---|---|---|---|
| R1 | Claude's `--disallowedTools` for chat safety | B (open gap) → C (conductor change) | Likely needs a new `call(..., deny_tools=[...])` kwarg on conductor's claude adapter |
| R2 | Gemini `--approval-mode plan` read-only | — | Conductor's gemini adapter already does this — verify in Slice D |
| R3 | Per-role timeout isolation | B | Adapter takes `timeout_sec` in ctor; router constructs per-role adapters |
| R4 | Task-aware `DEFAULT_RULES` | C | Router keeps them; they run before the adapter call and just pick `model=` |
| R5 | `max_turns` propagation | B → C | Pipe through to conductor's shell-out `exec()`; HTTP loops may need a kwarg too |
| R6 | `stderr` + `raw_stdout` on all paths | B | Adapter populates from `CallResponse.raw` |
| R7 | Cost on non-zero exit | B | Conductor already preserves usage on ProviderHTTPError |
| R8 | OpenAI NDJSON parsing | — | Conductor's codex adapter already does this — delete sentinel's parser in Slice E |
| R9 | Ollama endpoint config | B (open gap) | Conductor today reads `OLLAMA_BASE_URL` env only; may need a kwarg for per-request override |
| R10 | Tool-use loop ownership | B | Delegated to conductor's shell-out for claude/codex (which delegates to CLI); HTTP loop for local/kimi — new capability |

## Success criteria

1. **A real `sentinel work` cycle on autumn-mail completes with `SENTINEL_USE_CONDUCTOR=1`** — all 5 roles executed, no regression in output quality vs the pre-migration run
2. **`sentinel providers` shows identical output** pre- and post-migration (same providers detected, same capabilities reported)
3. **Test count within ±50 of pre-migration** — we delete ~2000 lines of provider tests but add ~500 lines of adapter tests; net reduction is the win
4. **Conductor gets 1-3 small PRs** filed as gaps surface (e.g., R1 `--disallowedTools`, R9 per-instance ollama endpoint) — all bounded, none requiring architectural change
5. **Slice E deletion is clean** — no remaining imports of `sentinel.providers.claude` etc. in the tree
6. **sentinel v0.4.0 cut**: brew formula bumped, release published

## Out of scope explicitly

- **Changing sentinel's role logic.** Not touching `roles/*.py`. The migration is under the provider interface.
- **Replacing sentinel's budget tracking.** `_abort_if_budget_exhausted` and `_journal_call` stay.
- **Rewriting sentinel's config schema.** Users' `.sentinel/config.toml` stays unchanged.
- **Adding conductor features beyond the surfaced gaps.** If Slice D uncovers an R1/R9-class gap, that's one small PR. Not "rewrite conductor to suit sentinel."

## Relationship to the master integration plan

Finishes Stage 5 of `plans/touchstone-conductor-integration` (explicitly blocked on conductor v0.3 HTTP tool-use — shipped 2026-04-23). Closes the "trio→quartet→collapse" transformation from a code-architecture perspective; the trio comment in state.md becomes literally true.
