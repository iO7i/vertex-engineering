# Vertex Engineering

Selected architecture, reliability patterns, and engineering lessons from Vertex, designed and built by Hosam Talbi Al-Khairat.

Vertex is a private multi-tenant commerce intelligence and execution platform. It integrates external commerce systems, including Zid and Salla, across shared evidence, reasoning, authority, and execution infrastructure. The commercial implementation remains private; this repository documents selected engineering decisions behind it.

## Thesis

> Evidence is not reasoning.
> Reasoning is not authority.
> Authority is not execution.
> Execution success is not outcome verification.

```text
EVIDENCE != REASONING != AUTHORITY != ORCHESTRATION != EXECUTION != VERIFICATION
```

These are different questions: what happened, what does it mean, what should be done, may it happen, how does work progress through failure, and did the intended effect become true? Vertex gives those questions different architectural owners.

## How to verify the public claims

This repository is an architecture dossier, not a runnable copy of the private Vertex implementation. Its public evidence is the linked design record, failure model, and certification strategy. The independently runnable proof of the generalized reliability primitives lives in [Faultline](https://github.com/iO7i/faultline): after `pnpm install --frozen-lockfile`, run `pnpm demo`.

Faultline demonstrates stale-authority rejection and acknowledgement-loss reconciliation against a synthetic simulator. That is evidence for the published engineering primitives, not a production certification of Vertex, its providers, or its commercial applications.

## Engineering notes

- [Why `/version` can lie](notes/01-deployment-provenance.md)
- [Logout is an authority problem](notes/02-logout-is-authority.md)
- [One intended effect under ambiguous completion](notes/03-one-intended-effect.md)

## Engineering incidents

Vertex is a commercial system, but some of the engineering lessons behind it are worth sharing.

These deliberately sanitized notes document selected correctness and reliability failures: the failed invariant, the changed mental model, and the systemic correction.

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

The architectural purpose is separation: provider observations become evidence; evidence informs reasoning; reasoning may propose; current authority admits or rejects an action; durable work makes bounded progress; readback turns an observed result into new evidence.

## Ambiguous completion

```text
T0  operation is authorized
T1  worker dispatches provider mutation
T2  provider commits the mutation
T3  connection fails before Vertex records acknowledgement
T4  durable workflow resumes
```

At T4, blind retry is incorrect because the provider may have executed the effect twice. Blind success is also incorrect because Vertex has not established that the intended effect exists. The system retains bounded progress, then uses operation identity, idempotency where supported, readback, and reconciliation to decide how work can safely continue.

## Generation fencing

```text
Generation 17 worker starts
          |
          | installation lifecycle changes
          v
Generation 18 becomes current
          |
          v
old Generation 17 worker resumes -> attempts provider mutation -> REJECTED
```

Installation identity alone is insufficient. Authority granted under an earlier lifecycle generation must not silently survive into the current one.

```text
installation = same
generation   = different
authority    = stale
```

## Core planes

| Plane | Primary question | It is not |
| --- | --- | --- |
| Merchant Data Plane (MDP) | What is true, and what evidence supports that claim? | A UI cache or execution authority |
| Brain | What does the available evidence mean? | Provider mutation authority |
| Serving Plane | What derived state should products read efficiently? | Canonical merchant truth |
| Control Plane | May this principal perform this operation in this context? | A recommendation engine |
| Durable execution | How does admitted work make progress across failure? | Evidence or authorization authority |
| Kernel | How do we communicate safely with a provider? | Merchant truth or business intelligence |
| Platforms | How does a canonical Vertex operation map to a provider? | Application-specific business logic |
| Contracts | How does the system agree on versioned meaning? | Runtime authority |
| Identity | Who is acting, and in which merchant context? | Authorization by itself |

## Public evidence

This repository exposes the engineering record without exposing the commercial implementation.

- **Implemented foundations:** Contracts, Kernel, Platforms, Identity, Merchant Data Plane, Brain, Serving Plane, Control Plane, and durable workflow infrastructure are documented here as distinct shared system surfaces.
- **Provider scope:** Vertex has provider-specific Zid and Salla integration surfaces, including lifecycle and normalized read paths.
- **Application scope:** documented domains include SEO, Store Audit, Joho, CRO, Upsell, Reviews, Loyalty, Refer, Recur, Social, Signals, Digital Downloads, Matrix, and Profit. Names do not imply identical implementation depth, release maturity, or feature coverage.
- **Validation scope:** relevant components include deterministic, integration, database-backed, provider-specific, and browser-journey test paths. This is not a claim about aggregate coverage, customer count, availability, or current release status.

**IMPLEMENTED** means present in the private Vertex implementation. **VALIDATED** means exercised through evidence appropriate to the relevant scope. **DESIGN PRINCIPLE** denotes an architectural rule whose implementation coverage may vary by application.

## Reliability principles

```text
Retries are unavoidable.
Duplicate effects require explicit containment.

External acknowledgement is not equivalent to business outcome.
Stale authority must fail closed.
Previous installation generations must not silently remain authorized.
Canonical evidence and derived serving state are different things.
A model recommendation does not create execution authority.
Workflow durability does not create authorization.
Provider side effects require reconciliation.
Failure paths deserve first-class tests.
Version compatibility is part of correctness.
Deployment provenance is part of correctness.
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
