<!-- Trusted provenance: F-0fe4d29cf9f4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0fe4d29cf9f4

## Purpose

F-0fe4d29cf9f4 performs a cleanup-shaped sequence over an input object: a gated preliminary operation, callbacks over a counted collection, conditional traversal of a second structure, and a final storage-related call.

“Cleanup” is inferred intent. The available body does not establish destruction, deallocation, ownership transfer, or release of the input object.

Normal completion returns the original input value. A path through a dependency annotated as non-returning does not normally reach the final storage-related call or the return, subject to the accuracy of that annotation.

## Interface

### Input and result

The function accepts one 32-bit value used as an object address. On normal return, it returns that original 32-bit value.

There is no visible null-input check, storage validation, explicit error result, or recovery path.

The supplied representation uses:

- Four-byte addresses, references, object fields, and table entries.
- An unsigned 32-bit count and index for the first traversal.
- Unsigned 32-bit arithmetic for that traversal’s slot-address calculation.
- Signed 32-bit representations for several other address calculations and the final subtraction and right shift.

These are properties of the supplied representation, not a portability guarantee.

### Input-object layout

Offsets are byte offsets from the input address.

| Offset | Width | Use |
|---|---:|---|
| +0x08 | 4 bytes | First explicit argument to the enabled preliminary operation. |
| +0x0c | 4 bytes | Third explicit argument to that operation. |
| +0x10 | 4 bytes | Fourth explicit argument to that operation. |
| +0x14 | 4 bytes | Copied into the first word of shared temporary storage on the enabled path. |
| +0x18 | 4 bytes | Copied into the adjacent word of that storage on the enabled path. |
| +0x1c | 4 bytes | Controls entry to the second-structure phase and is reread within that phase. |
| +0x20 | 4 bytes | First collection’s base; this field’s address is also passed to a dependency. |
| +0x24 | 4 bytes | Repeatedly read endpoint for the final address walk. |
| +0x28 | 4 bytes | Boundary used in the final call’s second argument. |

A processed nonnull element must provide a readable first-word table reference and a callable entry at byte offset +4 in that table. The complete callback ABI, including any implicit receiver convention, is unresolved.

### Temporary storage and aliasing

Two adjacent four-byte temporary words are shared across phases. The enabled preliminary path populates the first from input +0x14 and the second from input +0x18. Later, every call to F-9947a2c8a627 receives the address of that same first word.

This establishes an eight-byte local layout, not an eight-byte output-only dependency interface. The dependency’s access extent and whether it reads, writes, or both are unknown. If the preliminary path is skipped, the caller performs no corresponding initialization of these words before their address may be passed onward. No preservation guarantee establishes their contents after intervening calls or earlier uses.

The caller also uses adjacent four-byte pairs for initializer destinations, a working cursor, and a captured endpoint. Passing the address of a pair’s first word does not establish how many bytes the receiving dependency accesses or initializes.

### Preconditions

Valid execution requires readable storage for every object field, collection slot, table entry, and cursor-relative location actually accessed. Dependencies require whatever additional preconditions their unavailable contracts impose.

Normal termination additionally requires returning dependencies, progress in the second traversal, and reachability of the final walk’s endpoint. The function does not enforce these requirements.

## Behavior

### 1. Gate and perform the preliminary operation

F-b8fd684a82f2 is called without explicit arguments.

A zero result skips the preliminary operation, including the reads from +0x14 and +0x18. A nonzero result causes these ordered actions:

1. Read input +0x14 into the first shared temporary word.
2. Read input +0x18 into its adjacent word.
3. Invoke F-395ef1af5853 with four explicit arguments: the value from +0x08, a global value, the value from +0x0c, and the value from +0x10.

The global operand is a value read for the enabled call, not a fixed or immutable constant. It can reflect changes made by F-b8fd684a82f2.

The supplied expression does not establish the relative evaluation order of the global read and the three argument-field reads. All belong to the enabled call after the two explicit temporary-populating reads. The +0x14 and +0x18 values are not explicit arguments to F-395ef1af5853.

Neither dependency has an established preservation contract. Subsequent observations must not be assumed to see entry-state object fields, globals, or memory.

### 2. Traverse the first collection with a captured count

F-0c14ee61f55d receives the address of input +0x20. Its result is captured as an unsigned 32-bit count.

Indices run upward from zero while strictly less than that captured count. Each iteration:

- Rereads the base from +0x20.
- Computes the slot address using unsigned 32-bit multiplication of the index by four and unsigned 32-bit addition of the loaded base.
- Reads one four-byte reference from that address.
- Skips the callback if the reference is zero.
- Otherwise reads the referenced object’s first-word table address, obtains the callable entry at table offset +4, and invokes it.

The slot-address arithmetic wraps modulo \(2^{32}\). Equivalently, the effective address is the low 32 bits of the loaded base plus four times the index. This established unsigned behavior is distinct from the unresolved signed-overflow behavior of other expressions.

The captured count is not recomputed. The base and each slot are observed when their iteration is reached. A callback can therefore redirect later iterations by changing +0x20 without changing the captured iteration bound.

There is no deduplication. Repeated nonnull references produce repeated callbacks if execution reaches their slots. The caller does not visibly clear elements or reset collection fields.

Even the largest unsigned count does not require the loop index to wrap: after processing index \(2^{32}-2\), incrementing to \(2^{32}-1\) makes the strict comparison false when that is the count. Slot-address wrap can occur much earlier.

### 3. Enter and initialize the second traversal

After the first traversal, input +0x1c is read.

If zero, the entire second-structure phase is skipped, including its later field recheck and non-returning path. Execution proceeds directly to the final phase.

If nonzero:

1. F-bab66a21209f receives the address of the first word of a local two-word initializer destination.
2. Both words of that destination are copied, in order, into the working cursor pair.
3. F-d6fd6932ce90 receives another initializer destination’s address and a newly read value from +0x1c.
4. Both words of that destination are copied, in order, into the endpoint pair.

F-bab66a21209f has no explicit second-structure argument. Its context source is unknown. Its effects can change the +0x1c value subsequently supplied to F-d6fd6932ce90, including making it zero.

The caller copies both words from each destination, but this does not prove that either dependency initializes both words. Missing initialization is not repaired by the caller.

The endpoint is captured once. The visible caller does not rebuild it after callbacks or later changes to the structure.

### 4. Traverse using the cursor’s second word

Only the second words of the cursor and endpoint pairs participate in the termination comparison. Equal values skip the loop regardless of their first words.

On each unequal comparison:

1. Read a four-byte reference at the address represented by the cursor’s second word minus eight bytes.
2. If that read is zero, skip the callback.
3. Otherwise, read the reference at that cursor-relative location again, obtain its first-word table address, and invoke the entry at table offset +4.
4. Call F-9947a2c8a627 with the shared temporary storage’s first-word address, the working cursor pair’s first-word address, and zero.
5. Compare the cursor’s then-current second word with the captured endpoint’s second word again.

The nonnull path contains two reads of the cursor-relative reference, unlike the first traversal’s single slot read. With stable storage they select the same reference. The first read is not a guarantee that the second read remains nonzero or valid under mutation.

F-9947a2c8a627 receives the same temporary location previously populated from +0x14, adjacent to the location populated from +0x18. Calling it does not establish that those values remain present, that they are inputs, or that the area is output-only.

The caller does not copy that shared storage back into the cursor. Any progress must arise from dependency effects on the passed cursor or other unresolved effects. If the relevant cursor word remains unequal and unchanged, the loop has no visible independent advancement mechanism.

### 5. Recheck the second-structure field

Only after an entered second traversal finishes does the function reread +0x1c.

If that read is zero, execution proceeds to the final phase.

If nonzero, the function:

- Reads +0x1c again for the next call’s argument.
- Calls F-e7559f7e821e with that argument value plus 0x10.
- Passes its returned 32-bit value minus 0x10 to F-feee9156a257.

The branch-testing read and argument read are separate. Stable storage makes their values agree; the caller does not capture one value for both purposes.

F-feee9156a257 is annotated as non-returning in the supplied representation. Under that annotation, this path never reaches the final phase or normal return. Its mechanism—such as process termination, exception transfer, or another nonlocal exit—is unresolved.

The additions and subtraction here use signed representations. Behavior at signed-overflow boundaries is not established as modular arithmetic.

### 6. Walk addresses, make the final call, and return

The final phase captures +0x20 once as its base and starts a walking address at that value.

Each comparison rereads +0x24. Equality ends the walk; inequality causes an increment of four followed by another comparison. The body performs no element read, callback, or object-field write.

With a stable endpoint reachable by non-overflowing increments of four, the walk terminates. An unreachable endpoint is not rounded or rejected. Signed-overflow behavior is unresolved; unsigned wrap semantics from the first traversal must not be imported here.

After equality, the function reads +0x28 and calls F-ff1bdc376dcd with:

1. The captured base.
2. The signed 32-bit difference between +0x28 and that base, right-shifted by two.
3. Zero.

For a representable, nonnegative difference divisible by four, the second argument is that difference divided by four. Negative right-shift behavior and overflowing subtraction are not given a portable interpretation. The function does not validate an allocation size.

If F-ff1bdc376dcd returns normally, the function returns the original input value.

## Side effects

The caller directly reads object fields, a global value, collection references, callback tables, and cursor-relative storage. Its explicit writes affect temporary storage and traversal state, not input-object fields or collection slots.

Dependencies can have additional effects:

- F-0c14ee61f55d receives an object-field address.
- Initializers receive temporary-storage addresses.
- F-9947a2c8a627 receives shared temporary storage and cursor storage.
- Callbacks and other dependencies may mutate accessible objects, globals, or allocation state.

No preservation, ownership, retention, release, or access-extent contract follows merely from these arguments. In particular, normal return does not prove that the input object or any referenced storage remains valid.

Faults, dependency failures, and nonlocal exits can occur after partial progress. No rollback, error translation, or recovery is visible. Concurrent and reentrant mutations have no established synchronization contract.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-b8fd684a82f2 | Supplies the preliminary zero/nonzero gate. |
| F-395ef1af5853 | Receives three object-field values and one global value read for the enabled call. |
| F-0c14ee61f55d | Receives the address of +0x20; returns the captured unsigned traversal count. |
| F-bab66a21209f | Receives the first cursor-initialization destination’s address. |
| F-d6fd6932ce90 | Receives the endpoint-initialization destination’s address and a fresh +0x1c value. |
| F-9947a2c8a627 | Receives shared temporary storage, working cursor storage, and zero after each second-traversal iteration. |
| F-e7559f7e821e | Receives a fresh +0x1c value adjusted by +0x10. |
| F-feee9156a257 | Receives the preceding result adjusted by −0x10; annotated non-returning. |
| F-ff1bdc376dcd | Receives the captured collection base, shifted boundary difference, and zero. |

Indirect callbacks have no supplied opaque IDs. Their table-dispatch layout is established; their full signatures and effects are documented_only.

## Acceptance examples

These scenarios are conditional consequences of static inspection, not executed tests.

1. **Empty normal path.** If the gate and captured count are zero, +0x1c is zero, and +0x24 equals the final captured base, no callbacks occur. The second phase is skipped. The final dependency is called, and a normal return from it produces the original input value.

2. **Global changed by the gate.** If the gate changes the global operand and returns nonzero, the enabled preliminary call reads the global after that gate call. Its value is not constrained to the entry-state value. No relative ordering is promised among this read and the three argument-field reads.

3. **Shared temporary storage.** If the enabled path reads two particular words from +0x14 and +0x18, they initially occupy the adjacent shared locations. A later F-9947a2c8a627 call receives the first location’s address. Their continued presence or consumption requires additional dependency-preservation and access assumptions.

4. **Unsigned slot-address wrap.** With a loaded base of 0xfffffffc, index one selects address 0x00000000. This follows from unsigned 32-bit addition, not signed-overflow interpretation. A valid read at that address is an additional precondition, not guaranteed by the function.

5. **Nulls, duplicates, and base mutation.** For a captured count of three, stable slots containing a nonnull reference, zero, and the same reference produce two callbacks. If the first callback changes +0x20, later slots instead come from the changed base at their respective wrapped offsets.

6. **Equal cursor and endpoint.** Equal second words after initialization cause no cursor-relative reads or F-9947a2c8a627 calls. The post-traversal +0x1c recheck still occurs because the phase was entered.

7. **No cursor progress.** If unequal second words remain unchanged and callbacks and helpers keep returning without altering the path, the same cursor-relative location recurs. Calling F-9947a2c8a627 alone does not establish termination.

8. **Second-structure branch outcomes.** If an entered traversal terminates and its field recheck observes zero, the final phase is reached. If it observes nonzero, the subsequent freshly read value determines F-e7559f7e821e’s adjusted argument. Under the non-returning annotation, F-feee9156a257 prevents normal completion.

9. **Ordinary boundary calculation.** If +0x28 is 64 bytes above the captured base with representable subtraction, the final call’s second argument is 16. The independently observed +0x24 value controls the preceding walk, not this calculation.

## Uncertainties

The following remain documented_only:

- Semantic object and collection types; cleanup intent.
- Callback ABI, effects, and ownership behavior.
- Global operand meaning and dependency preservation guarantees.
- Purpose and later contents of the shared temporary words.
- F-9947a2c8a627’s read/write behavior and access extent.
- Initializer context, access extent, and completeness.
- Cursor advancement and termination guarantees.
- Purpose of the empty final walk.
- Accuracy and mechanism of the non-returning annotation.
- Behavior for invalid storage, signed overflow, negative shifts, and unsynchronized mutation.

These unknowns do not weaken the established unsigned wrapping rule for first-traversal slot addresses.

## Evidence

Established static evidence comprises the supplied body’s field offsets, operand widths, explicit arguments, local-storage reuse, read ordering where sequentially expressed, unsigned slot-address arithmetic, callback dispatch layout, captured versus reread values, branches, and original-input return value.

The non-returning property is an annotation in the supplied representation, not an independently verified runtime fact. Cleanup, collection, and cursor terminology describe inferred roles without adding dependency postconditions.

No runtime tests have been performed. The acceptance examples are conditional specification cases, not reports of execution.
