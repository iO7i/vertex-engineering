# Vertex Engineering

**Selected architecture, reliability patterns, and engineering lessons from Vertex, designed and built by Hosam Al-Khairat.**

Vertex is a private multi-tenant commerce intelligence and execution platform integrating external commerce systems, including **Zid** and **Salla**, across shared evidence, reasoning, authority, and execution infrastructure.

The commercial implementation remains private. This repository documents the engineering decisions behind it.

> **Evidence is not reasoning.**
> **Reasoning is not authority.**
> **Authority is not execution.**
> **Execution success is not outcome verification.**

---

## What is Vertex?

Vertex is not fourteen independent commerce applications connected to the same database.

It is a shared operational system underneath them.

The applications sit on common foundations for provider integration, identity, evidence, intelligence, authorization, durable work, and verification.

At the center of the architecture is one rule:

```text
EVIDENCE != REASONING != AUTHORITY != ORCHESTRATION != EXECUTION != VERIFICATION
```

These are different questions:

```text
What happened?

What does it mean?

What should we do?

Are we allowed to do it?

How do we make durable progress?

Did the intended effect actually become true?
```

Vertex deliberately gives those questions different architectural owners.

That separation is the central engineering decision documented in this repository.

---

## Why this repository exists

Vertex is a commercial system, so its implementation is private.

That should not require the engineering behind it to be invisible.

This repository makes selected architecture and reliability decisions publicly inspectable without exposing proprietary source code, customer data, credentials, internal infrastructure, commercial capability logic, or security-sensitive implementation details.

The focus is on problems such as:

* multi-tenant authority,
* canonical evidence,
* stale work,
* provider inconsistency,
* retries and ambiguous completion,
* installation lifecycle changes,
* durable workflows,
* contract compatibility,
* provenance,
* readback,
* reconciliation,
* and release evidence.

This is an **engineering dossier**, not an open-source release of Vertex.

---

# Architectural thesis

A commerce platform connected to external systems operates under partial knowledge.

An API response may be lost.

A webhook may arrive twice.

An event may arrive late.

A worker may crash after the provider has already committed an effect.

An installation may change while old work is still queued.

An AI recommendation may be reasonable but unauthorized.

A successful provider request may not produce the state the application intended.

Collapsing all of those concerns into a single application service turns ordinary distributed-system failures into duplicated, stale, cross-tenant, or unexplainable effects.

Vertex instead separates them.

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
                           |                       |
                           | "What is true?"       |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | BRAIN                 |
                           |                       |
                           | "What does it mean?"  |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | SERVING PLANE         |
                           |                       |
                           | prepared product      |
                           | state                 |
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
                           |                       |
                           | "May this happen?"    |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | DURABLE WORKFLOW      |
                           |                       |
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
                           | READBACK              |
                           | + RECONCILIATION      |
                           +-----------+-----------+
                                       |
                                       v
                           +-----------------------+
                           | NEW MDP EVIDENCE      |
                           +-----------+-----------+
                                       |
                                       +--------------------+
                                                            |
                                                            v
                                                   NEXT DECISION CYCLE
```

The full architecture, trust boundaries, and read/action paths are documented in [System Overview](architecture/system-overview.md).

---

# The closed loop

Vertex treats an external mutation as a **hypothesis that must be checked**.

```text
observe
   |
   v
preserve evidence
   |
   v
interpret
   |
   v
propose
   |
   v
authorize
   |
   v
execute
   |
   v
read back
   |
   v
reconcile
   |
   v
preserve new evidence
   |
   v
interpret again
```

A successful request is therefore not the end of an operation.

The resulting state becomes new evidence.

See:

* [Closed-loop execution](architecture/closed-loop-execution.md)
* [Readback verification](engineering/readback-verification.md)

---

# Why these boundaries exist

The architecture is easiest to understand through the failures it is designed to contain.

## 1. Ambiguous provider completion

Consider this sequence:

```text
T0  operation is authorized

T1  worker dispatches provider mutation

T2  provider commits the mutation

T3  connection fails before Vertex records acknowledgement

T4  durable workflow resumes
```

At `T4`, two naive choices are both wrong.

```text
BLIND RETRY
    |
    +--> provider may execute the effect twice


BLIND SUCCESS
    |
    +--> Vertex has not established that the intended effect exists
```

Vertex separates:

```text
INTENT
   |
   v
DISPATCH
   |
   v
PROVIDER ACKNOWLEDGEMENT
   |
   v
OBSERVED PROVIDER STATE
   |
   v
RECONCILED OUTCOME
```

Durable progress, idempotency, readback, and reconciliation exist because external side effects do not share Vertex's transaction boundary.

See [Idempotency](engineering/idempotency.md) and [Provider Reconciliation](engineering/provider-reconciliation.md).

---

## 2. Stale installation authority

Now consider an installation lifecycle race:

```text
Generation 17
     |
     | worker starts
     |
     |          installation lifecycle changes
     |                       |
     |                       v
     |                 Generation 18
     |
     v
old Generation 17 worker resumes
     |
     v
attempts provider mutation
     |
     X
REJECTED
```

Installation identity alone is insufficient.

Authority granted under an earlier lifecycle generation must not silently survive into the current one.

Conceptually:

```text
installation = same
generation   = different
authority    = stale
```

This is the role of **generation fencing**.

See [Generation Fencing](engineering/generation-fencing.md).

---

# Core planes

| Plane                         | Primary question                                           | It is not                               |
| ----------------------------- | ---------------------------------------------------------- | --------------------------------------- |
| **Merchant Data Plane (MDP)** | What is true, and what evidence supports that claim?       | A UI cache or execution authority       |
| **Brain**                     | What does the available evidence mean?                     | Provider mutation authority             |
| **Serving Plane**             | What derived state should products read efficiently?       | Canonical merchant truth                |
| **Control Plane**             | May this principal perform this operation in this context? | A recommendation engine                 |
| **Durable execution**         | How does admitted work make progress across failure?       | Evidence or authorization authority     |
| **Kernel**                    | How do we communicate safely with a provider?              | Merchant truth or business intelligence |
| **Platforms**                 | How does a canonical Vertex operation map to a provider?   | Application-specific business logic     |
| **Contracts**                 | How does the system agree on versioned meaning?            | Runtime authority                       |
| **Identity**                  | Who is acting, and in which merchant context?              | Authorization by itself                 |

The important property is not the number of components.

It is that their responsibilities do not silently collapse into one another.

---

# Merchant Data Plane

The **Merchant Data Plane** exists to preserve canonical observations and their provenance.

Conceptually:

```text
provider state
     |
     v
normalized observation
     |
     v
canonical evidence
     |
     +--> source
     +--> provider identity
     +--> merchant/store scope
     +--> observation time
     +--> ingestion time
     +--> contract version
     +--> evidence identity
     |
     v
reasoning input
```

MDP answers:

> **What does Vertex currently have evidence to say is true?**

It does not decide what should happen next.

That belongs elsewhere.

---

# Brain

Brain consumes evidence and produces versioned interpretation.

Depending on the capability, that may involve:

```text
diagnosis
ranking
scoring
prediction
recommendation
classification
opportunity detection
action proposal
```

Conceptually:

```text
MDP evidence
     |
     v
capability
     |
     v
reasoning
     |
     v
versioned decision artifact
     |
     +--> evidence references
     +--> capability version
     +--> result
     +--> explanation / metadata
```

The architectural rule is simple:

```text
BRAIN MAY PROPOSE.

BRAIN DOES NOT SELF-AUTHORIZE.
```

A high-confidence recommendation does not create its own permission to mutate an external merchant system.

See [Evidence, Reasoning, and Authority](architecture/evidence-reasoning-authority.md).

---

# Serving Plane

The Serving Plane prepares derived product-facing state.

It exists because the optimal representation for:

```text
canonical evidence
```

is not necessarily the optimal representation for:

```text
interactive product reads
```

The Serving Plane may combine:

```text
MDP evidence
+
Brain output
+
version bindings
+
prepared projections
```

into state suited to application consumption.

Its defining property is:

> **Serving state is derived.**

It is not the canonical source of merchant truth and should be rebuildable from authoritative upstream state.

---

# Control Plane

The Control Plane answers a fundamentally different question from Brain:

> **May this action happen now?**

An action can be intelligent and still fail authorization.

The relevant context may include:

```text
human authority session
account
merchant
store
installation
installation generation
application
capability
purpose
entitlement
policy
requested operation
```

Conceptually:

```text
ACTION PROPOSAL
      |
      v
+-----------------------+
| CONTROL PLANE         |
|-----------------------|
| Who?                  |
| Which account?        |
| Which store?          |
| Which installation?   |
| Which generation?     |
| Which capability?     |
| Which purpose?        |
| Which entitlement?    |
| Which policy?         |
+-----------+-----------+
            |
       +----+----+
       |         |
       v         v
     DENY      ADMIT
                 |
                 v
          bounded authority
```

Neither Brain, an application, nor the workflow engine can manufacture this authority for itself.

---

# Durable execution

External commerce operations are often asynchronous and failure-prone.

Durable workflow infrastructure exists to coordinate:

```text
retry
resume
checkpoint
timeout
reconciliation
long-running work
worker restart
external side effects
```

But workflow state is deliberately not treated as merchant truth.

```text
Temporal / workflow state  !=  MDP evidence
Temporal / workflow state  !=  Control Plane authority
Temporal / workflow state  !=  Brain interpretation
```

Each of those concerns survives for a different reason.

See [Durable Workflows](engineering/durable-workflows.md).

---

# Kernel and Platforms

Provider differences should not leak independently into every Vertex application.

The shared provider boundary is conceptually split between:

```text
KERNEL
   |
   +--> transport
   +--> provider authentication primitives
   +--> API behavior
   +--> pagination
   +--> signatures
   +--> lifecycle primitives
   +--> error normalization


PLATFORMS
   |
   +--> canonical provider mapping
   +--> Zid normalization
   +--> Salla normalization
   +--> canonical -> provider operation mapping
   +--> provider-specific capability differences
   +--> provider receipts / read paths
```

The intended direction is:

```text
                    canonical Vertex operation
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
               ZID MAP                 SALLA MAP
                  |                       |
                  v                       v
                 Zid                    Salla
```

not:

```text
SEO -------------> custom Zid logic
Reviews ---------> different Zid logic
Loyalty ---------> another Zid implementation
CRO -------------> yet another provider client
```

Applications should contribute domain behavior, not recreate the provider platform.

---

# Contracts

Vertex uses shared versioned contracts so independently deployed components agree on meaning.

They cover concepts such as:

```text
tenant identity
merchant identity
store identity
installation identity
installation generation

evidence
decisions
capability requests
capability results

action intents
authority
execution receipts
outcomes

errors
versions
compatibility
```

The goal is to prevent semantic drift between producers and consumers.

A serializer mismatch or lifecycle interpretation mismatch across repositories is a system correctness problem, not merely a typing inconvenience.

---

# Identity and multi-tenancy

A human login does not imply ambient authority over everything attached to that identity.

The logical scope narrows:

```text
HUMAN
  |
  v
ACCOUNT
  |
  v
MERCHANT / STORE
  |
  v
INSTALLATION
  |
  v
CURRENT GENERATION
  |
  v
CAPABILITY / OPERATION
```

Questions such as these remain distinct:

```text
Is this the same human?

Is this human a member of this account?

Does this account contain this store?

Is this installation current?

Is this the current installation generation?

Is this capability authorized?

Is this operation permitted for this purpose?
```

That distinction is central to Vertex's multi-tenant boundary.

See [Multi-tenant Boundaries](architecture/multi-tenant-boundaries.md).

---

# Engineering surface

The private Vertex implementation includes the following high-level shared surfaces:

| Surface                             | Responsibility                                                                   |
| ----------------------------------- | -------------------------------------------------------------------------------- |
| **Contracts**                       | Shared versioned vocabulary across the suite                                     |
| **Kernel**                          | Provider communication and lifecycle primitives                                  |
| **Platforms**                       | Canonical provider mapping and normalization                                     |
| **Identity**                        | Human, account, merchant, store, and workload identity boundaries                |
| **Merchant Data Plane**             | Canonical observations, evidence, provenance, receipts, and reconciliation       |
| **Brain**                           | Versioned intelligence and capability outputs derived from evidence              |
| **Serving Plane**                   | Prepared application-facing state and intelligence                               |
| **Control Plane**                   | Session, membership, installation, generation, entitlement, and policy authority |
| **Durable workflow infrastructure** | Retryable and resumable external-provider work                                   |
| **Provider integrations**           | Zid and Salla integration surfaces                                               |

Vertex currently spans **14 documented application domains**:

```text
SEO
Store Audit
Joho
CRO
Upsell
Reviews
Loyalty
Refer
Recur
Social
Signals
Digital Downloads
Matrix
Profit
```

These names describe application domains within Vertex.

They do **not** imply identical implementation depth, release maturity, or feature coverage.

The suite-level architectural intent is consistent:

```text
APPLICATION
    |
    +--> contributes domain behavior

APPLICATION
    |
    X--> should not recreate identity
    X--> should not recreate provider authentication
    X--> should not recreate provider semantics
    X--> should not redefine canonical merchant truth
    X--> should not redefine shared contracts
    X--> should not bypass authority
    X--> should not reinvent durable execution
```

---

# Evidence vocabulary

This repository uses three terms deliberately.

### IMPLEMENTED

Present in the private Vertex implementation.

### VALIDATED

Exercised through deterministic, integration, provider, database-backed, or browser-journey evidence for the relevant scope.

### DESIGN PRINCIPLE

An architectural rule whose implementation coverage may vary across applications or system areas.

These terms are intentionally narrower than words such as *proven*, *safe*, or *guaranteed*.

The repository does not treat a passing test as evidence for claims outside that test's scope.

---

# Reliability principles

Vertex development follows several recurring rules.

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

---

# Testing and certification philosophy

Correctness in a distributed application does not stop at:

```text
unit tests pass
```

The broader verification model is layered:

```text
SOURCE
  |
  v
BUILD
  |
  v
UNIT TESTS
  |
  v
CONTRACT TESTS
  |
  v
DATABASE-BACKED PATHS
  |
  v
INTEGRATION TESTS
  |
  v
FAILURE SCENARIOS
  |
  v
BROWSER JOURNEYS
  |
  v
PROVIDER-SPECIFIC PATHS
  |
  v
READBACK / RECONCILIATION
  |
  v
DEPLOYMENT VERSION + READINESS
  |
  v
RELEASE EVIDENCE
```

Different components exercise different subsets of that ladder.

The principle is more important than the specific harness:

> **A release claim should not exceed the evidence supporting it.**

See:

* [Failure Model](reliability/failure-model.md)
* [Certification Strategy](reliability/certification-strategy.md)
* [Test Philosophy](reliability/test-philosophy.md)

---

# Selected engineering problems

The repository goes deeper on individual mechanisms here:

### Architecture

* [System Overview](architecture/system-overview.md)
* [Evidence, Reasoning, and Authority](architecture/evidence-reasoning-authority.md)
* [Closed-loop Execution](architecture/closed-loop-execution.md)
* [Multi-tenant Boundaries](architecture/multi-tenant-boundaries.md)

### Engineering

* [Idempotency](engineering/idempotency.md)
* [Generation Fencing](engineering/generation-fencing.md)
* [Provider Reconciliation](engineering/provider-reconciliation.md)
* [Durable Workflows](engineering/durable-workflows.md)
* [Readback Verification](engineering/readback-verification.md)

### Reliability

* [Failure Model](reliability/failure-model.md)
* [Certification Strategy](reliability/certification-strategy.md)
* [Test Philosophy](reliability/test-philosophy.md)

---

# What must never collapse

A concise version of the architecture:

```text
WRONG                                      INTENDED

Brain -----------------> provider          Brain -> proposal -> authority -> execution

LLM output ------------> evidence          LLM output -> interpretation referencing evidence

App DB ----------------> canonical truth   App -> shared canonical evidence

Serving Plane ---------> authority         Serving Plane -> rebuildable derived state

Workflow retry --------> repeat mutation   retry -> identify / reconcile / safely continue

HTTP 200 --------------> outcome           acknowledgement -> readback -> outcome

old installation ------> still trusted     stale generation -> rejected

credential ------------> ambient access    credential -> bounded execution context

each application ------> custom provider   apps -> shared Kernel + Platforms
```

The entire system can be summarized as:

```text
OBSERVE
   |
   v
EVIDENCE
   |
   v
REASON
   |
   v
PROPOSE
   |
   v
AUTHORIZE
   |
   v
EXECUTE
   |
   v
OBSERVE AGAIN
   |
   v
VERIFY
   |
   +--------------------------> NEXT DECISION
```

---

# Repository map

```text
vertex-engineering/
│
├── README.md
│
├── architecture/
│   ├── system-overview.md
│   ├── evidence-reasoning-authority.md
│   ├── closed-loop-execution.md
│   └── multi-tenant-boundaries.md
│
├── engineering/
│   ├── idempotency.md
│   ├── generation-fencing.md
│   ├── provider-reconciliation.md
│   ├── durable-workflows.md
│   └── readback-verification.md
│
├── reliability/
│   ├── failure-model.md
│   ├── certification-strategy.md
│   └── test-philosophy.md
│
├── media/
│   ├── diagrams/
│   └── screenshots/
│
└── LICENSE
```

---

# Private implementation boundary

Vertex is a commercial product.

This repository intentionally does **not** contain:

```text
commercial implementation source code
raw private schemas or migrations
customer or merchant data
real tenant identifiers
provider credentials
tokens or secrets
private infrastructure endpoints
internal deployment configuration
security-sensitive topology
private provider payloads
commercial capability logic
private prompts or model-routing configuration
```

The diagrams are conceptual.

Example pseudocode is newly written to explain engineering principles rather than copied from the private implementation.

The objective is to expose enough architecture to make the engineering inspectable without turning this repository into a blueprint for reproducing the commercial system.

---

# Scope

This repository documents selected engineering decisions from a system I built.

It does not claim:

```text
universal correctness
perfect provider consistency
exactly-once execution
zero failure
uniform maturity across every application
customer count
availability
SLA performance
complete production status
```

Where a mechanism is not established as a shared implementation surface, it is described as a design principle rather than a guarantee.

That distinction is intentional.

The same principle underlies Vertex itself:

> **Preserve what the evidence establishes. Keep inference, authority, execution, and verification separate.**

---

**Hosam Al-Khairat**
Builder of Vertex
