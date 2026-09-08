# System overview

Vertex is organized around a question that is often hidden inside application code: which subsystem is entitled to answer which question? The diagram below is a logical map, not a deployment diagram or a list of network endpoints.

```text
                                 +----------------+
                                 |    Merchant    |
                                 +----------------+
                                        |      ^
                                        |      | product views / requests
                                        v      |
 +-----------+       +-----------------------------+       +-----------+
 |  Shopper   |----->|      Vertex app suite        |<----->|  Merchant |
 +-----------+       | SEO / Audit / Joho / ...     |       +-----------+
                     +---------------+-------------+
                                     | prepared reads / action requests
                                     v
 +----------------+          +------------------+          +----------------+
 | Identity       |--------->|  Control Plane   |<---------| Contracts      |
 | human/session  |          | admission policy |          | shared meaning |
 | membership     |          +--------+---------+          +----------------+
 +----------------+                   | bounded permit
                                      | v
                                     | +------------------+       +----------------+
                                     | | Durable work     |------>| specialist     |
                                     | | workflow state   |       | executors      |
                                     | +------------------+       +-------+--------+
                                     |                                      |
                                     |                                      v
 +-----------+   normalized calls    |                             provider mutation
 | Zid       |<----------------------+                                     |
 +-----------+                         +------------------+               |
 | Salla     |<------------------------| Kernel + Platforms|<--------------+
 +-----------+  webhooks / reads       | transport/mapping |
 | other     |------------------------>| receipts/readback |
 +-----------+                         +--------+---------+
                                              | observations / receipts
                                              v
                           +------------------------------------+
                           | Merchant Data Plane (MDP)          |
                           | canonical observations, evidence,  |
                           | provenance, snapshots, outcomes    |
                           +---------------+--------------------+
                                           | evidence references
                                           v
                           +------------------------------------+
                           | Brain                              |
                           | interpretation, scoring, proposals |
                           +---------------+--------------------+
                                           | versioned derived artifacts
                                           v
                           +------------------------------------+
                           | Serving Plane                      |
                           | rebuildable product projections     |
                           +---------------+--------------------+
                                           |
                                           +-----> Vertex app suite

Readback/reconciliation flows from Kernel + Platforms back to MDP, then begins
the next decision cycle. Brain never supplies an execution permit by itself.
```

## Read path

```text
provider/source -> Kernel + Platforms -> MDP evidence -> Brain interpretation
               -> Serving Plane projection -> application read
```

The MDP maintains the distinction between an observation and a conclusion drawn from it. A low-latency serving projection is therefore treated as derived state: it should be replaceable from authoritative upstream evidence and decision artifacts.

## Action path

```text
application request -> Control Plane evaluates current context
                    -> durable workflow receives bounded work
                    -> specialist executor -> provider request
```

An app may request an action; it does not gain global provider authority. The Control Plane's logical decision binds principal, tenant/store context, installation, current generation, action/purpose, and policy state. Durable work carries progress; it does not manufacture authority.

## Verification path

```text
provider response -> dispatch receipt -> subsequent provider read
                  -> expected-versus-observed reconciliation -> MDP evidence
```

The outcome path is intentionally separate from dispatch. A response may be useful evidence, but it is not automatically proof that the intended merchant state exists.

## Trust boundaries

### Provider boundary

Providers are independent systems. They may delay consistency, duplicate or omit webhooks, rate-limit calls, accept a request before a later read reflects it, or leave a request's completion ambiguous. Kernel and Platforms normalize communication and retain bounded receipts; they do not turn provider behavior into a guarantee.

### Tenant and identity boundary

Human identity, account membership, selected store, installation membership, installation generation, and purpose are distinct facts. A human session alone is insufficient evidence for an action on any arbitrary installation.

### Evidence and intelligence boundary

Brain uses evidence but does not rewrite it into truth. It creates interpretation artifacts and proposals that can be versioned, reviewed, superseded, or suppressed when evidence is stale, missing, or conflicted.

### Authority and execution boundary

Control Plane makes an admission decision from current policy and context. An executor receives only the narrow context necessary to perform admitted work. Provider credentials and transport details are not ambient authority for arbitrary app logic.

## What must never collapse

```text
Brain                    != authority
Serving Plane            != canonical truth
durable workflow engine  != authority
provider response        != verified outcome
LLM output               != evidence
application database     != canonical merchant truth
credential               != ambient global authority
installation identity    != every installation generation
```

## Design questions

**Which state is canonical?** Provider-derived observations and their bounded provenance live in the MDP; serving state is derived.

**Which components are rebuildable?** Serving projections and many product views should be rebuildable from their versioned upstream inputs.

**Where may credentials exist?** At the bounded provider-communication layer, never as a general-purpose application capability described by this public record.
