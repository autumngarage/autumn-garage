# Branding system — pastel tone-on-tone wordmark across the quartet

**Date:** 2026-05-01
**Type:** decision
**Trigger:** T1.1 (touches `.cortex/doctrine/` — adds Doctrine 0007)
**Cites:** doctrine/0007-branding-system

> Codified the Autumn Garage branding system as Doctrine 0007 and dispatched a four-agent swarm to apply it: each tool gets a figlet `standard`-font wordmark, a pastel tone-on-tone color from a locked palette, a `by Autumn Garage` attribution line, and a CLI banner contract modeled on conductor's existing `banner.py`. Autumn-garage's own README updated in this commit.

## Context

User asked to standardize ASCII branding across all four tools and the umbrella repo. Conductor v0.3.x had already shipped the most polished surface — a Python `banner.py` with embedded figlet glyphs, ANSI 256-color, TTY-aware emit, plus a caller-attribution one-liner. Touchstone had a smaller bash `tk_hero` (runtime figlet + gum). Cortex and Sentinel had no ASCII branding at all. Autumn-garage's README was plain markdown.

User's two constraints:

1. Pastel + tone-on-tone (no saturated terminal colors).
2. Same wordmark/palette/attribution everywhere — README *and* CLI splash.

## What we decided

**Doctrine 0007** locks in:

- **Wordmark.** figlet `standard`-font glyphs, embedded as string literals (no runtime `figlet` dependency). Canonical glyph blocks for all five names (Touchstone, Cortex, Sentinel, Conductor, Autumn Garage) live in the doctrine.
- **Palette.** One pastel hue family per tool, two ANSI 256 codes each (primary for glyphs, subtitle for tagline / version / `by Autumn Garage`).

| Tool | Primary | Subtitle | Family |
|---|---|---|---|
| Touchstone | `216` | `223` | peach |
| Cortex | `152` | `159` | pale aqua |
| Sentinel | `151` | `157` | sage |
| Conductor | `147` | `183` | lavender |
| Autumn Garage | `181` | `217` | dusty rose |

  Conductor's existing palette (saturated `99` deep purple, bold) gets bumped to lavender `147`/lilac `183` (no bold) as part of this sweep — the cost of a one-time visual nudge to bring the family onto the same softer axis.

- **Attribution.** `by Autumn Garage` line in every banner (CLI: subtitle color; README: italicized + linked).
- **CLI contract.** `render_banner()` / `print_banner()` shape modeled on conductor's `src/conductor/banner.py`. Stderr only; honors `NO_COLOR`, `CLICOLOR=0`, non-TTY. Animated reveal opt-out, ≤250ms total, only on `init`.
- **README treatment.** Fenced ```text``` glyph block at top, then tagline + attribution line linking the family.

## Why pastel + tone-on-tone

Saturated terminal colors fight other tooling output. Pastels recede into the TTY palette — branding without shouting. Two values from the same hue family let the eye read the wordmark + subtitle as one branded block rather than two competing colored regions. One hue family per tool means a user shelling between two tools in one terminal locates themselves instantly.

## What got shipped in this commit

- `.cortex/doctrine/0007-branding-system.md` — the full spec, with the five canonical glyph blocks embedded so they're never lost to a missing `figlet` install.
- `README.md` — autumn-garage's own README now leads with the canonical wordmark + tagline + attribution + family links.

## What's in flight (dispatched but not yet merged)

Four codex-coding-agent swarms running in background, one PR per tool repo, all titled `branding: ASCII wordmark + Autumn Garage attribution` (conductor's titled `branding: pastel palette + Autumn Garage attribution + README wordmark`):

- **autumngarage/touchstone** — README header swap + `lib/ui.sh` `tk_hero()` reworked to embedded glyphs + pastel peach.
- **autumngarage/cortex** — new `src/cortex/banner.py` + wired into `init` + `doctor` + README header + tests.
- **autumngarage/sentinel** — new `src/sentinel/banner.py` + wired into `sentinel work` + README header + tests.
- **autumngarage/conductor** — color swap (99/bold → 147 lavender) + add `_ANSI_LILAC` for subtitle + `by Autumn Garage` line in `render_banner()` + comment update citing 0007 + README header + test updates.

PR URLs to be appended here when each agent reports back.

## How this doctrine evolves

Branding is decorative; the doctrine is durable. Add a new tool → edit 0007 in place to extend the palette table and add its glyph block, write a journal entry. Change the figlet font choice or swap the palette → write a new doctrine entry that supersedes 0007 (immutable-with-supersede per Cortex Protocol § 4.2).
