# Multi-tenant boundaries

Multi-tenancy is not a single tenant ID copied through requests. Vertex treats tenant context as a chain of relationships with independently changing state.

```text
HUMAN
  |
  v
CENTRAL IDENTITY / SESSION
  |
  v
ACCOUNT MEMBERSHIP
  |
  +------------------+------------------+
  |                                     |
  v                                     v
STORE A                               STORE B
  |                                     |
  v                                     v
INSTALLATION                         INSTALLATION
  |                                     |
  +--> GENERATION N                   +--> GENERATION M
  |                                     |
  v                                     v
human authority session            app / capability context
```

The available implementation evidence includes explicit installation-authority records, installation generations, human authority sessions, account and installation memberships, policy revisions, and scoped workload behavior. This document describes their logical role, not their storage schema.

## Distinctions that matter

```text
same human           != same store
same account         != every installation
same installation ID != every generation
previous generation  != current authority
```

Each equality that is assumed without verification can become a cross-tenant leakage, stale-authority, or unintended-action defect.

## Scope as a tuple

A public conceptual scope can contain:

```text
human | account | merchant | store | provider | application
installation | installation generation | purpose | operation
```

Not every operation needs every field, but omitting a relevant field should be a decision—not an accident. Provider reads and mutations are particularly sensitive to binding the external store, installation, generation, and purpose to trusted authority rather than accepting caller-supplied values at face value.

## Cross-tenant leakage as a failure class

Cross-tenant leakage includes more than a wrong database query. It includes stale cached projections, an executor retaining a prior store binding, a readback attached to another installation, a credential used outside its purpose, or a session continuing after membership was revoked. The architecture responds by making scope explicit in contracts, independently evaluating authority, and retaining evidence/provenance that can expose mismatches.

## Identity lifecycle

The logical path is:

```text
human -> central identity/session -> account membership -> store selection
      -> installation membership -> current installation generation
      -> merchant context
```

The implementation snapshot includes checks for active sessions and memberships at action evaluation, along with identity security-event handling. This supports a fail-closed approach: if current state cannot establish the required relationship, the action is not admitted. This public dossier intentionally does not publish identity-provider configuration, session formats, or revocation internals.

## Design questions

**Why bind generation as well as installation?** An installation can be replaced, revoked, or re-established. The identity may remain recognizable while its prior authority should not.

**Why not make an app database the tenant authority?** Applications should add domain value; a local convenience model is not a substitute for suite-wide membership, installation, and lifecycle truth.
