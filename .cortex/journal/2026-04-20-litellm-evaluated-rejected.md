# LiteLLM evaluated as Sentinel's provider abstraction; rejected

**Date:** 2026-04-20
**Type:** decision
**Trigger:** T2.2 (failed-approach: a path considered and rejected, with reasoning preserved)
**Cites:** doctrine/0004-conductor-as-fourth-peer, plans/conductor-bootstrap, doctrine/0003-llm-providers-compose-by-contract, https://github.com/BerriAI/litellm, https://docs.litellm.ai/blog/security-update-march-2026

**failed-approach:** true

> Investigated LiteLLM as a possible drop-in for Sentinel's provider layer (and possibly a Python shim Touchstone could shell out to). Concluded: don't adopt. Three load-bearing reasons — the March 2026 PyPI supply-chain compromise that shipped a credential stealer to ~47k installs; LiteLLM's API-key-first design which would force Sentinel to abandon its "user authenticates with their CLI, Sentinel never sees keys" invariant for every provider; and specific known Moonshot bugs (reasoning_content stripping in multi-turn tool calls — LiteLLM #21672) that would bite Sentinel on day one. The decision led directly to extracting Conductor as our own thin abstraction.

## Context

After Doctrine 0003 shipped (cross-tool provider contract, file-only) and the two side-quest plans extracted, the user asked whether OSS tools could simplify the work. LiteLLM was the obvious candidate: 100+ provider support, OpenAI-compatible request shape, well-known.

A deep investigation looked at:
- Architecture (SDK + FastAPI gateway, single package)
- Production adoption (Rocket Money, Samsara, Lemonade, etc., per third-party reviews)
- Provider quality for our specific stack (Anthropic, Codex CLI, Gemini, Ollama, Moonshot)
- Known footguns (PyPI compromise, memory leaks, cache cost accounting, reasoning_content stripping)
- Maintenance posture (BerriAI: 8 employees, single dominant maintainer with 15k commits, near-daily releases, no disclosed Series A)
- Migration testimonials (much stronger evidence of orgs migrating *away* than *to*)

## What we decided

**Don't adopt LiteLLM.** Build Conductor instead (see `doctrine/0004-conductor-as-fourth-peer` and `plans/conductor-bootstrap`).

The three load-bearing reasons:

1. **Supply-chain risk.** March 2026: LiteLLM PyPI versions 1.82.7 and 1.82.8 contained a credential-stealing payload — `litellm_init.pth` that ran on every Python interpreter startup, exfiltrating env vars, SSH keys, AWS/GCP/Azure creds, kube configs, Docker creds, CI secrets, and crypto wallets. Live ~46 minutes; ~47k downloads in the window. Root cause was a chained compromise (Trivy CI → maintainer's PyPI creds). LiteLLM responded well, but the structural risk (8-person seed-stage company, single dominant maintainer, near-daily releases via that one credential) is real for any tool that would handle every API key Sentinel uses. Sources: [LiteLLM advisory](https://docs.litellm.ai/blog/security-update-march-2026), [Snyk](https://snyk.io/blog/poisoned-security-scanner-backdooring-litellm/), [Wiz](https://www.wiz.io/blog/threes-a-crowd-teampcp-trojanizes-litellm-in-continuation-of-campaign), [Simon Willison](https://simonwillison.net/2026/Mar/25/litellm-hack/).
2. **Auth doctrine break.** LiteLLM is API-key-first. Adopting it means Sentinel reads `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `OPENAI_API_KEY`, `MOONSHOT_API_KEY` directly — abandoning the deliberate "user authenticates with their CLI, Sentinel never sees keys" invariant for *every* provider, not just Moonshot. That's not a refactor; it's a different security posture for the whole tool.
3. **Specific Moonshot bugs would bite us.** [LiteLLM #21672](https://github.com/BerriAI/litellm/issues/21672) — `reasoning_content` stripped from assistant tool-call messages, breaks multi-turn tool calling on Kimi K2.5/K2 thinking. Reproduced across other projects: [strands-agents/sdk-python#1150](https://github.com/strands-agents/sdk-python/issues/1150), [google/adk-python#3983](https://github.com/google/adk-python/issues/3983). That's precisely the path Sentinel's Researcher/Coder roles exercise. We'd be debugging someone else's translation layer instead of our own ~80 lines.

Bonus signal: more documented migrations *away from* LiteLLM ([CrewAI removal guide](https://docs.crewai.com/en/learn/litellm-removal-guide), [HKUDS/nanobot post-hack](https://github.com/HKUDS/nanobot/discussions/2445), [dev.to "I built a simpler LLM gateway"](https://dev.to/devansh365/litellm-got-hacked-i-built-a-simpler-llm-gateway-you-can-actually-audit-3hia)) than substantive testimonials *to* it.

Considered and rejected within the LiteLLM evaluation:

- **Use LiteLLM only for Moonshot, not other providers.** Pays the full dependency cost (16 MB wheel, ~30 transitive deps, hard-pinned `openai==2.24.0`) for one provider with the most documented bugs. Worst-of-both.
- **Vendor a stripped-down LiteLLM fork.** Vendoring 1 GB of source to extract a few hundred useful lines is overhead, and we'd be on the hook for security tracking forever.

## Consequences / action items

- [x] Investigation report saved (in subagent output; not persisted as a separate file — the relevant findings are inlined here and in Doctrine 0004's Context section).
- [x] Decision recorded in Doctrine 0004 and `plans/conductor-bootstrap.md`.
- [ ] If LiteLLM ever becomes relevant again (e.g., they're acquired, the company posture changes, the supply-chain story matures): revisit this entry first; the rejection wasn't blanket — it was reasoned against this codebase's specific constraints at this specific time.
- [ ] If we ship Conductor v0.1 and find ourselves reimplementing what LiteLLM solves (cost tracking, retries, fallback chains, streaming), revisit before building each one — sometimes the right answer is "use the OSS thing for this narrow concern even if we don't adopt the whole library."

## Reflection

The investigation took ~3 minutes via subagent and saved what could easily have been weeks of LiteLLM adoption followed by months of debugging-someone-else's-bugs and panic-rolling-back-after-the-next-supply-chain-incident. The discipline of "pause to research a load-bearing decision before committing" is worth the latency every time. Worth journaling because the rejection itself is durable knowledge — anyone in the future who asks "why didn't we just use LiteLLM?" deserves an honest, sourced answer.
