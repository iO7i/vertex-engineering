# Failure model

Reliability work begins by naming failure classes at each boundary. The table below is a taxonomy and design expectation, not a claim that every listed containment is implemented uniformly across every application.

| Boundary | Failure | Risk | Expected containment | Expected evidence | Recovery direction |
| --- | --- | --- | --- | --- | --- |
| Identity | Expired/revoked session | Action after identity changed | Current-session evaluation, fail closed | decision/rejection record | reauthenticate/re-evaluate |
| Tenant | Wrong account/store/installation | Cross-tenant data or effect | Explicit scope and membership binding | scope/provenance mismatch | stop, isolate, investigate |
| Generation | Stale lifecycle work | Mutation under old authority | Current-generation fence | stale decision/receipt | reject or re-admit |
| Contract | Schema/version drift | Misread or unsafe request | Versioned contracts, compatibility tests | validation failure | deploy compatible consumer/producer |
| Evidence | Stale/missing/conflicted data | Unsupported conclusion | Readiness/coverage state | evidence snapshot/receipt | refresh, suppress, qualify |
| Brain | Stale or invalid decision | Bad recommendation served | Version binding and input validation | decision version | recompute or suppress |
| Serving | Stale/wrong projection | Product shows wrong tenant/state | Rebuildable scoped projection | projection revision | rebuild/repair |
| Authority | Wrong purpose/role/policy | Unauthorized mutation | Policy-scoped admission | allow/deny decision | deny, update policy/context |
| Execution | Crash/timeout/retry | Duplicate or lost work | Durable progress and idempotency | workflow/operation state | resume/reconcile |
| Provider | Rate limit/outage/partial effect | Delayed or ambiguous result | Bounded retry/backoff/readback | receipt + observed state | retry, defer, reconcile |
| Verification | Ack without effect | False success | Expected-versus-observed comparison | readback/outcome artifact | wait, repair, report mismatch |
| Deployment | Wrong build/migration mismatch | Behavior differs from release | provenance and readiness gates | build/version/readiness evidence | halt/rollback/remediate |

## Identity failures

Examples include expired sessions, stale handoffs, wrong store selection, and logout inconsistency. A safe action path does not treat an old UI state as continuing proof of membership. The reviewed Control Plane has explicit current-state checks for human sessions, memberships, installation memberships, lifecycle state, and policy. This dossier does not publish session protocol details.

## Evidence failures

Evidence can be missing, stale, duplicated, out of order, or contradictory. The MDP evidence model observed in the local snapshot includes coverage, missingness, reconciliation states, conflict state, and readiness assessment. The important behavior is to represent uncertainty, not to flatten it into a score without context.

## Authority and execution failures

An authority decision can become stale while work is queued. A worker can crash after an external system commits. The architecture contains these failures through scoped admission, generation fences, durable work, logical operation identity, and reconciliation—not by claiming exactly-once delivery or permanent permission.

## Provider and verification failures

Provider ACK, webhook, polling read, and merchant-visible state can disagree. The expected evidence is a bounded receipt and later observation, sufficient to classify a result without exposing raw payloads or credentials in this public record.

## Deployment failures

Correct source is not enough when an API, worker, migration, and contract package are deployed in incompatible combinations. Build identity, migration compatibility, readiness verification, and cross-component contract checks are treated as correctness evidence in the release methodology.
