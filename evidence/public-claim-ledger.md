# Public claim ledger

This ledger keeps the dossier's claims bounded by what an outsider can inspect.

| Claim | Public evidence | Current status |
| --- | --- | --- |
| Evidence, reasoning, authority, execution, and verification are separate concerns | [Evidence, Reasoning, and Authority](../architecture/evidence-reasoning-authority.md), [System Overview](../architecture/system-overview.md) | Documented design principle |
| Stale installation generations must not silently retain authority | [Generation Fencing](../engineering/generation-fencing.md), [Failure Model](../reliability/failure-model.md) | Documented design principle |
| Provider effects require readback or reconciliation after ambiguous completion | [Provider Reconciliation](../engineering/provider-reconciliation.md), [Readback Verification](../engineering/readback-verification.md) | Documented design principle |
| The generalized primitives can reject stale authority and reconcile lost acknowledgement | [Faultline](https://github.com/iO7i/faultline), `pnpm demo` | Independently runnable synthetic proof |
| Vertex production scale, uptime, customer count, or universal exactly-once behavior | None published here | Deliberately not claimed |

The commercial implementation, customer data, provider credentials, private infrastructure, and application-specific business logic remain outside this repository.
