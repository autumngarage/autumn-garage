# Stacked squash-merges broke; bundled recovery landed all three tools

**Date:** 2026-04-18
**Type:** incident
**Trigger:** T1.2
**Cites:** https://github.com/autumngarage/touchstone/pull/50, https://github.com/autumngarage/cortex/pull/20, https://github.com/autumngarage/sentinel/pull/75

> User clarified they don't review diffs — Touchstone's PR-review discipline was overhead they wanted removed. First attempt to bulk-merge the 12 stacked PRs failed because squash-merging stacked PRs collapses their ancestor commits into each other's branches. Recovery: close the stacked PRs, push each tool's tip branch as a single bundled PR against main, squash-merge those. All three trio repos now have R1-R4 (and T1.6 for Sentinel) on main.

## Impact

- Nothing shipped that shouldn't have; the content was correct.
- Time cost: ~20 min of recovery work across three repos.
- Three PRs superseded cleanly (cortex #16-#19, sentinel #71-#74); one tool partially landed normally (touchstone #46) and had its stacked tail recovered via a rebased bundle (touchstone #50).

## Timeline

1. User: "I don't need to review diffs. Is this something Touchstone is recommending because we can remove it?"
2. I flipped the default: future dispatches use `scripts/open-pr.sh --auto-merge`.
3. Tried to bulk-merge the 12 open PRs in stack order via `gh pr merge --squash`.
4. First Touchstone merge (PR #46, R1) succeeded. `--delete-branch` removed `feat/new-project-r1-improvements`.
5. Second Touchstone merge (PR #47, R2) failed: "not mergeable: the merge commit cannot be cleanly created." GitHub's auto-retarget of #47 to main (after #46's base branch was deleted) produced a diff that couldn't cleanly squash — #47's commits included #46's original (unsquashed) commits, which conflicted with main's single squashed commit.
6. Third Touchstone merge (PR #48, R3) "succeeded" — but merged INTO #47's branch, not into main, because #48's base was #47's branch. Same mistake for #49.
7. Checked main: only R1 landed. R2-R4 were stranded.
8. Pivot: for each tool, find the tip branch (`feat/registry-...` for touchstone, `feat/stub-guidance` for cortex, `feat/cortex-t16-integration` for sentinel), rebase onto current main (dropping R1 for touchstone since it's already there), open ONE PR per tool against main.
9. Bundled PRs: touchstone #50 (R2+R3+R4), cortex #20 (R1+R2+R3+R4), sentinel #75 (R1+R2+R3+T1.6). All three squash-merged cleanly.
10. Closed the original stacked PRs with a superseded-by comment.

## What went wrong (root cause)

**Stacked PRs + `gh pr merge --squash` is an unsound combination.** The squash collapses commits into one on main, but the stacked PR's branch still has the original un-squashed commits as ancestors. When GitHub auto-retargets the stacked PR to main after the base branch is deleted, the diff is computed against a tree that git doesn't recognize as ancestor-equivalent — hence "merge commit cannot be cleanly created."

The `gh pr merge --auto --squash` flag would have faced the same problem, just deferred.

## What worked

**Single-PR-per-tool bundled against main.** Each tool's tip branch already contains the full stack of R1-R4 commits linearly. Targeting a PR from that tip directly at main (skipping the stack hierarchy) produces a clean 3-5-commit PR that squashes cleanly.

## Lesson — codify this

For future multi-round work across a repo, prefer one of these patterns:

1. **Bundle from the start.** Each round's PR is a fresh branch off main with N commits replaying the prior rounds' work. No stacking. Reviewer sees the whole round as one PR. Simpler merge.
2. **`open-pr.sh --auto-merge` per PR, serialized.** Each PR lands before the next is opened. No stack exists. But slower — each round waits for the previous to land.
3. **True stacked PRs with Graphite or similar.** Actual stacked-PR tooling (graphite.dev, gerrit) handles the squash-merge-ladder cleanly because it rewrites parent pointers as each level lands. `gh pr merge` doesn't. Using real stacked-PR tools would be a third option, but adopting a new tool for this is heavier than the first two options.

## New TODOs surfaced

- [ ] Touchstone: document the "don't stack PRs with `gh pr merge --squash`" gotcha in principles/git-workflow.md. Alternative: auto-detect stack context in `open-pr.sh` and warn the user.
- [ ] Coordination playbook: when the user says "ship it all," prefer bundled rounds from the start rather than stacked. Faster review, cleaner merge, fewer moving parts.

## Consequences / action items

- [x] All three tools on main with R1-R4 (+T1.6 for sentinel).
- [ ] `brew upgrade` the tools locally to pick up the merged versions.
- [ ] Regression pass against the brew-installed binaries to confirm the end-to-end story works on shipped artifacts.
- [ ] Autumn-mail first real cycle.
