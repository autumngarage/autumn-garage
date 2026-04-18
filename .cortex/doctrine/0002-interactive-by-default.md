# 0002 — First-run commands are interactive by default; flags are overrides, not the primary interface

> New-user commands in the Autumn Garage trio (`touchstone new`, `touchstone init`, `cortex init`, `sentinel init`) are interactive on a TTY. Flags remain as overrides for scripting and experts. A `--yes` / `-y` flag accepts all defaults. Non-TTY environments (CI, hooks, subprocesses) fall back to flag-driven behavior with sensible defaults and warnings rather than hanging on prompts. Every wizard prints its equivalent flag-form at the end so scripters learn the flags by using the tool.

**Status:** Accepted
**Date:** 2026-04-18
**Promoted-from:** journal/2026-04-18-setup-reflection
**Load-priority:** always

## Context

First dogfood of the full stack — scaffolding `autumn-mail` via `touchstone new autumn-mail --type swift --reviewer codex --no-register`, then `cortex init`, then implicit `sentinel work` auto-init — surfaced a consistent pattern of friction:

- Users must memorize flags they don't yet know exist (`--type`, `--reviewer`, `--review-routing`, `--no-register`, `--small-review-lines`, `--gitbutler-mcp`).
- Defaults are sometimes invisible and sometimes surprising (e.g., Touchstone silently registers new projects in `~/.touchstone-projects` unless the user knows `--no-register`; Sentinel's first-run config pairs coder + reviewer on the same provider, violating its own "different providers" design rule).
- Discoverability fails new users. The first-time friction is paid every time a new person (or the author in a new context) adopts the tools.
- Scripting value is real but secondary. The primary user in the solo-dev scenario the trio targets is a human starting something new, not a CI job bootstrapping.

Meanwhile, the prior-art for interactive first-run is strong and user-proven: `gh repo create` without args, `npm init`, `cargo new`'s confirm prompts, `create-vite`, `yarn create`. All of these keep their flag-forms but default-to-prompt. None have been harmed by the addition.

## Decision

Every first-run command in the trio MUST:

1. **Detect TTY.** If interactive, prompt for the ambiguous choices. If non-interactive (CI, redirected stdin, `CI=true`), fall back to the current flag-driven behavior without prompting.
2. **Prompt only for the ambiguous choices.** Anything the tool can detect (project type from files present, default branch from `git config init.defaultBranch`, gh auth from `gh auth status`) should be auto-detected and presented for confirmation, not asked blindly.
3. **Preserve flag precedence.** Any flag explicitly passed on the command line skips its corresponding prompt. `touchstone new foo --type swift` only prompts for the choices `--type` didn't answer.
4. **Provide `--yes` / `-y`.** Accepts all interactive defaults without prompting. Equivalent to "run the wizard, press Enter at every step."
5. **Print the equivalent flag-form at end.** After a successful wizard, print the single-line flag command that would reproduce the result. Teach-by-doing beats documentation for scripting onboarding.
6. **Be reversible / clean up on cancel.** `Ctrl-C` or `q` during the wizard must not leave partial scaffolds, stray git init, or registry entries behind.
7. **Save answers to config** where applicable, so re-running is idempotent and the user never re-answers the same question without reason.

Inside scope: `touchstone new`, `touchstone init`, `cortex init`, `sentinel init` (new explicit command — promoted from the implicit auto-init in `sentinel work`).

Outside scope: non-first-run commands (`touchstone sync`, `cortex doctor`, `sentinel work`). These remain flag-driven or already benefit from context they have. Interactive prompts mid-cycle would interrupt flow.

## Consequences

- **What becomes easier:** first-run UX collapses from "read docs, memorize flags, hope for the best" to "type the command, answer a few questions, get a good result." Discoverability rises — users learn what the tool can do by running it.
- **What becomes harder:** flag maintenance must stay current alongside prompts. Non-TTY fallback adds a test surface. Prompt order and phrasing must be carefully designed; interrogation fatigue is real.
- **What this forecloses:** CLIs in the trio may not introduce new first-run commands that are flag-only (no interactive alternative). Every new first-run must ship with both.

---

Falsification condition: if telemetry (or user feedback) shows the wizard is *slower* than flag-driven usage for 80%+ of runs even among novice users, the default flips back to flag-driven and interactive becomes opt-in via a subcommand (`touchstone wizard` instead of `touchstone new`).
