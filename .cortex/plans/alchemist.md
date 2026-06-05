---
ID: plans/alchemist
Title: Alchemist — issue-driven transmuter for the Autumn Garage family
Status: active
Written: 2026-05-06
Author: Henry
Goal-hash: 979ffb41
Updated-by:
  - 2026-05-06T00:00 Henry (created)
  - 2026-06-04T23:09-04:00 codex (schema repair)
Date: 2026-05-06
Owner: Henry
Workstream: alchemist-bootstrap
Cites: doctrine/0001, doctrine/0003, doctrine/0004, doctrine/0007
---

# Alchemist — issue-driven transmuter for the Autumn Garage family

> Autumn Garage grows from a quartet to a quintet. **Alchemist** is a thin always-on tool that watches every Autumn-Garage-family repo for GitHub issues, dispatches them as work to **Conductor**'s agentic loop, runs **Touchstone**'s review gate over the resulting diff, and opens a PR. Alchemist owns no engineering opinions of its own — it is a chassis around the existing tools. Its job is *transmutation*: open issue in, reviewed PR out.

## Why (grounding)

This plan is grounded in doctrine/0001 (Autumn Garage coordinates cross-tool work without becoming a monorepo), doctrine/0003 and doctrine/0004 (tools compose through contracts and Conductor is the LLM routing peer), doctrine/0007 (shared branded surface), and journal/2026-05-06-alchemist-quintet-build-kickoff.md (the kickoff that framed Alchemist as the fifth family tool).

## Vision in one paragraph

A user files a GitHub issue on any garage-family repo and labels it `alchemist-dispatch`. Within ~15 minutes, Alchemist (running as a Railway cron) sees the label, pulls the issue body, calls `conductor exec` with workspace-write tools to produce a candidate fix on a feature branch, runs `touchstone review` over the diff, and opens a PR with the issue, the fix, and the review summary in the body. The human merges (or rejects). Alchemist never merges autonomously, never iterates against the reviewer's comments, and never speaks to the LLM directly — every model call goes through Conductor, every quality judgment goes through Touchstone. Alchemist is the messenger between GitHub and the rest of the garage.

## Why now

Three pressures converge on the same answer:

1. **Cross-repo "issue → PR" is a pattern, not a one-off.** The four quartet repos plus autumn-mail plus future repos all want the same loop. Building it once, configurably, beats N copies of a bash script.
2. **Conductor v0.3.x already ships the agentic loop.** `conductor exec --tools Read,Edit,Write,Bash --sandbox workspace-write` is the engine. Alchemist does not need to invent it.
3. **Touchstone already ships the review gate.** The codex/local-reviewer cascade is exactly the quality check we want before opening a PR. Alchemist does not need to invent that either.

What's missing is the glue: a process that watches GitHub, hands work to Conductor, hands diffs to Touchstone, and writes a PR. That glue is small enough to not be Sentinel — Sentinel's value is in *autonomous-finds-its-own-work* loops with budgets, reviewers, and backlogs. Alchemist is *reactive* — one issue, one fix, one PR, exit. Different posture, different shape, different lifecycle.

## Role in the family

| Tool | Role | Posture |
|---|---|---|
| **Touchstone** | the ground — scaffolding + AI review gate | invoked on demand |
| **Cortex** | the spine — project memory protocol | shared substrate |
| **Sentinel** | the hands — autonomous worker with backlog | runs proactively in one repo |
| **Conductor** | the voice — capability-aware LLM router | called by every other tool |
| **Alchemist** | the transmuter — issue → reviewed PR | reacts to GitHub events across all repos |

Alchemist sits *opposite* Sentinel in the same plane: Sentinel pushes (proactively finds and ships work in one repo); Alchemist pulls (reactively shapes work that humans have already framed as issues, across many repos). They coexist; neither replaces the other.

## Composition rules (Doctrine 0001 / 0003 / 0004 still hold)

Alchemist composes by file/CLI contract, never by code import. Inside an Alchemist run, the only Python code is the orchestration loop. Everything load-bearing happens in subprocesses:

- `gh issue list/view/edit` — GitHub I/O
- `conductor exec --with <provider> --tools Read,Edit,Write,Bash --sandbox workspace-write --brief-file <path>` — the agentic fix loop
- `touchstone review --base <ref> --head <branch>` — the AI review pass
- `git checkout/commit/push` — branch state
- `gh pr create` — surface the result

If any of those CLIs is unavailable (older garage install, missing provider, etc.), Alchemist fails fast with a structured error pointing at the missing tool. No fallbacks, no shims, no silent degradation.

## Approach

Build Alchemist as a small Python CLI and Railway cron service that owns orchestration only. The core loop polls a configured allowlist of GitHub repositories for `alchemist-dispatch`, checks out a branch from the target repo's default branch, renders a complete issue brief, invokes Conductor for the agentic edit loop, invokes Touchstone for the review gate, pushes the result, and opens a PR for human review. State is tracked through GitHub labels plus a small persistent cache so repeated cron ticks are idempotent.

Keep model selection, provider credentials, review standards, git merge authority, and tool-specific project memory in their owning tools. Alchemist should call `gh`, `git`, `conductor`, and `touchstone` through their public CLI contracts and fail fast when any required contract is unavailable.

## Scope

**In scope (v0.1):**
- Watch a configured list of repos (default: every `autumngarage/*` plus `autumn-mail`).
- Poll mode only. No webhook server, no GitHub App. `gh issue list --label alchemist-dispatch` per repo, every N minutes.
- Single-shot per issue: one Conductor call, one Touchstone review, one PR. No reviewer iteration loop (that's Sentinel territory).
- PR body includes: the issue link, the Touchstone review summary verbatim, the Conductor cost log, and the agentic loop's transcript link (or the transcript inline if short).
- State tracking by GitHub label transition: `alchemist-dispatch` → (working) → `alchemist-shipped` or `alchemist-error`. Idempotent — re-runs do nothing on already-handled issues.
- Per-issue budget cap (`--budget '$2'` default, configurable per repo).
- **Deployed on Railway as an always-on cron-triggered service.** Local-only Alchemist is a development surface, not a finish line. v0.1 is not done until the cron is live in Railway, picking up labelled issues without a human running the CLI.
- Branded surface per Doctrine 0007 (wordmark, palette, attribution, banner contract).

**Out of scope (v0.1):**
- Webhook receiver / GitHub App / sub-poll latency.
- Multi-pass coder↔reviewer iteration (refer issue back to Sentinel if needed).
- Auto-merge — humans always merge.
- Cross-issue planning (no backlog, no prioritization, no scan/plan layer).
- Non-fix issue handling (questions, discussion, RFC) — out of scope; if the issue isn't fixable code, label it `alchemist-skip`.
- Customer / non-garage repo support — restricted to autumn-garage-family repos in v0.1 by allowlist (see "Sandboxing posture" below).

**Out of scope permanently:**
- Direct LLM provider calls. All model use goes through Conductor.
- Direct quality judgment. All review goes through Touchstone.
- Hosted SaaS for other users. Alchemist is for the garage's own dogfood loop, not a product.

## Core flow

```
loop every N minutes:
  for repo in configured_repos:
    for issue in gh issue list --repo <repo> --label alchemist-dispatch --state open:
      branch = "alchemist/issue-<N>-<slug>"
      transition_label(issue, "alchemist-working")
      checkout_or_clone(repo, branch_from=default_branch)

      brief = render_brief(issue.title, issue.body, repo.context)
      conductor exec --with <provider> --tools Read,Edit,Write,Bash \
                     --sandbox workspace-write --brief-file <brief>

      review = touchstone review --base <default_branch> --head <branch>
      git push origin <branch>
      gh pr create \
        --title "fix: <issue title> (#<N>)" \
        --body "<rendered: issue + review + cost + transcript>"

      transition_label(issue, "alchemist-shipped")
```

Errors at any stage flip the label to `alchemist-error` and post a comment on the issue with the structured error. No retries within Alchemist — humans triage failures and re-label `alchemist-dispatch` to retry.

## Configuration surface

A single config file, `~/.alchemist/config.toml` (or `/etc/alchemist/config.toml` for the Railway deploy), with this shape:

```toml
[alchemist]
poll_interval_minutes = 15
default_budget = "$2"
default_provider = "claude"   # passed to `conductor exec --with`
github_token_env = "GITHUB_TOKEN"

[[repos]]
name = "autumngarage/sentinel"
budget = "$3"   # override default for harder repos

[[repos]]
name = "autumngarage/cortex"

[[repos]]
name = "autumngarage/touchstone"

[[repos]]
name = "autumngarage/conductor"

[[repos]]
name = "henrymodisett/autumn-mail"
```

No env-var sprawl. Provider credentials live in Conductor's keychain (Doctrine 0005 — credentials by reference). Touchstone reads its own config from the target repo. Alchemist's config is purely about *what to watch* and *how much to spend.*

## Deployment — Railway (v0.1 scope, not a follow-on)

The always-on host is **Railway**, on a new project under the existing `henrymodisett@gmail.com` account, *separate from* the `daring-strength` project (which is outrider/vanguard customer infrastructure and out of scope per the memory note).

### Project + service shape

- **New Railway project**: `autumn-garage-alchemist` (or similar — final name a v0.1 detail).
- **One service** initially: `alchemist-cron`. Dockerfile-based image bundling `gh`, `git`, `conductor`, `touchstone`, and `alchemist` itself.
- **Cron schedule** on the service: `*/15 * * * *` (every 15 minutes). Each tick runs `alchemist run-once`, which iterates the configured repos, processes any labelled issues, and exits. No long-running daemon, no public endpoint, no inbound port.
- **Persistent volume** mounted at `/var/alchemist/state` — holds the per-issue cache (last-seen ETag, processed-issue marker, PR URLs) so re-runs are cheap and idempotent across container restarts.
- **Environment variables** (Railway-managed, not committed): `GITHUB_TOKEN` (fine-grained PAT scoped to the configured repo allowlist only), Conductor provider keys (`ANTHROPIC_API_KEY`, etc., or whatever Conductor's keychain layer requires when running headless), `ALCHEMIST_CONFIG` (path to mounted config TOML).
- **Logs** land in Railway's dashboard for short-term debugging. Per-cycle JSONL transcript additionally pushed to an `alchemist-runs` branch on this autumn-garage repo for grep-ability and longer retention.

### Bootstrap steps (the path from "vision plan" to "first live tick")

1. `railway init -n autumn-garage-alchemist` from the alchemist repo root once it exists. Confirm in `railway list` that the new project sits alongside `daring-strength` without colliding.
2. Add the cron service via `railway service` / dashboard. Wire the Dockerfile.
3. Provision env vars via `railway variable set` (or dashboard) for `GITHUB_TOKEN` + Conductor provider creds. The fine-grained PAT must be scoped at creation to *only* the watched repos — Doctrine 0005 (credentials by reference) plus the principle of least privilege.
4. Attach a Railway volume to `/var/alchemist/state` and seed it with an empty config (or commit a default to the image and let the volume hold mutable state only).
5. Set the cron schedule on the service (`*/15 * * * *` to start; tunable down to 5 min later if the autumn-garage issue cadence justifies it).
6. Deploy. First tick should produce an empty cycle (no `alchemist-dispatch` issues yet) and a clean log. Verify by manually creating a trivial labelled issue on one watched repo and waiting one tick.
7. End-to-end smoke test: a real-but-small fix issue (e.g., a typo in a README) on a garage repo. Inspect the resulting PR, the Touchstone review summary, the cost log, and the label transitions.

### Cost shape

Railway hobby usage on this is small: the cron container is dormant between ticks (~0% CPU, ~0 GB RAM cost), and active for a few minutes per labelled issue. Per-tick LLM cost is bounded by `--budget '$2'` per issue. With ~5 dispatch issues per month (the falsification floor), monthly LLM spend stays under $10; Railway compute under the hobby tier minimum.

### Upgrade path (deferred, not v0.1)

When 15-min latency starts to bite, the upgrade is purely additive:
- Add a second Railway service: `alchemist-webhook`, an always-on HTTPS endpoint that validates GitHub webhook signatures and writes the issue payload to the shared volume's "pending" directory.
- The cron service drains pending events first, then falls through to its existing label scan. Same image, same env, same allowlist.
- Optionally register a GitHub App for first-class webhook delivery (vs. per-repo webhook config).

None of that is v0.1. v0.1 is the cron, and the cron alone.

## Sandboxing posture

Alchemist runs Conductor's agentic loop with `workspace-write` inside a Railway container. The container is the sandbox. This is acceptable for autumn-garage-family repos because:

- The `GITHUB_TOKEN` is fine-grained and allowlisted to those repos only — Alchemist cannot push elsewhere.
- The container has no access to customer data, customer Railway, or customer credentials.
- Conductor's tools enforce path validation; Bash time/memory rlimits apply (`--sandbox strict` available if needed).

This is **not** acceptable for customer or third-party repos. If Alchemist's allowlist ever extends beyond garage-family repos, the sandbox layer has to harden — E2B, Firecracker, or per-issue ephemeral microVMs. Doctrine 0005 (credentials by reference) constrains this: customer credentials must never live in the Alchemist Railway service.

## Branding (Doctrine 0007 compliance)

Alchemist needs the standard branded surface before v0.1 ships:

- **Wordmark** — figlet `standard` font for "Alchemist", embedded as a string literal in `alchemist/banner.py`.
- **Hue family** — pastel-amber / pale-gold proposed: ANSI **222** primary (`#ffd787`), **230** subtitle (`#ffffd7`). This does not collide with Touchstone (peach 216/223), Cortex (aqua 152/159), Sentinel (sage 151/157), Conductor (periwinkle 147/183), or Autumn Garage (rose 181/217), and matches the alchemical-gold metaphor.
- **Tagline** — candidate: *"transmute issues into pull requests."* Final wording deferred until the README is drafted.
- **Attribution + banner contract** — same as the four existing tools; reference `~/Repos/conductor/src/conductor/banner.py`.

This needs ratification in Doctrine 0007 (color table extension) when Alchemist v0.1 ships. The doctrine itself flags small additive branding changes as edit-in-place, journal-the-change.

## Repo and release shape

Mirror Conductor's bootstrap:

- New repo: `github.com/autumngarage/alchemist`.
- Python (matches Cortex, Sentinel, Conductor; Touchstone is bash). Hatch-vcs versioning.
- Brew tap: `autumngarage/homebrew-alchemist`. `brew install autumngarage/alchemist/alchemist`.
- The shared `homebrew-bump.yml` workflow in this repo handles the tap update on release.
- The `/deploy` skill in this repo extends to Alchemist's release flow (one more entry in the survey).
- Independent release cadence — Doctrine 0001 still holds.

## Open decisions (do not pre-commit)

1. **Single provider or auto-route?** v0.1 uses a per-repo configured provider (`default_provider = "claude"`). Auto-routing via `conductor exec --auto` is a v0.2 concern — let's see if "always claude" is good enough first.
2. **Touchstone reject behavior.** If `touchstone review` returns `changes-requested`, does Alchemist (a) open the PR anyway with the review attached, (b) skip opening and label `alchemist-needs-human`, or (c) run one Conductor revision pass? Recommend (a) for v0.1 — keeps the "thin" promise; humans triage.
3. **Per-repo bootstrap.** Should each watched repo carry an `.alchemist/` directory (analogous to `.sentinel/`) with repo-specific config (budget overrides, label conventions, prompt augmentations), or is the central config sufficient? Recommend central config for v0.1; per-repo only if it earns its keep.
4. **Cortex journal integration.** Alchemist runs are a natural fit for Cortex T1.6-style journal entries (one per run, in the *target repo's* `.cortex/journal/`). Wire this in v0.1 or defer to v0.2? Lean v0.1 — it's cheap and the value is high.
5. **Naming of the dispatch label.** `alchemist-dispatch` is verbose. Alternatives: `transmute`, `fix-me`, `alchemist`. Recommend `alchemist-dispatch` for unambiguous opt-in.

## Success Criteria

- Alchemist v0.1 is released and installable with `brew install autumngarage/alchemist/alchemist`.
- A Railway cron service runs `alchemist run-once` on the configured schedule and exits cleanly when there are zero eligible issues.
- A labelled live test issue in an Autumn Garage family repo produces exactly one PR containing the issue link, Touchstone review summary, Conductor cost log, and transcript reference.
- Label transitions are idempotent: `alchemist-dispatch` moves to `alchemist-working`, then to `alchemist-shipped` or `alchemist-error`; reruns do not duplicate branches or PRs for a completed issue.
- Missing `gh`, `git`, `conductor`, `touchstone`, GitHub credentials, or provider credentials fail non-zero with an actionable error and do not silently fall back.
- Alchemist never merges autonomously; merge authority remains human-controlled through the target repo's normal PR gate.

## Work items

- [ ] Create `github.com/autumngarage/alchemist` with the Python package, CLI entry point, banner, tests, and release wiring.
- [ ] Implement config loading for the repo allowlist, budget defaults, provider defaults, GitHub token reference, and persistent state path.
- [ ] Implement the GitHub polling and label-transition loop using `gh` with idempotent branch/PR detection.
- [ ] Implement brief rendering and Conductor invocation through the public `conductor exec` contract.
- [ ] Implement Touchstone review invocation and PR body rendering with review summary, cost log, and transcript reference.
- [ ] Package the Railway cron container with `gh`, `git`, Conductor, Touchstone, and Alchemist.
- [ ] Run an end-to-end smoke test on a small labelled issue and verify the resulting PR, labels, logs, and failure behavior.

## Falsification clause

If, six months after Alchemist v0.1 ships, fewer than half of the closed dispatched issues across the watched repos resulted in a merged Alchemist PR (i.e., the human consistently rejects or rewrites the output), Alchemist failed to be useful and should be wound down. The fix would be in Conductor's agentic loop quality or in the brief-rendering, not in adding more autonomy to Alchemist.

If the dispatch label sees fewer than ~5 issues per month across all watched repos, Alchemist is solving a problem that doesn't exist at this scale, and should be deferred until the garage produces issues at a rate that justifies the always-on infrastructure. Polling on the user's laptop ad-hoc would suffice instead.

## Relationship to existing tools

- **Sentinel** is unaffected. Sentinel's `--from-issues` flag (a separate plan) is still worth shipping, because Sentinel-driven cycles are autonomous in a way Alchemist explicitly is not. The two tools end up overlapping in the "issue exists, agent ships PR" surface, but Alchemist is single-shot reactive while Sentinel is multi-pass proactive. They can both be wired up; the user picks per-repo.
- **Conductor** gains a new caller. No changes required to Conductor for v0.1 — the existing `exec --tools --sandbox` surface is already what Alchemist needs.
- **Touchstone** gains a new caller. Touchstone's `review` subcommand needs to be invokable headless against an arbitrary `<base>..<head>` range and emit structured output (JSON) — verify this is already the case before Alchemist v0.1 ships; if not, file an issue on Touchstone first.
- **Cortex** gains an optional journal author. Alchemist writes a T1.6-style cycle entry to the target repo's `.cortex/journal/` after each run, when `.cortex/` exists.

## What ratifies this plan

This plan is active. Outstanding ratification items: draft a successor Doctrine entry to ratify the quintet framing (parallel to Doctrine 0004 ratifying the quartet), and add an additive amendment to Doctrine 0007 for the Alchemist hue.
