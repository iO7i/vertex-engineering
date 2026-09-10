# Incident 001 — When Authentication Wasn't a Platform Invariant

## Summary

During suite-wide identity and journey certification, we found inconsistent
authenticated behavior across applications that shared identity/session
infrastructure. The failures did not come from one invalid token. They came
from the rest of the journey being treated as local application state.

This was a correctness and readiness problem discovered through engineering
work, not a public vulnerability disclosure or a claim of a broad outage.

## Impact

The user-visible failure class included:

- authentication appearing to succeed while the application journey did not
  complete correctly
- account or store selection losing continuity during a handoff
- a session appearing expired while selection was being resumed
- logout clearing one layer of state without reliably ending the broader
  authenticated journey
- locale state becoming inconsistent across an authentication transition

The same underlying identity problem could therefore look like several
different application bugs.

## Expected invariant

An authenticated journey is complete only when the application has re-established
current authority for the exact identity, application context, and selected
tenant/account context it is about to use.

```text
authentication
      ↓
current human authority
      ↓
tenant/account selection
      ↓
handoff and locale continuity
      ↓
application action
```

Logout must also have an explicit revocation scope. Clearing local browser
state is not, by itself, proof that a shared identity can no longer act.

## What we observed

Different applications made locally reasonable assumptions about cookies,
redirects, selection state, and the point at which a session was considered
valid. Those assumptions diverged at the seams between applications.

A token could be valid while the selected account context was stale, a
continuation was incomplete, or another application still held usable local
state. Locale propagation had the same seam: it was part of a journey, but was
not always treated as part of the shared contract.

## What was actually wrong

The platform had allowed several related decisions to be owned independently:

- authentication was treated as token acceptance
- account/store selection was treated as a UI handoff
- logout was treated as deletion of one application cookie
- locale was treated as presentation state

In a multi-application system, those are coupled protocol states. The failure
was therefore a contract-boundary problem, not a missing conditional in one
application.

## Why the original model was insufficient

```text
valid session token
        !=
complete authenticated application state
```

The original model collapsed authentication, authority, tenant selection,
continuation, logout, and locale into a single boolean notion of “logged in.”
It did not account for retries, stale handoffs, membership changes, or
different revocation scopes.

Fixing each application independently would have preserved the inconsistency:
the same identity transition could still be interpreted differently at the
next application boundary.

## Systemic correction

The correction moved to the shared identity and platform-contract boundary.
The resulting design treats the following as explicit protocol concerns:

- current human-session authority rather than token validity alone
- current membership and installation authority for the selected context
- retry-safe account/store selection
- fresh continuation semantics at handoff boundaries
- centralized logout semantics with explicit revocation scope
- clearing local state before terminating shared identity state
- deterministic locale propagation across the journey

Applications consume the shared contract and re-check current authority at
consequential boundaries. A stale session, membership, or lifecycle context
fails closed instead of being trusted because an earlier UI step succeeded.

## Regression strategy

Regression coverage was expanded from single logout redirects to suite-wide
journeys and negative paths. It exercises, at the appropriate abstraction
level:

- stale and revoked sessions
- membership changes and installation supersession
- retry and resume during account/store selection
- replayed security events with idempotent handling
- cross-application and cross-realm isolation
- logout mutation protection and local-state clearing
- locale-safe provider return and continuation

The test question changed from “did this app delete its cookie?” to “can a
consequential request still establish current authority for the exact context
it is using?”

## What we deliberately do not disclose

This note omits session identifiers, cookie names, token formats, endpoint
paths, signing or provider configuration, internal service names, private
contract structures, customer or tenant identifiers, and attack-relevant
timing details.

## Engineering lesson

Authentication is not simply token validation in a multi-application platform.
Identity, authority, tenant selection, continuation, logout, and locale form a
distributed protocol. Its invariants belong at the platform boundary and must
be tested across the suite.

Related reading: [Logout is an authority problem](../notes/02-logout-is-authority.md)
and [Generation fencing](../engineering/generation-fencing.md).
