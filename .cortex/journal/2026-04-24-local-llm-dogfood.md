---
type: decision
Date: 2026-04-24T01:00
Author: claude-code (Henry Modisett)
Cites: plans/conductor-http-tool-use, journal/2026-04-23-conductor-v0.3-http-tool-use
---

# Local LLM path dogfooded end-to-end — three conductor fixes, one recipe

## Context

Coming out of Stage 3 (HTTP tool-use loop shipped in conductor v0.3.0–v0.3.2), the garage's "local LLM support" was plumbed but never exercised against a real model. The audit called it `supported, not dogfooded`. This session closed that gap on an M4 Max 36GB: pulled qwen3.6:27b (dense) and qwen3.6:35b-a3b (MoE), ran conductor's HTTP tool-use loop against each with a real task (read `conductor/src/conductor/tools/registry.py`, explain the three attack patterns `_resolve_in_cwd` defends against), and compared with the existing default `qwen2.5-coder:7b`.

Three production issues surfaced immediately.

## Findings

### 1. qwen2.5-coder doesn't emit structured tool_calls in ollama

Direct `/api/chat` probe with a `tools` parameter: `tool_calls` field is `null`, call shows up as a ```json``` markdown block inside `message.content`. Conductor's HTTP loop reads `message.tool_calls` (the spec field), sees empty, breaks out of the loop, and returns the model's prose as the final answer. That prose is a **hallucination** — the model never ran the Read tool, so it's answering from training-data familiarity with the function's name.

Verified non-deterministic: two runs produced two different incorrect third defenses ("URL redirection" and "Path Injection" — neither is what the code actually does). Always 2/3 correct (path traversal, symlink) by training-data luck on the function-name, wrong on the third.

**Severity**: high. Users configuring conductor with the out-of-the-box default (`qwen2.5-coder:14b`) get confidently-wrong answers from tool-use workloads with no indication the tool didn't run.

### 2. qwen3.6:27b dense is too slow to complete conductor's default timeout

Clean isolated run (no concurrent pulls, model loaded): 5.85 tok/s observed on a simple "what is 2+2" prompt. Full agent loop timed out at 560s with a single `httpx` timeout mid-iteration. Pattern: model + our multi-K-token prompt took >180s (conductor's default per-request timeout) to generate one turn's response, so even the first iteration died.

Diagnosis: 27B dense loaded as 23GB GPU memory. 36GB total Mac RAM leaves only 13GB for OS + everything else, so the system pages under pressure. Not a model bug — a hardware-fit issue.

### 3. qwen3.6:35b-a3b MoE is the right default despite being "bigger"

Same 2+2 probe: 41.76 tok/s (7× faster than 27b dense). Full agent loop: 939s, exit 0, **correct 3/3 answer** verbatim matching the docstring in substance. This is the first local model to actually complete conductor's v0.3.x tool-use loop end-to-end with a non-hallucinated answer.

The MoE architecture (3B active params per token from a 35B total) means generation speed scales with the active count, not the total. For any Mac under ~64GB, MoE is the right class of local model.

## Hardware reference point

M4 Max, 36GB, 10P+4E cores. Tokens/sec numbers above are from this machine. Expect better on 64GB+ Macs with more headroom; expect worse on 16GB Macs where neither 27B nor 35B-A3B will fit without heavy paging.

## What shipped — conductor v0.3.3

PR #12. Three changes:

- **Default model bump**: `qwen2.5-coder:14b` → `qwen3.6:35b-a3b`. Tests reference `OLLAMA_DEFAULT_MODEL` symbol so future bumps are one-line. Guardrail test locks against accidental rollback to the silent-fail model.
- **Timeout bump**: 180s → 600s, env-overridable via `CONDUCTOR_OLLAMA_TIMEOUT_SEC`. Explicit kwarg wins over env; invalid env values fall back rather than raising.
- **Silent-fail guard**: when `tool_calls` is empty but `content` contains a JSON block whose `name` matches a sent tool, prepend a visible `[conductor: ...]` diagnostic so users never get hallucinated prose without knowing it. Balanced-brace walker for detection; handles nested `{"arguments": {...}}`.

+13 tests. All 368 pass.

## Recipe for future reference (candidate doctrine)

For local tool-use loops on Mac:

- **Use `qwen3.6:35b-a3b` or `qwen3.5+ MoE` variants.** Avoid qwen2.5-coder for tool-use — it silently fails in ollama. OK for one-shot `call()` completions.
- **Avoid 27B+ dense models on < 64GB Macs.** Memory pressure makes them functionally unusable even though they load.
- **Budget time: 10-20 minutes for a tool-use loop that touches one file.** Local is not interactive. Useful for batch/background (sentinel cycles overnight), not for touchstone reviews blocking a developer.
- **Disable "thinking mode" when possible.** qwen3.6 emits many thinking tokens by default (observed 124 thinking tokens just to answer "4"). This compounds in multi-turn loops. Not yet exposed via conductor — follow-up TODO.

## Next steps filed but not taken this session

- Investigate whether ollama exposes a `thinking: false` payload flag or equivalent to skip the reasoning step on qwen3.6 — could be a major speedup
- Write a benchmark command (`conductor bench --with ollama`) so users can get numbers for their own hardware without reinventing the harness
- Curated-model list / doctrine entry pinned from this journal so new users don't have to rediscover the qwen-coder silent-fail pattern
- Apply the silent-fail guard pattern to the kimi provider too (same OpenAI-compatible shape; same class of bug possible with other model families)
