# Autumn Garage — Coordinated Upgrade Plan (v3)

> **⚠️ STALE — superseded for sequencing (annotated 2026-05-02).** This document was written 2026-04-18 against Cortex's old "Phase B/D/E" labels. Cortex has since reorganized around the Tier-1→Tier-4→v0.7.0→v0.9.0→v1.0 release sequence. **For the current Cortex launch sequencing, read [`autumngarage/cortex/.cortex/state.md`](https://github.com/autumngarage/cortex/blob/main/.cortex/state.md) and the master plan [`autumngarage/cortex/.cortex/plans/cortex-v1.md`](https://github.com/autumngarage/cortex/blob/main/.cortex/plans/cortex-v1.md).** Those are the canonical sources, kept current by Cortex itself.
>
> The cross-tool *integration contracts* described below (CIC, install scenarios, hook discovery rules, environment permutations) remain conceptually valid — but any specific phase/version/timeline claim about Cortex below is stale; cross-check against Cortex's current `.cortex/state.md` before relying on it.
>
> **Bigger problem this file represents:** per Cortex's [Doctrine 0007](https://github.com/autumngarage/cortex/blob/main/.cortex/doctrine/0007-canonical-ownership-of-state-and-plans.md) (canonical ownership of "where are we" / "what's next" lives in `.cortex/`, not at repo root), this very file is the anti-pattern Doctrine 0007 names — a repo-root coordination plan whose forward-looking content should live in `autumn-garage/.cortex/plans/` as a proper plan with `Status: active` frontmatter, while this root-level file should either be slimmed to a thin narrative index that links to the canonical sources, or deleted in favor of them. Cortex v0.6.0 will ship a `cortex doctor` warning that flags exactly this. Tracked as follow-up against autumn-garage; flagged but not blocking.

**Date:** 2026-04-18
**Scope:** Touchstone (v1.1.0) · Cortex (v0.1.0, shipped on Homebrew 2026-04-18) · Sentinel (v0.2.0)
**Revision note:** v3 updates for reality. Cortex Phase B shipped today as v0.1.0 on `brew install autumngarage/cortex/cortex`, which invalidates v2's assumption that Phase B was weeks out. The "Cortex Integration Contract" sketched in v2 is largely already specified in Cortex Protocol v0.2.0 — T1.6 sentinel-cycle, T1.7 touchstone-arch-diff, T1.9 pr-merged, with templates under `.cortex/templates/`. Remaining work is *operationalization* in Cortex Phase E (consumers writing those entries) and authoring helpers in Cortex Phase D (`cortex journal draft`, `cortex plan spawn`). This plan now lives inside the `autumn-garage` coordination repo; the canonical decision trail is in `.cortex/` (doctrine + plans + journal). This file is a narrative index — prefer `.cortex/state.md` for current priorities.

**v3 changes from v2:** timeline collapsed (Phase B not pending); CIC v0 reframed as Cortex Protocol v0.2.0 + Phase E operationalization; coordination repo now exists (`autumn-garage`); dogfood target named (`autumn-mail`, SwiftUI Gmail client via `gws` + MLX Swift). v2 sections below (install scenarios, environment permutations, operational concerns) remain valid and are retained without re-editing.

---

## v2 content (retained, still correct)

The sections below were written against v2 assumptions (Cortex pending). Read them with the v3 caveat that Cortex v0.1.0 already ships, and that "CIC v0" has largely resolved into Cortex Protocol v0.2.0.

---

---

## 1. Thesis

Three CLIs, file-contract–coupled, no code imports, each usable standalone, opt-in composition. Cortex is the spine; Touchstone the ground; Sentinel the hands.

Position: the open, file-native, provider-agnostic stack. No vendor lock-in, no hosted service, git-diffable at every layer. Closest commercial Frankenstein is Cursor + Mem0 + CodeRabbit; closest OSS Frankenstein is OpenHands + AGENTS.md + copier.

---

## 2. Install-scenario matrix

### 2.1 Tool permutations

| # | Installed | Behavior |
|---|---|---|
| 0 | none | git + manual review |
| 1 | T | Scaffolding + multi-provider pre-push review + cross-repo sync |
| 2 | C | `.cortex/` in any repo; manual authoring; feeds `cortex manifest` into CLAUDE.md/AGENTS.md |
| 3 | S | Autonomous assess→plan→delegate cycles, budget caps |
| 4 | T + C | Scaffolded projects include `.cortex/`; review-hook opt-in writes journal entries |
| 5 | T + S | Scaffolded + autonomous; Touchstone review gate fires on Sentinel pushes via git (already works) |
| 6 | C + S | Sentinel reads `.cortex/state.md` at scan start, writes cycle-end journal entries |
| 7 | T + C + S | Full loop: scaffold → cycle → journal → doctrine promotion → smarter next cycle |

**Degradation rule for all tools:** standalone install paths must never regress. No warnings about missing integrations unless the user enabled them.

### 2.2 Environment permutations (added per Codex critique)

These cross-cut the tool matrix and must be handled by every tool:

- **Monorepos.** `.cortex/` at git-root by default. A `.cortex-root` marker file may re-anchor search to a subdirectory (for per-package memory). Nested `.cortex/` is v1+ work; for v0, one `.cortex/` per git-root, full stop.
- **Git worktrees.** Detection must resolve via `git rev-parse --show-toplevel` not `pwd`. Worktrees share one logical `.cortex/` at the repo root (not the worktree root).
- **Submodules.** Skip descent into submodules during `.cortex/` discovery. Treat submodules as standalone projects if they have their own `.cortex/`.
- **CI runners.** Ephemeral checkouts must not attempt to install, auto-update, or write pending journal entries by default. Every tool exposes `--ci` (or respects `CI=true` env) which disables auto-update, prompts, and optional writes.
- **Devcontainers / Codespaces.** Each tool's homebrew install path must have a `curl | bash` fallback or documented manual install. Don't block on PATH mutations.
- **Corp-locked / offline machines.** Installs from source tarball (GitHub release assets). Document explicitly. No auto-update; version-pin via lockfile/env.
- **Read-only workspaces.** If `.cortex/` isn't writable, Cortex runs in read-only mode (manifest + grep + doctor work; init/journal/refresh fail cleanly).
- **Multi-user shared machines.** Journal entries carry `Author: <git user.email>`; file permissions respect umask; never embed secrets. Shared repos should gitignore pending/rejected (see §6).
- **Windows / WSL.** Cortex and Sentinel are Python 3.11+ so cross-platform. Touchstone is bash and will not support native Windows — WSL only, documented.
- **Hooks running from non-repo cwd.** All scripts must resolve repo root explicitly before reading/writing `.cortex/`.

### 2.3 Upgrade paths

- **T → T+C:** `touchstone init-cortex` (new idempotent subcommand that shells `cortex init`).
- **T → T+S:** `brew install autumngarage/sentinel/sentinel && sentinel init`. No Touchstone change needed.
- **S → C+S:** `brew install autumngarage/cortex/cortex && cortex init`; Sentinel auto-detects next run.
- **Any partial → Full:** documented one-liner in `autumngarage` pinned README. **Deferred:** `touchstone doctor --suggest-garage` (nice but out of v0 scope).
- **Cross-tool version drift:** each tool declares its supported Cortex Integration Contract (CIC) version range in `version --verbose`. Mismatch prints a warning, never blocks.

---

## 3. The Cortex Integration Contract (CIC v0)

**Versioning note:** CIC is independently versioned from Cortex SPEC and Cortex CLI. A single CIC version may span multiple SPEC/CLI releases. Bumping CIC is a coordinated cross-tool event; bumping SPEC/CLI is not.

### 3.1 Read interface (stable, machine-parseable)

- **`.cortex/state.md`** at repo root. Consumers read as markdown; required frontmatter keys: `Spec-version`, `Generated`, `Sources`, `Incomplete`.
- **`.cortex/protocol.md`** — agents may import via `@.cortex/protocol.md`.
- Consumers MUST tolerate unknown frontmatter keys and unknown markdown sections (forward compat).
- Discovery rule: walk up from cwd to git-root; honor `.cortex-root` marker if present.

### 3.2 Write interface — **CLI-primary** (revised from v1)

Public write path is a CLI subcommand with synchronous validation:

```
cortex journal append \
  --type <incident|cycle-complete|pr-merged|decision-draft|test-broke|arch-diff> \
  --source <tool@version> \
  --trigger <T1.1|T1.6|...> \
  --body-file <path or -> \
  [--dry-run] [--json]
```

- Exits 0 on success (entry written to `.cortex/journal/<YYYY-MM>/<ts>-<type>.md`).
- Exits non-zero with structured error (`--json`) on validation failure. The caller knows immediately.
- Normalizes timestamp, de-duplicates on `(source, trigger, content-hash)` within the current day.
- Atomic write (temp + rename) handled internally.
- Adds `Author: <git user.email>` automatically if not provided.

**Why CLI-primary (not file-drop):** Codex correctly noted file-drop moves validation, locking, dedup, and error surface into async ingestion. That's hidden coupling. A CLI gives immediate feedback at the write site; bash consumers shell out; sandboxed processes can still call the CLI.

**File-drop is retained as an internal fallback only**, behind `cortex journal append --defer` — used when the CLI can't reach the repo synchronously (e.g., inside a locked-down hook where shell-out is blocked). Deferred entries land in `.cortex/journal/pending/` and are processed by `cortex doctor --ingest-pending`. Deferred writes are explicitly second-class, documented as such, and may be removed in CIC v1.

### 3.3 Error taxonomy

Rejected entries MUST include:
- Which producer wrote the file (`Source:` from frontmatter).
- Which field failed validation (e.g., `missing: Trigger`).
- Suggested fix (`hint:`).
- Whether the source integration should be disabled (`action: review-config | skip | disable-integration`).

`cortex doctor` surfaces rejections grouped by producer. A user with ten bad Sentinel entries sees "sentinel@0.2.0 produced 10 invalid entries; most common failure: missing trigger."

### 3.4 Detection precedence (revised for consistency)

Each tool, at startup:

1. Check PATH for sibling CLI (`shutil.which` / `command -v`).
2. Resolve git-root; check for `.cortex/`, `.sentinel/`, `.touchstone-config` at that root (honoring `.cortex-root` marker).
3. **Write-integration only activates when both CLI and dir are present.** If the dir exists without the CLI, the tool logs once ("cortex directory detected but CLI not installed; integration disabled") and proceeds standalone. No orphan writes.
4. Read-integration is permitted dir-only (e.g., Sentinel can read `.cortex/state.md` without shelling to Cortex).

### 3.5 What CIC v0 deliberately omits

- No shared Python library / no code imports.
- No unified config file.
- No meta-CLI.
- No cross-tool auth.
- No shared daemon.
- No nested-`.cortex/` semantics.
- No pending-entry format as a public contract (internal only).

---

## 4. Per-tool work (reduced MVP scope)

Per Codex critique, cut to the minimum that proves the loop. Everything else moves to "deferred."

### 4.1 Cortex (the spine) — **ships first**

1. Finish Phase B exit: interactive flow, refresh-map, refresh-state. Dogfood against sigint + self.
2. **After** Phase B exit, draft CIC v0 based on what's actually stable. Do not freeze before.
3. Implement `cortex journal append` as the public write CLI (synchronous validation, atomic, dedup, author injection).
4. Implement `cortex doctor --ingest-pending` for the deferred-write fallback.
5. Document CIC v0 in a dedicated file (`SPEC-INTEGRATION.md` or similar), versioned independently.
6. Tag CIC v0 when both consumers (Sentinel, Touchstone) have dogfooded writes.

**Exit gate:** One real Sentinel cycle and one real Touchstone hook invocation produce well-formed journal entries via `cortex journal append` with zero rejections.

### 4.2 Sentinel (the hands) — **ships second**

1. Cortex detection: `sentinel.integrations.cortex.detect()` returns `(cli_present, dir_present, cic_version)`.
2. Scan-start: if dir present, read `.cortex/state.md`, inject into Monitor's project-context prompt.
3. Cycle-end: if both CLI and dir, call `cortex journal append` with `--type cycle-complete` (lenses, findings count, verdict, PR link, spend).
4. Config: `[cortex] enabled = "auto"  # auto | on | off`, `supports_cic = "~0"`.
5. CI-safe: respects `CI=true`; skips write in CI by default.

**Exit gate:** `sentinel work` in a Cortex-enabled repo writes a journal entry with `cortex journal append` exit 0, no manual edits.

### 4.3 Touchstone (the ground) — **ships third, minimum scope**

1. `touchstone init-cortex` idempotent subcommand (shells `cortex init` in current project; opt-in, never auto-runs).
2. `touchstone init --with-cortex` flag for new projects (runs `cortex init` after scaffold).
3. `hooks/codex-review.sh`: on blocking finding, if `cortex` on PATH AND `.cortex/` present AND `.codex-review.toml` has `write_to_cortex_journal = true` (default false), call `cortex journal append --type incident`.
4. Add `cortex` + `sentinel` detection to `lib/detect.sh` for future use.

**Deferred (explicitly cut from v0):** `touchstone status` integration surfacing, `touchstone doctor --suggest-garage`, Full Garage install script, unified docs site. Revisit after MVP dogfood.

**Exit gate:** `touchstone new foo --with-cortex` + `sentinel work` in the new project produces a clean journal entry, clean `cortex doctor`, no manual edits.

---

## 5. Release sequencing (10 weeks, revised from 6)

Extended per Codex critique: Cortex is v0.1-dev; freezing contract before Phase B exit is backwards. Contract freeze moves to week 5-6, after dogfood has shaped it.

| Week | Cortex | Sentinel | Touchstone |
|---|---|---|---|
| 1-2 | Finish Phase B: interactive flow, refresh commands | Design cortex-detect module | — |
| 3 | Dogfood Phase B on sigint + self; file gaps | — | — |
| 4 | Address Phase B gaps; prototype `journal append` | Prototype read-integration (`state.md` ingestion) | — |
| 5 | Draft CIC v0 spec doc | Prototype write-integration | Draft `--with-cortex` flag |
| 6 | Tag Cortex 0.1.0 + CIC v0; publish Homebrew | Integrate writes against real Cortex | — |
| 7 | Dogfood feedback cycle; bugfix 0.1.1 | Tag Sentinel 0.3.0 with CIC v0 support | Prototype review-hook write |
| 8 | — | Publish Sentinel 0.3.0 Homebrew | Integrate review-hook; dogfood |
| 9 | — | Feedback cycle on Sentinel integration | Tag Touchstone 1.2.0 with CIC v0 |
| 10 | — | — | Publish Touchstone 1.2.0; expect 1 contract revision (CIC v0.1) if needed |

**Coordination rules:**

- Cortex must hit its Phase B dogfood exit before CIC v0 is drafted. No exceptions.
- CIC bumps are coordinated: all three tools declare their supported range; a breaking bump requires synchronized releases.
- CIC v0 → v1 is expected within 6 months of v0. Don't promise stability longer than that.
- Standalone install paths are tested as first-class CI in every tool's test suite.

---

## 6. Operational concerns (week-1 reality, per Codex critique)

### 6.1 gitignore policy

- `.cortex/journal/pending/` — **gitignored** (transient; ingest processes them).
- `.cortex/journal/rejected/` — **gitignored** (diagnostic only; shared via `cortex doctor` output, not VCS).
- `.cortex/journal/<YYYY-MM>/` — **committed** (this is the durable log).
- `.cortex/state.md`, `.cortex/doctrine/`, `.cortex/plans/`, `.cortex/map.md`, `.cortex/procedures/` — **committed**.
- `cortex init` writes a `.gitignore` inside `.cortex/` that encodes these rules.

### 6.2 Identity & multi-user

- Every journal entry includes `Author: <git user.email>` (auto-injected by `cortex journal append`).
- Entries also include `Source: <tool@version>` and `Host: <hostname>` (optional, off by default, opt-in via config for shared machines).
- Permissions respect umask; never chmod.

### 6.3 Uninstall / disable

- `cortex init --uninstall` removes `.cortex/` after confirmation.
- Each consumer's config has an off-switch (`[cortex] enabled = "off"`) that short-circuits integration without uninstalling anything.
- Documentation covers: "how do I stop Sentinel from writing to Cortex" and "how do I throw away the journal."

### 6.4 CI behavior

- Every tool respects `CI=true` env: no auto-update, no prompts, no optional writes unless explicit `--ci-enable-writes`.
- Cortex doctor is CI-safe (read-only by default).

### 6.5 Clock skew & collisions

- `cortex journal append` uses UTC, second-precision. On same-second collisions it appends a suffix (`-1`, `-2`). Client-provided timestamps are ignored (only accepted for backfill via `--backfill`).
- Dedup is by `(source, trigger, content-hash, day)` — same cycle rerun twice in a day collapses to one entry.

### 6.6 Partial upgrades

- Upgrade Cortex but not Sentinel: Sentinel's CIC range check emits a warning if CIC bumped. Writes still work if the wire format is backward compat; if not, writes fall back to deferred (`--defer`) with a loud warning.
- Upgrade Sentinel but not Cortex: Sentinel warns, disables write-integration, runs standalone.
- All three tools expose `<tool> version --verbose` showing CIC range; `cortex doctor` cross-checks detected siblings.

### 6.7 Noise management

- Write-integrations ship default-off for the first release of each consumer. User enables per-project via config.
- `cortex doctor --journal-stats` surfaces entries-per-day per source so users can spot noisy producers.

---

## 7. Open decisions

- **D1 — Principles vs Doctrine boundary.** *Decision:* Touchstone principles = universal (synced cross-repo); Cortex Doctrine = per-project. Doctrine entries can declare `Grounds-in: touchstone-principle-<slug>`. Write up as a SPEC appendix + Touchstone principle. *Owner:* Cortex SPEC, week 3.
- **D2 — CLI-primary vs file-drop writes.** *Decision:* CLI-primary (revised). File-drop retained as second-class fallback (`--defer`). See §3.2.
- **D3 — Umbrella branding.** *Decision:* GitHub org `autumngarage` with a pinned README pointing at the three tools. No monorepo, no meta-CLI.
- **D4 — Homebrew taps.** *Decision:* Keep three separate taps. Document a 3-line install snippet.
- **D5 — Compat surfacing.** *Decision:* Each tool's `version --verbose` prints CIC range. `cortex doctor` cross-checks.
- **D6 — Review-hook findings in journal.** *Decision:* Opt-in per project. Default off. Revisit after MVP dogfood.
- **D7 (new) — Monorepo nested `.cortex/`.** *Decision:* Not in v0. Git-root only, `.cortex-root` marker permitted to re-anchor. Revisit in CIC v1.
- **D8 (new) — Deferred (file-drop) path longevity.** *Decision:* Ship in CIC v0 as second-class. If unused after 3 months, remove in CIC v1.

---

## 8. Risks

- **CIC spec churn breaks consumers.** *Mitigation:* freeze CIC *after* Phase B dogfood, not before. Separate CIC version from SPEC/CLI. Expect one revision inside 6 months.
- **File-bus hidden coupling** (Codex's #1 concern). *Mitigation:* CLI-primary writes with sync validation + structured errors solve the majority; `--defer` path is explicitly second-class with loud warnings.
- **Journal noise.** *Mitigation:* default-off writes, `--journal-stats`, per-source rejection grouping in `cortex doctor`.
- **Touchstone bash ↔ Cortex Python.** *Mitigation:* `cortex journal append` is the only write surface bash needs; shell-out is tractable.
- **User confusion over tool roles.** *Mitigation:* one-page mental model in each README: "Touchstone sets up. Cortex remembers. Sentinel acts."
- **Monorepo root discovery misfires.** *Mitigation:* `.cortex-root` marker + `git rev-parse --show-toplevel` baseline. Tested.
- **Partial upgrades corrupt writes.** *Mitigation:* explicit CIC version range checks; fallback to deferred-writes with warning.

---

## 9. Success metrics (at week 10)

- 0 rejected journal entries in a one-week continuous Sentinel dogfood against sigint.
- `touchstone new foo --with-cortex` + one Sentinel cycle produces a clean PR, clean doctor run, no manual edits.
- Each tool's standalone install passes its own test matrix on macOS + Linux (Ubuntu 22.04 + 24.04) + devcontainer. No warnings about absent siblings.
- ≥1 external (non-Henry) user successfully runs the Full Garage against their own repo.
- CIC v0 stable (no breaking changes) for ≥6 weeks after Week 10.

---

## 10. Explicitly deferred (out of v0 scope)

- Unified CLI.
- Shared Python library.
- Cross-tool auth / secret sharing.
- Hosted dashboard.
- Semantic search / vector layer (excluded by Cortex Doctrine 0005).
- `touchstone status` cortex/sentinel surfacing.
- `touchstone doctor --suggest-garage`.
- `cortex doctor --suggest` install-prompts.
- Full Garage install script.
- Unified docs site.
- Nested `.cortex/` in monorepos.
- Native Windows support for Touchstone (WSL only).
- Journal promotion automation (human-in-loop only for v0).

---

## 11. Codex critique — applied changes

Codex review of v1 raised 12 points. Disposition:

1. ✅ File-drop as public contract → demoted to `--defer` fallback; CLI-primary write path (§3.2).
2. ✅ Atomic-write rules, uniqueness, UTF-8, size → moved into `cortex journal append` internals (§3.2).
3. ✅ "Works without cortex on PATH" inconsistency → resolved via §3.4 rule: writes require both CLI + dir.
4. ✅ Rejected entries UX → structured error taxonomy + grouped-by-producer in `cortex doctor` (§3.3).
5. ✅ Environment permutations missing → §2.2 added (CI, devcontainer, corp-locked, WSL, worktrees, submodules, read-only, multi-user, monorepo).
6. ✅ Monorepo root discovery → `.cortex-root` marker + git-root baseline; nested deferred (D7).
7. ✅ Multi-user identity → `Author:` auto-injection, gitignore pending/rejected (§6.1–6.2).
8. ✅ 6 weeks too aggressive → extended to 10 weeks (§5).
9. ✅ Freeze before Phase B exit is backwards → freeze moved to week 5-6, post-dogfood (§4.1, §5).
10. ✅ Version naming confusion → "Cortex Integration Contract (CIC)" versioned independently (§3).
11. ✅ Weakest assumption (file bus as reliable integration) → addressed by CLI-primary switch (§3.2).
12. ✅ Scope cut → `touchstone status`/`doctor --suggest`/install script/docs site all moved to §10 deferred.
