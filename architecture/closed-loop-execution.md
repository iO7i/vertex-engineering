# Closed-loop execution

An action is not complete when a request is sent. Vertex models a provider-facing action as a closed loop that starts and ends with observed state.

1. Reality changes at a provider or external source.
2. Vertex observes the change through a supported source path.
3. The observation enters canonical evidence with scope and provenance references.
4. Brain interprets the currently available evidence.
5. Brain produces a versioned decision or proposal.
6. The product exposes that derived result to an application or merchant.
7. A merchant or system requests an action.
8. Control Plane evaluates current authority for the exact scope and purpose.
9. Durable workflow state begins or resumes bounded work.
10. A specialist executor receives only its admitted context.
11. The executor requests a provider mutation.
12. The provider response is retained as a dispatch receipt, not a final outcome.
13. Readback observes later provider state.
14. Expected and observed state are reconciled.
15. The result becomes new canonical evidence, including mismatch or ambiguity where present.
16. Brain reasons from the updated evidence in the next cycle.

```text
observe -> evidence -> interpret -> propose -> authorize -> execute
   ^                                                     |
   |                                                     v
   +------ reconcile <- read back <- provider response <-+
```

## Request success is not business outcome

```text
request success != business outcome
```

A provider can acknowledge a request while a readback later shows a different state. The discrepancy could be eventual consistency, a partial provider effect, a mapping issue, an independent actor overwriting a value, or a bad expectation. This dossier does not assume a single cause. It records the distinction so the system can reconcile rather than silently equate HTTP-level success with merchant-level completion.

## Ambiguous completion

Consider a mutation that is committed by a provider but whose response is lost:

```text
T0  Control Plane admits an action.
T1  Worker sends provider mutation.
T2  Provider commits the mutation.
T3  Network connection fails before the worker records success.
T4  Workflow sees timeout or crash.
T5  Retry becomes eligible.
```

Blindly retrying at T5 can create a duplicate effect. Treating the timeout as failure can leave a completed effect unrecorded. The resolution is a deliberate combination of operation identity, idempotency where the provider supports it, durable retry state, and provider readback/reconciliation. None of those independently creates exactly-once delivery.

## Bounded context at execution

The executor must not infer authority from workflow age, a cached proposal, or access to a provider client. Admission binds context such as installation and generation; the execution path checks that the operation still belongs to an allowed current context before a mutation is made. This is particularly important when work resumes after a long delay.

## Design questions

**What if readback is delayed?** Retain the dispatch state as pending/ambiguous rather than translating absence of immediate confirmation into success.

**What if current evidence advances while a proposal is served?** A proposal can be marked version-bound, refreshed, suppressed, or independently re-authorized. It should not overwrite newer canonical evidence.
