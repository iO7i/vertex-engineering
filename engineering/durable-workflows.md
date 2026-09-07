# Durable workflows

External commerce operations can outlive a request handler. Workers restart, providers throttle, long enumerations time out, and a recovery must resume without silently forgetting what already happened. Vertex uses durable workflow infrastructure in representative applications to carry workflow state, retries, timeouts, replay-compatible execution, and long-running progress.

```text
Temporal remembers workflow progress.
MDP remembers merchant evidence.
Control Plane determines authority.
Brain determines interpretation.
```

Those responsibilities are deliberately different.

## What durable work is for

- Track progress across process restarts.
- Retry bounded activities with explicit timeout and backoff policies.
- Coordinate long-running sync and reconciliation work.
- Limit provider pressure through scheduling and concurrency choices.
- Preserve enough history to replay or diagnose workflow behavior.
- Continue unfinished work rather than relying on an in-memory queue.

The available SEO application snapshot, for example, contains separate workflow, worker, activity, schedule, replay-evidence, and restart-probe concerns. Its workflow definitions use bounded retry policies and avoid a workflow-level fallback that could mutate a projection after ownership moved. That informs the principle below without disclosing workflow names, queues, or deployment configuration.

## Crash and recovery sequence

```text
T0  workflow starts a provider synchronization activity
T1  activity checkpoints/records bounded progress as designed
T2  worker process stops
T3  workflow infrastructure reschedules eligible work
T4  activity resumes or retries under its configured policy
T5  provider-facing effects are reconciled before duplicate dispatch
```

Durability prevents loss of workflow progress; it does not prove that an external side effect was not already committed. That is why workflow retries and the [idempotency](idempotency.md) / [reconciliation](provider-reconciliation.md) boundaries must work together.

## Replay and compatibility

Long-lived workflow histories introduce a versioning problem: changing code must not reinterpret existing history unpredictably. Vertex includes replay-oriented verification and explicit retention of selected compatibility paths. The public lesson is modest: workflow code should evolve with history compatibility in mind, and release evidence should include relevant replay or compatibility checks when a system uses durable execution.

## Authority on resume

A workflow's original start time is not a permanent permit. Before sensitive external effects, the relevant current authority and generation must still be honored. Temporal—or any durable engine—does not create business authorization merely by replaying a prior command.

## Design questions

**What does a workflow own?** Progress, deadlines, retries, and coordination.

**What does it not own?** Canonical merchant truth, authorization policy, provider semantics, or the meaning of a Brain recommendation.
