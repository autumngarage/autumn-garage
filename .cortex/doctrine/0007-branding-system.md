---
ID: 0007
Title: Branding system — wordmark, palette, attribution
Date: 2026-05-01
Status: Active
Load-priority: always
---

# Doctrine 0007 — Branding system

Every Autumn Garage tool ships with the same branded surface: a figlet `standard`-font wordmark, a pastel tone-on-tone color, an attribution line, and a CLI banner that prints on splash moments. The four tools and the umbrella repo share one visual vocabulary so that a user dropping into any of them recognizes the family.

Conductor v0.3.x shipped the first banner (`src/conductor/banner.py`). This doctrine generalizes that pattern and locks in the cross-tool palette so future tools (and updates to existing ones) stay aligned.

## The five surfaces

Each repo gets the same five branded surfaces:

1. **README header** — fenced ```text``` block with the wordmark, immediately followed by the tagline and the attribution line.
2. **CLI splash banner** — printed to stderr on `init`, `doctor`, and `version --verbose`. Color when stdout is a TTY and `NO_COLOR` is unset; plain otherwise. Subtitle + version + `by Autumn Garage`.
3. **Caller-attribution one-liner** (CLI tools that get shelled out by other tools) — `▸ <Caller> is using <Tool> → <provider/target>`. Conductor already ships this; other tools adopt the same pattern when relevant.
4. **`--version` short form** — single line, no banner, just `<tool> <version> · Autumn Garage`.
5. **Animated reveal on `init`** — the wordmark prints one row at a time, ~50ms apart, totalling ~250ms. Skipped when `NO_COLOR` is set, when stderr is not a TTY, or when `--no-banner` is passed. Optional flourish; tools can drop it without breaking the doctrine.

## Wordmark — figlet `standard` font

All five wordmarks are figlet `standard` (the default). Glyphs are embedded as string literals in each tool — no figlet runtime dependency. The canonical renderings:

### Touchstone

```text
 _____                _         _                   
|_   _|__  _   _  ___| |__  ___| |_ ___  _ __   ___ 
  | |/ _ \| | | |/ __| '_ \/ __| __/ _ \| '_ \ / _ \
  | | (_) | |_| | (__| | | \__ \ || (_) | | | |  __/
  |_|\___/ \__,_|\___|_| |_|___/\__\___/|_| |_|\___|
```

### Cortex

```text
  ____           _            
 / ___|___  _ __| |_ _____  __
| |   / _ \| '__| __/ _ \ \/ /
| |__| (_) | |  | ||  __/>  < 
 \____\___/|_|   \__\___/_/\_\
```

### Sentinel

```text
 ____             _   _            _ 
/ ___|  ___ _ __ | |_(_)_ __   ___| |
\___ \ / _ \ '_ \| __| | '_ \ / _ \ |
 ___) |  __/ | | | |_| | | | |  __/ |
|____/ \___|_| |_|\__|_|_| |_|\___|_|
```

### Conductor

```text
  ____                _            _             
 / ___|___  _ __   __| |_   _  ___| |_ ___  _ __ 
| |   / _ \| '_ \ / _` | | | |/ __| __/ _ \| '__|
| |__| (_) | | | | (_| | |_| | (__| || (_) | |   
 \____\___/|_| |_|\__,_|\__,_|\___|\__\___/|_|   
```

### Autumn Garage

```text
    _         _                            ____                            
   / \  _   _| |_ _   _ _ __ ___  _ __    / ___| __ _ _ __ __ _  __ _  ___ 
  / _ \| | | | __| | | | '_ ` _ \| '_ \  | |  _ / _` | '__/ _` |/ _` |/ _ \
 / ___ \ |_| | |_| |_| | | | | | | | | | | |_| | (_| | | | (_| | (_| |  __/
/_/   \_\__,_|\__|\__,_|_| |_| |_|_| |_|  \____|\__,_|_|  \__,_|\__, |\___|
                                                                |___/      
```

> Glyphs are the canonical artwork. To regenerate or extend, use `figlet -f standard "<Name>"` and update both this doctrine and the relevant tool's banner module in the same change.

## Palette — pastel, tone-on-tone

Each tool owns a pastel hue family. Within that family, two ANSI 256-color codes are picked: a **primary** for the wordmark glyphs and a **subtitle** dim companion for tagline / version / attribution. Tone-on-tone — the two values share a hue and differ in luminance/saturation.

| Tool | Primary | Subtitle | Hex (approx) | Notes |
|---|---|---|---|---|
| **Touchstone** | `216` | `223` | `#ffaf87` / `#ffd7af` | peach → wheat |
| **Cortex** | `152` | `159` | `#afd7d7` / `#afffff` | pale aqua → paler cyan |
| **Sentinel** | `151` | `157` | `#afd7af` / `#afffaf` | sage → pale mint |
| **Conductor** | `147` | `183` | `#afafff` / `#d7afff` | periwinkle → lilac |
| **Autumn Garage** | `181` | `217` | `#d7afaf` / `#ffafaf` | dusty rose → pale rose |

ANSI escape format: `\033[38;5;<n>m` for foreground; `\033[0m` to reset. No bold, no background. Banner code inspects `NO_COLOR`, `CLICOLOR=0`, and `stream.isatty()` before emitting any escape.

**Why pastel.** Saturated terminal colors fight other tooling output. Pastels recede into a TTY palette and read as "decorative chrome", not "this is alarming". The whole branding moment should be quiet.

**Why tone-on-tone within a family.** The subtitle pulls the same hue as the wordmark so the eye reads the block as one element rather than two competing colored regions.

**Why one family per tool.** A user shelling between conductor and sentinel in the same terminal sees different hues and locates themselves instantly. Conductor's banner.py already named this rationale at `src/conductor/banner.py:27-28`.

## Attribution line

Every banner and every README ends with:

```text
by Autumn Garage
```

In CLI: dim/subtitle color, on its own line, beneath the version. In README: italicized line beneath the tagline, with **Autumn Garage** linking to `https://github.com/autumngarage/autumn-garage`. Autumn Garage's own README uses the same line but without the link (you're already there).

## CLI banner contract

Every tool's banner module exposes:

- `render_banner(subtitle: str | None, version: str | None, *, use_color: bool = True) -> list[str]`
- `print_banner(subtitle: str | None, version: str | None, *, stream=None) -> None`
- A `SUBTITLE_INIT` and `SUBTITLE_DOCTOR` constant (or per-tool equivalent), so call sites don't recompute taglines.

Go-to reference implementation: `~/Repos/conductor/src/conductor/banner.py`. Touchstone's bash equivalent (currently `tk_hero`) tracks the same surface in shell.

The banner writes to **stderr**, never stdout. Pipelines that capture tool stdout must be undisturbed.

## README treatment

Every README opens with:

````markdown
```text
<wordmark glyphs>
```

> *<one-line tagline>*
>
> by **Autumn Garage** · part of the [Touchstone](…) · [Cortex](…) · [Sentinel](…) · [Conductor](…) family
````

Then the existing prose (Status / Install / Quick start / etc.). The wordmark is plain markdown — no inline color — relying purely on monospace rendering for the figlet glyphs.

## Animation policy

- Animation is opt-out, not opt-in. It runs by default in interactive `init`.
- Honors `NO_COLOR`, non-TTY stderr, and `--no-banner`.
- ≤250ms total. Five lines, ~50ms each.
- No animation in `doctor`, `version`, or any CI-likely command.
- No animation when the banner is reused as a status overlay (e.g., during long-running operations).

## Why this doctrine and not just per-tool implementations

Three reasons:

1. **A new tool joining the family** (or an old one replatforming) should pick up the spec without reverse-engineering it from conductor's banner.py.
2. **The palette is load-bearing across tools.** If two tools picked the same hue, they'd lose the locate-yourself property. Locking the table here prevents drift.
3. **Updates compound.** When we want to add (e.g.) a `--brand-bw` flag or change the attribution string, the change happens once here and ripples to every tool's PR.

## How to update this doctrine

Branding is decorative; the doctrine is durable. Treat it as immutable-with-supersede per the Cortex Protocol § 4.2 — to revise, write a new doctrine entry that supersedes 0007 (e.g., changing the figlet font choice or replacing the palette). Small additive changes (a new tool's color, a new wordmark) edit this entry directly and write a journal entry recording what changed.

## History

Articulated 2026-05-01 in response to the request to standardize ASCII branding across all four tools and the umbrella repo, building on conductor's existing banner system (`src/conductor/banner.py` v0.3.x). Pastel + tone-on-tone palette chosen explicitly over saturated colors. See `journal/2026-05-01-branding-system.md` for the implementation sweep.
