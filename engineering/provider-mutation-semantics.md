# Provider mutation semantics

Provider mutations are not a single operation type. Each provider endpoint has different support for request identity, acknowledgements, readback, ordering, and visibility. Vertex treats those properties as execution semantics to be evaluated for an individual provider operation, not as global provider promises.

This document defines the public reasoning model. It does not publish a provider-by-provider capability matrix, private payloads, endpoint names, or implementation configuration.

## Mutation identity

A logical operation identifies one intended business effect within a bounded merchant, installation, generation, purpose, and action scope. It is not merely a retry counter.

```text
logical operation identity
        + exact intended arguments
        + current authority / generation
        + provider request identity, when supported
        = one bounded mutation attempt family
```

The same logical operation must not be repurposed with altered arguments. A changed intent is a new operation that requires its own current authorization and execution path.

## Operation capability profile

Before a provider-facing mutation is designed, the relevant operation is assessed along these dimensions:

| Dimension | Question | Safe response when absent or uncertain |
| --- | --- | --- |
| Request identity | Can a stable caller identity be supplied and recognized by the provider? | Do not equate a retry with a new mutation; retain the ambiguity and reconcile. |
| Receipt | What request, response, or provider reference can be retained? | Record the local dispatch attempt and avoid inventing a provider acknowledgement. |
| Readback | Can the intended effect later be observed through a scoped read path? | Qualify the outcome as unavailable or unresolved rather than verified. |
| Visibility | Can the provider acknowledge before the effect is visible to a later read? | Use bounded pending/reconciliation states; do not infer failure from immediate absence. |
| Ordering | Can webhooks, reads, and responses arrive in a different order? | Preserve source timing and reconcile observations rather than assuming a total order. |
| Reversal | Is a bounded compensating action meaningful if an observed result is wrong? | Escalate or repair through explicit authority; never treat compensation as automatic success. |

`Supported`, `unavailable`, and `unknown` are materially different states. Unknown capability is not permission to assume the most convenient behavior.

## Ambiguous completion

```text
T0  current authority admits one mutation
T1  worker records intent and dispatch attempt
T2  provider may commit the effect
T3  acknowledgement is unavailable to Vertex
T4  workflow resumes
T5  Vertex correlates receipt and reads bounded provider state
```

At T4, retrying may duplicate an existing effect. Declaring success may conceal a missing or divergent effect. The execution path therefore distinguishes **dispatched**, **acknowledged**, **observed**, **reconciled**, and **unresolved** rather than collapsing them into one boolean.

Where a provider honors a stable request identity, the same identity may make a repeat dispatch safe within that provider's documented semantics. It still does not establish the merchant-visible outcome; readback remains a separate question.

## Public design rules

- Scope operation identity to the merchant and installation context; do not use a globally ambient key.
- Bind an execution attempt to current generation and authority at the moment of mutation.
- Keep request identity and intended arguments immutable across retries.
- Record transport acknowledgement separately from observed state.
- Prefer provider readback or an explicitly qualified unresolved result over inferred success.
- Keep endpoint-specific behavior behind the Kernel and Platforms boundaries.

## What this does not claim

This model does not claim uniform idempotency, ordering, readback, or compensation behavior across Zid, Salla, or any other provider. It also does not claim exactly-once execution. It describes the boundary Vertex uses to make provider differences explicit and recoverable.
