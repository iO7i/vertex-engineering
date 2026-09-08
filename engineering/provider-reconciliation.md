# Provider reconciliation

Providers are independent systems. Vertex does not control their delivery guarantees, consistency windows, webhook behavior, rate limits, or the semantic gap between an API acknowledgement and a merchant-visible result.

```text
INTENT
  |
  v
REQUEST
  |
  v
PROVIDER ACK
  |
  X  <-- not enough
  |
  v
OBSERVED PROVIDER STATE
  |
  v
RECONCILED OUTCOME
```

## Sources of disagreement

- Webhooks can be duplicate, delayed, out of order, or missing.
- Provider APIs can be eventually consistent.
- A request can time out after the provider committed it.
- A response can describe a partial effect.
- Different provider IDs and object shapes require canonical mapping.
- A later read can observe an intervening change by another actor.

The response is not to pretend all providers behave alike. Platforms translate provider-specific objects and capabilities into canonical contracts, while preserving enough source and receipt information to interpret differences later.

## Acknowledgement is transport evidence

A successful provider response establishes that a request reached some provider boundary under some conditions. It does not automatically establish that the intended state exists, that no other field changed, or that the state is visible to the merchant. Reconciliation compares expected and observed state after a suitable readback path.

```text
expected state + observed state + receipt/provenance -> outcome classification
```

Possible classifications are intentionally conceptual: confirmed, pending visibility, mismatched, conflicted, unavailable, or ambiguous. The exact schema is private because it is implementation and product-specific.

## Readbacks as bounded evidence

The reviewed Kernel provider-readback interfaces bind a read to merchant/store/provider, installation, generation, purpose, and a bounded time window. They retain normalized data and digests/receipts instead of exposing credentials or raw provider responses to callers. This supports provenance without turning a provider client into ambient authority.

## Failure story: webhook before API read

An event can arrive before a polling/API read reflects the associated mutation. If the system assumes global ordering, it may incorrectly roll back a valid conclusion. A reconciliation design records source timing and coverage, waits or retries within a policy, and represents unresolved disagreement rather than fabricating certainty.

## Design questions

**Why not trust webhooks over reads?** Both are observations with different failure modes. Their relationship must be reconciled, not assumed.

**What if the provider cannot support a required readback?** The result should be qualified or unavailable for that outcome, rather than treated as verified.
