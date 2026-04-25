---
name: deploy
description: Survey the four autumn-garage tools (conductor, cortex, sentinel, touchstone) for unpublished commits past their latest tag and ship a release for each one that has pending work. Each tool's release-published event auto-bumps the corresponding homebrew-<tool> tap formula via .github/workflows/release.yml → autumn-garage/.github/workflows/homebrew-bump.yml@v1. Use when the user says "deploy", "ship the unpublished work", "release everything pending", or asks for a tap-version bump on one or more of the tools.
---

# /deploy — release any unpublished updates to the autumn-garage tools

The four tools (conductor, cortex, sentinel, touchstone) each have a paired Homebrew tap (`homebrew-<tool>`) updated automatically by GitHub Actions whenever a release is published in the source repo. This skill detects per-tool drift between the latest tag and `origin/main`, and cuts releases accordingly.

## Per-tool release shape

| Tool | Local clone | Versioning | Release command |
|---|---|---|---|
| conductor | `~/Repos/conductor` | `hatch-vcs` (derived from tag) | `git tag vX.Y.Z && git push origin vX.Y.Z && gh release create vX.Y.Z --generate-notes` |
| cortex | `~/Repos/cortex` | manual bumps in `src/cortex/__init__.py` + `pyproject.toml` (+ `uv.lock` regen + `README.md` version refs), shipped via PR | bump all files on a release branch, push, open PR with `scripts/open-pr.sh --auto-merge`, then tag the merge commit + `gh release create` (cortex blocks direct commits to `main`) |
| sentinel | `~/Repos/sentinel` | `hatch-vcs` | same as conductor |
| touchstone | `~/Repos/touchstone` | `VERSION` file bumped by helper | `bin/touchstone release --patch \| --minor \| --major` (this helper does VERSION bump + commit + tag + push + `gh release create` in one call — do NOT run the steps separately) |

Touchstone's helper exists; do not duplicate its work. Conductor/cortex/sentinel get tagged manually.

## Run order

### 0. Pre-flight: cross-repo visibility + Actions access (do this once, before surveying)

The release-published event in each tool repo calls `autumn-garage/.github/workflows/homebrew-bump.yml@v1` as a reusable workflow. GitHub blocks the call silently if either of these isn't right, leaving GitHub Releases in place but skipping the tap bump.

```bash
# 1. autumn-garage must allow its workflows to be called from the org's repos
gh api repos/autumngarage/autumn-garage/actions/permissions/access --jq .access_level
# expected: "organization" (or "all"). If "none", run:
#   gh api -X PUT repos/autumngarage/autumn-garage/actions/permissions/access -f access_level=organization

# 2. Visibility constraint: a public consumer cannot call a private source workflow.
#    All five repos (autumn-garage + the four tools) must be at the same visibility,
#    OR autumn-garage must be at least as public as the tools.
GARAGE_VIS=$(gh repo view autumngarage/autumn-garage --json visibility -q .visibility)
for t in conductor cortex sentinel touchstone; do
  TOOL_VIS=$(gh repo view autumngarage/$t --json visibility -q .visibility)
  echo "$t: tool=$TOOL_VIS  garage=$GARAGE_VIS"
done
# If any tool is PUBLIC while garage is PRIVATE → chain will break. Surface and stop.
```

If either check fails, surface it to the user and stop. Don't try to "fix" by toggling visibility — that's a user decision.

### 1. Survey (do this first, before any destructive action)

For each tool, in `~/Repos/<tool>/`, run:

```bash
git fetch --tags origin                        # get latest tags + main
LATEST_TAG="$(git tag -l --sort=-v:refname 'v*' | head -1)"
PENDING="$(git log "$LATEST_TAG..origin/main" --oneline)"
```

If `PENDING` is empty → tool is up to date, skip it in the deploy step.

If non-empty → record (tool, latest_tag, pending commits) for the next step.

### 2. Pre-flight checks (per tool with pending work)

Before touching a tool, verify:

- Working tree is clean: `git status --porcelain` empty
- On `main` branch: `git rev-parse --abbrev-ref HEAD` == `main`
- Local main is in sync with origin: `git rev-list --left-right --count origin/main...main` returns `0\t0`

If any check fails, surface the problem to the user and skip that tool. Don't try to "fix" with destructive commands — let the user decide.

### 3. Suggest a version bump per pending tool

Parse the pending commit messages and recommend a bump:

- Any `feat!:`, `BREAKING CHANGE:`, or `:` after a type-with-`!` → **major**
- Otherwise any `feat:` → **minor**
- Only `fix:` / `chore:` / `docs:` / `refactor:` / `test:` / `ci:` / `build:` / `style:` → **patch**

Print the pending commit list and the suggested bump. **Always confirm with the user before cutting** — don't auto-deploy. Show:

```
=== <tool> ===
Latest tag: vX.Y.Z   (released YYYY-MM-DD)
Pending: 5 commits since latest tag
  abc1234 feat: …
  def5678 fix: …
  …
Suggested bump: minor → vX.Y+1.0
```

The user may override (e.g., "patch instead", "skip this one"). Accept their direction.

### 4. Cut the release per tool

Use the table above. For conductor / sentinel:

```bash
cd ~/Repos/<tool>
git tag vX.Y.Z
git push origin vX.Y.Z
gh release create vX.Y.Z --generate-notes
```

For cortex (PR-based — direct commits to `main` are blocked by a pre-commit hook):

```bash
cd ~/Repos/cortex
git checkout -b chore/vX.Y.Z-release

# Bump version in all four places — Codex review will block if README is stale.
# - src/cortex/__init__.py: __version__ = "X.Y.Z"
# - pyproject.toml:         version = "X.Y.Z"
# - README.md:              every "v<prev>" / "currently on v<prev>" version ref
# - uv.lock:                regenerate with `uv lock`
uv lock
git add src/cortex/__init__.py pyproject.toml README.md uv.lock
git commit -m "chore: vX.Y.Z release prep — bump version"
git push -u origin chore/vX.Y.Z-release

# Auto-merging PR — Codex reviews; merges on clean.
scripts/open-pr.sh --auto-merge \
  --title "chore: vX.Y.Z release prep — bump version" \
  --body "Bumps version to X.Y.Z to ship pending main commits."

# After merge, tag the resulting commit on main (NOT the branch HEAD — the PR squash-merges).
git fetch origin main
git tag vX.Y.Z origin/main
git push origin vX.Y.Z
gh release create vX.Y.Z --generate-notes
```

If Codex blocks on stale README version refs, fix them on the same branch and push again; `--auto-merge` re-runs.

For touchstone:

```bash
cd ~/Repos/touchstone
TOUCHSTONE_NO_AUTO_UPDATE=1 bin/touchstone release --patch    # or --minor / --major
```

### 5. Verify the auto-deploy chain fired

For each released tool:

```bash
# The release-published event triggers .github/workflows/release.yml in the tool repo,
# which calls autumn-garage's shared homebrew-bump.yml. Check the run started.
gh run list --workflow=release.yml --repo autumngarage/<tool> --limit 1
```

Wait ~30–60s, then verify the tap got the new commit:

```bash
# Tap repos may not be cloned locally; query the API instead.
gh api "repos/autumngarage/homebrew-<tool>/commits?per_page=1" --jq '.[0] | "\(.sha[:7]) \(.commit.message | split("\n")[0])"'
```

The latest commit message should start with `<tool> X.Y.Z` (the format the shared workflow's `commit-message` template produces).

### 6. Final summary

Report back to the user, one line per tool:

- `✓ <tool>: vX.Y.Z released, tap bumped (homebrew-<tool>@<sha>)`
- `– <tool>: up to date (no pending commits)`
- `✗ <tool>: failed at <step> — <reason>`

If any tool failed mid-flight, the GitHub Release exists (via `gh release create`) but the tap may not have been bumped. The escape hatch:

```bash
gh workflow run release.yml --repo autumngarage/<tool> -f tag_name=vX.Y.Z
```

This re-fires the homebrew-bump workflow for an existing tag (idempotent — if the tap is already at that version, the action no-ops).

## Don't do

- Don't deploy without surveying first. Even if the user says "deploy everything," run step 1 and confirm.
- Don't combine tools' releases into one operation. Each release has its own tag, its own tarball, its own tap commit. They're independent.
- Don't bypass pre-flight checks (clean tree, on main, in sync). They exist because tagging a wrong commit is hard to undo.
- Don't skip the verification step. The release-published event fires reliably, but the workflow run can fail (PAT expired, tap repo permissions changed). Verify before declaring success.

## Required prerequisites (already in place as of 2026-04-25)

- `HOMEBREW_TAP_PAT` repo secret on each of the four tool repos
- `.github/workflows/release.yml` wrapper present in each tool repo, pinned `@v1` to autumn-garage's shared workflow
- `autumn-garage` tag `v1` exists, pointing at `.github/workflows/homebrew-bump.yml`
- `autumn-garage` Actions reusable-workflow access set to `organization` (see step 0)
- `autumn-garage` visibility ≥ each tool repo's visibility (a public tool cannot call a private workflow source — see step 0)

If any of these are missing, the auto-bump chain breaks silently. Surface that as an error rather than proceeding.
