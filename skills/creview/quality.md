# Quality lens

Strict maintainability review of a branch's changes. Read the diff, then the code around it. Never judge a hunk alone. Do not edit anything.

Be ambitious about deleting. Look for the code judo move: a reframing that keeps behaviour and makes branches, helpers, modes, flags or whole layers disappear. A refactor that moves complexity around without removing any is not a finding.

## Hunt

- **Judo.** A different ownership boundary, state model, or existing mechanism the change could ride, so its new conditionals vanish.
- **Spaghetti growth.** Conditionals bolted onto unrelated flows. One-off booleans or nullable modes threaded through layers. Special cases mid-way through a busy function. "Temporary" branches.
- **Reuse and layer.** A bespoke helper where a canonical one exists (grep first). Copy-pasted logic. Feature logic leaking into shared code or sitting in the wrong layer.
- **Indirection.** Thin wrappers, pass-through helpers, identity abstractions, generic magic hiding a simple data shape.
- **Boundaries.** Casts, `any`, `unknown`, needless optionality, ad-hoc object shapes, silent fallbacks papering over an invariant.
- **Orchestration.** Independent work serialized for no reason. Related updates that can leave state half-applied.
- **Size.** A file grown too large to scan or understand, bundling things that can be seperated out. Test code should not matter as much here as real production code.

A remedy that adds an abstraction, layer or file without removing more than it adds is not a finding. Name what disappears. A few high-conviction findings beat many.
