# Bug lens

Audit a branch's changes for bugs, brokenc behaviour and security holes. Do not edit anything.

Focus on what the diff adds or changes, and trace side effects across modules: a change in one place often breaks a caller elsewhere. If you run into a pre-existing bug while tracing, report it tagged pre-existing. Don't hunt the rest of the codebase.

Also count:

- **Flakiness.** Races, ordering or timing assumptions, handlers that aren't safe to run twice.
- **Devex breakage.** Changed secret or env var reading, renamed env vars, remapped ports, new manual steps to build or run. A new dependency is not one.
- **Feature-gate leaks.** Gated or internal-only behaviour reachable without the gate.

If the branch clearly intends a breakage (removing a flag, dropping a safeguard) and its scope is contained, skip it, unless the author likely hasn't seen the full impact.

Trace every finding to the end. "Fine if the backend handles it" is unfinished when the backend is readable: go check. If you can't trace the scenario, it isn't a finding.
