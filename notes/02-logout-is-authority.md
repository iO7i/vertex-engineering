# Engineering note: logout is an authority problem

## Symptom

The suite-wide logout problem looked like a browser-session problem: one app
cleared its cookie and redirected to the identity provider, yet other surfaces
could still have stale session context, cached installation selection, or an
unclear revocation scope. Clearing one local cookie does not answer whether the
presented identity is still allowed to act.

## Naive model

Logout means “delete the app cookie.” A shared human identity is treated as one
global boolean, and membership or installation state is trusted from the last
successful UI resolution. This makes exact-session revocation, user-wide
deletion, app-specific authorization, and membership changes indistinguishable.

## Real invariant

Every consequential request must re-establish current authority for the exact
identity, app, membership, store, installation, and generation it is using.
Revocation scope must be explicit: a `session.revoked` fact denies that exact
session; a `user.deleted` fact denies every session for that issuer and subject.
Neither event silently deletes merchant data or changes unrelated accounts.

## Architecture / fix

Keep the identity provider responsible for authentication and the Control Plane
responsible for current membership and installation authority. Record security
events idempotently by provider event identity. Resolve app sessions through the
current membership/install joins instead of accepting browser-provided account
or store claims. The app-owned logout bridge requires the browser-mutation
check, clears the host-only cookie, and then sends the user through the
provider logout path. A stale session fails closed at the next authority check.

## Regression proof

The logout route tests cover CSRF rejection, cookie clearing, and locale-safe
provider return. Control Plane tests cover replayed security events, exact
session revocation, and user-wide revocation of other sessions. The lifecycle
model also exercises stale sessions, membership changes, installation
supersession, and cross-realm isolation rather than testing logout as a single
HTTP redirect.

Private identity-provider configuration and internal session protocol details
are intentionally not included here.
