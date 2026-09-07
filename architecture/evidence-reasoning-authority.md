# Evidence, reasoning, and authority

```text
Evidence != Reasoning != Authority
```

This is Vertex's most important boundary. A system becomes difficult to correct when a fetched provider object, a model's interpretation, and permission to mutate a store are all represented by the same loosely scoped application state.

```text
PROVIDER OBSERVATION
       |
       v
CANONICAL EVIDENCE
       |
       v
BRAIN INTERPRETATION
       |
       v
ACTION PROPOSAL
       |
       v
AUTHORITY CHECK
       |
       v
EXECUTION
       |
       v
READBACK
       |
       v
NEW EVIDENCE
```

## Evidence: what was observed?

Evidence is a bounded claim about external or system state with context: tenant/store scope, source, time, coverage, provenance or receipt references, and quality conditions such as missingness, staleness, or conflict. Vertex's MDP includes evidence snapshots, coverage, reconciliation state, metric-computation receipts, evidence digests, and readiness assessments.

An evidence artifact should be able to say "this is unavailable," "this is stale," or "these sources conflict." Suppressing a conclusion can be more correct than presenting a confident but unsupported one.

Evidence is not an LLM answer. A model can summarize or interpret evidence, but its output does not make the underlying observation more authoritative.

## Reasoning: what does the evidence mean?

Brain produces interpretations: diagnosis, ranking, scoring, predicted value, recommendations, capability results, or action proposals. These artifacts should identify the decision version and the evidence or snapshot version on which they depend.

Reasoning must remain revisable. New evidence, changed coverage, a newer model, or a corrected interpretation can supersede a decision without rewriting the historical evidence record.

```text
BRAIN PROPOSES.
BRAIN DOES NOT SELF-AUTHORIZE.
```

## Authority: may this happen now?

Authority answers a different question: whether a particular principal or workload may perform a particular action for a particular scope at the time of admission. Vertex's Control Plane evaluates current session and membership state, role policy, installation identity, current generation, and action/capability policy rather than accepting an old recommendation as a permit.

Conceptually, a permit is bounded by:

```text
principal + account/store + app + installation + generation + action/purpose
+ policy revision + time
```

The exact public representation is intentionally not specified. The principle is that a credential, cached UI state, or model output must not become broad, durable authority.

## Why the separation matters

### Failure story: an old recommendation

1. Brain interprets current evidence and recommends action X.
2. The recommendation remains visible or queued.
3. Merchant membership, store selection, installation state, or policy changes.
4. The old proposal is submitted for execution.

The correct response is not to trust the historical recommendation. Authority must be evaluated independently at execution admission, against current scope and policy. If the context no longer matches, work is denied or becomes stale; it does not inherit permission from an earlier inference.

### Failure story: a confident but incomplete model answer

A model may produce a plausible conclusion when a required provider domain is unavailable or stale. The remedy is not a more persuasive response. The evidence layer must expose readiness and missingness so the consuming capability can suppress, degrade, or qualify the result.

## Versioning and references

Evidence references matter because they give a decision an inspectable basis. Decision versions matter because conclusions are time-bound. Policy revisions matter because authority is evaluated under a changing policy. Together, these references make it possible to ask: *what was known, what did the system infer, and why was this action allowed?*

## Design questions

**Why not let Brain call a provider directly?** Interpretation quality is not an authorization model, and direct calls would couple model behavior to a broad credential boundary.

**Why record readback as evidence rather than only a job result?** The observed state is input to the next decision cycle; it is more than a transient execution log.
