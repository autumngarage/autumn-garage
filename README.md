```text
    _         _                            ____
   / \  _   _| |_ _   _ _ __ ___  _ __    / ___| __ _ _ __ __ _  __ _  ___
  / _ \| | | | __| | | | '_ ` _ \| '_ \  | |  _ / _` | '__/ _` |/ _` |/ _ \
 / ___ \ |_| | |_| |_| | | | | | | | | | | |_| | (_| | | | (_| | (_| |  __/
/_/   \_\__,_|\__|\__,_|_| |_| |_|_| |_|  \____|\__,_|_|  \__,_|\__, |\___|
                                                                |___/
```

> *The umbrella for four small CLIs that compose into one workflow.*
>
> by **Autumn Garage** · home of [Touchstone](https://github.com/autumngarage/touchstone) · [Cortex](https://github.com/autumngarage/cortex) · [Sentinel](https://github.com/autumngarage/sentinel) · [Conductor](https://github.com/autumngarage/conductor)

# Autumn Garage

Coordination repo for the Autumn Garage quartet:

- **[Touchstone](https://github.com/autumngarage/touchstone)** — scaffolding + pre-push AI review gate + cross-repo sync. *The ground.*
- **[Cortex](https://github.com/autumngarage/cortex)** — portable file-format protocol for project memory. *The spine.*
- **[Sentinel](https://github.com/autumngarage/sentinel)** — autonomous assess→plan→delegate→review loop across multiple CLI providers. *The hands.*
- **[Conductor](https://github.com/autumngarage/conductor)** — capability-aware router across LLM providers (`claude`, `codex`, `gemini`, `kimi`, `ollama`). *The voice.*

Each tool installs independently. Installed together, they compose: scaffold a project with Touchstone, gain memory with Cortex, let Sentinel cycle through work, and have every LLM call route through Conductor — Touchstone reviewing every push along the way.

Composition is by **file contract, not code import** — the load-bearing invariant. Each tool shells out to the others; none imports another as a library. See `.cortex/doctrine/0003-llm-providers-compose-by-contract.md`.

## What lives here

Coordination markdown:

- `autumn-garage-plan.md` — current coordinated upgrade plan (versioned in-document).
- `.cortex/state.md` — current priorities at a glance (the canonical "what's pending" doc).
- `.cortex/doctrine/` — cross-tool decisions (one tool's decisions live in that tool's own repo).
- `.cortex/plans/` — coordinated workstreams.
- `.cortex/journal/` — running record of coordination decisions and dogfood findings.
- `integration/` — cross-tool integration specs (e.g., `providers.md`, the canonical env-var → provider mapping for Conductor's adapters).

Shared infrastructure (active artifacts that touch every tool's release path):

- `.github/workflows/homebrew-bump.yml` — reusable workflow each tool's `release.yml` calls (pinned `@v1`) to auto-bump its `homebrew-<tool>` Formula on release publish.
- `.claude/skills/deploy/SKILL.md` — `/deploy` skill that surveys the four tools for unpublished work and ships the pending releases.

## What does *not* live here

- **Application code.** The four tools live in their own repos. The dogfood lives in [`autumn-mail`](https://github.com/autumngarage/autumn-mail).
- **Single-tool decisions.** Those belong in the relevant tool's own `.cortex/`.
- **Shared libraries.** Forbidden by Cortex Doctrine 0003 — the tools compose through file contracts, never code imports.

(Cross-cutting CI workflows and shared agent skills are not "application code" — those are welcome.)

## Dogfood target

[`autumn-mail`](https://github.com/autumngarage/autumn-mail) — a minimal SwiftUI macOS app that reads and replies to Gmail using a local LLM. The real test of whether the quartet survives contact with a non-Python, CLI-wrapping, local-LLM project. Findings feed back here as journal entries.

## Install the full garage

```sh
brew install \
  autumngarage/touchstone/touchstone \
  autumngarage/cortex/cortex \
  autumngarage/sentinel/sentinel \
  autumngarage/conductor/conductor
```

## Releasing

Each tool's `homebrew-<tool>` tap auto-bumps when you publish a GitHub Release in the tool's repo — the per-tool `.github/workflows/release.yml` calls the shared workflow here. To ship pending work across the quartet, work in this repo and use the `/deploy` skill (or ask Claude to "deploy pending work" — the skill description matches that phrasing).

## License

MIT (or whatever each tool ships under — this repo inherits).
