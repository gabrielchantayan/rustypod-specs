<!-- Trusted provenance: F-8a5aadac202f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8a5aadac202f

## Purpose

F-8a5aadac202f conditionally releases a reference represented by a caller-supplied address slot. Some references are exempt from release. For other references, it either updates a reference-state word and returns or enters a destruction path.

The destruction path invokes a dependency, obtains a management-context address from a global source, validates allocation metadata, updates management counters, and invokes two final dependencies.

**Established behavior:** the function contains reference-state changes and allocation-accounting changes. It has no explicit result value.

**Inferred intent:** the metadata and counter operations resemble reference-counted object release followed by allocator bookkeeping. That interpretation does not establish what the dependencies destroy, whether backing memory is actually freed, or whether the caller’s address remains usable.

## Interface

### Input and result

The sole argument is the address of a slot containing an object-data address.

The function returns no explicit value. Observable outcomes are:

- Return without a write by this function.
- Update a reference-state word, then return.
- Enter the destruction path, update accounting state, invoke dependencies, then return if those dependencies return.
- Invoke F-a817d8d58c10 on invalid allocation metadata. The supplied evidence marks that dependency as nonreturning.

No status code, replacement address, or explicit error object is produced.

### Widths and address interpretation

The supplied representation uses:

- 32-bit signed words for reference state and several counters.
- 32-bit unsigned words for allocation metadata and masked quantities.
- 16-bit unsigned allocation-category indices.
- Address arithmetic expressed through integer-valued addresses.

This contract describes the evident 32-bit target model. Exact target arithmetic semantics, alignment rules, and behavior on invalid addresses are not independently established. In particular, the representation alone does not justify imposing portable signed-language overflow semantics on the underlying operations.

For ordinary valid executions, all addresses and arithmetic described below must be meaningful in that target model.

### Object metadata layout

Relative to an object-data address, the function accesses:

| Relative byte offset | Width | Role in this function |
|---|---:|---|
| −20 | 32 bits | Allocation descriptor; validated and masked for accounting |
| −16 | 16 bits, unsigned | First accounting-category index |
| −14 | 16 bits, unsigned | Second accounting-category index |
| −12 | 32 bits | Reference-state word; also the address passed to the first destruction dependency |

The function does not directly read object-data bytes at offset zero or above.

The reference-state address is twelve bytes before the address loaded from the caller’s slot. The allocation-descriptor address is twenty bytes before the address captured for destruction.

### External state

The function uses three externally supplied values:

1. A distinguished object-data address that causes an immediate return.
2. A distinguished reference-state address that exempts some references from decrementing.
3. A global source holding a management-context address, read after F-2fd8826557a4 returns.

The identities, initialization, and lifetime of these values are not supplied.

### Preconditions

For a defined, normally returning execution, every reached memory access must be valid. This includes the input slot, metadata, the management-context source, and all selected accounting words.

The function does not explicitly check:

- Whether the input argument is null.
- Whether the object-data address is null.
- Whether metadata is readable or writable.
- Whether category indices fit the actual management-context allocation.
- Whether accounting counters can be decremented without wrapping.
- Whether dependencies preserve metadata, globals, or backing storage.

These are preconditions or unresolved dependency contracts, not guarantees provided by this function.

## Behavior

### 1. Exempt the distinguished object-data address

The function first reads the caller’s slot and compares its value with the distinguished object-data address.

If equal, it returns immediately. This path performs no explicit metadata access, metadata write, accounting update, or dependency call.

If unequal, the function reads the caller’s slot again and derives a reference-state address by subtracting twelve bytes. The comparison and address derivation therefore use separate reads, not a single captured value.

### 2. Examine and possibly decrement reference state

The function reads the 32-bit reference-state word.

If that word is −1:

- It does not compare the reference-state address against the distinguished reference-state address.
- It does not decrement the word.
- It proceeds to the destruction path.

If the word is not −1, the function compares the derived reference-state address against the distinguished reference-state address. Equality causes an immediate return, without a reference-state write.

Otherwise, it reads the reference-state word again, subtracts one, and stores the result to the same location. It then examines the computed result:

- A result other than −1 causes a return.
- A result of −1 enters the destruction path.

For stable ordinary metadata, a starting value of zero is thus changed to −1 and enters destruction. A starting value of one is changed to zero and returns. A starting value already equal to −1 enters destruction without a decrement.

The two reference-state reads on the decrement branch are distinct. If the word can change between them, the first read controls branch selection while the later read supplies the decrement operand. Synchronization behavior is not established.

### 3. Capture the address used for destruction

On entry to the destruction path, the function reads the caller’s slot again. This newly captured object-data address is used for the remainder of its direct object-metadata addressing.

It derives an address twelve bytes before the captured address. If the derived address is zero, the function returns without calling a dependency or changing management counters.

This is not a general null-object check. In the ordinary address model, a captured object-data address of twelve produces a zero derived address; a captured address of zero does not.

The address used here need not equal the one used for the earlier reference-state decision if the caller’s slot changes or aliases a location modified along the way.

### 4. Invoke the first destruction dependency and select context

For a nonzero derived address, the function calls F-2fd8826557a4 with:

1. The derived address twelve bytes before object data.
2. The integer value 2.
3. The integer value 0.

Only after that call returns does the function read the management-context address from its global source.

This order is significant. A change to that global source during F-2fd8826557a4 affects which management context is selected.

The function does not reload the caller’s slot after this call. Its subsequent object-metadata accesses use the previously captured object-data address.

### 5. Validate allocation metadata

The function considers the 32-bit allocation descriptor twenty bytes before the captured object-data address.

Validation proceeds as follows:

- It reads the descriptor and rejects a value of zero.
- If the first read was nonzero, it reads the descriptor again and rejects a value whose least significant bit is set.

Either rejection invokes F-a817d8d58c10, which is marked nonreturning. No explicit accounting update precedes that invocation. However, F-2fd8826557a4 has already been called, and any earlier reference-state decrement has already occurred.

For stable metadata, validation therefore accepts exactly nonzero descriptor values with bit zero clear. It does not require the later masked accounting quantity to be nonzero.

The separate validation reads matter if metadata changes between them. The supplied function provides no established synchronization guarantee for this interval.

### 6. Begin management bookkeeping

After validation, the function calls F-c0fc6d033caa with the selected management-context address.

Only after that call returns does it read the first 16-bit category index and obtain the allocation descriptor used for the first group of accounting updates.

Let:

- The first category index be the unsigned 16-bit value at object-data offset −16.
- The second category index be the unsigned 16-bit value at object-data offset −14, read later.
- The accounting quantity be the allocation descriptor masked with `0x3ffffffc`.

The mask retains bits 2 through 29 and clears bits 0, 1, 30, and 31. Its result lies from zero through `0x3ffffffc` and is a multiple of four.

The function updates four 32-bit accounting words in this order:

1. Subtract the captured accounting quantity from the word at management-context offset `0x224 + 4 × first category index`.
2. Subtract the same quantity from the word at offset `0x30c`.
3. Read the second category index and subtract the same quantity from the word at offset `0xe0 + 4 × second category index`.
4. Subtract the same quantity from the word at offset `0x220`.

Each operation reads the current destination word and then writes its subtraction result. The first four updates share one captured masked quantity; they do not each reread the descriptor.

Neither category index is explicitly range-checked. Each can represent values from 0 through 65,535, producing a category displacement from zero through 262,140 bytes before the fixed offset is added.

### 7. Update count-related accounting

The function next calls F-09dcb55e8e0a with no explicit arguments. Its returned integer selects another accounting word.

After that dependency returns, the function performs these updates in order:

1. Decrement the 32-bit word at management-context offset `0x310 + 4 × returned integer`.
2. Decrement the 32-bit word at management-context offset `0x390`.
3. Reread the allocation descriptor, mask it with `0x3ffffffc`, and subtract that newly obtained quantity from the 32-bit word at management-context offset `0x08`.

There is no explicit validation or bounds check on the integer returned by F-09dcb55e8e0a.

The last subtraction need not use the same quantity as the first four subtractions. The descriptor is reread after the intervening dependency call and accounting writes.

### 8. Invoke final dependencies

After all explicit accounting updates, the function calls:

1. F-cc6c9b4a0f68 with the selected management-context address and the allocation-descriptor address.
2. F-e586bff64fb7 with the selected management-context address.

It then returns if both dependencies return.

No explicit rollback restores reference state or accounting state if an intervening dependency does not return normally.

## Side effects

### Direct writes

The function can directly write:

- One reference-state word, on the decrement branch.
- Seven management-accounting words, on a completed bookkeeping path.

There is no explicit assignment to the caller’s address slot and no explicit clearing of the object-data address.

This does not guarantee that the slot remains unchanged: it may alias written storage, or a dependency may modify it indirectly.

### Ordering and aliasing

Accounting destinations are not established to be disjoint. If selected destinations overlap, later operations observe the results of earlier writes. The contract does not permit treating all accounting updates as simultaneous or as operating on independent counters.

The selected management-context address remains the one captured after F-2fd8826557a4. Later changes to the global source do not cause the function to select a different context.

### Ownership and concurrency

The visible operations suggest release and allocation bookkeeping, but actual ownership transfer, destruction, deallocation, and post-return validity depend on the opaque dependencies.

There is no visible atomic reference-count operation or explicit synchronization around the early reference-state accesses. F-c0fc6d033caa and F-e586bff64fb7 might delimit synchronization, but that is inferred intent only. Their names and behavior are not available as evidence.

## Dependencies

| Opaque ID | Invocation contract visible here |
|---|---|
| F-2fd8826557a4 | Called with the nonzero reference-state address and integer arguments 2 and 0, before management-context selection and descriptor validation. |
| F-a817d8d58c10 | Called when descriptor validation rejects metadata; marked nonreturning. |
| F-c0fc6d033caa | Called with the selected management context after validation and before accounting metadata is read for updates. |
| F-09dcb55e8e0a | Called without explicit arguments; returns the integer used to select a count-related accounting word. |
| F-cc6c9b4a0f68 | Called after accounting updates with the selected context and allocation-descriptor address. |
| F-e586bff64fb7 | Called last with the selected context. |

No dependency is assumed to preserve globals, preserve metadata, acquire or release locks, own memory, free memory, or return normally unless explicitly stated above.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Distinguished object-data address.** If the first input-slot read equals the distinguished object-data address, the function returns without reading reference-state metadata or invoking dependencies.

2. **Ordinary retained reference.** With stable valid storage, a nonexempt reference-state word of 1 becomes 0. No destruction dependency is invoked.

3. **Transition into destruction.** With stable valid storage and a nonexempt reference-state word of 0, the word becomes −1 and destruction begins, subject to the later zero-address guard.

4. **Already negative-one state.** A reference-state word of −1 is not decremented. Even if its address equals the distinguished reference-state address, that address comparison is bypassed and destruction is reached.

5. **Distinguished reference-state address.** If the reference-state word is not −1 and its address equals the distinguished reference-state address, the function returns without decrementing it.

6. **Rejected descriptor.** A stable descriptor of zero or a stable odd descriptor invokes F-a817d8d58c10 after F-2fd8826557a4. The normal accounting sequence is not reached.

7. **Accepted zero accounting quantity.** A stable descriptor of 2 passes validation but produces a masked quantity of zero. The explicit subtraction operations still occur, count-related words are decremented, and final dependencies are invoked.

8. **Nonzero accounting quantity.** With a stable descriptor of `0x42`, the masked quantity is `0x40`. Assuming dependencies return and addresses remain valid, the first four accounting updates subtract 64; the final size-related update also subtracts 64 if the descriptor remains unchanged.

9. **Descriptor changes during a dependency.** If the descriptor is `0x40` for the first accounting group but becomes `0x80` during F-09dcb55e8e0a, the first four subtractions use 64 and the subtraction at context offset `0x08` uses 128. No second validation is shown.

## Uncertainties

The following are **documented_only**:

- Actual dependency semantics and failure modes.
- Whether any dependency releases the allocation before later metadata reads.
- Whether the context has a fixed valid extent and what category ranges it supports.
- The meaning and permitted range of the index returned by F-09dcb55e8e0a.
- Whether metadata or accounting destinations can alias the input slot or each other.
- Concurrency, memory ordering, and locking guarantees.
- Target behavior for arithmetic overflow, address wraparound, misalignment, and invalid memory accesses.
- The semantic meanings of descriptor flags outside the explicitly tested and masked bits.
- Whether a returned call leaves the caller with a dangling reference, a valid shared reference, or another state.

## Evidence

The contract is based solely on the supplied static function representation and dependency-ID mapping. Its established claims concern visible branches, memory accesses, argument values, arithmetic masks, and call ordering.

Reference-counting, allocator bookkeeping, and possible synchronization are interpretations, not independently verified dependency contracts.

No implementation access, runtime execution, smoke checks, or tests were performed. The acceptance examples describe consequences of the supplied static representation under their stated conditions; they are not observed results.
