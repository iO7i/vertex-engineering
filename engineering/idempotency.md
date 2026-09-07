# Idempotency

```text
Retries are unavoidable. Duplicate effects are not.
```

Networked systems commonly deliver the same event or retry the same operation more than once. A worker can crash after the provider commits. A provider can retry a webhook. A durable workflow can resume after a timeout. The engineering objective is not a slogan of exactly-once execution; it is to contain duplicate effects while retaining enough evidence to recover from ambiguity.

## The distinction

Exactly-once is a strong end-to-end claim and is not made here. Vertex instead uses an *effectively-once* design where supported: identify a logical operation, use provider idempotency facilities when available, make internal handlers safe under re-delivery, record progress durably, and reconcile observed provider state before repeating an ambiguous mutation.

```text
logical operation ID
        +
provider idempotency key (when supported)
        +
durable retry / receipt state
        +
readback reconciliation
        = bounded duplicate-risk strategy
```

## A failure timeline

```text
T0  action admitted
T1  worker sends provider mutation
T2  provider commits mutation
T3  network fails
T4  worker sees timeout
T5  workflow retries
```

At T5, a retry cannot assume failure. It should determine what is known about the prior attempt: operation identity, any provider acknowledgement, existing receipt, and observed provider state. If the effect is already present, the operation can converge without dispatching a blind duplicate. If the state is not yet observable, it may remain pending while a bounded reconciliation policy runs.

## Event delivery

Vertex includes a per-subscription idempotency abstraction that records successful event handling after the handler completes. This deliberately preserves at-least-once behavior: a crash before the success mark permits re-delivery instead of silently losing the event. It also means handlers must tolerate a rare double-run; deduplication is useful, but cannot be the sole correctness mechanism.

## What an idempotency boundary needs

- A stable logical operation or event identity.
- Scope binding: a key must not collide across tenant or purpose boundaries.
- A clear completion state: dispatched, acknowledged, observed, reconciled, or terminally unresolved are different states.
- A transactional or durable place to record internal progress where appropriate.
- A provider-specific policy for APIs that support idempotency keys, those that do not, and those with weak completion semantics.
- Readback before repeating an externally visible mutation after ambiguous completion.

## Failure story: duplicate webhooks

A provider may deliver a webhook twice, perhaps minutes apart. If a consumer only asks whether the event "looks familiar," two workers can race. A durable successful-processing record and idempotent handler reduce that risk. If the downstream effect is itself external, it still needs its own operation identity and reconciliation boundary.

## Design questions

**Does an idempotency key prove an outcome?** No. It can suppress repeated dispatch; readback establishes what was observed afterward.

**Why mark a message after successful handling?** Marking it first can turn a crash into permanent loss. The tradeoff is deliberate: at-least-once delivery requires idempotent consumers.
