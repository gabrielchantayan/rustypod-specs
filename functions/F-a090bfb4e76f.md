<!-- Trusted provenance: F-a090bfb4e76f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a090bfb4e76f

## Purpose

F-a090bfb4e76f recognizes one of two event categories and conditionally invokes the corresponding action when a supplied 32-bit value differs from a shared state value. It returns a 32-bit recognition result.

The established behavior distinguishes three outcomes:

- The first event category is recognized: conditionally invoke F-5ae09c305385, then return 1.
- The second event category is recognized: conditionally invoke F-18f063f11849, then return 1.
- Neither category is recognized: return 0 without invoking either action.

A recognized event returns 1 even when its action is skipped because the supplied value equals the relevant shared-state value. The result therefore indicates recognition, not necessarily a state change or successful action.

The embedded naming suggests selection and highlighting behavior involving cities. That interpretation is inferred intent, not an established contract for the event representation, action semantics, or shared state.

## Interface

### Inputs

The function accepts three inputs:

| Input | Representation | Established use |
|---|---|---|
| Context | Opaque 32-bit value | Passed unchanged as the first argument to whichever action is invoked. Not otherwise inspected by this function. |
| Event descriptor | Opaque 32-bit value | Passed as the first argument to the event-comparison dependency, once or twice. |
| Candidate value | 32-bit integer | Compared for inequality with a shared 32-bit state value and, when an action is required, passed unchanged as that action’s second argument. |

The 32-bit interpretation follows the supplied source representation: explicitly four-byte opaque values and integer-valued comparisons. The underlying calling convention, register assignments, and any pointer interpretation are not established.

Equality is the only operation applied to the candidate value. Signed ordering, range checks, arithmetic, truncation, and widening do not participate in the visible decision. All 32-bit candidate patterns are therefore treated alike by the visible logic: only equality or inequality with the selected shared-state observation matters. Dependencies may impose additional input restrictions.

The context and event descriptor might encode addresses, handles, integers, or another representation. Their validity requirements are unknown. The function does not establish that zero is a valid or invalid value for either input.

### Shared input

The function reads one shared 32-bit state location. This document calls it the **shared state**.

The source does not establish:

- Its initialization requirements.
- Whether it represents a selected item, highlighted item, identifier, or sentinel.
- Whether dependencies can modify it.
- Whether concurrent access is permitted.
- Whether reads have volatile or synchronization semantics.

The placement of reads in the supplied source is nevertheless significant and is specified below.

### Output

On every visible normal-return path, the function returns a four-byte value whose numeric value is either:

- **1:** one of the event categories was recognized.
- **0:** both event comparisons produced nonzero results.

No other normal return value is visible.

These results are conditional on called dependencies returning normally. The function does not visibly handle dependency failure, exceptions, nonlocal exits, or nontermination.

## Behavior

### Event recognition

The function uses two fixed event tokens:

- **Token A:** the token used for the first event comparison.
- **Token B:** the token used for the second event comparison.

Their bytes, lengths, storage format, and termination rules are not provided. The surrounding naming suggests that Token A represents selection and Token B represents highlighting, but the contract does not require particular textual spellings.

For both comparisons, a zero result means that the corresponding category is recognized. Any nonzero result means it is not recognized. The function does not distinguish positive from negative nonzero results.

The event descriptor is supplied unchanged to both comparisons. This does not imply that any storage it refers to remains unchanged between calls.

### Common initial sequence

Every invocation that progresses normally begins with these observable operations, in order:

1. Call the comparison dependency with the event descriptor and Token A.
2. After that call returns, read the shared state and retain that value as a snapshot.
3. Select the first-category or fallback path according to whether the first comparison returned zero.

Consequently, the snapshot is obtained after any side effects of the first comparison. It is not an entry-time snapshot.

The snapshot read is present even when the first category is recognized and that snapshot will not determine the action decision.

### First category recognized

When the first comparison returns zero:

- The function obtains the shared state again for the candidate-value equality decision.
- If the candidate differs from this second shared-state observation, it calls F-5ae09c305385 with the context and candidate value, in that order.
- If the candidate equals this second observation, it invokes neither action.
- It returns 1 after any invoked action returns normally.

Token B is not compared on this path.

The retained snapshot from the common initial sequence does not control this branch. The distinction between the snapshot read and the subsequent shared-state read matters if the shared state can change between them.

The function does not recheck the shared state after the action. It does not inspect the action’s return value.

### First category not recognized

When the first comparison returns nonzero:

- The function calls the same comparison dependency with the event descriptor and Token B.
- If that second comparison returns nonzero, the function returns 0.
- If the second comparison returns zero, the function compares the candidate value with the previously retained shared-state snapshot.
- If those values differ, it calls F-18f063f11849 with the context and candidate value, in that order.
- If those values are equal, it invokes neither action.
- It returns 1 after any invoked action returns normally.

This path does not reread the shared state for its action decision. In particular, mutations performed by the second comparison do not replace the retained snapshot.

### Precedence and exclusivity

The first category has precedence. Once its comparison returns zero, the function does not ask whether the second category also matches.

At most one action is invoked per call. The two action dependencies cannot both be reached during one normally executing invocation of the visible function body.

For each recognized category, equality suppresses only the associated action. It does not change the recognition result from 1 to 0.

### State changes during comparisons

Dependency side effects must not be assumed absent:

- A shared-state mutation during the first comparison can affect the snapshot and the first-category equality decision, because both reads occur afterward.
- A mutation during the second comparison does not alter the retained value used by the second-category equality decision.
- A change between the two shared-state reads on the first-category path can make the second read differ from the snapshot.

The supplied source establishes these read placements at its represented level. It does not establish machine-level atomicity or a concurrency model.

## Side effects

### Direct effects

The visible function body:

- Performs one event-comparison call unconditionally.
- Performs a second event-comparison call only if the first returned nonzero.
- Reads the shared state once on the fallback path and twice on the first-category path.
- May invoke one action.
- Does not explicitly write the shared state.
- Does not explicitly allocate, release, copy, or transfer ownership of memory.
- Does not explicitly produce logging, user-interface output, or another external result beyond its return value.

The absence of direct writes does not mean the overall invocation is side-effect-free. All dependencies may have effects not shown here.

### Ownership and lifetime

The visible function passes the context, event descriptor, and candidate as values. It contains no explicit retain, release, destruction, or ownership-transfer operation.

No ownership conclusion follows for resources encoded by those values. In particular:

- The comparison dependency may read or modify memory reachable through the event descriptor.
- An action may read or modify resources reachable through the context.
- A dependency may allocate or release resources.
- A dependency may change the shared state or other global state.

Whether any such behavior actually occurs is unknown. Callers must satisfy the real dependency contracts; this function provides no visible validation layer.

### Errors and abnormal completion

There is no visible error-recovery path. A nonzero comparison result is treated as nonrecognition, not separately classified as an error.

Action results are ignored. A recognized event can therefore produce a return value of 1 regardless of an action’s returned status, provided the action returns normally.

If a dependency does not return normally, the visible function provides no established guarantee that it returns a recognition result or completes later operations.

## Dependencies

| Opaque identifier | Role | Invocation contract visible here |
|---|---|---|
| F-unmapped-01 | Event comparison; document-local opaque ID because no mapped ID was supplied | Receives the event descriptor and one fixed token. Called first with Token A and conditionally with Token B. Zero selects the corresponding category; all nonzero results reject it. |
| F-5ae09c305385 | First-category action | Receives the unchanged context and candidate value. Called only after Token A is recognized and the candidate differs from the second shared-state observation. Its result is not consumed. |
| F-18f063f11849 | Second-category action | Receives the unchanged context and candidate value. Called only after Token A is rejected, Token B is recognized, and the candidate differs from the retained snapshot. Its result is not consumed. |

F-unmapped-01 is a label local to this specification, not an asserted published identity.

No dependency is assumed to preserve globals, preserve pointed-to contents, own arguments, release arguments, or return successfully. The exact token representation and dependency signatures beyond the visible arguments remain undocumented.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Each assumes that invoked dependencies return normally.

### First category recognized, candidate unchanged

Conditions:

- The Token A comparison returns zero.
- The candidate equals the second shared-state observation.

Required behavior:

- Read the shared state once for the snapshot and again for the equality decision.
- Do not compare against Token B.
- Invoke neither action.
- Return 1.

### First category recognized, candidate differs

Conditions:

- The Token A comparison returns zero.
- The candidate differs from the second shared-state observation.

Required behavior:

- Invoke F-5ae09c305385 exactly once with the context and candidate.
- Do not invoke F-18f063f11849.
- Return 1 after the action returns, without interpreting its result.

### Second category recognized, candidate unchanged

Conditions:

- The Token A comparison returns nonzero.
- The snapshot equals the candidate.
- The Token B comparison returns zero.

Required behavior:

- Invoke neither action.
- Return 1.
- Do not substitute a later shared-state value for the snapshot.

### Second comparison changes the shared state

Conditions:

- The candidate is 12.
- The first comparison rejects Token A.
- The subsequent snapshot is 9.
- The second comparison recognizes Token B and changes the shared state to 12.

Required behavior:

- Invoke F-18f063f11849 with candidate 12 because the retained snapshot was 9.
- Return 1.

The action is not suppressed merely because the shared state now equals the candidate.

### Second comparison changes an initially equal state

Conditions:

- The candidate is 12.
- The snapshot is 12.
- The second comparison recognizes Token B and changes the shared state to 9.

Required behavior:

- Do not invoke F-18f063f11849.
- Return 1.

The decision uses the snapshot, not the changed state.

### First-path reads differ

Conditions:

- The Token A comparison returns zero.
- The snapshot is 9.
- The later shared-state observation is 12.
- The candidate is 12.

Required behavior:

- Skip the first-category action.
- Return 1.

This case is conditional on the environment permitting the shared state to change between the represented reads.

### Neither category recognized

Conditions:

- Both comparisons return nonzero, with any signs or magnitudes.

Required behavior:

- Read the shared state after the first comparison and before the second.
- Invoke neither action.
- Return 0.

### Candidate boundary patterns

Conditions:

- The candidate has the 32-bit pattern `0xFFFFFFFF`.
- The relevant shared-state observation has the same pattern.

Required behavior:

- Suppress the corresponding action after recognition.

If the relevant shared-state pattern instead differs, invoke the corresponding action. No signed range check, increment, subtraction, or overflow handling is involved.

## Uncertainties

The following are **documented_only**, not behavior to invent or normalize:

- Exact event-token contents and encoding.
- Whether comparison means string equality, token identity, or another relation.
- The published identity and full contract of F-unmapped-01.
- Pointer, handle, and lifetime requirements for the opaque inputs.
- The meaning and allowed domain of the candidate and shared state.
- The actions’ effects, error conventions, and ownership behavior.
- Shared-state initialization, synchronization, volatility, and atomicity.
- Reentrancy and concurrency support.
- Behavior following invalid input or abnormal dependency completion.
- Machine-level ABI details beyond the represented 32-bit values.

The selection/highlighting interpretation is inferred from naming only. It must not be used to add state updates, validation, notifications, or resource operations absent from the visible behavior.

## Evidence

**Established static evidence:** The supplied function body shows two ordered comparisons, a shared-state snapshot after the first comparison, a separate shared-state observation on the first-category path, conditional calls to the two mapped action dependencies, and normal returns restricted to 0 and 1. It contains no explicit shared-state write or action-result check.

**Inferred intent:** The naming associated with the two event tokens suggests selection and highlighting. The candidate/shared-state inequality checks suggest suppression of redundant actions. Neither inference establishes the underlying application semantics.

**Unknown:** Dependency implementations, token data, shared-state declarations, ABI details, and runtime behavior were not supplied.

No runtime tests have been performed. The acceptance examples specify consequences of the visible static behavior; they are not observations of execution.
