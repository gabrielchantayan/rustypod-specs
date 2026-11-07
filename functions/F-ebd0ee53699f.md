<!-- Trusted provenance: F-ebd0ee53699f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebd0ee53699f

## Purpose

F-ebd0ee53699f conditionally dispatches four caller-supplied 32-bit values to eligible objects obtained from a container-associated collection. It visits collection positions in descending order, maintains a container-resident dispatch-depth field, and may perform a separate cleanup pass after the depth returns to zero.

The established behavior is selective callback dispatch followed by conditional cleanup. Interpreting the container as a listener registry, its depth as a reentrancy counter, and its change marker as a deferred-removal indicator is plausible intent, not a proven external contract.

This function does not directly return a result, allocate storage, or release storage. Dependencies and callbacks may have additional effects that are not established here.

## Interface

### Inputs and return

The interface has five arguments:

| Argument | Established role |
|---|---|
| Container address | A 32-bit address-like value used to access container fields and derive the collection address. |
| Value 1 | An opaque 32-bit value forwarded to each selected callback. |
| Value 2 | An opaque 32-bit value forwarded to each selected callback. |
| Value 3 | An opaque 32-bit value forwarded to each selected callback. |
| Value 4 | An opaque 32-bit value forwarded to each selected callback. |

There is no defined return value.

The four forwarded values are not directly interpreted or dereferenced by this function. Their signedness, pointer status, ownership, and application meaning are unspecified.

### Container layout

Offsets are byte offsets from the container address.

| Offset | Width | Established use |
|---|---:|---|
| +4 | 32 bits | Entry gate: must equal zero for dispatch processing to proceed. |
| +8 | 32 bits | Depth-like state, incremented on entry to the active region and decremented on normal exit from it. |
| +12 | 32 bits | Change marker, saved before dispatch and conditionally compared afterward. |
| +16 | Address within container | Address passed to collection lookup and cleanup dependencies. |
| +24 | 32 bits | Initial traversal count for each pass. |

The field at +24 is treated as a signed integer for positive-count loop tests. Its relationship to the representation beginning at +16 is not otherwise established.

No behavior is established for unused offsets.

### Lookup-result layout

A nonzero result from F-89ebe7aabbb8 is used as an address with at least these accessible fields:

| Offset | Width | Established use |
|---|---:|---|
| +0 | 32 bits | Object address passed to the eligibility dependency and subsequently reloaded. |
| +4 | 32-bit read during dispatch | Converted to a character-sized value to determine whether the entry is eligible for further processing. |
| +4 | 8-bit read during cleanup | A nonzero byte triggers a cleanup dependency call. |

The dispatch condition examines the character-width conversion of the 32-bit field, not whether the whole 32-bit field is zero. Cleanup explicitly examines the byte at offset +4. Treating these as the same flag requires the target’s conversion and byte-order conventions.

### Object layout

An object selected for callback dispatch is accessed at these offsets:

| Offset | Width | Established use |
|---|---:|---|
| +4 | 32 bits | Must equal zero for the callback to be invoked. |
| +48 | 32 bits | Callable address. |
| +52 | 32 bits | Opaque fifth callback argument. |

The callback receives, in order, Value 1, Value 2, Value 3, Value 4, and the object’s +52 value. The object address itself is not separately supplied as an argument by this function.

### Preconditions

For an execution that reaches a memory access or indirect call, the corresponding address must support that operation. In particular:

- Container fields that are reached must be readable, and the depth field must be writable.
- A nonzero lookup result must remain usable for all subsequent accesses that are actually performed.
- After a successful eligibility call, the reloaded object address must support the object-field accesses.
- A selected callback address must be callable with the five-argument interface.
- Dependencies and callbacks must return normally if normal completion is required.

There is no general null-container check. However, if the initial dependency returns zero, the function does not itself access the container.

## Behavior

### 1. Entry gating

The first action is a call to F-918216db7a4d with no explicit arguments.

The function proceeds only when all of the following hold, evaluated in this order:

1. F-918216db7a4d returns nonzero.
2. The container’s 32-bit field at +4 equals zero.
3. The container’s 32-bit field at +24 is nonzero.

Short-circuit behavior matters:

- A zero dependency result prevents both container reads.
- A nonzero container field at +4 prevents the +24 read.
- A zero count prevents the depth update, dispatch traversal, and cleanup decision.

A negative count is nonzero and therefore passes the third gate. It does not produce dispatch iterations, because iteration requires a strictly positive count.

The gate result and the container field at +4 are not rechecked during dispatch.

### 2. Entering the active region

Once the gates pass:

1. The current change marker at container offset +12 is saved.
2. The depth at +8 is read, incremented by one, and written back.

The initial dispatch count has already been loaded before these operations.

For representable arithmetic, the depth write stores the prior depth plus one. No overflow handling is present. Exact behavior when the signed increment or decrement exceeds its representable range is not established by the supplied evidence.

### 3. Descending dispatch traversal

For a positive initial count \(N\), lookup calls use positions \(N, N-1, \ldots, 1\). A separate local traversal value controls this sequence; the container’s count is not reread to determine each next dispatch position.

At each position:

1. F-89ebe7aabbb8 is called with the collection address, container +16, and the current position.
2. A zero lookup result skips the entry.
3. Otherwise, the result’s 32-bit field at +4 is read and converted to a character-sized value.
4. A nonzero converted value skips the entry.
5. Otherwise, the object address at result offset +0 is read and passed to F-d94e68c9f6c2.
6. A zero eligibility result skips the entry.
7. After a nonzero eligibility result, the object address at result offset +0 is read again.
8. The reloaded object’s 32-bit field at +4 is read. A nonzero value skips callback invocation.
9. Otherwise, the callback address at object +48 and opaque value at object +52 are obtained, and the callback is invoked with the four original forwarded values followed by that opaque value.

The object-address reload after F-d94e68c9f6c2 is significant. The address tested by that dependency need not be the address subsequently inspected and dispatched to if the dependency changes the lookup result.

No null check is performed on that reloaded object address before accessing its +4 field. A successful eligibility result alone does not establish that the later address remains valid.

The entry’s tag is not rechecked after eligibility evaluation. The object’s +4 field is checked after eligibility evaluation, not before it.

The relative evaluation order of the callback-address read and fifth-argument read is not established. Both supply the same invocation, but an acceptance condition must not require one to precede the other.

### 4. Leaving the active region

After all dispatch iterations finish normally:

1. The current depth at container +8 is read.
2. One is subtracted.
3. The result is written back.

This uses the depth value present after all dependency and callback activity, rather than restoring the original depth.

Cleanup is considered only if the newly stored depth is exactly zero. If it is not zero, the change marker is not read for this decision.

When the new depth is zero, the current marker at +12 is read and compared with the saved entry value. Equality ends processing. Inequality starts cleanup.

Only final equality is tested. Intermediate marker changes that return to the saved value do not independently cause cleanup.

### 5. Conditional cleanup traversal

At cleanup entry, the current container count at +24 is read again. This count can differ from the count used for dispatch.

For a positive cleanup count \(M\), lookup positions are \(M, M-1, \ldots, 1\). A zero or negative cleanup count produces no cleanup iterations.

At each position:

1. F-89ebe7aabbb8 is called with container +16 and the current position.
2. A zero result causes no tag read and no cleanup call.
3. For a nonzero result, the byte at result +4 is read.
4. If that byte is nonzero, F-63ecaced33d1 is called with container +16 and the literal integer **1**.

The second cleanup argument is always 1. It is not the current lookup position, the lookup-result address, the object address, or the tag value.

There is no eligibility call and no object-state check during cleanup. A cleanup result is not used. The function neither retries a position nor restarts traversal after a cleanup call.

## Side effects

Direct writes are limited to the container’s depth field:

- One increment after entry gating succeeds.
- One decrement after dispatch completes normally.

The function directly reads container, lookup-result, and object fields as described above. It does not directly clear tags, reset the change marker, update the count, or overwrite callback fields.

Indirect side effects may occur through every dependency and callback. Their effects on the container, collection, lookup-result storage, objects, global state, and memory ownership are not established.

Consequently:

- The function provides no snapshot guarantee for collection contents.
- A fixed descending lookup sequence does not prove that each logical object is visited once.
- Nested dispatch or mutation can affect the final depth, marker comparison, and cleanup count.
- No ownership transfer, retention, destruction, or release is established by a lookup, eligibility check, callback, or cleanup call.
- No synchronization or thread-safety guarantee is visible.
- No abnormal-exit recovery is visible. If a callback or dependency does not return normally, the later depth decrement and cleanup are not established to occur.

## Dependencies

| Opaque ID | Invocation and use |
|---|---|
| F-918216db7a4d | Called first, without explicit arguments. A zero result disables all subsequent work. |
| F-89ebe7aabbb8 | Called with the collection address and a descending positive position. Its result is treated as a nullable entry address in both passes. |
| F-d94e68c9f6c2 | Called with an entry’s object address during dispatch, after the entry-tag check. A nonzero result permits further processing, including reloading the entry’s object address. |
| F-63ecaced33d1 | Called during cleanup for each nonnull lookup result whose byte at +4 is nonzero. Arguments are the collection address and literal 1. |
| Object-supplied callback | Loaded from the selected object at +48. Receives the four forwarded values and the object’s +52 value. No return value is consumed. |

The names and argument patterns do not establish any dependency’s internal behavior. In particular, F-63ecaced33d1 is not proven to remove the entry just examined or to release any memory.

## Acceptance examples

These are conditional behavioral expectations derived from static evidence, not executed tests. Except where stated otherwise, assume valid memory, representable arithmetic, normally returning calls, and no dependency mutations beyond those specified.

1. **Global gate closed.**  
   If F-918216db7a4d returns zero, there are no direct container accesses, collection lookups, callback invocations, or depth writes.

2. **Container gate closed.**  
   If the initial dependency returns nonzero and container +4 is nonzero, the count is not read and the depth is unchanged.

3. **Zero versus negative count.**  
   A zero count exits before the marker is saved or the depth is updated. A negative count enters the active region and performs the depth increment and decrement, but performs no dispatch lookups. Cleanup remains possible if the final depth and marker comparison require it.

4. **Descending traversal.**  
   With initial count three, dispatch lookup positions are three, two, and one. A null result at position two does not suppress the lookup at position one.

5. **Character-sized dispatch tag.**  
   If a nonnull entry’s converted character-sized tag is nonzero, eligibility is not called. A nonzero 32-bit tag whose character conversion is zero does not, by itself, exclude the entry.

6. **Eligibility mutation.**  
   Suppose an entry initially references object A. F-d94e68c9f6c2 receives A, changes the entry to reference object B, and returns nonzero. The subsequent state check and possible callback use B, not a saved copy of A.

7. **Blocked object.**  
   If eligibility succeeds but the reloaded object’s +4 field is nonzero, no callback is invoked for that entry.

8. **Callback arguments.**  
   For an otherwise eligible entry, the callback receives the four caller-supplied 32-bit values unchanged, followed by the selected object’s +52 value. It does not receive the collection position as an additional argument.

9. **Nested depth suppresses cleanup.**  
   If the depth is one on entry and remains otherwise unchanged, the function writes two on entry and one on exit. Even if the marker changes, this invocation does not perform cleanup.

10. **Final marker equality controls cleanup.**  
    If the marker changes during dispatch but equals its saved value when checked at depth zero, cleanup is skipped. A final unequal marker triggers cleanup.

11. **Cleanup uses a fresh count.**  
    If dispatch begins with count two but the count is four when cleanup begins, cleanup lookup positions are four, three, two, and one.

12. **Cleanup passes literal one.**  
    If the cleanup lookup at position four returns a nonnull entry with a nonzero byte at +4, F-63ecaced33d1 receives the collection address and 1, not 4.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the entry gate, depth, change marker, tags, count, and object state.
- The target ABI’s full pointer, character-conversion, alignment, and byte-order rules. The supplied representation supports 32-bit fields and 8-bit byte accesses; exact dispatch-tag/cleanup-tag equivalence requires target conventions.
- The representation and invariants of the collection.
- Whether dependencies preserve any container field, entry, object, or global state.
- Whether callbacks can reenter this function, alter collections, or invalidate storage.
- The lifetime and ownership of all addresses supplied or returned.
- Signed-overflow behavior and any intended bounds on depth or count.
- Dependency failure modes, callback validity guarantees, and abnormal-control-flow behavior.
- Whether cleanup performs removal, compaction, reclamation, or some other operation.

No additional validation, retry, fallback, exception handling, or memory-management guarantee is inferred.

## Evidence

The supplied static representation establishes the gating sequence, field offsets, operand widths, descending loop conditions, conditional calls, object-address reload, depth writes, final marker comparison, and literal cleanup argument.

The dependency mapping establishes the opaque IDs used here. Dependency implementations and callback implementations were not supplied.

“Dispatch,” “depth,” “change marker,” and “cleanup” are descriptive labels for observed roles. Their application-level interpretations remain inferred.

No runtime tests have been performed. The acceptance examples express conditional obligations supported by static evidence, not observed execution results.
