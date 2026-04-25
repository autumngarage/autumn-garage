---
name: deploy
description: Survey the four autumn-garage tools (conductor, cortex, sentinel, touchstone) for unpublished commits past their latest tag and ship a release for each one that has pending work. Each tool exposes `scripts/release.sh --patch|--minor|--major` and owns its own release shape; this skill is the uniform driver. Use when the user says "deploy", "ship the unpublished work", "release everything pending", or asks for a tap-version bump on one or more of the tools.
---

# /deploy — release any unpublished updates to the autumn-garage tools

Each tool (conductor, cortex, sentinel, touchstone) exposes `scripts/release.sh --patch|--minor|--major`. The helper handles whatever quirks that tool has (hatch-vcs vs manual version bumps, pre-commit hooks, README refs). This skill is the uniform driver — survey for pending work, get user confirmation on the bump, call each tool's helper, verify the tap chain.

## Run order

### 0. Pre-flight: cross-repo visibility (do this once, before surveying)

The release-published event in each tool repo calls `autumn-garage/.github/workflows/homebrew-bump.yml@v1` as a reusable workflow. GitHub blocks the call silently if the source repo is less public than the consumers, leaving GitHub Releases in place but skipping the tap bump.

```bash
GARAGE_VIS=$(gh repo view autumngarage/autumn-garage --json visibility -q .visibility)
for t in conductor cortex sentinel touchstone; do
  TOOL_VIS=$(gh repo view autumngarage/$t --json visibility -q .visibility)
  if [ "$GARAGE_VIS" = "PRIVATE" ] && [ "$TOOL_VIS" = "PUBLIC" ]; then
    echo "BLOCKED: $t is PUBLIC but autumn-garage is PRIVATE — workflow chain will silently fail."
    exit 1
  fi
done
```

If the check fails, surface it and stop. Don't try to "fix" by toggling visibility — that's a user decision.

### 1. Survey

```bash
for tool in conductor cortex sentinel touchstone; do
  cd ~/Repos/$tool
  git fetch --tags origin >/dev/null
  LATEST="$(git tag -l --sort=-v:refname 'v*' | head -1)"
  PENDING="$(git log "$LATEST..origin/main" --oneline)"
  # record (tool, LATEST, PENDING) for tools where PENDING is non-empty
done
```

### 2. Suggest a version bump per pending tool, confirm with user

Parse the pending commit messages:

- Any `feat!:`, `BREAKING CHANGE:`, or type-with-`!` → **major**
- Otherwise any `feat:` → **minor**
- Only `fix:` / `chore:` / `docs:` / `refactor:` / `test:` / `ci:` / `build:` / `style:` → **patch**

Show the user:

```
=== <tool> ===
Latest tag: vX.Y.Z
Pending: 5 commits
  abc1234 feat: …
  def5678 fix: …
Suggested bump: minor → vX.Y+1.0
```

User confirms or overrides per tool. **Don't auto-deploy without confirmation.**

### 3. Cut releases — uniform loop

Each tool's `scripts/release.sh` runs its own pre-flight (clean tree, on main, in sync with origin) and handles its own release shape. The skill just calls it.

```bash
for tool_and_bump in $confirmed; do
  cd ~/Repos/$tool
  scripts/release.sh --$bump
done
```

If a helper fails, surface the error and stop — don't move to the next tool until the user decides.

### 4. Verify the tap chain fired

For each tool released:

```bash
gh run list --workflow=release.yml --repo autumngarage/$tool --limit 1
```

Wait ~30–60s, then:

```bash
gh api "repos/autumngarage/homebrew-$tool/commits?per_page=1" \
  --jq '.[0] | "\(.sha[:7]) \(.commit.message | split("\n")[0])"'
```

The latest tap commit message should start with `<tool> X.Y.Z`.

If a workflow run failed, the GitHub Release exists but the tap may not be bumped. Re-fire:

```bash
gh workflow run release.yml --repo autumngarage/$tool -f tag_name=vX.Y.Z
```

Idempotent — if the tap is already at that version, the action no-ops.

### 5. Final summary

One line per tool:

- `✓ <tool>: vX.Y.Z released, tap bumped (homebrew-<tool>@<sha>)`
- `– <tool>: up to date (no pending commits)`
- `✗ <tool>: failed at <step> — <reason>`

## Don't do

- Don't deploy without surveying first. Even if the user says "deploy everything," run step 1 and confirm.
- Don't combine tools into one operation. Each release is its own tag, tarball, tap commit. They're independent.
- Don't reach into a tool's internals. If `scripts/release.sh` doesn't do what you need, fix the helper in the tool's repo — don't work around it from this skill.
- Don't skip step 4. The release-published event normally fires reliably, but the workflow can fail (PAT expired, tap permissions changed, source/consumer visibility mismatch). Verify before declaring success.

## Required prerequisites (verified 2026-04-25)

- `HOMEBREW_TAP_PAT` repo secret on each of the four tool repos
- `.github/workflows/release.yml` wrapper present in each tool repo, pinned `@v1` to autumn-garage's shared workflow
- `autumn-garage` tag `v1` exists, pointing at `.github/workflows/homebrew-bump.yml`
- `autumn-garage` Actions reusable-workflow access set to `organization` (only relevant if autumn-garage becomes private — public repos don't enforce this)
- `autumn-garage` visibility ≥ each tool repo's visibility (see step 0)
- `scripts/release.sh` exists and is executable in each of the four tool repos

If any of these are missing, the chain breaks. Surface as an error rather than proceeding.
