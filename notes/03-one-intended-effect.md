# Engineering note: one intended effect under ambiguous completion

## Symptom

A mutating provider call can commit its business effect and lose the response
before the caller records completion. The caller then has two dangerous choices:
blindly retry and create a duplicate, or treat the timeout as failure and lose
the ability to establish what happened.

## Naive model

Interpret transport acknowledgement as business outcome: `200` means done,
timeout means safe to retry, and a provider retry policy supplies “exactly once.”
That model confuses delivery with effect and cannot distinguish a duplicate
identity from a conflicting reuse of the same identity.

## Real invariant

The application may guarantee one intended business effect per admitted
operation identity within its own authority boundary. It must represent
completion as `unknown` when acknowledgement is lost, bind the operation to
scope and canonical input, reject conflicting reuse, and reconcile against
durable receipt/readback evidence before deciding whether another dispatch is
safe. This is not a claim of exactly-once provider delivery.

## Architecture / fix

Persist the logical operation identity, canonical input digest, authority scope,
dispatch state, and receipt before allowing recovery to proceed. Use provider
idempotency where available. Otherwise, record the ambiguous state, read back
the provider or simulator by operation identity, classify the result, and
suppress redispatch once the intended effect is established. Downstream
at-least-once delivery remains acceptable when every consumer applies the same
identity and conflict rules.

## Regression proof

[Faultline](https://github.com/iO7i/faultline) runs a real synthetic simulator:
lost acknowledgement becomes `COMPLETION_UNKNOWN`, reconciliation records one
effect, and the second dispatch is suppressed. Its bounded checker also records
and replays a stale-authority counterexample. [Vertex Replay Lab](https://github.com/iO7i/vertex-replay-lab)
adds canonical-digest, duplicate, conflict, tenant, fail-closed, restart, and
replay-digest tests, plus a duplicate-schedule property check across varied
order and multiplicity.

The invariant is deliberately scoped to intended application effects; it does
not imply provider exactly-once semantics or production throughput.
