# Incident 003 — When Future Data Counted as Fresh

## Summary

During correctness review of synchronized analytics data, we found that
freshness logic could classify data as current without first establishing that
its timestamp was meaningful. Two related cases exposed the gap: data older
than the intended freshness window could remain current, and a timestamp from
the future could also satisfy the freshness check.

This was discovered during engineering validation. The examples below are
illustrative, not customer records or production measurements.

## Impact

Downstream analytics, recommendations, or operational decisions could consume
data presented as current when it was stale or temporally unresolved.

The problem was not the visual wording of a freshness badge. It was that a
derived state could claim a level of temporal certainty that the input did not
support.

## Expected invariant

Freshness requires both a meaningful timestamp and a valid age calculation:

```text
timestamp is valid
        ∧
timestamp is not in the future
        ∧
age is within the declared window
        ⇒ CURRENT
```

Illustrative boundary cases:

```text
Current time:          12:00
Last successful sync:  03:00
Expected:              STALE
```

```text
Current time:          12:00
Last successful sync:  12:30
Expected:              INVALID / UNRESOLVED
```

These are synthetic examples used to explain the invariant.

## What we observed

The historical predicate was effectively concerned with whether an age was
less than a threshold. A future timestamp produces a negative age, which can
pass that predicate. Older data could also remain in a current classification
when the state transition or boundary condition was not evaluated explicitly.

Timestamps without sufficient timezone meaning introduced a second ambiguity:
the system could be tempted to assign a timezone rather than preserve an
unresolved state.

## What was actually wrong

Freshness had been implemented as presentation-adjacent metadata rather than
as a validated data contract. The logic trusted the timestamp and evaluated a
single inequality, but did not first validate temporal direction, timezone
meaning, or the complete set of state transitions.

## Why the original model was insufficient

```text
age <= threshold
```

is not a complete freshness contract. It assumes the timestamp is real,
comparable, correctly anchored in time, and not from the future.

That assumption is unsafe because time is input data. A negative age is not
evidence of freshness; it is evidence that the timestamp or clock relationship
needs interpretation.

## Systemic correction

Freshness was made an explicit state with conservative semantics. The system
now distinguishes, at the appropriate boundary, between states such as:

- current
- stale
- invalid
- unresolved

Future or malformed timestamps are rejected or quarantined rather than treated
as current. Timezone ambiguity is handled explicitly; the system does not
silently invent a timezone to manufacture certainty. Stale data cannot prove
current state, and downstream consumers receive the freshness state as part of
the contract rather than only a display label.

## Regression strategy

The regression suite uses deterministic clocks and synthetic fixtures to cover:

- just-inside and just-outside freshness-window boundaries
- clearly stale data
- future timestamps
- missing or ambiguous timezone information
- malformed timestamps
- clock skew and replayed time-based inputs
- transitions between current, stale, invalid, and unresolved states

The tests verify both the classification and the behavior of consumers that
must suppress, qualify, or refresh uncertain data.

## What we deliberately do not disclose

This note omits merchant records, provider payloads, production timestamps,
database queries and schemas, source-system identifiers, private data paths,
and implementation-specific freshness configuration.

## Engineering lesson

Time is input data. It needs validation like any other untrusted input. If
recommendations, analytics, or operational decisions depend on freshness, then
temporal semantics belong in the data contract—not in a UI badge.
