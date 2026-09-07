# Vertex Engineering

Selected architecture, reliability patterns, and engineering lessons from building Vertex, a private commerce intelligence system.

> Evidence is not reasoning.  
> Reasoning is not authority.  
> Authority is not execution.  
> Execution success is not outcome verification.

## What is Vertex?

Vertex is a private commercial system for operating commerce applications on shared foundations. Its public architectural record is more useful when it is precise about boundaries than when it tries to reveal implementation detail.

The system separates provider-derived evidence, interpretation, authorization, durable work, provider interaction, and verification. That separation is the central design decision documented here.

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

## Application suite

The local implementation snapshot contains worktrees or integration evidence for application areas including SEO, Store Audit, Joho, CRO, Upsell/CrossSell, Reviews, Loyalty, Refer, Recur, Social, Signals, Digital Downloads, Matrix, and Profit. This is an architectural observation, not a feature inventory or a statement that every application has identical maturity.

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

The private implementation remains private. This repository intentionally omits implementation code, raw schemas and migrations, credentials, provider payloads, real tenant identifiers, private URLs, operational dashboards, prompts, model routing, and detailed security-sensitive surfaces. The diagrams are conceptual and the pseudocode is newly written.

## Relationship to Faultline

No public Faultline repository was verified while this dossier was assembled, so this repository intentionally includes no link. If a public Faultline project is released later, it may describe how revision-bound authority, evidence provenance, and readback verification were generalized into a separate industrial AI assurance setting. It would not contain or depend on Vertex's commercial implementation.

## Scope and evidence note

This dossier was based on an available local implementation snapshot, including shared Contracts, Control Plane, Kernel, Platform/MDP, Identity, and representative application worktrees. It documents supported architectural direction and observed engineering patterns, not confidential production status, customer count, availability metrics, or universal feature coverage. Where implementation evidence did not establish a claim, this dossier either omits it or calls it a design principle.
