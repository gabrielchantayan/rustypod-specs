<!-- Trusted provenance: F-5ba1d1c1de3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5ba1d1c1de3f

## Purpose

F-5ba1d1c1de3f conditionally requests an object-state change, then selects one of three actions on an associated owner object.

The change decision compares an unsigned 32-bit input against the inverted low bit of a byte in an object supplied by a lookup dependency. The input is not normalized to Boolean.

The lookup supplies the object pointer through a writable local pointer slot. The function reads this slot at multiple stages rather than capturing one immutable target identity. Consequently, the object inspected initially, the object used in a later identity comparison, and the object receiving the change request need not be the same if dependencies modify the slot.

“Owner,” “target,” “capture source,” and “refresh” describe operational roles only. Their application-level meanings are not established.

## Interface

### Arguments and result

| Argument | Representation | Established use |
|---|---|---|
| First through fifth | Five separate 32-bit values | Forwarded in order to F-d0c0f5ff3914, without visible conversion |
| Sixth | Unsigned 32-bit integer | Compared with a derived zero-or-one value and forwarded unchanged to the target-change operation |

The first five arguments have no established signedness or semantic type. Their interpretation as identifiers, addresses, flags, or other values is unknown.

The function has no explicit return value. Normal completion does not report whether lookup failed, the initial state matched, or a change was requested. No explicit error code or exception translation is present.

### Lookup output storage

F-d0c0f5ff3914 receives the first five arguments followed by:

1. The address of a local pointer slot.
2. The address of a separate four-byte temporary region.

Neither output region is visibly initialized before this call. The lookup’s return value is treated as an owner pointer.

The pointer slot and the pointer stored in it are distinct:

- The slot is writable local storage whose address is passed to the lookup.
- Its contents identify an object when subsequently read.
- The lookup’s retention or sharing of the slot address is unknown.
- Intervening dependencies may therefore modify the slot while this invocation remains active; the contract cannot assume otherwise.
- A later read of the slot is not necessarily the pointer used for the initial state-byte read.

The four-byte temporary region is not visibly read after lookup. Its contents and resource implications are unknown.

### Established layout and widths

The supplied representation uses four-byte object words and a 32-bit pointer and dispatch-table model.

| Storage | Byte offset | Established access |
|---|---:|---|
| Object selected for initial inspection | 4 | One-byte state source |
| Object receiving an indirect call | 0 | Dispatch-table pointer |
| Owner | 480, or 0x1E0 | Four-byte value captured for two queries |
| Dispatch tables | Listed under Dependencies | Callable entries |

The owner field at offset 480 is captured as a 32-bit value. Query results and the optional captured result are treated as signed 32-bit integers. The saved result’s final branch test distinguishes only zero from nonzero.

These observations do not establish a complete object layout or a portable type declaration.

## Behavior

### Lookup and initial inspection

The function calls F-d0c0f5ff3914 once with the five forwarded inputs and the two output addresses.

If the returned owner pointer is null, the function returns without inspecting the pointer slot’s referenced object or calling any later dependency. Lookup itself may still have side effects.

For a non-null owner, the function reads the pointer slot and uses the pointer obtained at this stage to read one byte at object offset 4. This initial pointer must identify storage readable at that offset. There is no visible null check or other validation of it.

The state value derived from that byte is:

| Low bit of the byte | Derived unsigned 32-bit value |
|---|---:|
| Clear | 1 |
| Set | 0 |

The byte is converted to an unsigned-width value, complemented, and masked with 1. Other bits cannot affect the resulting zero or one.

The derived value is compared against the entire sixth argument. If equal, the function returns without the subsequent owner-field read, queries, capture checks, change request, refresh, or final action.

Thus:

- Input 0 matches only a set low bit.
- Input 1 matches only a clear low bit.
- Every other unsigned 32-bit input necessarily causes a mismatch.

Out-of-range Boolean values are not rejected.

### Capture the query input

On a mismatch, the function reads the owner’s 32-bit field at offset 480 and saves that value for both query calls.

The field is not reread for the second query. Changes to the field by intervening dependencies do not alter the saved query input.

A separate signed 32-bit saved result is initialized to zero. It changes only if the optional capture operation is reached.

The owner pointer itself is the value returned by lookup. The function does not resolve a replacement owner later. This does not guarantee that its referenced storage remains valid.

### Optional pre-change capture

The first call to F-004c5654fd0a receives the captured owner-field value.

Only a signed result strictly greater than zero enables the following sequence:

1. Invoke F-indirect-01 through the owner’s dispatch-table entry at byte offset 0x188, explicitly passing the owner pointer.
2. Continue only if its 32-bit result is nonzero.
3. Call F-914c751ce49b with the owner pointer.
4. Compare the returned pointer with the pointer slot’s then-current contents.
5. If they differ, invoke F-indirect-02 through the returned object’s dispatch-table entry at byte offset 0x54, saving its signed 32-bit result.

Each condition short-circuits the remaining capture steps when false.

The identity comparison occurs after F-914c751ce49b returns. It does not necessarily compare against the object whose state byte was initially inspected. The first query, the owner gate, or F-914c751ce49b may have affected the pointer slot through an address retained or shared by lookup.

The comparison examines pointer identity only. It does not compare object contents. The pointer stored in the slot at this comparison is not dereferenced merely to perform the comparison.

If the pointers are equal, capture is skipped and the saved result remains zero. If they differ, the pointer returned by F-914c751ce49b must support the required dispatch-table access and invocation. No visible null check precedes that access.

F-indirect-02 has no explicit arguments in the supplied representation. In particular, this contract does not specify the capture-source pointer as an explicit argument. Any implicit receiver mechanism is unknown.

A zero capture result remains indistinguishable, for final branch selection, from capture having been skipped.

### Request the change using the current slot contents

After the optional capture sequence, the function again uses the pointer slot’s contents to select the target-change operation. It accesses the selected object’s dispatch table and invokes F-indirect-03 at table byte offset 0x3C, explicitly supplying:

- The target pointer obtained from the slot at this dispatch stage.
- The original unsigned 32-bit sixth argument.

The return value is unused.

This stage does not necessarily act on either:

- The object inspected for the initial state decision.
- The object represented by the slot during the earlier identity comparison.

In particular, F-indirect-02 can run between the comparison and this dispatch. If it changes the pointer slot, the ensuing request uses the changed contents. Earlier dependencies may also change the slot.

The dispatch-table access and explicit target argument both use the slot in the supplied representation. There is no intervening dependency call shown between preparing those uses and invoking F-indirect-03. Their precise evaluation order, and behavior under asynchronous modification, are not established.

Normal execution requires valid storage and a callable dispatch entry for the pointer used at this stage. Initial validity of the inspected object does not satisfy that requirement if the slot has since changed. No visible validation, state-byte reread, or renewed match decision occurs for a replacement target.

### Refresh and second query

After F-indirect-03 returns, the function calls F-65ddf49a9824 with the original owner pointer. Its return value is unused.

The function then calls F-004c5654fd0a a second time with the previously captured owner-field value.

This second result is independent for branching purposes. The function does not reuse the first query result, reread the owner field, or verify that the requested state change occurred.

### Select the final action

Exactly one final action follows a normally returning second query:

| Second signed query result | Saved result | Final action |
|---|---|---|
| Less than 1 | Any | F-indirect-04 on the owner |
| Greater than zero | Zero | F-indirect-05 on the owner, with additional arguments 0 and 1 |
| Greater than zero | Nonzero | F-a18dfa95693f with the owner, saved result, and 8 |

F-indirect-04 uses owner table offset 0x198 and explicitly receives the owner.

F-indirect-05 uses owner table offset 400 decimal, equivalently 0x190, and explicitly receives the owner, zero, and one.

Negative query results follow the same final branch as zero. A negative saved result follows the same branch as every other nonzero saved result.

No subsequent work is visible before return.

## Side effects

### Direct access and call ordering

The function directly reads:

- The pointer slot for initial inspection, the conditional identity comparison, and target-change dispatch.
- The initial inspected object’s state byte.
- The owner field at offset 480 on the mismatch path.
- Dispatch-table pointers and entries required by enabled calls.

It maintains local output storage, the captured query input, and the saved result. No direct write to an owner or target field is visible. External mutations are delegated to dependencies.

On a complete change path, the established sequence is:

1. Lookup.
2. Initial slot read and state-byte inspection.
3. Mismatch decision.
4. Owner-field capture.
5. First query.
6. Any enabled owner gate, pointer-returning lookup, current-slot comparison, and capture dispatch.
7. Current-slot target-change dispatch.
8. Owner refresh.
9. Second query using the captured field value.
10. One final action.

The optional identity comparison reads the slot after F-914c751ce49b. The change dispatch uses it after any capture operation. These later accesses are material behavior, not references to a guaranteed fixed target identity.

### Ownership and lifetime

No explicit allocation, retention, release, or ownership transfer is visible.

The function requires the owner and each object actually accessed to remain valid at the relevant access. It does not establish that dependencies preserve object lifetimes, fields, dispatch tables, or globals.

The lookup receives addresses of local storage. Whether it retains or shares those addresses is unknown. Modification of the pointer slot during this invocation is not ruled out. Safe use of retained addresses after the invocation ends is not established.

The temporary output and captured result cannot be classified as owning or borrowed resources from this evidence. Absence of an explicit release in this function does not establish what dependencies do with them.

### Errors and abnormal completion

A null owner return is explicitly tolerated. Other invalid pointers or dispatch entries are not visibly handled.

There is no visible retry, rollback, exception handler, or recovery branch. If a dependency does not return normally, subsequent calls are not established as occurring, and earlier effects are not visibly undone.

## Dependencies

| Opaque ID | Established interface role |
|---|---|
| F-d0c0f5ff3914 | Receives five 32-bit values and two writable output addresses; returns the owner pointer |
| F-004c5654fd0a | Receives the captured 32-bit owner-field value; produces a signed 32-bit query result |
| F-914c751ce49b | Receives the owner; returns a pointer for identity comparison and possible capture |
| F-65ddf49a9824 | Receives the owner after the target-change request |
| F-a18dfa95693f | Receives the owner, a nonzero saved result, and 8 in one final branch |

The following opaque IDs identify indirect call sites, not resolved implementations:

| Opaque ID | Table selection | Explicit arguments | Result use |
|---|---|---|---|
| F-indirect-01 | Owner, offset 0x188 | Owner | Nonzero enables later capture checks |
| F-indirect-02 | Returned capture-source object, offset 0x54 | None visible | Saved signed 32-bit result |
| F-indirect-03 | Target selected from the slot at dispatch, offset 0x3C | Target, original sixth argument | Ignored |
| F-indirect-04 | Owner, offset 0x198 | Owner | Ignored |
| F-indirect-05 | Owner, offset 0x190 | Owner, 0, 1 | Ignored |

Indirect implementations may vary with object state. These IDs do not imply distinct implementations, stable tables, memory ownership, or preservation of the pointer slot.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example concerns invalid storage, required memory is assumed valid and dependencies are assumed to return normally.

1. **Null owner.** Lookup runs once. No target-state read or later dependency call occurs, regardless of the pointer-slot contents.

2. **Matching initial state.** If the initial inspected byte has a clear low bit and the sixth argument is 1, the function returns after inspection. A set low bit and argument 0 produce the same early return.

3. **Non-Boolean request.** An argument of 2 always mismatches. F-indirect-03 receives 2 unchanged. The same full-width rule applies to the maximum unsigned 32-bit value.

4. **First query zero, second query positive.** Capture checks are skipped. After change dispatch and refresh, F-indirect-05 receives the owner, 0, and 1.

5. **Positive first query, zero owner gate.** F-914c751ce49b and F-indirect-02 are skipped. The final action still depends on the second query.

6. **Current identity suppresses capture.** Initial inspection uses object A. Before the identity comparison, a dependency changes the slot to object B, and F-914c751ce49b returns B. Capture is skipped because B equals the current slot contents, even though B differs from A.

7. **Initial identity does not suppress capture.** Initial inspection uses A. The slot later contains B, while F-914c751ce49b returns A. Because A differs from B, F-indirect-02 is invoked through A, assuming A remains valid.

8. **Capture redirects the change request.** The comparison finds different pointers and enables F-indirect-02. That dependency changes the slot to valid object C. F-indirect-03 is then selected through C and receives C, without inspecting C’s state byte or repeating the match decision.

9. **Later slot becomes invalid.** Initial inspection succeeds, but a dependency leaves the slot null before change dispatch. There is no visible null-handling branch for this stage; normal dispatch requires are not satisfied.

10. **Nonzero capture, nonpositive second query.** F-indirect-04 runs. The saved value does not override the nonpositive-query branch.

11. **Negative capture, positive second query.** F-a18dfa95693f receives the negative saved 32-bit value and 8. The saved value need not be positive.

12. **Owner-field mutation.** An intervening dependency changes owner offset 480. Both queries nevertheless receive the original captured value; their results may differ for other dependency-defined reasons.

## Uncertainties

### Inferred intent

The sequence is consistent with changing state while optionally preserving context for a later owner adjustment. It does not establish whether the state represents selection, activation, visibility, focus, or another concept.

### documented_only

The following remain unresolved:

- Semantic types and meanings of the first five inputs.
- Lookup matching rules and the temporary output’s meaning.
- Whether lookup retains or shares either output address.
- Which dependencies, if any, modify the pointer slot in actual executions.
- Query semantics beyond signed comparisons with zero.
- The meaning and ownership of the captured owner-field value and saved result.
- Actual effects or success conditions of change, refresh, and final operations.
- Any implicit receiver convention for F-indirect-02.
- Dependency effects on globals, object fields, dispatch tables, and lifetimes.
- Evaluation order among target-slot uses within the final indirect-call expression.
- Concurrency, synchronization, reentrancy, and exception behavior.

These unknowns do not justify assuming a stable target pointer, preserved memory, successful state realization, or any unobserved resource management.

## Evidence

The contract rests on the supplied static function representation and opaque dependency mapping.

Established evidence includes the argument widths, layout offsets, inverted low-bit calculation, full-width comparison, conditional capture sequence, signed query boundaries, captured query-input reuse, repeated pointer-slot accesses, and three-way final dispatch.

The writable slot address is passed to lookup. Later pointer expressions read that slot after intervening calls, including after the optional capture operation. This establishes the distinction between initial inspection identity and later target selection without establishing that any dependency actually changes the slot.

No dependency bodies, runtime traces, independent layout declarations, or execution results were supplied. No runtime tests have been performed.
