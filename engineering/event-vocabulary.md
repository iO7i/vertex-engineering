# Event vocabulary

Vertex uses domain events to make state transitions, asynchronous work, and reconciliation inspectable. This is a compact public vocabulary for explaining those transitions. It is not a wire format, a complete production schema, or an inventory of private event names.

## Event envelope

Every event should be interpretable without exposing merchant payloads. The conceptual envelope is:

| Field | Purpose |
| --- | --- |
| `event.name` | A stable, low-cardinality description of the occurrence. |
| `occurred_at` | When the occurrence happened, distinct from ingestion or processing time. |
| `merchant_scope` | A privacy-preserving scoped reference, not a customer identifier or raw payload. |
| `installation_generation` | The lifecycle generation against which the event was evaluated. |
| `logical_operation_id` | Present only for work related to one bounded external effect. |
| `correlation_id` | Connects related observations and workflow progress without becoming authority. |
| `outcome` | A constrained result such as admitted, rejected, acknowledged, observed, or unresolved. |
| `reason_code` | A stable explanation category; free-form diagnostic detail remains bounded. |

Identifiers and event attributes must never contain credentials, access tokens, raw provider payloads, private infrastructure addresses, or merchant content.

## Conceptual event families

| Family | Example occurrence | Why it exists |
| --- | --- | --- |
| `evidence.*` | `evidence.observed`, `evidence.conflicted` | Distinguishes external observations from interpretation. |
| `reasoning.*` | `reasoning.proposal.created`, `reasoning.proposal.suppressed` | Shows that a proposal is not an execution decision. |
| `authority.*` | `authority.action.admitted`, `authority.generation.rejected` | Records current-scope authorization decisions. |
| `workflow.*` | `workflow.activity.retry_scheduled`, `workflow.resume_started` | Makes progress, retries, and resume behavior observable. |
| `provider.*` | `provider.mutation.dispatched`, `provider.acknowledgement.unknown` | Separates a transport attempt from a verified outcome. |
| `reconciliation.*` | `reconciliation.readback.observed`, `reconciliation.outcome.unresolved` | Represents disagreement or absence without fabricating certainty. |
| `serving.*` | `serving.projection.rebuilt`, `serving.projection.suppressed` | Keeps derived product state distinct from canonical evidence. |

The example names are explanatory. They are not a promise that every application emits every event or that these are the private implementation names.

## Naming and evolution

- Name an event for a completed occurrence, not an implementation detail or a dynamic identifier.
- Put changing identifiers in scoped attributes, never in the event name.
- Add an event only when there is a defined question it helps operators or engineers answer.
- Version semantics deliberately when a consumer could otherwise reinterpret historic evidence.
- Deprecate a meaning with a migration note rather than silently reusing a familiar name.

## A recovery trace

```text
authority.action.admitted
provider.mutation.dispatched
provider.acknowledgement.unknown
workflow.resume_started
reconciliation.readback.observed
reconciliation.outcome.reconciled
```

This trace does not turn event history into authority. The Control Plane remains responsible for whether a resumed action may proceed, and readback remains evidence rather than a retroactive transport acknowledgement.

## What this does not claim

The vocabulary does not guarantee that events are complete, ordered, durable, or globally consistent. Those properties depend on the relevant component and provider boundary. Its purpose is narrower: name important transitions so ambiguity, rejection, and reconciliation remain observable.
