# Certification strategy

Tests passing answers a narrow question: did the tested code behave as expected in that run? Release confidence needs a chain of evidence for the behaviors the system actually depends on.

```text
SOURCE
  |
BUILD
  |
UNIT
  |
INTEGRATION
  |
CONTRACT COMPATIBILITY
  |
DATABASE
  |
SCENARIO HARNESS
  |
AUTHENTICATED JOURNEY
  |
PROVIDER READBACK
  |
DEPLOYMENT PROVENANCE
  |
RELEASE EVIDENCE
```

This is a methodology, not a statement that every release has satisfied every rung or that a successful run is a guarantee of production behavior.

## Layers of evidence

### Source and build

Type checks, generated-artifact checks, package compatibility, and build provenance establish a reproducible candidate. Source review alone cannot establish runtime authority behavior or provider semantics.

### Unit and integration

Unit tests target deterministic rules: contract parsing, scope validation, policy predicates, idempotency behavior, and state transitions. Integration tests exercise component seams such as database persistence, HTTP adapters, and provider-normalization boundaries.

### Contract and database

Shared Contracts are a first-class compatibility boundary. The local implementation snapshot contains contract tests, lifecycle/merchant-data architecture checks, PostgreSQL-backed tests, migration verification, and consumer-boundary checks across representative repositories. The public claim is limited to the presence of those engineering practices in the reviewed snapshot; no pass counts or release status are published.

### Scenario and journey evidence

Some failures only appear across a sequence: installation, session transition, authority change, durable retry, provider receipt, and readback. Scenario certification and authenticated journeys are useful when they exercise the real boundary under controlled scope. Negative paths are as important as happy paths.

### Provider and deployment evidence

Where safe and authorized, a provider canary or subsequent readback gives stronger evidence than a synthetic transport mock for a narrow supported behavior. Deployment evidence links the candidate source/build, runtime version, readiness state, and compatible migrations. This repository intentionally excludes canary credentials, URLs, outputs, and operational results.

## Production gates as questions

Rather than a universal checkbox list, a release asks:

- Which contracts changed, and which consumers must remain compatible?
- Which migration and worker/API combinations are valid?
- Which failure scenarios became newly relevant?
- Which provider-facing effects require controlled readback?
- Can the deployed runtime identify its provenance and readiness without leaking sensitive configuration?

## Design questions

**Why not call a test suite a certification?** Certification implies a declared behavioral claim and evidence appropriate to that claim. A test suite is one evidence source among several.

**Why include negative cases?** Fail-closed authorization, stale-generation rejection, and ambiguous completion cannot be established by success-only tests.
