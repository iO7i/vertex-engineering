# Incident 002 — When Deployment Provenance Could Not Prove Itself

## Summary

During deployment-readiness work, we examined a provenance mechanism intended
to answer a simple question:

> What canonical source revision is this running service actually attributable
> to?

The mechanism could report an identity derived from the deployment or upload
path rather than the canonical source revision we intended it to attest. A
healthy response and a plausible version string were therefore not enough to
prove the relationship between reviewed source, built artifact, and running
service.

The issue was found during verification rather than presented as a customer
outage. The important failure was in the assurance mechanism itself.

## Impact

Release verification could have accepted a plausible runtime identity without
being able to independently establish that the reviewed source produced the
artifact serving traffic.

That weakens the evidence chain even when the process is healthy. It also makes
it harder to distinguish a source mismatch from a normal application-version
change.

## Expected invariant

```text
reported source identity
        ==
canonical source revision that produced the deployed artifact
```

The stronger release chain is:

```text
canonical source
      ↓
reproducible build inputs
      ↓
immutable artifact identity
      ↓
deployment record
      ↓
runtime provenance and readiness
```

Liveness, version labeling, readiness, and provenance answer different
questions. They should not be collapsed into one endpoint or one string.

## What we observed

A version-style response could return successfully and look credible while the
source/artifact relationship remained ambiguous. Metadata supplied by the
deployment path could be mistaken for identity of the canonical source.

The symptom was subtle: nothing had to crash. The response merely appeared to
prove more than it actually established.

## What was actually wrong

The provenance producer trusted metadata available at deployment time without
requiring that metadata to be bound to the canonical source and immutable
artifact under review.

In effect, the path that uploaded or launched the artifact could influence the
identity later used to verify it. The evidence source was therefore not
independent of the process it was supposed to assess.

## Why the original model was insufficient

The original model was:

```text
version endpoint returns 200
        +
version looks plausible
        =
deployment is proven
```

That confuses process liveness, package identity, build identity, deployment
identity, and readiness. A process can be healthy and accurately report a
configured label while still running an artifact that is not the intended
source revision.

When an endpoint is used as release evidence, the endpoint's own identity
semantics become part of the trust boundary.

## Systemic correction

The correction established one canonical source of truth for revision identity
and separated the evidence types:

- health reports whether a process responds
- readiness reports whether required runtime conditions hold
- version reports declared application identity
- provenance binds source, build inputs, and immutable artifact identity

Release verification now compares expected and observed provenance instead of
accepting a plausible label. Missing or conflicting provenance fails closed.
The build records an auditable provenance document, while deployment evidence
records the observed artifact and readiness relationship without exposing
private infrastructure details.

## Regression strategy

The regression strategy tests the assurance mechanism as carefully as the
runtime:

- missing required release identity is rejected
- source and build inputs are bound to the immutable artifact identity
- tampered provenance data is rejected
- expected and observed release identities must agree
- readiness remains distinct from provenance
- the release checklist requires source, artifact, deployment, readiness, and
  compatibility evidence together

The relevant question is no longer “does `/version` look right?” It is “can the
release evidence establish the source-to-runtime relationship without relying
on a silently substitutable label?”

## What we deliberately do not disclose

This note omits provider and cloud resource names, production hostnames,
deployment identifiers, commit hashes, private repository paths, environment
variable names, credentials, exact build configuration, package pins, and
internal artifact formats.

## Engineering lesson

Observability becomes part of the trust boundary when it is used as evidence.
A system cannot claim to prove what was deployed if the mechanism reporting the
identity can be silently substituted by deployment machinery.

Related reading: [Why `/version` can lie](../notes/01-deployment-provenance.md)
and [Certification strategy](../reliability/certification-strategy.md).
