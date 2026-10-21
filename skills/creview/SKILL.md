---
name: creview
description: Reviews the current branch with two fresh-context subagents (code quality, bugs), fixes findings that remove a bug or shrink the code, and asks about the rest. Built so repeated runs converge instead of drifting beyond the change. Use when the user wants a review-and-fix pass, says "creview" or "review and fix", or a change is done and should be tightened before a PR.
---

# creview

Review, then fix. Fewer lines is the goal, and only a bug earns more. That pressure is what makes repeated runs converge.

## 1. Review

Base: `git merge-base origin/<default branch> HEAD`. The diff is base to working tree, uncommitted and untracked files included. Empty diff: say so, stop.

Spawn two read-only subagents in parallel. Each prompt holds:

- the path to its lens: `<this skill's directory>/quality.md` or `bugs.md`, read in full
- the repo root and base SHA
- §2 and the finding format below, verbatim
- findings dropped or declined earlier in this session: do not raise again

No subagents available: apply both lenses yourself.

```
- path:line | bug|perf|loc|complexity | introduced|pre-existing
  claim: one sentence
  proof: bug/perf: inputs or state → wrong result. loc/complexity: what disappears, net lines.
```

## 2. The bar

A finding must do one of:

1. Fix a bug, flakiness, or anything else that hurts users or operators (perf, data loss, security). Needs a concrete failure scenario. "Could happen" is not one.
2. Reduce production lines. Tests don't count.
3. Reduce complexity without adding lines. Added lines are the usual sign of overengineering.

Only 1 may add lines. Never cut lines by golfing or by deleting tests, validation or error handling. Drop the rest: style, naming, speculative, already decided.

## 3. Fix or ask

Verify each finding in the code before acting. Re-trace every bug scenario yourself.

Fix it when the fix is obvious and contained. Ask (`AskUserQuestion`, batched, after fixing) when it is opinionated, high impact, cuts against the repo's conventions, or lands outside the diff, pre-existing bugs included.

A bug fix gets one regression test where the code already has tests nearby. Run the repo's fast checks on what you touched. Revert a fix that breaks them.

## 4. Report

```
creview: N fixed, N asked, N dropped. Net production LoC: -X
Fixed: path:line, one line each
Dropped: one line each, why
```
