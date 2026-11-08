<!-- Trusted provenance: F-c7fef7dba7a3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c7fef7dba7a3

## Purpose

F-c7fef7dba7a3 acquires storage for an opaque context and starting position, then performs word operations corresponding to a requested number of logical bits.

**Established static behavior:** storage is addressed as 32-bit words within regions of 128 words, representing 4,096 logical bit positions. Partial-word operations OR a dependency-produced value into an existing word. Complete-word operations replace a word with a dependency-produced value.

**Inferred intent:** the function appears intended to mark a contiguous logical bit range. That interpretation requires suitable acquisition, mask transformation, storage-lifetime, and finalization behavior. None of those dependency semantics is established merely by this function.

A central distinction is between the **mutable region-reference slot** and the **cached word cursor**. Acquisition receives the address of the slot and may retain it. Later dependencies may therefore change the slot indirectly. Direct word operations use cached pointers; finalization uses the slot’s value at the relevant call. These need not continue to identify the same storage.

## Interface

| Input or output | Representation and meaning |
|---|---|
| Context | Opaque 32-bit value passed unchanged to each acquisition. Pointer, handle, and identifier interpretations are unresolved. |
| Starting position | Unsigned 32-bit value. Its full value is supplied to initial acquisition; its low 12 bits select the initial word and bit offset. |
| Length | Unsigned 32-bit logical bit count. Zero is accepted but can still cause dependency calls and a direct word update. |
| Return status | Signed 32-bit integer in the supplied representation. Zero permits progress; nonzero stops acquisition or transition processing. No individual error-code meanings are established. |

The directly used storage layout is:

- Unsigned 32-bit words, with a four-byte stride.
- 128 word positions per region: 512 bytes representing 4,096 logical bits.
- Initial word index: starting-position bits 5 through 11, yielding 0 through 127.
- Initial bit offset: starting-position bits 0 through 4, yielding 0 through 31.
- Logical bit zero of a word corresponds to its numeric most-significant bit before transformation.

These are addressing and numeric-mask conventions, not an established byte-order contract.

The function creates a region-reference slot initially containing null. Acquisition receives its address. The slot is distinct from the word pointers subsequently derived from it.

For defined direct accesses, each cached pointer must address suitably aligned, accessible 32-bit storage for every word actually accessed. Such storage must remain usable across intervening dependency calls. A success status alone does not establish these conditions, and the function performs no visible validation.

Ownership is **documented_only**: the evidence does not establish allocation, borrowing, locking, mapping, persistence, or release responsibilities. No explicit allocation or deallocation is performed directly by this function.

## Behavior

### Acquisition, preparation, and cursor establishment

The first dependency call is F-1f96b6eaec84, receiving the context, original starting position, and address of the initially null region-reference slot.

If acquisition returns nonzero, processing proceeds directly to exit cleanup. There is no preparation, mask transformation, or direct word access on this path. Cleanup examines the slot’s then-current value; a nonnull value is passed to finalization. The acquisition failure remains the returned status.

If acquisition returns zero:

1. F-d5b4823d3579 receives the region-reference value present when that call is made.
2. Preparation returns.
3. The function reads the region-reference slot to establish its initial cached word pointer, offset by the starting word index.

Consequently, preparation may receive one pointer while the subsequently established cursor uses another. This can occur if preparation changes the escaped slot indirectly. The function neither checks for that change nor prepares the replacement value again.

A null slot on acquisition success is not rejected. Preparation receives that null value if it remains null at the call. Preparation could change the slot before cursor establishment; whether it does so, and whether null is acceptable to it, are unknown.

The current word capacity is tracked independently of later slot changes. Initially, the number of available words is 128 minus the starting word index.

### Initial partial-word operation

When the starting bit offset is nonzero, an initial partial-word operation always occurs, including for length zero.

The consumed span is the smaller of the remaining length and the number of positions from the initial offset to the word’s end. Its raw mask selects that run using most-significant-bit-first numeric numbering. A zero span produces a zero mask.

The ordering is:

- Establish the selected word pointer before transformation.
- Call F-b5c826d2ab04 with the raw 32-bit mask.
- After transformation returns, read the word through that already cached pointer.
- OR the returned 32-bit value with the word read.
- Write the result through the same cached pointer.

The word pointer is not reloaded from the region-reference slot after transformation. A slot change during transformation affects later slot-based operations, not this destination.

This operation cannot clear bits present in the word as read after transformation. It does not guarantee preservation of the word’s earlier contents: transformation itself may alter storage. Nor does it prevent setting bits outside the raw mask, because the transformation’s mapping is unknown.

After this operation, the remaining length decreases by the consumed span, the next cached destination advances by four bytes, and the available-word count decreases by one. The advance and capacity decrement occur even when the consumed span is zero.

### Complete-word operations

While at least 32 positions remain, complete words are processed in increasing cached-address order.

Before each complete-word operation, an exhausted region capacity causes the transition described below. Otherwise:

- F-b5c826d2ab04 receives `0xFFFFFFFF`.
- After it returns, its 32-bit result replaces the word at the cached destination.
- The cursor advances four bytes.
- Available capacity decreases by one.
- Remaining length decreases by 32.

There is no direct read of the destination’s old value for this replacement. The transformation dependency may itself read or modify storage; its behavior is not constrained here.

Transformation is called separately for every complete word. Results are not reused. If transformation changes the region-reference slot, the replacement still uses the cached cursor.

### Transitions before complete words

A transition occurs only when another operation is needed and the available-word count is zero. Reaching a boundary with no remaining positions does not itself cause another acquisition.

For a transition before a complete word:

1. The running acquisition position increases by 4,096 using unsigned 32-bit arithmetic.
2. F-1e9d170e476a receives the region-reference slot’s current value and a second argument of zero.
3. A nonzero status stops processing and proceeds to cleanup.
4. On zero status, the function explicitly sets the region-reference slot to null.
5. F-1f96b6eaec84 receives the original context, advanced position, and the slot’s address.
6. A nonzero acquisition status stops processing and proceeds to cleanup.
7. On success, preparation receives the slot’s current value.
8. After preparation returns, the slot is read to establish a new cursor at word zero, with 128 available words.

The transition finalization call is not conditional on a nonnull slot. If an earlier dependency cleared the slot, this call receives null.

If successful transition finalization changes the slot, its replacement value is overwritten by the explicit null assignment before acquisition. No additional finalization of that replacement value is performed at this point.

Acquisition positions advance from the preceding acquisition position, not from a region-aligned base. Their low 12 bits remain unchanged. Correct interpretation as adjacent regions is a dependency assumption.

### Final partial-word operation

After complete-word processing, a remainder from 1 through 31 causes one final partial-word operation. A zero remainder causes none.

If capacity is exhausted, the function first finalizes using the slot’s current value. On success it clears the slot, acquires using the running position plus 4,096, prepares, and then reads the slot to establish a cursor at word zero. This final acquisition argument is calculated without updating the running position, because no further transition is needed.

The raw mask consists of the remainder’s number of high-order ones. Transformation returns before the destination word is read through the cached cursor. The function then ORs the transformed value into that word.

As with the initial partial word, a transformation-induced slot change does not redirect the cached access.

### Exit cleanup and status

All normal exits reach one cleanup check:

- Read the region-reference slot’s then-current value.
- If nonnull, pass that value and zero to F-1e9d170e476a.
- Ignore that call’s return value.
- Return the saved processing status.

Cleanup does not repeat if its own finalization call changes the slot.

The returned status is the initial acquisition failure, the acquisition or finalization failure that stopped a transition, or zero if those checked operations succeeded. Preparation and transformation supply no checked failure status. Cleanup failure does not replace the saved status.

A failed transition finalization does **not** guarantee another call on the same pointer. During that failed call, the dependency could change the escaped slot:

- If the slot is then null, cleanup makes no call.
- If it contains another pointer, cleanup finalizes that pointer.
- Only if the slot still contains the original pointer does cleanup finalize that same pointer again.

There is no visible rollback of earlier direct writes. Their persistence is unresolved.

## Side effects

Directly observable operations include:

- Acquisition and, on success, preparation even for zero-length requests.
- One transformation call per word operation.
- Read–OR–write operations for partial words.
- Replacements without a direct old-value read for complete words.
- Checked finalization at required region transitions.
- A conditional, unchecked finalization at exit.
- Explicit clearing of the region-reference slot after successful transition finalization.

The slot/cursor distinction affects all these observations. After a cursor is established, a dependency can change the slot without changing that cached pointer. Word operations can therefore target storage different from the pointer later supplied to finalization. Conversely, preparation can change the slot before a cursor is established, causing subsequent direct operations to use a different base from preparation’s argument.

No dependency is assumed to preserve globals, the escaped slot, storage contents, or storage lifetime. If a dependency invalidates a cached destination before its access, safe execution is not established.

Repeated finalization of an identical pointer is possible, including after a transition failure under slot preservation. It is not unconditional.

No synchronization, atomic read–modify–write, transactional behavior, or thread-safety guarantee is visible. Position advancement wraps modulo 2³², and no overall endpoint or range validation occurs before writes begin.

## Dependencies

| Opaque ID | Observed interface | Established handling |
|---|---|---|
| F-1f96b6eaec84 | Context, unsigned 32-bit position, address of mutable region-reference slot. | Zero permits preparation; nonzero stops processing. Cleanup uses the slot’s value afterward. Retention of the slot address is not excluded. |
| F-d5b4823d3579 | Region-reference value at call time. | Return value is not used. Cursor establishment reads the slot after this call returns. |
| F-b5c826d2ab04 | Raw 32-bit mask. | Supplies the 32-bit word-update value. The subsequent access uses an already cached pointer, not a refreshed slot value. |
| F-1e9d170e476a | Region-reference value at call time, followed by zero. | Transition results are checked; exit-cleanup results are ignored. Indirect changes to the escaped slot are not excluded. |

“Acquisition,” “preparation,” “transformation,” and “finalization” describe call roles, not established ownership or persistence semantics.

Neither successful nor failed finalization establishes whether its argument remains valid. In particular, the function’s possible repeated call on one pointer is not proof that such repetition is supported by the dependency.

## Acceptance examples

Except where explicitly varied, these examples assume normally returning dependencies, usable storage, and slot preservation by preparation, transformation, and finalization. Acquisition supplies the stated region. These are conditional scenarios, not inferred dependency guarantees.

### Aligned zero length

For position 0 and length 0:

- Acquire at position 0 and prepare.
- Perform no transformation or direct word access.
- Finalize the nonnull slot value during cleanup.
- Return zero even if cleanup finalization returns nonzero.

### Unaligned zero length

For position 1 and length 0:

- Acquire and prepare.
- Transform raw mask zero.
- Read cached word 0 after transformation and write its value ORed with the transformed result.
- Finalize during cleanup.

The word remains unchanged only if transformation returns zero and does not otherwise change the relevant storage.

### Partial and complete words

For position 5 and length 3, transformation receives `0x07000000`; its result is ORed into cached word 0. With identity transformation and no relevant side effects, numeric bits 26, 25, and 24 are set.

For position 32 and length 32, transformation receives `0xFFFFFFFF`; its result replaces word 1. A result of `0x12345678` is written even if this clears previously set bits.

### Crossing a region boundary

For position 4,095 and length 2:

- Acquire at 4,095 and prepare.
- Transform `0x00000001` and OR into word 127.
- Finalize for the transition.
- On success, acquire at 8,191, not 4,096.
- Prepare, establish a new cursor, transform `0x80000000`, and OR into word 0.
- Perform exit cleanup.

These acquisition positions do not establish how the dependency maps them to storage.

### Slot changes around cursor establishment

For position 0 and length 1, suppose acquisition supplies region A.

If preparation receives A but changes the slot to B before returning, the cursor is established from B. If transformation then changes the slot to C, the direct partial-word operation still accesses B’s word 0. Cleanup finalizes C if C is nonnull.

This scenario requires B to remain valid for the access. No identity between A, B, and C is required by the visible function.

### Transition failure

For position 0 and length 4,097, 128 complete-word operations precede a transition for the final position.

Suppose transition finalization receives A and returns nonzero status E:

- No next acquisition or final partial-word operation occurs.
- If the slot still contains A, cleanup finalizes A again.
- If the slot now contains B, cleanup finalizes B.
- If the slot is null, cleanup performs no finalization.
- The returned status is E regardless of cleanup’s result.

### Acquisition failure

If acquisition returns nonzero status E and leaves a nonnull slot value B, no preparation or direct word access follows that acquisition. Cleanup finalizes B and the function returns E.

### Position wrap

For position `0xFFFFF000` and length 4,097, successful boundary processing acquires next at position zero. The unsigned addition wraps; no overflow error is generated by the visible arithmetic.

## Uncertainties

The following remain **documented_only**:

- Context validity and meaning.
- Exact pointer representation and platform alignment requirements.
- Acquisition’s position-to-storage mapping and possible aliasing.
- Whether acquisition retains the slot address, and which later dependencies can mutate it.
- Storage lifetime across preparation, transformation, and finalization.
- Ownership, locking, release, and persistence responsibilities.
- Transformation’s mathematical mapping, byte-order purpose, and side effects.
- Behavior of dependencies receiving null or repeated pointers.
- Error-code meanings beyond zero versus nonzero.
- Behavior after invalid memory access or a dependency that does not return normally.
- Whether position wrap is intentional.

The visible arithmetic bounds are established: word indices are 0–127, bit offsets are 0–31, and evaluated mask shifts have counts from 1 through 31. Initial consumption cannot exceed remaining length; complete-word subtraction occurs only with at least 32 positions remaining. Position addition alone can wrap as described above.

## Evidence

This contract is based solely on static examination of the supplied function representation and opaque dependency mapping. No runtime tests have been performed.

Established evidence covers widths, numeric masks, addressing, conditional branches, dependency-call order, slot reads and writes, cached-pointer use, cleanup conditions, and status propagation.

Slot stability is not established: acquisition receives the slot’s address, and subsequent calls are not shown to preserve it. The contract therefore distinguishes preparation-before-cursor-establishment, transformation-before-cached-access, and slot-based finalization rather than assuming one stable region identity.

Contiguous bit-range marking, persistence, ownership transfer, and resource release remain interpretations or dependency requirements, not unconditional outcomes.
