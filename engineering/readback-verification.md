# Readback verification

```text
A successful tool or API response is not the intended outcome.
```

Provider-facing work has at least four distinct artifacts:

```text
dispatch receipt -> provider acknowledgement -> observed state -> outcome reconciliation
```

Collapsing them makes operational reports look cleaner while obscuring the question a merchant actually cares about: did the expected state occur?

## Generic example

```text
Intent:   change product attribute to X
Provider: HTTP 200
Readback: attribute remains Y

Conclusion:
transport succeeded
desired outcome did not become observed truth
```

The difference can arise from eventual consistency, unsupported semantics, a partial effect, a later overwrite, or an incorrect expectation. A verification model need not assert the cause immediately. It must retain the evidence needed to represent the mismatch and decide whether to retry, wait, escalate, or stop.

## The loop

```text
PROPOSE
  |
AUTHORIZE
  |
EXECUTE
  |
OBSERVE
  |
VERIFY
  |
LEARN
  +-----> next decision
```

Readback becomes new evidence because it updates what the system can responsibly claim about merchant reality. It is not merely a logging step at the end of a job.

## Expected versus observed

Verification should bind expected state to the operation and scope that created it. Readback should be normalized and provenance-aware. Comparisons should account for the supported provider capability and consistency behavior rather than using a generic equality test across all provider objects.

Vertex's provider-readback interfaces bind source reads to scope, installation generation, purpose, and response receipts. That supports the architecture; detailed comparison logic and provider payloads are intentionally excluded.

## Failure story: a lost acknowledgement

If a worker loses a response after submitting a mutation, an acknowledgement may be absent even though the desired state exists. Readback can establish the observed state and allow the workflow to converge without a duplicate request. Conversely, an acknowledgement with no observed effect should not be reported as a completed business outcome.

## Design questions

**Can every action be verified immediately?** No. Verification may be delayed, unavailable, or subject to provider consistency. Those are states to model, not excuses to invent success.

**Does readback make provider state canonical?** It contributes provider-derived evidence to MDP. Vertex's canonical record includes provenance and reconciliation context, not an unqualified copy of arbitrary provider data.
