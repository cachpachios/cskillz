---
name: ship-review
description: Turn code review findings that already exist in context into a posted GitHub PR review written in the user's voice. Grills the user finding-by-finding on severity and inclusion, re-verifies each finding against repo norms first, drafts short question-led comments, then posts with inline anchors after explicit approval. Use when findings from a review skill (thermo-nuclear-code-quality-review, quality-review, security-review, code-review) are in context and the user wants them turned into an actual PR review, or says "post this as a review", "comment these on the PR", "put this on github".
---

# Ship Review

Takes findings that already exist and turns them into a posted review. Does **not** produce findings.

If there are no review findings in context, stop and say so. Do not audit the diff yourself to fill the gap.

## 1. Grill for severity and inclusion

The user decides what ships. You supply context and a recommendation.

- Use `AskUserQuestion`. Batch 1-3 findings per round, grouped by what couples them (same subsystem, same root cause). Provide a context ahead of the question to give altitude, as reviewer it is hard to know every line and its context of a large codebase.
- Recommended option, if present, first, labelled `(Recommended)`.
- Almost always offer "Drop it" as an option, with the honest case for dropping.
- Give the context they may not have: what the mechanism is, what the blast radius is, whether it is pre-existing. Two or three sentences.
- Offer severity _shapes_, not just yes/no: flag and ask if intentional / note without any ask / drop.
- Nits get one batched question at the end, multi-select.
- Answer their questions straight. If they ask whether something buys anything and it does not, say it does not.

**Important!** Prefer changes that **reduce** complexity, preferably with reduced line count as an effect. Avoid adding unnecessary layers, indirection, or verbosity, unless there is a critical and likely bug that would be solved by it.

## 2. Draft in the user's voice

- **Two sentences: one observation, one question.** A third has to earn it, a fourth is a rewrite. Dont restate a mechanism the author can read in their own diff, name it and stop ("read-before-mutate, then re-prune on 167"). A restructure proposal is one clause, not a paragraph.
- **Ask, do not assert.** "Intentional?" "Align?" "Or whats your thoughts?" "...right?" A question invites the author's context; a statement invites defending.
- **No emdashes. Ever.** Commas, periods, or parens.
- **Casual and lowercase-ish.** Contractions without apostrophes read natural here: doesnt, thats, havent, isnt. "Hmm," to open a doubt. abit, mby, rn.
- **`nit:` prefix** for cosmetics. "Not blocking" / "fine as a followup" for optional asks.
- **Show real uncertainty where it exists.** "Dont know the legacy data well enough to say." "Dunno X well." Never fake it, and never hedge a verified fact.
- **Say when something is pre-existing.** It changes how the author hears it.
- Direct about real problems ("This is spaghetti.") but never dismissive of the person.
- Sparing emoji: 👍 🤔 😄
- Credit what the PR got right. Vague praise reads as filler.

Try to aim for a single short sentence/question per comment. Short and sweet over verbosity! People can infer context themselves that is in the code.

The body should be very short, more often than not it can be absent unless there is something structural about most comments.

## 3. Verdict bar

- **APPROVE with comments** is the default, including for substantive optional asks the author can take or leave.
- **COMMENT** when something genuinely needs an answer or addressing before merge. Or when quality is so low that it warrants a real rereview before merge.
- **REQUEST_CHANGES** only for damaging issues: data loss, security, a new defect with real consequences that this PR introduces. Inherited warts never reach this bar.

Summary body is optional, but if needed follows the same voice. Max a line or two.

## 4. Approve, then post

Cut before you present. Reread each comment and delete every clause that doesnt change the ask: justification for the ask, restatement of the diff, softening preamble. If nothing got shorter, you didnt cut.

Present every comment verbatim with its `path:line` anchor, plus the summary body and verdict. Then ask for the go via `AskUserQuestion` with a "let me edit first" option. Never post unasked.

Anchor lines must exist in the diff or GitHub silently drops the comment. Read the file with line numbers to pin anchors; do not count by eye.

```bash
gh pr list --head "$(git branch --show-current)" --state all --json number,url
# write review.json to the scratchpad: {event, body, comments:[{path, line, side:"RIGHT", body}]}
gh api /repos/OWNER/REPO/pulls/N/reviews -X POST --input review.json -q '{state, url: .html_url}'
```

Then read back what actually landed, and report any anchor that went missing:

```bash
gh api "/repos/OWNER/REPO/pulls/N/comments?per_page=100" -q '.[] | select(.user.login=="LOGIN") | "\(.path | split("/") | last):\(.line)"'
```

Close by telling the user what shipped and what got dropped, and flag anything worth a ticket rather than a PR fix.
