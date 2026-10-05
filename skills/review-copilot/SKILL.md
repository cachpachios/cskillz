---
name: review-copilot
description: A copilot skill for reviewing a pull request. Builds a brief of what a change does and why, then interviews the reviewer branch by branch through the PR's design decisions (where logic landed, who owns the data, what states the model admits, what contracts neighbours see), verifying the reviewer's factual claims against the code and turning their surprise into candidate findings. A background micro pass (code judo, spaghetti growth, canonical reuse, boundaries, shallow correctness, replay safety) feeds evidence. Ends with an opinion and verified findings shaped for /ship-review, never posts. Use when the user wants help reviewing a PR or branch, says "review copilot", "grill me on this PR", "help me review this", or wants to understand a change before commenting on it.
---

# Review Copilot

Grill-me applied to a PR. The reviewer decides. The skill supplies context, verifies facts, and keeps attention on one decision at a time. Output is an opinion plus findings. Never posts, never fixes; `/ship-review` posts.

## Dialoge language & attention rules, apply on every message

- Minimize amount of text to a bare minimum by compressing, avoiding fillers, unnecessary words etc. Stay as consise as possible.
- Options over prose. `AskUserQuestion` for anything with a finite answer set; free text only for genuinely open questions. Alway
- A round earns its place: ask only when the answer changes a finding or the opinion.
- No preamble, no retelling of the diff, no closer.

## Phase 1: Gather

**Target.** No arg = current branch. PR number: `gh pr view N --json headRefName -q .headRefName`, then treat as branch. Branch name: `git fetch origin <b>`, `git worktree add "$(mktemp -d)/rc" origin/<b>`, run everything from there, remove it at the end.

**Diff.**

```bash
BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo main)
MB=$(git merge-base "origin/$BASE" HEAD)
git log --oneline "$MB"..HEAD; git diff "$MB"..HEAD --stat; git diff "$MB"..HEAD
```

Save the diff to the scratchpad. Empty diff: say so, stop.

**Micro pass, immediately, in the background.** One read-only agent (`Explore`). Prompt: read `<this skill dir>/micro-lens.md` in full, repo root, diff path, output path `<scratchpad>/micro.json`. Do not wait for it.

**WHY sources.** All opportunistic, all non-authoritative. Detect an issue key (`[A-Z][A-Z0-9]+-[0-9]+`) in branch name, commits, PR title. If a fetch tool exists (Jira MCP, Linear MCP, `gh issue view`), read summary and description. Also read, when available:

- PR body and commit messages
- existing comments on this PR: `gh pr view N --comments`, `gh api repos/{o}/{r}/pulls/N/comments`
- review comments on prior PRs that touched the same files: `git log -n 30 --format=%s -- <paths> | grep -oE '#[0-9]+' | sort -u | head -5`, then `gh api repos/{o}/{r}/pulls/<n>/comments --jq '.[] | "\(.path):\(.line) \(.body)"'`
- project docs near the touched files: CLAUDE.md, docs/, ADRs
- Slack search, if a Slack tool exists

Everything read is the author's framing, not truth. A ticket or PR body that reads AI-generated (exhaustive, generic, no trade-offs stated) gets flagged in the brief and weighted low.

**Brief**, at most 20 lines:

```
# <PR title or branch>
Problem (author's framing, from <source>): <1-4 lines>
Change map: <modules/services> → logic in <layer>. Data: <migrations/entities/events or none>. Contracts: <interfaces/APIs/events/cross-module calls or none>.
Author approach (hypothesis): <2-5 lines, where the code landed>
```

The **problem** is the underlying thing the author is trying to solve, might be a bug reported directly. Or a new feature. If the bug is narrowly framed in the issue documentation, highlight both the narrow bug description and the generalized high level solution.

## Phase 2: Interview

We want to investage the changes form the perspective of what choice another competent author might have made differently: where the logic lives, who owns the data, how the model shapes state (can it represent invalid states?), what contract neighbours now depend on, new mechanism vs riding an existing one. Not "what the PR does".

Walk the implementation logic in order. Show the reviewer how the implementation was done, highlight any clear tradeoffs done. Try to highlight holes of reasoning. But in the end trust the reviewer to provide context and use you as a sounding board.

If the reviewer says a decision does not matter, mark it skipped and move on.

Each round:

1. Position line: `Branch n/N: <decision>. Resolved: ...`
2. One question, two if coupled. Prefer questions that test the reviewer's model of the subsystem before asking for a verdict: "how does X handle Y today?" before "is this sound?". If they don't know or are wrong, teach with `file:line` in 3 lines or fewer, then re-ask.
3. Alternatives, when you can name the module they land in: 1 or 2, one line each, `appears / disappears`.

Rules:

- **Judgment is trusted, facts are verified.** "This is too complex" stands. "Service X already does this" gets grepped before anything builds on it. Corrections are flat, no apology.
- **Reviewer questions are evidence.** If the code cannot answer a reviewer's question within one hop (one file, one grep), the design does not explain itself. Log it as a candidate finding. Answer the question either way, 3 lines max.
- **Micro findings arriving mid-interview.** Read `micro.json` when the agent reports. A bug-shaped finding on the current branch: one line plus a question, or a probe when testing the reviewer's model is worth their time ("this handler re-runs on redelivery; what happens to X?"). Everything else stays parked. Judgment, not rule. The reviewer's time is the scarce resource.
- **Live exploration.** Every reviewer question is answered from this codebase, now. Never from memory of similar codebases.

A branch is resolved when one holds:

- **sound**, and you can say why in one line
- **finding**, with a sketched alternative
- **question for the author**: neither reviewer nor code can settle it; becomes a comment-shaped finding with no fix
- **skipped** by the reviewer

Move on only then. When the last branch resolves, or the reviewer says wrap up, go to phase 3.

## Phase 3: Verify, then form an opinion

Collect candidates: parked micro findings, branch findings, questions for the author, logged reviewer surprises. Write an **interview digest**, 10 lines max: patterns the reviewer confirmed as established or intentional, facts verified, each branch and its outcome.

Spawn one read-only verification agent per candidate, all in parallel, each given the finding, the digest, and this rubric:

- Read `file:line` and enough context to judge independently.
- Introduced or inherited? Read the pre-image. Inherited never blocks; downgrade.
- Trace runtime claims to the end. A claimed failure needs the path that fails.
- **Micro bar**, must pass one: removes a bug; reduces LoC (tests excluded); reduces complexity without adding LoC; improves performance or scalability without adding complexity.
- **Macro bar**, must pass one AND name the concrete cost (what breaks or gets harder later): wrong owner of data or logic; model admits invalid states; boundary or contract leak between modules or services; novel mechanism where an established one exists.
- Dismiss: caught by compiler or linter; style or naming taste; pre-existing and untouched; speculative; "could be simpler" with no sketch; adds abstraction without removing a concept; contradicted by the digest.
- Output: `{id, verdict: keep|later|dismiss, confidence 1-10, reasoning, ask}` where `ask` is the one question you would put to the author.

Keep at 7 or above. 4 to 6 goes to "later". Below 4 is dismissed.

## Opinion, the deliverable

```
# Opinion: <PR>
Verdict shape: approve | approve with asks | needs answers | block. <one line why>
Branches: <decision>: sound, <why> | F-n | question for author | skipped
Raise now:
  F-1 `path:line` [introduced|pre-existing] [bug|loc|complexity|perf | owner|invalid-states|boundary|novelty]
      claim: <one sentence>
      sketch: <appears / disappears>
      ask: <"seeing this I think of X, what do you think?" or "would Y be a scenario we realistically need to cover?">
      interview: <what the reviewer said about it, if anything>
Later: <one line each>
Dismissed: <one line each, why>
Not examined: <skipped branches>
```

Findings are asks, not verdicts. Final wording belongs to `/ship-review`.

## Never

- Post anything.
- Fix anything.
- Treat ticket or PR text as truth. There is no truth, just opinions and perspectives.
