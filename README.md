# Vertex Engineering

Selected architecture, reliability patterns, and engineering lessons from Vertex, designed and built by Hosam Al-Khairat.

> Evidence is not reasoning.  
> Reasoning is not authority.  
> Authority is not execution.  
> Execution success is not outcome verification.

## What is Vertex?

Vertex is a private multi-tenant commerce intelligence and execution platform. It integrates external commerce systems across shared evidence, reasoning, authority, and execution infrastructure. The private implementation includes provider-specific integration surfaces for Zid and Salla.

I designed Vertex to separate provider-derived evidence, interpretation, authorization, durable work, provider interaction, and verification. That separation is the central engineering decision documented here. The commercial implementation remains private.

## Why this repository exists

This repository lets an engineer inspect the problems Vertex is designed around: multi-tenant scope, stale authority, provider inconsistency, durable progress, contract compatibility, and evidence provenance. It is a technical dossier, not a source release.

## What this repository is not

- Not an open-source version of Vertex.
- Not a source-code mirror, deployment guide, or endpoint catalog.
- Not a record of customer data, provider credentials, private infrastructure, or commercial capability logic.
- Not a claim that every pattern described here is universal or complete.

## The architectural thesis

```text
EVIDENCE != REASONING != AUTHORITY != ORCHESTRATION != EXECUTION != VERIFICATION
```

A system can observe a provider accurately and still make a poor recommendation. A strong recommendation may still be unauthorized. An authorized request can still fail halfway through an external side effect. A provider acknowledgement can still differ from the state a later read observes. Treating those questions as one thing is how incidental failures become cross-tenant, duplicated, or unexplainable effects.

## System overview

```text
External commerce reality
          |
          v
  Providers and external sources
          |
          v
Kernel + Platforms -----> Merchant Data Plane (what is true?)
                                  |
                                  v
                         Brain (what does it mean?)
                                  |
                                  v
                 Serving Plane (what should be fast to read?)
                                  |
                                  v
                    Vertex application / capability request
                                  |
                                  v
                    Control Plane (may it happen?)
                                  |
                                  v
                 Durable workflow -> bounded specialist executor
                                  |
                                  v
                             Provider mutation
                                  |
                                  v
                     Readback + reconciliation -> MDP
```

The fuller diagram, trust boundaries, and read/action/verification paths are in [system overview](architecture/system-overview.md).

## The closed loop

Vertex treats a provider mutation as a hypothesis that has to be checked. The loop is:

```text
observe -> preserve evidence -> interpret -> propose -> authorize -> execute
       -> read back -> reconcile -> preserve new evidence -> interpret again
```

See [closed-loop execution](architecture/closed-loop-execution.md) and [readback verification](engineering/readback-verification.md).

## Core planes

| Plane | Primary question | It is not |
| --- | --- | --- |
| Merchant Data Plane (MDP) | What is true, and what supports that claim? | A UI cache or execution authority |
| Brain | What does the available evidence mean? | Provider mutation authority |
| Serving Plane | What derived state should the product read quickly? | Canonical merchant truth |
| Control Plane | May this principal perform this action in this context? | A recommendation engine |
| Durable execution | How does work make bounded progress across failure? | Evidence or authorization authority |
| Kernel and Platforms | How do canonical operations communicate with providers? | Business truth or ambient authority |

Detailed responsibility boundaries appear in [evidence, reasoning, and authority](architecture/evidence-reasoning-authority.md) and [multi-tenant boundaries](architecture/multi-tenant-boundaries.md).

## Reliability principles

- Retries are unavoidable; duplicate effects must be deliberately contained.
- Provider acknowledgement is evidence of transport, not necessarily business outcome.
- Current authority is evaluated at admission and bound to a narrow context.
- A previous installation generation must not silently retain authority.
- Derived serving state should be rebuildable from upstream authoritative state.
- Failure cases deserve named contracts, deterministic tests, and recovery paths.

## Engineering surface

The private implementation includes the following high-level surfaces:

| Surface | Role |
| --- | --- |
| Contracts | Shared, versioned vocabulary for tenancy, lifecycle, evidence, authority, and application integration. |
| Kernel | Provider communication primitives, lifecycle/session support, webhook handling, and bounded readback. |
| Platforms | Canonical mapping, provider normalization, and shared transport/data-plane capabilities. |
| Identity | Human and workload identity boundaries. |
| Merchant Data Plane | Canonical observations, evidence, receipts, coverage, and reconciliation state. |
| Brain | Versioned interpretation and merchant-intelligence artifacts derived from evidence. |
| Serving Plane | Derived, product-facing intelligence snapshots and prepared reads. |
| Control Plane | Current-session, membership, installation, generation, policy, and entitlement decisions. |
| Durable workflow infrastructure | Retryable, resumable workflow execution for long-running provider work. |
| Provider integrations | Zid and Salla adapters and provider-specific lifecycle/read paths. |

**Evidence vocabulary.** **IMPLEMENTED** means present in the private Vertex codebase. **VALIDATED** means exercised through deterministic, integration, provider, or journey evidence available for the relevant scope. **DESIGN PRINCIPLE** is an architectural rule; its coverage may vary by application. These terms are used selectively so they retain meaning.

**IMPLEMENTED:** the shared Contracts catalog contains 13 cataloged application entries. **VALIDATED:** the codebase includes provider-specific Zid and Salla test paths and browser-journey coverage in representative application work. These are engineering-surface indicators only; they are not customer, uptime, coverage, or release-maturity claims.

## Application suite

Vertex application domains include SEO, Store Audit, Joho, CRO, Upsell/CrossSell, Reviews, Loyalty, Refer, Recur, Social, Signals, Digital Downloads, Matrix, and Profit. The names describe documented application domains, not a feature inventory; they do not imply identical release maturity or implementation coverage.

The intended suite boundary is consistent: an application contributes its domain behavior; it should not independently rebuild identity, provider authentication, provider semantics, canonical merchant evidence, shared contracts, authority policy, or durable-work primitives.

## Selected engineering problems

- [Idempotency](engineering/idempotency.md)
- [Generation fencing](engineering/generation-fencing.md)
- [Provider reconciliation](engineering/provider-reconciliation.md)
- [Durable workflows](engineering/durable-workflows.md)
- [Readback verification](engineering/readback-verification.md)
- [Failure model](reliability/failure-model.md)
- [Certification strategy](reliability/certification-strategy.md)

## Repository map

```text
architecture/  system responsibility, authority, multi-tenancy, closed loop
engineering/   concentrated lessons from external side effects and recovery
reliability/   failure taxonomy, release evidence, testing principles
media/         safe, inspectable source diagrams; no product screenshots
```

## Private implementation boundary

The commercial implementation remains private. This repository intentionally omits implementation code, raw schemas and migrations, credentials, provider payloads, real tenant identifiers, private URLs, operational dashboards, prompts, model routing, and detailed security-sensitive surfaces. The diagrams are conceptual and the pseudocode is newly written.

## Two failure traces

### Ambiguous provider completion

```text
T0  authorize
T1  dispatch
T2  provider commits
T3  connection fails before acknowledgement is recorded
T4  workflow resumes
```

Blind retry is incorrect because T2 may already have created the effect. Blind success is incorrect because the system has not observed the result. The workflow retains bounded progress, then reconciliation/readback determines whether the expected provider state exists before another mutation is attempted.

### Stale installation generation

```text
G17 worker starts
merchant installation lifecycle changes
G18 becomes current
G17 worker resumes
G17 authority is rejected
```

Installation identity alone is not enough: the prior generation's authority must not silently survive a lifecycle change. Generation fencing makes the current lifecycle revision part of the side-effect boundary.

## Scope note

This public record documents selected decisions from the system I built. It intentionally makes no confidential claims about customer count, availability, production status, or universal feature coverage. Where a mechanism is not established as a shared implementation surface, this repository describes it as a design principle rather than a guarantee.
