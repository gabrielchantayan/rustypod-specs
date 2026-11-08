<!-- Trusted provenance: F-fff6431b6bd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fff6431b6bd8

## Purpose

F-fff6431b6bd8 performs a conditional, multi-stage operation using a caller-supplied context and either an explicit target or a target acquired through a dependency. It may clear an existing context-associated value, select a preparation path, acquire an intermediate value, invoke a context-selected callback, and store a newly produced associated value.

A normal return of **1** means that final creation returned nonzero and all subsequent operations, including cleanup, completed normally. A normal return of **0** means that this success point was not reached. Neither result independently establishes resource validity, ownership transfer, or broader application-level success.

The sequence is not transactional: earlier writes and dependency effects are not rolled back when a later stage fails.

“Target,” “preparation,” “intermediate,” and “creation” describe roles in the visible sequence. Their application-level meanings are not established.

## Interface

### Inputs and result

The visible interface has two 32-bit inputs:

| Input | Interpretation |
|---|---|
| Context address | Address of storage containing the fields below. No context-null check precedes accesses. |
| Requested target | A nonzero value is used directly. Zero requests acquisition through F-2a7d78c39bfc. |

Target selection uses zero versus nonzero, not a positivity test. A word with its high bit set is not rejected merely because its signed interpretation would be negative.

The normal return is a 32-bit word:

- **0:** the final-creation success path was not completed.
- **1:** final creation returned nonzero, and the subsequent success-state operations and cleanup completed normally.

No target, intermediate value, callback result, or associated value is returned directly.

### Context and reachable layouts

The supplied representation uses 32-bit addresses and the following 32-bit context fields:

| Byte offset | Observed role |
|---|---|
| +0x08 | Address of a dispatch structure. |
| +0x10 | Shared argument read separately at several stages. |
| +0x14 | Callback argument. |
| +0x18 | Optional descriptor address. |
| +0x3C | Flags updated on the success path. |
| +0x40 | Additional callback argument. |
| +0x50 | Existing or newly produced associated value. |

The dispatch structure supplies a callable entry at byte offset **+0x18**.

An active descriptor supplies:

| Byte offset | Width | Observed role |
|---|---:|---|
| +0x00 | 32 bits | Nonzero activation value and first argument to F-0784f4f6b095. |
| +0x08 | 32 bits | Quantity shifted left by two positions for that dependency’s second argument. |

On success, the value reread from context +0x50 is treated as an address. A 32-bit word is written at that address plus **0x10**.

Two distinct unresolved global locations supply the third and fourth arguments to F-0784f4f6b095. This contract calls them **global input A** and **global input B**, respectively. These labels identify argument sources, not constants. Their declarations, initial contents, storage widths, and semantic types are not supplied.

### Preconditions

Normal completion requires every address actually accessed to designate accessible storage of the required kind. Depending on the path, this includes the context, descriptor, dispatch structure, callable entry, unresolved global locations, and final associated object.

Each invoked dependency and callback must return normally for execution to continue. No explicit fault recovery, address validation, bounds checking, or synchronization appears.

The implicit-target-unavailable path performs no direct context access. It does not establish that dependencies themselves avoid accessing context-related or global state.

## Behavior

### Target resolution

The pending return value begins at zero.

A nonzero requested target becomes the effective target without acquisition. Otherwise, F-2a7d78c39bfc is called without explicit arguments, and its result becomes the effective target.

If acquisition returns zero, all context processing is skipped. F-4bd8a1faf697 is still called with zero. F-87ceecb12e6c is not called because the effective and requested targets are equal. Normal completion returns zero.

All remaining processing requires a nonzero effective target.

### Existing association

The function reads context +0x50. If this value is nonzero:

1. F-3eadc90e0ec1 is called without explicit arguments.
2. After it returns, zero is written to context +0x50.

The old field value is not visibly passed to this dependency. The call therefore does not establish that the old value is released or destroyed.

The explicit zero write occurs after the call, including if the dependency changed the field. If the initial read was zero, both the call and this write are skipped.

### Operation value and preparation selection

F-ea92bf179106 receives the effective target. After it returns, F-834e12b6dfde receives that same target. The latter’s 32-bit result is retained as the operation value.

F-68e8df570ef9 is then called without explicit arguments.

If its result is nonzero, the direct preparation path is selected without inspecting the descriptor. F-6eaf99ef0ef2 receives:

1. The retained operation value.
2. The current value of context +0x10.

If the mode result is zero, the function reads context +0x18:

- A zero descriptor address selects direct preparation.
- A nonzero descriptor address causes its first 32-bit word to be read.
- A zero first word also selects direct preparation.
- A nonzero first word selects descriptor-assisted preparation.

The descriptor-assisted path calls F-0784f4f6b095 with:

1. The descriptor’s nonzero first word.
2. The descriptor word at +0x08 shifted left by two bit positions.
3. The current value read from global input A.
4. The current value read from global input B.

The global values are read while preparing this call, after the preceding dependency calls have returned. They are not established constants or entry-time snapshots. Preceding dependencies may have modified either location.

The supplied representation does not establish the relative evaluation order of the two global reads, or their order relative to other argument reads for this call. It also does not establish an atomic snapshot across them.

After F-0784f4f6b095 returns, F-5d152df5657d receives the operation value and a fresh read of context +0x10. Its result is the descriptor-assisted preparation result. F-0784f4f6b095’s return value is not used as the preparation gate.

Thus, preparation is gated by the result of either F-6eaf99ef0ef2 or F-5d152df5657d.

### Intermediate acquisition

A zero preparation result skips intermediate acquisition, callback invocation, final creation, and success-state writes. Execution proceeds to terminal cleanup with the pending return value still zero.

For a nonzero preparation result, F-48a5e637b2ef receives these arguments in positional order:

1. Zero.
2. The retained operation value.
3. The current value of context +0x10.
4. The effective target.

Its result becomes the intermediate value.

A zero intermediate value skips the callback and final creation. It also skips F-2642535311f0. No additional validity test is visible.

### Indirect callback

For a nonzero intermediate value, a callback is selected through the address at context +0x08 and the callable entry at dispatch offset +0x18.

Its six positional arguments are:

1. The operation value.
2. The same operation value again.
3. The current value of context +0x14.
4. The current value of context +0x10.
5. The effective target.
6. The current value of context +0x40.

The intermediate value is not visibly passed.

A zero callback result prevents final creation. A nonzero callback result permits it. No additional meaning for either result is established.

The representation does not establish a total evaluation order among the callback’s independent argument reads and dispatch-entry reads.

### Final creation and success state

After a nonzero callback result, F-79055c71edcb receives:

1. The retained operation value.
2. The nonzero intermediate value.
3. A fresh read of context +0x10.

Its result is written to context +0x50, even if that result is zero.

A zero result leaves the pending return value at zero and skips the success-state operations.

For a nonzero result, the visible order is:

1. Call F-7eadd9e17dc6 without explicit arguments.
2. Set the pending return value to one.
3. Reread context +0x50.
4. Write F-7eadd9e17dc6’s result as a 32-bit word at the reread address plus 0x10.
5. Read the 32-bit flags at context +0x3C and write back a value with bit 7 cleared and bit 3 set.

The flags transformation preserves every other bit from that particular read. Its mask is **0xFFFFFF7F**, followed by setting **0x00000008**.

The metadata destination is not necessarily the value originally returned by F-79055c71edcb. F-7eadd9e17dc6 runs before context +0x50 is reread and may change that field. There is no new nonzero check before the metadata write.

A fault during these operations does not constitute a normal return of one.

### Cleanup and return

If F-48a5e637b2ef returned a nonzero intermediate value, F-2642535311f0 receives that value after callback-related processing. This call occurs whether the callback rejected, final creation returned zero, or the success path completed, provided execution reaches cleanup normally.

F-4bd8a1faf697 then receives the effective target on every normally completing path, including implicit acquisition returning zero.

After that call, F-87ceecb12e6c receives the effective target only when it differs from the original requested target:

| Target selection | F-87ceecb12e6c |
|---|---|
| Explicit nonzero target | Not called. |
| Implicitly acquired nonzero target | Called with the acquired target. |
| Implicit acquisition returning zero | Not called. |

Cleanup return values are not used to change the pending result. Cleanup side effects are otherwise unknown.

## Side effects

### Direct writes and persistence

The function’s explicit writes, when their branches are reached, occur in this order:

1. Clear context +0x50 after F-3eadc90e0ec1.
2. Store the final creation result at context +0x50.
3. Write the metadata word through the subsequently reread associated address.
4. Update context +0x3C.

Earlier writes are not rolled back on later failure. An old association can be cleared even if preparation fails.

Failure does not explicitly clear bit 3 of the flags. A zero return therefore does not imply that this bit is clear. Conversely, a success-path flags write is not a guarantee about the flags at return: later dependencies may change them.

### Read timing and external effects

Context +0x10 is read separately for preparation, intermediate acquisition, callback invocation, and final creation. It is not captured once. A dependency or callback can change the value seen by a later stage.

Likewise, global inputs A and B are read on the active-descriptor branch when preparing F-0784f4f6b095’s arguments. The direct path does not explicitly read them. Their values may reflect effects of preceding dependencies. Their relative read order is unspecified by the supplied representation.

No dependency or callback is assumed to preserve globals, context fields, reachable memory, or resource state. Accordingly, the contract distinguishes this function’s explicit operations from the complete state visible after external calls.

Aliasing may make a direct write affect another field or storage location described here. No non-aliasing guarantee is established.

### Ownership

The call sequence suggests acquisition and cleanup roles, but does not establish ownership semantics:

- F-3eadc90e0ec1 receives no explicit old associated value.
- Every nonzero intermediate value receives one F-2642535311f0 call on a normally completing path.
- F-87ceecb12e6c is selected by target inequality, not an independently established ownership flag.
- No visible call explicitly disposes of the newly stored associated value after successful creation.

These are call obligations, not guarantees of allocation, reference counting, destruction, or release.

## Dependencies

| Opaque function ID | Established role |
|---|---|
| F-2a7d78c39bfc | Supplies the effective target when the requested target is zero. |
| F-3eadc90e0ec1 | Called without explicit arguments before clearing an initially nonzero associated field. |
| F-ea92bf179106 | Receives the effective target before operation-value retrieval. |
| F-834e12b6dfde | Supplies the retained operation value. |
| F-68e8df570ef9 | Selects direct preparation or descriptor inspection. |
| F-0784f4f6b095 | Receives two descriptor-derived arguments and current reads from two unresolved global locations; its result does not gate continuation. |
| F-6eaf99ef0ef2 | Supplies the direct preparation result. |
| F-5d152df5657d | Supplies the descriptor-assisted preparation result. |
| F-48a5e637b2ef | Supplies the intermediate value after nonzero preparation. |
| F-79055c71edcb | Supplies the value stored at context +0x50 after callback approval. |
| F-7eadd9e17dc6 | Supplies the metadata word written through the reread associated address. |
| F-2642535311f0 | Receives a nonzero intermediate value after callback-related processing. |
| F-4bd8a1faf697 | Receives the effective target during terminal cleanup. |
| F-87ceecb12e6c | Receives the effective target when it differs from the requested target. |

The indirect callback has no supplied opaque identity. Its dispatch location, argument positions, and zero/nonzero gating role are established; its internal behavior is not.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid accesses and normal returns unless stated otherwise.

1. **Unavailable implicit target.** Requested target is zero and F-2a7d78c39bfc returns zero. Expect return zero, no direct context access, F-4bd8a1faf697 with zero, and no F-87ceecb12e6c.

2. **Clear before failure.** The effective target and initial context +0x50 are nonzero. Expect F-3eadc90e0ec1 followed by a zero write to +0x50. If preparation returns zero, expect no intermediate acquisition or callback. The clear is not rolled back.

3. **Mode bypass.** F-68e8df570ef9 returns nonzero. Expect direct preparation without explicit descriptor inspection or reads of global inputs A and B by this function.

4. **Inactive descriptor.** Mode is zero and the descriptor address or first word is zero. Expect direct preparation. The descriptor +0x08 quantity and global inputs are not needed on this branch.

5. **Active descriptor with updated globals.** Mode is zero, the descriptor’s first word is nonzero, and its +0x08 word is 3. Suppose preceding dependencies leave global inputs A and B holding values A1 and B1, which remain unchanged during argument preparation. Expect F-0784f4f6b095 to receive the first word, 12, A1, and B1, followed by F-5d152df5657d. Entry-time global contents are irrelevant if overwritten before these reads.

6. **No global-read ordering guarantee.** If an external actor can modify either global during argument preparation, this contract does not require the third and fourth arguments to reflect one simultaneous snapshot or a specified read order.

7. **Zero intermediate.** Preparation is nonzero but F-48a5e637b2ef returns zero. Expect no callback, no final creation, no F-2642535311f0, and return zero.

8. **Callback rejection.** A nonzero intermediate is followed by a zero callback result. Expect no final creation, but expect F-2642535311f0 before terminal target cleanup.

9. **Creation failure.** Callback approval is followed by F-79055c71edcb returning zero. Expect zero stored at context +0x50, no metadata-producing call, no explicit success-flags update, and return zero.

10. **Success and destination reread.** Final creation returns nonzero. If F-7eadd9e17dc6 changes context +0x50 to another valid address, the metadata write uses that changed address. If the flags read equals **0x000000A5**, the flags write is **0x0000002D**. Normal completion returns one.

11. **Fresh operation argument.** The callback changes context +0x10 and returns nonzero. Final creation receives the changed value rather than necessarily receiving the value used for preparation.

## Uncertainties

The following remain **documented_only**:

- Semantic types, validity rules, and application meaning of targets, operation values, intermediates, and associated values.
- Dependency ownership, allocation, release, global-state effects, and error-reporting conventions.
- The initial contents, declarations, storage widths, permitted mutations, and meanings of global inputs A and B. Their observed use is as current global reads, not proven constants.
- The callback’s identity and internal effects.
- Synchronization requirements and concurrent mutation behavior.
- Behavior after invalid memory access, abnormal dependency termination, or an invalid callback target.

The descriptor quantity is a 32-bit word shifted left by two positions. When the result fits without losing high bits, its numerical value is four times the original nonnegative quantity. No range check is visible. The representation does not independently resolve signed-overflow language semantics or justify unrestricted mathematical multiplication for all inputs.

The apparent resource-management purpose is inferred from call placement. It does not establish that any dependency owns, frees, or preserves a resource.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping.

Established evidence comprises the branch predicates, argument positions, context and descriptor offsets, 32-bit field operations, global reads used for descriptor-helper arguments, ordering across completed calls, direct writes, cleanup conditions, and zero-or-one normal result.

In particular, the active-descriptor call takes its third and fourth arguments from two unresolved global locations. The representation establishes reads at call preparation, not constant values, entry-time capture, or a relative evaluation order.

Semantic role names are explanatory inferences. Unavailable dependency behavior and unresolved data declarations remain documented rather than assumed.

No implementation access, runtime execution, or runtime tests have been performed.
