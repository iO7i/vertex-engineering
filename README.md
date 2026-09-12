# Vertex Engineering

Vertex is a private multi-tenant commerce intelligence and execution platform. It integrates external commerce systems, including Zid and Salla, with shared data, reasoning, identity, workflow, and provider-integration layers. This repository contains selected engineering decisions and failure analyses; the commercial implementation remains private.

## Start here

This repository is an architecture dossier, not a runnable copy of Vertex. The generalized reliability patterns have an independently runnable reference implementation in [Faultline](https://github.com/iO7i/faultline):

```bash
pnpm install --frozen-lockfile
pnpm demo
```

Faultline runs a synthetic simulator that rejects stale authority and reconciles a lost acknowledgement. Those results support the published patterns; they do not certify Vertex, its providers, or its commercial applications.

## The failure case

Suppose a provider accepts a mutation, then the connection drops before Vertex records the response:

```text
T0  operation is authorized
T1  worker sends provider mutation
T2  provider commits it
T3  connection fails before the acknowledgement is recorded
T4  workflow resumes
```

Blind retry can apply the effect twice. Assuming success is also unsafe because the intended outcome has not been checked. The design records operation identity, uses provider idempotency where available, reads back state, and keeps retry behavior bounded.

Authority has a similar lifecycle problem. A worker from an earlier installation generation must not remain authorized after the current generation changes:

```text
Generation 17 starts
          |
          | installation lifecycle changes
          v
Generation 18 becomes current
          |
          v
Generation 17 resumes -> provider mutation rejected
```

## Engineering notes

- [Why `/version` can lie](notes/01-deployment-provenance.md)
- [Logout is an authority problem](notes/02-logout-is-authority.md)
- [One intended effect under ambiguous completion](notes/03-one-intended-effect.md)

## Engineering incidents

Vertex is a commercial system, but some of the engineering lessons behind it are worth sharing.

These deliberately sanitized notes document selected correctness and reliability failures, the changed mental model, and the resulting platform-level fixes.

→ [Read the incident notes](incidents/README.md)

## Architecture

```text
                                      VERTEX

                               EXTERNAL REALITY
                                      |
                                      v
                              ZID / SALLA / SOURCES
                                      |
                                      v
                           +-----------------------+
                           | KERNEL + PLATFORMS    |
                           | provider boundary     |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | MERCHANT DATA PLANE   |
                           | "What is true?"       |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | BRAIN                 |
                           | "What does it mean?"  |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | SERVING PLANE         |
                           | prepared product state|
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | VERTEX APPLICATION    |
                           | / CAPABILITY          |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | CONTROL PLANE         |
                           | "May this happen?"    |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | DURABLE WORKFLOW      |
                           | retry / resume /      |
                           | reconciliation        |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | BOUNDED EXECUTOR      |
                           +-----------+-----------+
                                       |
                                       v
                               PROVIDER MUTATION
                                       |
                                       v
                           +-----------------------+
                           | READBACK +            |
                           | RECONCILIATION         |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | NEW MDP EVIDENCE      |
                           +-----------+-----------+
                                       |
                                       +--------------------> NEXT DECISION CYCLE
```

Provider observations enter the Merchant Data Plane as evidence. The Brain interprets that data, applications propose work, and the Control Plane decides whether the current principal may perform it. Durable workflow carries admitted work through failure. Readback turns the observed result into new data for the next cycle.

## Core planes

| Plane | Primary question | What it does not provide |
| --- | --- | --- |
| Merchant Data Plane (MDP) | What is true, and what supports that claim? | UI cache or mutation permission |
| Brain | What does the available evidence mean? | Provider mutation permission |
| Serving Plane | What derived state should products read efficiently? | Canonical merchant truth |
| Control Plane | May this principal perform this operation here? | Recommendation logic |
| Durable execution | How does admitted work make progress across failure? | Data or authorization |
| Kernel | How do we communicate safely with a provider? | Merchant truth or business intelligence |
| Platforms | How does a canonical Vertex operation map to a provider? | Application-specific business logic |
| Contracts | How does the system agree on versioned meaning? | Runtime permission |
| Identity | Who is acting, and in which merchant context? | Authorization by itself |

## Public evidence

This repository exposes the engineering record without exposing the commercial implementation.

- **Implemented foundations:** Contracts, Kernel, Platforms, Identity, Merchant Data Plane, Brain, Serving Plane, Control Plane, and durable workflow infrastructure are documented here as distinct shared system surfaces.
- **Provider scope:** Vertex has provider-specific Zid and Salla integration surfaces, including lifecycle and normalized read paths.
- **Application scope:** documented domains include SEO, Store Audit, Joho, CRO, Upsell, Reviews, Loyalty, Refer, Recur, Social, Signals, Digital Downloads, Matrix, and Profit. Names do not imply identical implementation depth, release maturity, or feature coverage.
- **Validation scope:** relevant components include deterministic, integration, database-backed, provider-specific, and browser-journey test paths. This is not a claim about aggregate coverage, customer count, availability, or current release status.

**IMPLEMENTED** means present in the private Vertex implementation. **VALIDATED** means exercised through evidence appropriate to the relevant scope. **DESIGN PRINCIPLE** denotes an architectural rule whose implementation coverage may vary by application.

## Rules the system relies on

```text
Retries are unavoidable, so duplicate effects need containment.
Provider acknowledgement is not the business outcome until readback or reconciliation confirms it.
Stale authority and old installation generations are rejected.
Canonical merchant data and derived serving state are kept separate.
A model recommendation does not grant permission, and durable workflow does not grant permission.
Provider side effects are followed by reconciliation.
Failure paths, version compatibility, and deployment provenance are tested as correctness concerns.
```

## Deeper reading

- [System Overview](architecture/system-overview.md)
- [Evidence, Reasoning, and Authority](architecture/evidence-reasoning-authority.md)
- [Closed-loop Execution](architecture/closed-loop-execution.md)
- [Multi-tenant Boundaries](architecture/multi-tenant-boundaries.md)
- [Idempotency](engineering/idempotency.md)
- [Provider Mutation Semantics](engineering/provider-mutation-semantics.md)
- [Generation Fencing](engineering/generation-fencing.md)
- [Provider Reconciliation](engineering/provider-reconciliation.md)
- [Durable Workflows](engineering/durable-workflows.md)
- [Readback Verification](engineering/readback-verification.md)
- [Event Vocabulary](engineering/event-vocabulary.md)
- [Failure Model](reliability/failure-model.md)
- [Certification Strategy](reliability/certification-strategy.md)
- [Test Philosophy](reliability/test-philosophy.md)

## Private implementation boundary

Vertex is a commercial product. This repository intentionally excludes commercial implementation source code, raw private schemas or migrations, customer or merchant data, real tenant identifiers, credentials, tokens, private infrastructure endpoints, internal deployment configuration, security-sensitive topology, private provider payloads, commercial capability logic, and private prompt or model-routing configuration.

The diagrams are conceptual. Example pseudocode is newly written to explain engineering principles rather than reproduce the private implementation.

This dossier does not claim universal correctness, perfect provider consistency, exactly-once execution, zero failure, uniform maturity across applications, customer count, availability, SLA performance, or complete production status.

**Hosam Talbi Al-Khairat**
Builder of Vertex
