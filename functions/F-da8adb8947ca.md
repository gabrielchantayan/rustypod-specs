<!-- Trusted provenance: F-da8adb8947ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da8adb8947ca

## Purpose

F-da8adb8947ca changes a four-word state by either removing its first linked entry or advancing its current position by 12. When no linked entry is available and the current position equals the recorded end, it first obtains two values through F-871e64ddc6de and uses them to establish a new descriptor and region.

The function has no explicit return value. It does not return the removed entry address or the position consumed by the advance.

Interpreting this operation as allocation from a reusable-entry list or a contiguous pool is **inferred intent**. Likewise, interpreting the dependency calls as storage requests is inferred, not established by the dependency implementation or an external contract.

## Interface

### Inputs and result

The inputs are:

- A state address identifying four consecutive 32-bit words.
- A signed 32-bit selector. Only equality with zero affects behavior; all nonzero values select the same branch.

The result is mutation of accessible memory. There is no explicit success indication, error result, or returned address.

### State representation

The following labels describe roles within this function, not independently established data-type contracts.

| Byte offset from state address | Width | Label | Use |
|---|---:|---|---|
| 0 | 32 bits | Descriptor head | Tested for zero, used to locate an earlier capacity, and updated during expansion |
| 4 | 32 bits | Reusable head | Tested first; when nonzero, identifies a next-link word |
| 8 | 32 bits | Current position | Compared with the end and ultimately advanced by 12 |
| 12 | 32 bits | Region end | Compared with the current position and replaced during expansion |

A descriptor is interpreted through three consecutive 32-bit words:

| Byte offset from descriptor address | Width | Label |
|---|---:|---|
| 0 | 32 bits | Previous descriptor |
| 4 | 32 bits | Capacity |
| 8 | 32 bits | Region base |

A reusable entry need only expose a readable 32-bit word at its address for the accesses established here. No remaining entry layout is established.

Addresses are carried in 32-bit words. The representation does not establish portable behavior on systems whose pointers cannot be represented that way.

### Preconditions and scope of guarantees

Every address accessed on the selected path must support the corresponding read or write. In particular:

- The reusable path requires a readable next-link word at the nonzero reusable-head address.
- Expansion with a zero selector and a nonzero descriptor head requires a readable capacity word at that descriptor’s offset 4.
- Expansion requires the first dependency result to support the descriptor accesses.
- The state must remain accessible throughout the operation.

Non-overlap is **not checked**. It is a condition for the ordinary interpretation that the state, descriptor, reusable entries, and earlier descriptors retain distinct roles. Without sufficient separation, a write through one role can change another role.

The function itself does not access the contents of the region identified by the second dependency result. Its suitability as usable storage, its extent, and its alignment require an external contract.

## Behavior

### Reusable-entry path

The first decision tests the reusable-head word for zero.

If it is nonzero, the function reads the 32-bit word at the address represented by that head, stores the fetched value into the state’s reusable-head word, and returns.

The next-link read precedes the state write. A self-referential link therefore produces the same head value. No cycle detection, address validation, entry clearing, or release operation appears.

This path does not inspect the selector, compare the current position with the end, or call F-871e64ddc6de. Its only explicit write destination is the state word at offset 4. The other three state words are not explicit write destinations.

If the entry’s link word overlaps that destination, the write also accesses entry storage. Consequently, “the removed entry is untouched” is only valid when that entry storage does not overlap the written state word.

### Expansion decision

When the reusable head is zero, the function compares the current-position and region-end words.

Equality triggers expansion. Inequality skips expansion and proceeds directly to the final position advance.

There is no less-than comparison, remaining-space calculation, or check for 12 available bytes. Thus:

- A position six bytes below the end still advances by 12 without expansion.
- A position above the end also advances without expansion.
- Equal zero values trigger expansion.

### Capacity selection

Capacity is selected only on expansion and is held as an unsigned 32-bit value.

For a nonzero selector, the selected capacity is 1. No earlier descriptor capacity is consulted.

For a zero selector:

- A zero descriptor head selects capacity 32.
- A nonzero descriptor head causes a read of its unsigned 32-bit capacity word at offset 4.

For an existing capacity, two candidates are calculated:

- The old capacity plus 32.
- The old capacity plus its logical right shift by one, plus its logical right shift by three.

The shifts apply separately to the original capacity. The second candidate therefore adds separately rounded-down half and eighth portions.

All these additions use unsigned 32-bit arithmetic. Each candidate is reduced modulo \(2^{32}\) before the unsigned comparison. The larger candidate is selected; equality selects the second candidate without changing the numeric result.

This is not an unbounded arithmetic maximum. Overflow can select a capacity smaller than the old capacity. No monotonic-growth or nonzero-result validation is present.

### Dependency calls

After selecting capacity, the function calls F-871e64ddc6de twice, sequentially:

| Call | First argument | Second argument | Subsequent use of result |
|---|---|---|---|
| First | 12 | 0 | Descriptor address |
| Second | Low 32 bits of capacity multiplied by 12 | 0 | Region-base value |

These argument values and their order are established. Their meaning as byte counts is **inferred** from the surrounding use, not established by the available dependency evidence. In particular, the function alone does not prove that the calls request or return storage of those exact byte lengths.

The second call occurs before any explicit initialization of the first result by this function. Neither result receives an explicit failure check.

If the first call returns zero normally, execution still reaches the second call before attempting descriptor initialization. If either call does not return normally, the subsequent sequence is not reached.

### Descriptor and state updates

After both calls return, the following ordered accesses are established. Here, “descriptor address” means the first call’s returned value; “region result” means the second call’s returned value.

1. Write the region result to the word at descriptor address plus 8.
2. Read the word at state address plus 0 and write that fetched value to descriptor address plus 0.
3. Write the selected capacity to descriptor address plus 4.
4. Write the descriptor address to state address plus 0.
5. Read the word at descriptor address plus 8.
6. Write that reread value to state address plus 8.
7. Write the reread value plus the unsigned 32-bit capacity-times-12 product to state address plus 12.

The addition in the seventh step uses unsigned 32-bit arithmetic. Its result is reduced modulo \(2^{32}\). No ordering or extent check compares the calculated end with the base.

Two distinctions are essential:

- The descriptor-head value used in the second step is read after both dependency calls **and after the first descriptor write**. Either dependency activity or an overlapping first write can change the value fetched.
- The region-base value used for the state updates is reread after descriptor publication. With overlapping storage, intervening writes may have changed it; it need not equal the second dependency result.

Under separation from the state and earlier live objects, with no interfering external mutation, this sequence initializes a descriptor with the region result, the post-call descriptor head, and the selected capacity, then publishes it. Without that separation, the ordered destination-based description remains applicable, but the ordinary descriptor interpretation may not.

### Final position advance

Every path other than the reusable-entry path ends by reading the current-position word, adding 12, and writing the result back to that word.

This final addition is represented as signed 32-bit addition. A portable wraparound guarantee at signed overflow is not established. Concrete behavior at that boundary is **documented_only**.

For representable arithmetic and separated storage:

- Without expansion, the previous current position increases by 12.
- With expansion, the current position becomes the region result plus 12.
- With capacity 1, the resulting current position equals the newly calculated end.

These conclusions additionally require that dependency activity and other interference do not invalidate the relevant assumptions.

## Side effects

### Explicit write destinations

The reusable path writes state offset 4.

The non-expanding position path writes state offset 8.

The expansion path writes, in order:

- First dependency result plus 8.
- First dependency result plus 0.
- First dependency result plus 4.
- State offset 0.
- State offset 8.
- State offset 12.
- State offset 8 again for the final advance.

These are destination expressions, not unconditional claims that distinct logical fields are preserved.

For example, expansion contains no explicit write *addressed as* the reusable-head field. Nevertheless, if the first dependency result equals the state address, writing descriptor offset 4 writes the reusable-head field. A dependency can also modify that field independently.

Similarly, the final state-position write can overwrite descriptor contents when those locations overlap.

### Retention and ownership

With adequately separated storage and no interfering mutation, the resulting state retains the first dependency result as its descriptor head. That descriptor retains the second result as its region base and the head fetched during initialization as its previous link.

Without separation, publication or later state writes can overwrite those descriptor words. Retention of the original region result, preservation of the previous link, and preservation of earlier descriptor chains are therefore conditional—not unconditional guarantees.

No explicit release call occurs. This establishes neither ownership nor a duty to release. The dependency results might be allocated, borrowed, shared, or subject to some other lifetime protocol; the supplied evidence does not decide.

Removing a reusable-head link likewise does not establish an ownership transfer or how the removed entry is made available to any consumer.

### Errors and partial effects

There is no explicit error return, rollback, bounds check, link validation, arithmetic-overflow check, or synchronization mechanism.

An inaccessible descriptor result can cause an invalid access after both dependency calls have occurred. A zero or otherwise unusable region result is not locally rejected before being stored and used in arithmetic.

Effects already performed are not explicitly undone if a later operation fails. No transactional guarantee is established.

## Dependencies

### F-871e64ddc6de

This is the only explicit callee.

Established call-site facts are:

- Expansion makes two calls, in the stated order.
- Their argument pairs are 12 and zero, then the unsigned 32-bit capacity-times-12 product and zero.
- The first result is subsequently used as a descriptor address.
- The second result is stored as a region-base value.
- No local failure test separates the calls or guards the later writes.

A storage-acquisition role and byte-count interpretation are **inferred intent**.

The dependency’s allocation policy, actual argument semantics, alignment, initialization, lifetime, failure behavior, global effects, and reentrancy are **documented_only**. It must not be assumed to preserve the supplied state or return disjoint storage.

Capacity selection occurs before the calls. The later previous-link read can therefore observe a different descriptor head from the one used to select capacity.

## Acceptance examples

These are conditional static consequences, not runtime observations. Except for the explicit aliasing example, assume accessible separated storage, no dependency-induced state changes, no concurrent interference, and representable final signed addition.

### Reusable entry takes precedence

With a nonzero reusable head and a readable next-link word, the reusable head becomes that next-link value. Equal position and end values do not cause expansion. No dependency call occurs.

A zero next link empties the recorded reusable chain. A self-link leaves the head unchanged.

### Unequal position and end

With a zero reusable head and unequal position and end, only the current-position word is explicitly written, increasing by 12.

If the position initially lies six bytes below the end, it finishes six bytes above it. The selector does not affect this path.

### Nonzero selector

With an empty reusable chain, equal position and end, and any nonzero selector:

- The dependency argument pairs are 12 and zero, then 12 and zero.
- The selected capacity is 1.
- The new descriptor records capacity 1.
- The final position and calculated end both equal the region result plus 12.

Negative selector values have the same effect as positive nonzero values. Interpreting the two first arguments as 12-byte requests requires a dependency contract not supplied here.

### Initial default capacity

With a zero selector, zero descriptor head, empty reusable chain, and equal position and end:

- The selected capacity is 32.
- The dependency argument pairs are 12 and zero, then 384 and zero.
- The previous-descriptor word is zero.
- The final position is the region result plus 12.
- The end is the region result plus 384, using unsigned 32-bit arithmetic.

### Growth choices

For an existing capacity of 32, the candidates are 64 and 52. Capacity becomes 64, and the second call’s first argument is 768.

For an existing capacity of 64, the candidates are 96 and 104. Capacity becomes 104, and that argument is 1,248.

For an existing capacity of zero, the candidates are 32 and zero. Capacity becomes 32; the old zero capacity is not rejected.

### Wrapped growth

For an existing capacity of 4,294,967,295, the additive candidate wraps to 31. The other candidate wraps to 2,684,354,557 and wins the unsigned comparison.

Multiplication by 12 then produces a second-call first argument of 2,147,483,612 after reduction modulo \(2^{32}\). No local overflow error is reported.

### Dependency changes the head

If a dependency changes the descriptor head and storage is separated, the previous-descriptor word receives that changed head when initialization reads it. Capacity can still have been chosen using the earlier descriptor.

### Descriptor result aliases the state

Instead assume the first dependency result equals the state address. Let the initial reusable head and descriptor head be zero, position equal end, and selector be nonzero. Assume no dependency mutation, a suitable region result, and representable arithmetic.

The descriptor-capacity write sets the state’s reusable head to 1. Publishing the descriptor overwrites its previous-descriptor word with the state’s own address. The final position advance overwrites its region-base word with the region result plus 12.

This case demonstrates that distinct-field preservation, previous-link retention, and region-base retention do not hold without separation.

## Uncertainties

The following remain **documented_only**:

- The platform ABI and pointer semantics beyond the visible 32-bit representation.
- The dependency’s true argument meanings, including whether the first argument denotes bytes.
- Allocation extent, alignment, ownership, release, initialization, and failure signaling.
- Dependency-induced state changes, callbacks, or reentrant activity.
- Whether overlapping state, descriptors, entries, or returned storage are permitted by a surrounding protocol.
- Intended invariants connecting the reusable chain, descriptors, and current region.
- Concrete execution behavior at signed-overflow boundaries.
- Thread safety, atomicity, hardware memory ordering, and exact machine-level read counts.

The pool-allocation interpretation adds no guarantees to the established accesses.

## Evidence

The supplied static function representation establishes branch precedence, 32-bit field accesses, unsigned capacity arithmetic, dependency argument values and order, ordered descriptor and state accesses, and the final signed position addition.

It also establishes the absence of explicit local validation, release, rollback, and failure handling. It does not establish the dependency’s implementation or storage contract.

No runtime execution or runtime tests have been performed. The acceptance examples are derived from the static behavior under their stated conditions.
