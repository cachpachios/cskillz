# cskillz

My personal agent skills.

```bash
npx skills add cachpachios/cskillz
```

| Skill                                            | What it does                                                                                                                        |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| [review-copilot](skills/review-copilot/SKILL.md) | Grill-me for reviewing a PR. Walks the reviewer through the change's design decisions and ends with verified findings. Never posts. |
| [ship-review](skills/ship-review/SKILL.md)       | Turns review findings into a posted GitHub PR review, written in a short, question-led voice.                                       |
| [creview](skills/creview/SKILL.md)               | Reviews the branch (quality and bugs), fixes what removes a bug or shrinks the code, asks about the rest. Repeated runs converge.   |

They pair up: `/review-copilot` to form an opinion, then `/ship-review` to post it. `review-copilot` runs `creview`'s lenses for its micro pass, so install both.

These are opinionated. `ship-review` §2 hardcodes my review voice, so fork and edit it to make it yours.
