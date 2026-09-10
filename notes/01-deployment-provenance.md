# Engineering note: why `/version` can lie

## Symptom

During a deployment audit, a healthy version response was not enough to answer
the question that mattered: which source tree and build is actually serving
traffic? A process can report a configured package version, a copied environment
variable, or a platform label while the running image, migration head, or worker
is from a different release.

## Naive model

Treat `/version` as deployment proof. If it returns `200` and a plausible
semantic version, assume the source, image, deployment, and database are aligned.
That collapses liveness, package identity, build identity, and deployment identity
into one string.

## Real invariant

Runtime identity is a tuple, not a label: source commit/tree, declared build
inputs, immutable image or manifest digest, deployment identifier, migration
head, and readiness state must be inspectable and mutually compatible. A version
endpoint may describe a process; it cannot prove which artifact was deployed.

## Architecture / fix

Keep `/health`, `/ready`, and `/version`/`/provenance` separate. Require
production startup metadata instead of silently inventing it. Bind the exact
source tree, Dockerfile, lockfile, SBOM, and OCI manifest into a build-only
provenance document. At release verification, compare that document with the
immutable deployed image and record the deployment ID and readiness result.
An unprotected or unattested build is review evidence, not release provenance.

## Regression proof

The runtime tests reject missing production build SHA and deployment ID while
keeping readiness honest when persistence is absent. The provenance-generator
tests bind source commit/tree and build inputs to an OCI manifest and reject a
tampered manifest blob. The deployment runbook requires source commit, image
digest, deployment ID, `/provenance`, `/readiness`, migration checksums, and
runtime-role evidence together.

This note intentionally omits private service names, URLs, and deployment
identifiers.
