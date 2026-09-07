# Test philosophy

Vertex favors tests that make boundary failures explicit. A polished function-level suite is insufficient if tenant context, provider effects, or deployment compatibility can fail at the seams.

## Principles

- Test boundaries, not only functions.
- Treat negative paths as first-class behavior.
- Exercise retries and ambiguous completion deliberately.
- Make authority fail closed when current context cannot be established.
- Require cross-tenant and cross-store isolation cases.
- Share deterministic vectors across contract producers and consumers where possible.
- Do not treat provider acknowledgement as outcome-test completion.
- Reproduce generic historical failure classes as regression scenarios without publishing sensitive incident histories.
- Build deterministic harnesses for races such as stale generation and duplicate delivery.
- Test fresh installation, reinstall, logout, and session transitions as journeys.
- Treat version provenance and deployment compatibility as correctness concerns.

## Why deterministic scenarios matter

Many serious failures depend on timing. A deterministic harness can force the sequence: request dispatched, provider commits, response lost, retry begins; or worker starts under generation N, lifecycle advances to N+1, worker resumes. These tests make the intended containment observable without relying on random timing in a live environment.

## Testing evidence observed in the implementation snapshot

The private worktrees reviewed for this dossier include unit tests, integration tests, PostgreSQL-backed paths, migrations tests, contract/lifecycle tests, provider-oriented tests, browser journeys, Temporal replay/restart verification, and certification scripts in representative components. That evidence supports describing a layered test philosophy. It does not establish an aggregate coverage percentage, universal application adoption, or a permanent release state.

## Failure story: happy-path authorization

An authorization test that only asserts "an owner can act" misses the important questions: is the session current, does the membership still exist, is the selected installation current, did policy revision change, and is the workload bound to the intended app? The useful test matrix starts from the paths that must be denied.

## Design questions

**What makes a test valuable here?** It demonstrates containment at a boundary whose failure would otherwise produce an unauthorized action, cross-tenant result, lost work, duplicate effect, or false outcome.

**Why test deployment behavior?** A correct source tree paired with an incompatible worker, migration, or contract is still an incorrect system.
