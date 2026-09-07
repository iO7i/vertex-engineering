# Generation fencing

Installation identity alone is often too weak. A store can uninstall, reinstall, change authority, or otherwise move to a new lifecycle generation while old work is still queued.

```text
INSTALLATION
    |
    +-- generation 17   STALE
    |
    +-- generation 18   CURRENT
```

Vertex's Control Plane and application code include installation generations and tests/migrations concerned with reconciling or retiring superseded installation authority. This documentation abstracts the mechanism into a public principle.

```text
OLD AUTHORITY MUST NOT SILENTLY SURVIVE THE GENERATION CHANGE.
```

## The race

```text
T0  G17 worker starts with an admitted task.
T1  merchant reinstalls or authority changes.
T2  G18 becomes current.
T3  old G17 worker resumes after delay or retry.
T4  old worker attempts mutation.
```

Without a generation fence, the worker sees a familiar installation ID and proceeds using authority that was correct at T0 but no longer current at T4. A generation-bound system recognizes that identity continuity does not establish authority continuity.

## Conceptual fence

```text
permit = authorize(context, action)

if permit.generation != current_generation(context.installation):
    reject(STALE_AUTHORITY)

dispatch(bound_action)
```

The pseudocode is explanatory, not implementation code. In practice, the critical property is that the current generation is resolved from a trusted authority path and compared at the right side-effect boundary. Cached or caller-supplied generations do not create a reliable fence.

## Where generation should travel

Generation binding is relevant to authority decisions, workload identity, provider reads, credential leasing, queued work, execution receipts, and readback. The reviewed Kernel readback pattern, for example, binds installation and generation with merchant/store/provider and purpose context before accepting a provider response as a matching read.

## Failure story: credential after reinstall

Suppose a prior lifecycle's credential remains technically usable for a short period after a new installation is active. The fact that transport authentication works does not mean the old workflow is still allowed to use it. Generation fencing makes current lifecycle authority a separate condition from credential availability.

## Relationship to revision-bound authority

This is a general systems pattern: a permission should be bound to the revision of the world in which it was granted. Here, the relevant revision is installation generation. The same idea can apply to membership revisions, policy revisions, data snapshots, or any authorization state that can change while work is delayed.

## Design questions

**Can a workflow engine solve this by itself?** No. Workflow durability preserves progress; it does not know which installation generation is current.

**Should an old worker finish harmless read-only work?** That is a policy decision. For mutation, stale authority should fail closed unless a specifically designed policy says otherwise.
