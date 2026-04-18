# Autumn Garage

Coordination repo for the Autumn Garage trio:

- **[Touchstone](https://github.com/autumngarage/touchstone)** — scaffolding + pre-push AI review gate + cross-repo sync. The ground.
- **[Cortex](https://github.com/autumngarage/cortex)** — portable file-format protocol for project memory. The spine.
- **[Sentinel](https://github.com/autumngarage/sentinel)** — autonomous assess→plan→delegate→review loop across multiple CLI providers. The hands.

Each tool installs independently. Installed together, they compose: scaffold a project with Touchstone, gain memory with Cortex, and let Sentinel cycle through work while Touchstone reviews every push.

## What lives here

- `autumn-garage-plan.md` — the current coordinated upgrade plan (versioned in-document).
- `.cortex/doctrine/` — cross-tool decisions (one tool's decisions live in that tool's own repo).
- `.cortex/plans/` — coordinated workstreams.
- `.cortex/journal/` — the running record of coordination decisions and dogfood findings.
- `.cortex/state.md` — current priorities at a glance.

## What does *not* live here

- Code. The three tools live in their own repos. The dogfood application lives in [`autumn-mail`](https://github.com/autumngarage/autumn-mail).
- Single-tool decisions. Those belong in the relevant tool's own `.cortex/`.
- Shared libraries. Forbidden by Cortex Doctrine 0002 — the tools compose through file contracts, never code imports.

## Dogfood target

[`autumn-mail`](https://github.com/autumngarage/autumn-mail) — a minimal SwiftUI macOS app that reads and replies to Gmail using a local LLM. The real test of whether the trio survives contact with a non-Python, CLI-wrapping, local-LLM project. Findings feed back into this repo as journal entries.

## Install the full garage

```sh
brew install \
  autumngarage/touchstone/touchstone \
  autumngarage/cortex/cortex \
  autumngarage/sentinel/sentinel
```

## License

MIT (or whatever each tool ships under — this repo inherits).
