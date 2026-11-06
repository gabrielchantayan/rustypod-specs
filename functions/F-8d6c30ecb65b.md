<!-- Trusted provenance: F-8d6c30ecb65b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8d6c30ecb65b

## Purpose

F-8d6c30ecb65b conditionally passes a pointer-valued working state to a receiver, then performs counter-based cleanup of the state remaining in an exposed pointer slot.

Two discriminator checks select the processing path:

- The candidate’s 32-bit field at byte offset 4 must equal `0x00000017`.
- A candidate operation must return an address whose 32-bit field at byte offset 8 equals `0x52800000`.

A normally completed call returns zero when either discriminator fails and one when both match. One indicates selection of the processing path, not successful completion of an independently checked downstream operation. Results from the working-state dependency and receiver operation are not inspected.

Some cleanup paths invoke a dependency annotated as non-returning. Those paths have no established normal return.

**Inferred intent:** the counter decrement and conditional cleanup resemble reference management. The visible behavior does not establish a complete ownership or reference-counting protocol.

## Interface

The function receives four inputs in this order:

| Input | Established use |
|---|---|
| Receiver pointer | Supplies a dispatch table whose entry at byte offset `0x54` is invoked on the selected processing path. |
| Candidate pointer | Supplies the first discriminator and a dispatch table whose entry at byte offset `0x14` is conditionally invoked. |
| Opaque 32-bit value | Not read or forwarded by the visible function. |
| Initial working pointer | Copied into a local pointer slot after the first discriminator matches. |

The return value occupies 32 bits. Its established normal values are zero and one.

### Widths and accessed layouts

The supplied representation uses 32-bit integer words and 32-bit addresses. Offsets below are byte offsets.

| Storage | Offset | Observed interpretation |
|---|---:|---|
| Candidate | 0 | Dispatch-table address. |
| Candidate | 4 | 32-bit discriminator. |
| Receiver | 0 | Dispatch-table address. |
| Candidate operation result | 8 | 32-bit discriminator. |
| Working structure | 0 | Optional nested-object address. |
| Working structure | 4 | 32-bit counter, represented as signed. |
| Working structure | 8 | 32-bit cleanup selector and, on one path, a dependency argument. |
| Nested object | 0 | Dispatch-table address. |

These accesses do not establish complete object layouts, allocation sizes, alignment guarantees, or semantic field types beyond their observed uses.

### Validity requirements

The function does not validate its inputs comprehensively.

- The candidate’s offset-4 word must be readable before any candidate-null check occurs.
- An eligible candidate must support the dispatch through table offset `0x14`.
- The candidate operation’s result must support an immediate read at offset 8; no null or validity check protects that read.
- The receiver need only be usable if both discriminators match, but must then support the dispatch through table offset `0x54`.
- Each working pointer used for a direct access must designate storage valid for that access at that time.
- A nonnull nested-object pointer selected for dispatch must support its table lookup and operation at table offset 4.

Calls can change the exposed working-pointer slot or reachable memory. Consequently, validity before a call does not establish validity after it. In particular, the function does not repeat every null check after intervening calls.

Invalid pointers have no established clean rejection or error-return behavior.

## Behavior

### Selection and early returns

The prospective result begins at zero. The function first reads the candidate’s 32-bit discriminator at offset 4.

If the value differs from `0x17`, the function returns zero. It does not invoke the candidate operation, inspect the initial working structure, call the working-state dependency, or invoke the receiver operation.

If the value matches, the function copies the fourth input into a local working-pointer slot.

A candidate-null check occurs after the discriminator read. It does not make null candidates safe. The representation initializes the candidate-operation result address to zero and skips dispatch if that later check finds null; the subsequent discriminator read is still not guarded. This does not establish a usable null-input path.

For an eligible, nonnull candidate, the function invokes the candidate’s operation at dispatch-table offset `0x14`, supplying:

1. The candidate pointer.
2. The numeric value `0x6400`, equal to 25,600.

The operation’s return value is treated as an address. The function reads the 32-bit word at that address plus 8.

If the word differs from `0x52800000`, the function returns zero. It does not call the working-state dependency or receiver operation and performs no direct cleanup of the initial working structure.

No explicit release of the candidate operation’s result appears on either discriminator outcome. Its ownership and lifetime are unresolved.

### Working-state handoff

When the second discriminator matches, the prospective return value becomes one.

The function calls F-0ae694b85c09 with:

1. The address of the local working-pointer slot.
2. The candidate operation’s result address.

After that call returns, the function invokes the receiver’s operation at dispatch-table offset `0x54`, supplying:

1. The receiver pointer.
2. The address of the same working-pointer slot.

Neither call’s result is inspected. Both calls receive access to the slot and can replace its pointer value.

The function does not visibly compare successive pointer values, increment a counter when initializing the slot, or release a pointer merely because a call displaced it.

### Cleanup entry

After the receiver operation returns, the function tests the working pointer currently held in the slot.

If it is null, cleanup is skipped and the function normally returns one.

Otherwise, the function reads the working structure’s offset-8 word. If that word is nonzero, it calls F-338d16f9ce72 without explicit arguments. If the word is zero, it proceeds without that call.

The first cleanup field read therefore targets the receiver’s resulting pointer. That guarantee does not extend across F-338d16f9ce72: after the call, subsequent accesses use the then-current slot value.

There is no new null check between F-338d16f9ce72 and the counter access. If that dependency changes the slot, the replacement must remain valid for the following counter operations for execution to remain well-defined. An earlier nonnull test does not protect a replacement that becomes null or invalid.

### Counter decrement and branch decision

The function reads the 32-bit counter at offset 4 through the current working pointer, subtracts one, and stores the result at offset 4.

The branch decision uses the computed decrement result. It does not reread the stored counter to decide which cleanup branch to take.

For values where signed subtraction is representable:

| Counter before decrement | Stored value | Selected branch |
|---:|---:|---|
| 1 | 0 | Zero-result cleanup. |
| 3 | 2 | Nonzero-result cleanup. |
| 0 | −1 | Nonzero-result cleanup. |
| −1 | −2 | Nonzero-result cleanup. |

There is no lower-bound validation, saturation, or underflow error result.

The supplied representation uses signed 32-bit arithmetic. Subtraction from the signed minimum, −2,147,483,648, is not assigned a portable result by this contract. Neither wrapping nor a particular failure mode is established.

### Nonzero-result cleanup

If the computed decrement result is nonzero, the function calls F-66e1118d7fc7 with no explicit arguments shown.

If the dependency returns normally, the function returns one. There are no further visible cleanup accesses on this branch.

The absence of subsequent accesses does not imply that the dependency preserves memory, globals, or ownership state.

### Zero-result cleanup

If the computed decrement result is zero, the function reads the working structure’s first word.

If that word is nonzero, the function treats it as a nested-object pointer, obtains the nested object’s dispatch table, and invokes the entry at byte offset 4. No explicit arguments are shown for this indirect call.

The function then calls F-66e1118d7fc7, supplying the working pointer currently in the slot. If the nested-object operation changed the slot, this argument can differ from the pointer whose counter was decremented.

After F-66e1118d7fc7 returns, the function:

- Captures the slot’s current pointer value for possible later use.
- Tests the slot’s current pointer for null.

There is no intervening call between that capture and test.

If the pointer is null, the function normally returns one. Otherwise, it reads the current structure’s offset-8 word and selects one of two paths:

**Offset-8 word is zero.**  
The function calls F-feee9156a257 with the current working pointer.

**Offset-8 word is nonzero.**  
The function first calls F-a600efb7b035 without explicit arguments. After that call returns, it reads the offset-8 word through the pointer captured after F-66e1118d7fc7 and passes that 32-bit word to F-feee9156a257.

The latter argument read is distinct from the preceding nonzero test. F-a600efb7b035 may change the field between those reads. It may also change the slot; such a slot change does not redirect the field read, which uses the captured pointer. The captured storage must remain valid through that read.

Both F-feee9156a257 invocation sites are annotated as non-returning. Under those annotations, neither produces a normal return from F-8d6c30ecb65b.

The visible flow places the pointer-argument invocation after the conditional field-argument invocation. If the field-argument invocation unexpectedly returned, execution would reach the pointer-argument invocation using the then-current slot value. Such a return is not established behavior.

## Side effects

### Direct effects

The function directly:

- Reads candidate, receiver, working-structure, nested-object, and dispatch-table memory as selected by control flow.
- Initializes and subsequently reads a local working-pointer slot.
- Exposes the address of that slot to two calls.
- Writes one decremented 32-bit counter when nonnull cleanup reaches the counter operation.

No other direct write to caller-owned fields is visible. Dependencies and indirect operations may perform additional writes or other effects.

### Observable order and mutation boundaries

On the selected processing path, the significant order is:

1. Candidate operation and second discriminator read.
2. F-0ae694b85c09.
3. Receiver operation.
4. Working-pointer null test and initial offset-8 read.
5. Optional F-338d16f9ce72.
6. Counter read, decrement, and store.
7. Branch-specific cleanup.

On the zero-result branch, the optional nested-object operation precedes F-66e1118d7fc7. The subsequent pointer capture, null test, and offset-8 test precede any F-a600efb7b035 call.

No unknown callee is assumed to preserve globals, reachable structures, or the exposed slot. Access to the slot was provided earlier; whether dependencies retain that access is unknown. Descriptions of later accesses therefore distinguish the then-current slot value from the explicitly captured pointer.

### Ownership and errors

The visible counter update is established. The following are not:

- Whether the initial working pointer is borrowed or transferred.
- Whether F-0ae694b85c09 acquires a reference.
- Whether the receiver consumes or retains the working state.
- Whether displaced pointers require cleanup.
- Whether either cleanup dependency releases memory.
- Whether F-feee9156a257 deallocates, terminates, or transfers control elsewhere.

There is no visible error-code propagation from downstream calls. Exceptions, nonlocal exits, faults, and dependency-specific failures are unresolved rather than converted into zero or one by an established handler.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-0ae694b85c09 | Receives the slot address and candidate operation result after both discriminators match. | Slot mutation, ownership, failure behavior, and other effects. |
| F-338d16f9ce72 | Called without explicit arguments when the initial cleanup offset-8 read is nonzero. | Effects on the slot, structures, globals, and validity. |
| F-66e1118d7fc7 | Called without explicit arguments on the nonzero-decrement branch; receives the current working pointer on the zero-decrement branch. | Actual parameter contract and cleanup semantics. The two visible argument forms are not normalized. |
| F-a600efb7b035 | Called without explicit arguments after the later offset-8 test selects the nonzero path. | Effects before the field is reread through the captured pointer. |
| F-feee9156a257 | Receives either a working pointer or a 32-bit offset-8 field value; annotated non-returning. | Argument meaning and actual terminal behavior. |

Three additional dependencies are dynamically selected: the candidate operation at table offset `0x14`, receiver operation at `0x54`, and nested-object operation at offset 4. No opaque function IDs are supplied for their targets; table positions identify the observed dispatches without inventing target identities.

## Acceptance examples

These are conditional expectations from static evidence, not runtime test results. Unless stated otherwise, required memory accesses are valid and preceding calls return normally.

1. **First discriminator mismatch.**  
   A candidate with offset-4 word `0x16` produces zero. Neither indirect processing operation runs, and the initial working structure is not accessed.

2. **Second discriminator mismatch.**  
   For a candidate with discriminator `0x17`, the candidate operation receives the candidate pointer and `0x6400`. If its result has offset-8 word `0x52800001`, the function returns zero without calling F-0ae694b85c09 or the receiver operation.

3. **Receiver clears the slot.**  
   If both discriminators match and the receiver leaves the slot null, the function returns one without decrementing a counter.

4. **Receiver replaces the working pointer.**  
   If the receiver replaces the initial pointer with a distinct valid structure, the cleanup-entry null test and initial offset-8 read use that replacement. If the field is nonzero, F-338d16f9ce72 runs next and may change the slot. Subsequent counter accesses use the then-current pointer, not unconditionally the receiver’s replacement. No direct release solely because the initial pointer was displaced is established.

5. **Counter remains nonzero.**  
   If cleanup begins with offset-8 word zero and counter three, the function stores two, calls F-66e1118d7fc7 without explicit arguments, and returns one if the call returns normally.

6. **Counter begins at zero.**  
   Under the same conditions with counter zero, the function stores −1 and selects the nonzero-result branch. Zero is not treated as an already-completed release.

7. **Zero counter result followed by pointer-argument terminal call.**  
   Suppose the counter is one, the first word is zero, and the initial offset-8 word is zero. The function stores zero and calls F-66e1118d7fc7 with the current working pointer. If, after that call, the slot contains a valid nonnull pointer whose offset-8 word is zero, F-feee9156a257 receives that pointer. Nonnullness alone is insufficient: a nonzero post-call field instead selects F-a600efb7b035 and the field-argument path. A null post-call pointer permits normal return of one.

8. **Post-call state overrides earlier cleanup fields.**  
   If F-66e1118d7fc7 replaces the slot on the zero-result branch, the later null and offset-8 tests concern the replacement. Earlier field values do not determine this later branch.

9. **Captured-pointer field changes.**  
   If the later offset-8 test is nonzero, F-a600efb7b035 runs before the terminal argument read. If it changes the captured structure’s field to zero while preserving valid storage, F-feee9156a257 receives zero. Replacing the slot does not redirect that captured-pointer read.

10. **Invalid candidate-operation result.**  
    A null or unreadable result is not cleanly rejected. The function attempts the offset-8 discriminator read; a zero return is not established.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the discriminator constants and `0x6400`.
- Complete layouts, alignment requirements, allocation origins, and lifetimes.
- Calling conventions and hidden arguments for incompletely specified calls.
- Ownership, retention, release, and global-state effects of dependencies.
- Whether recipients retain access to the local slot.
- Signed-minimum decrement behavior.
- Dependency errors, exceptions, nonlocal exits, and memory-invalidating effects.
- Whether the non-return annotations always hold.
- Concurrency requirements. No atomic counter operation or explicit synchronization is established.

## Evidence

**Established static evidence:** the supplied function representation supports the discriminator values, field offsets, visible arguments, call ordering, slot exposure, counter write, post-call rereads, captured-pointer use, normal status values, and non-return annotations described above.

**Inferred intent:** category-specific processing followed by reference-like cleanup. This interpretation supplies no additional ownership guarantees.

**Unknown:** dependency implementations, complete execution conventions, and actual runtime outcomes.

No runtime tests have been performed.
