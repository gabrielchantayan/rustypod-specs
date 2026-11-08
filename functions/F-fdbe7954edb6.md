<!-- Trusted provenance: F-fdbe7954edb6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdbe7954edb6

## Purpose

F-fdbe7954edb6 conditionally reorganizes mutable object state. It obtains a comparison word from a dependency and returns immediately if that word matches an object field. Otherwise, it prepares temporary storage, traverses pointer-linked slots, explicitly removes nodes from those slots, delegates node handling, replaces selected object fields, and invokes two finishing dependencies.

The layout and operations suggest rebuilding a bucketed container. That is **inferred intent**, not an established guarantee. The supplied static evidence does not establish hashing rules, element preservation, insertion order, allocation policy, ownership transfer, or reclamation.

The contract below describes observable accesses and dependency calls under ordinary execution. Valid access, termination, and successful reconstruction require additional structural and dependency contracts. No dependency is assumed to preserve memory merely because that memory is absent from its explicit arguments.

## Interface

### Inputs and result

The function receives:

- **Object address:** A 32-bit address-like value identifying mutable storage.
- **Request word:** An opaque four-byte value supplied to the first invocation of F-0b31936eccf4.

There is no explicit return value. Results consist of memory changes and dependency effects.

The representation uses four-byte words and 32-bit address calculations. It does not establish an authoritative calling convention, a portable pointer model, or defined arithmetic-overflow behavior.

There is no visible validation of addresses, alignment, storage extent, chain structure, count consistency, or dependency success.

### Object layout

Each listed field occupies four bytes.

| Object-relative offset | Established use |
|---|---|
| +0x04 | Captured after temporary preparation and added to the object address for F-58cc241cfb6b. |
| +0x14 | Supplies the captured traversal base and initial working word; later exchanged directly with the first temporary field. |
| +0x18 | Compared with the initial dependency result; later contributes to the traversal boundary; finally passed by address to F-f673c27f3127. |
| +0x1C | Supplies a slot address; incremented after a zero head is observed; later replaced using the third temporary field. |
| +0x20 | Decremented after each explicit node removal; later replaced using the fourth temporary field. |

“Base,” “cursor,” and “count” describe these uses, not complete types or invariants.

### Temporary layout

The represented local storage has a leading four-byte area followed by four four-byte fields. Several dependencies receive the address of the leading area.

Preparation has the following established order:

1. The first field is set to zero.
2. F-0b31936eccf4 is invoked again, and its result is stored in the second field.
3. The third field is set to zero.
4. The fourth field is set to zero.
5. F-4b17fe508349 receives the address of the leading area.

The second invocation of F-0b31936eccf4 has no explicit argument in the supplied representation. Its effective input is **documented_only**. It must not be treated as necessarily receiving the original request word or producing the same result as the first invocation.

No explicit initialization of the leading area appears before it is passed to F-4b17fe508349. That dependency’s write footprint is unknown. Later temporary-field values need not remain equal to their initial values.

### Slot and node layout

A selected slot is a four-byte pointer cell. Its current zero value ends that inner-loop visit.

For a nonzero head:

- Words at node-relative offsets −0x10 and −0x0C supply metadata.
- The first word at a subsequently selected removal node supplies its successor.

These need not be observations of the same node: dependencies execute between the metadata observation and the removal-head observation.

The complete allocation layout and node payload are unknown. Valid execution requires accessible storage for every performed access, including negative-offset metadata reads and successor reads.

## Behavior

### Initial comparison

F-0b31936eccf4 is called with the request word. Only after it returns does the function read object field +0x18.

If the returned word equals that field, the function returns without temporary preparation, traversal, direct object writes, or further listed dependency calls.

This path is not established as side-effect-free. The first dependency has already executed, and the comparison observes +0x18 after any effects it caused.

### Preparation and captured state

On inequality, the function prepares temporary storage as described above and calls F-4b17fe508349.

After that call returns, it reads object fields +0x04, +0x14, and +0x18, in that order. It forms a traversal boundary from the captured +0x14 word plus four times the newly read +0x18 word.

The boundary is retained rather than recomputed during traversal. Later changes to either contributing field do not themselves change it.

The captured +0x04 word is retained for the derived-address argument to F-58cc241cfb6b. The captured +0x14 word also becomes the initial working word supplied to that dependency. Unlike the boundary, the working word is subsequently replaced by an additional register-derived output associated with node handling.

Calculations use 32-bit quantities and a four-byte slot stride. Valid execution requires suitable arithmetic and addresses. The evidence does not establish behavior for multiplication overflow, signed overflow, or address wraparound.

### Selecting a slot

At each outer-loop condition, the function freshly reads object field +0x1C and compares it with the captured boundary.

- Equality ends traversal.
- Inequality selects the read value as the slot address for an inner-loop visit.

This is an equality check, not an ordered bounds check. A cursor numerically beyond the boundary is not rejected merely for being beyond it.

The selected slot address remains fixed throughout that inner-loop visit. A dependency’s change to object field +0x1C does not by itself redirect the current visit.

### Metadata and destination processing

Each inner-loop condition reads the selected slot’s current head.

A zero head ends the inner loop. A nonzero head causes reads of the metadata words at offsets −0x10 and −0x0C relative to that observed head.

F-58cc241cfb6b then receives:

- The object address plus the captured +0x04 word.
- The current working word.
- The two metadata words.

Both metadata reads precede the call. Their relative order is not established by the supplied representation.

F-3ed23f1b4691 receives the temporary-area address and the result of F-58cc241cfb6b. Its result is retained as an argument for node handling.

Neither dependency is established to preserve the selected slot, the observed node’s successor, the count, the cursor, or other object state.

### Explicit removal

After both dependencies return, the function rereads the selected slot. This later head identifies the node explicitly removed.

The function then:

- Reads the first word at that node address.
- Writes the word just read into the selected slot.
- Reads object field +0x20, subtracts one, and writes the result back.
- Calls F-65514a6f4064 with the temporary-area address, the removed node address, and the result from F-3ed23f1b4691.

The removal uses the successor value present **when the successor read occurs**. An initially zero successor is insufficient to establish that zero is written: either preceding dependency may change that word while preserving both the selected slot and the count.

Likewise, preserving the initial metadata head is not implicit. If a preceding dependency replaces the slot head with another valid nonzero node, the removal uses the replacement node and its then-current successor. The earlier metadata can therefore describe a different node.

There is no visible nonzero check between the removal-head reread and its dereference. If a dependency clears the slot before that reread, the earlier nonzero observation provides no protection.

The count decrement uses the field value read after the two preceding dependencies. There is no zero-count guard. Behavior at arithmetic limits is unresolved.

### Working-word update and next observation

After F-65514a6f4064 returns, an additional machine-register output becomes the working word for subsequent F-58cc241cfb6b calls. This applies across both further nodes in the same slot and later slots.

The meaning and authoritative calling-convention status of that output are **documented_only**. The working word must not be described as an invariant copy of the original +0x14 field.

The next inner-loop condition freshly reads the selected slot. It does not rely on the successor value written before F-65514a6f4064.

Consequently, even if removal wrote zero, the loop ends only if the next head observation returns zero. F-65514a6f4064 may have repopulated the slot, and its preservation behavior is unknown.

### Cursor advancement

After an inner-loop condition observes a zero head, the function rereads object field +0x1C, adds four, and writes the result back.

This update uses the field’s current value, not necessarily the cached slot address. Dependency-induced cursor changes therefore affect the increment.

The next outer-loop condition compares another fresh cursor read with the original captured boundary.

### Final field operations

When the outer-loop condition observes equality with the boundary, the function performs these operations in order:

1. Saves object field +0x14, replaces it with the first temporary field, and stores the saved object value into that temporary field.
2. Calls F-f673c27f3127 with the addresses of the second temporary field and object field +0x18.
3. Saves object field +0x1C and replaces it with the third temporary field.
4. Saves object field +0x20 and replaces it with the fourth temporary field.
5. Stores the saved +0x1C value into the third temporary field.
6. Stores the saved +0x20 value into the fourth temporary field.
7. Calls F-dfc8ff5105e8 with the object address.
8. Calls F-e7559f7e821e with the temporary-area address.

The +0x14 operation is an explicit exchange. The operation on +0x18 is delegated; calling it an exchange requires F-f673c27f3127’s contract.

The final replacements are sequential, not an established atomic commit. No rollback is visible. Dependencies may affect values used by later operations.

## Side effects

### Direct writes

On the nonmatching path, direct writes can:

- Initialize and later overwrite temporary fields.
- Replace selected slot heads with successors read at removal time.
- Decrement object field +0x20 once per explicit removal.
- Advance object field +0x1C after each observed empty slot.
- Replace object fields +0x14, +0x1C, and +0x20 during final field operations.

No direct write to object field +0x18 is shown; its address is passed to a dependency.

These statements distinguish direct writes from dependency effects. They do not bound what a dependency can mutate.

### Ownership and lifetime

No explicit allocation or deallocation operation appears in this function. Passing a removed node to F-65514a6f4064 does not independently establish insertion, copying, destruction, or ownership transfer.

Similarly, passing temporary storage to F-e7559f7e821e does not establish which resources it releases, if any.

The temporary storage is local to this invocation. Whether dependencies retain references to it, and what lifetime obligations that would impose, remain **documented_only**.

### Errors, concurrency, and progress

There is no visible error result, retry, alternate cleanup path, synchronization, or recovery mechanism.

A failure after head replacement or count decrement can occur after partial mutation. Exception handling, traps, nonlocal exits, and dependency-failure propagation are unresolved.

Termination is not guaranteed for cyclic chains, unreachable boundaries, invalid arithmetic, or dependencies that repeatedly repopulate slots or redirect cursor state. Concurrent or reentrant access requires external contracts.

## Dependencies

| Opaque ID | Established participation | Unresolved behavior |
|---|---|---|
| F-0b31936eccf4 | Supplies the initial comparison word and, on the nonmatching path, a second word for temporary storage. | Transformation, determinism, second-call input, side effects. |
| F-4b17fe508349 | Receives temporary storage before traversal fields are captured. | Initialization extent, allocation, memory mutations. |
| F-58cc241cfb6b | Receives a derived object address, working word, and two metadata words. | Computation, result meaning, preservation guarantees. |
| F-3ed23f1b4691 | Receives temporary storage and the preceding result; supplies an argument for node handling. | Destination semantics, mutations, failures. |
| F-65514a6f4064 | Receives temporary storage, the removed node, and the preceding result after detachment and decrement. | Node ownership, slot mutation, resource handling, additional register output. |
| F-f673c27f3127 | Receives addresses of the second temporary field and object field +0x18. | Exchange or other transformation, write footprint. |
| F-dfc8ff5105e8 | Receives the object after final field replacements. | Bookkeeping, finalization, resource handling, mutations. |
| F-e7559f7e821e | Receives temporary storage after F-dfc8ff5105e8 returns. | Cleanup, destruction, retained references, ownership. |

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes valid accesses and normal returns from dependencies unless otherwise stated.

### Matching comparison

If the initial dependency result equals the subsequently read +0x18 word, the function returns without further listed dependency calls or direct object writes.

Effects of the initial dependency remain permitted.

### Initially exhausted traversal

If preparation completes and the first cursor observation equals the captured boundary, no slot-head, metadata, or successor reads occur.

Final field operations and both finishing calls still occur.

### Empty selected slot

If a selected slot reads as zero, that visit performs no node-processing calls. The function rereads +0x1C and increments its current value by four.

Only if the cursor field remained unchanged does this necessarily advance from the selected slot address.

### One-node chain and a preserved empty slot

Suppose the selected slot initially contains one valid node and:

- Both pre-removal dependencies preserve that slot head.
- The node’s successor is zero **when removal reads it**. Preserving an initially zero successor through that read is a sufficient premise.
- The count read during removal is a valid positive value for the decrement.

The function writes zero into the slot, decrements the count once, and calls F-65514a6f4064 with that node.

The next head observation ends the inner loop only if it returns zero. A sufficient additional premise is that F-65514a6f4064 and any other intervening actor leave the slot empty through that observation.

Preserving only the slot and count before removal is insufficient to establish the zero write.

### Successor changed before removal

Suppose the selected slot continues to point to the same node and its successor initially equals zero. If either pre-removal dependency changes that successor to a valid nonzero node address, and that value remains until the successor read, removal writes the changed value into the slot.

If F-65514a6f4064 then preserves that slot through the next observation, processing continues with the changed successor. The initial zero does not establish an empty slot.

### Repopulation after a zero write

Suppose the one-node example’s premises hold, including a zero successor at the removal read, so removal actually writes zero.

If F-65514a6f4064 then places another valid nonzero head in the selected slot and it remains through the next observation, processing continues in that slot. The earlier zero write does not trigger cursor advancement by itself.

### Replacement before removal

Suppose metadata is read from one head, but a pre-removal dependency replaces the selected slot with another valid nonzero head.

Removal uses the later head and its successor at removal time. The metadata need not describe the removed node.

If the replacement is zero, the function offers no intervening protective check before dereferencing it.

### Updated working word

If the additional register output after F-65514a6f4064 differs from the initial working word, the next F-58cc241cfb6b invocation uses the updated word, including when processing a later slot.

## Uncertainties

The following remain **documented_only**:

- Request semantics and valid ranges.
- The second F-0b31936eccf4 invocation’s effective input.
- Complete object, temporary, and node layouts.
- Alignment, aliasing, and arithmetic preconditions.
- Exact overflow behavior and authoritative signedness rules.
- Dependency write footprints and preservation guarantees, including preservation of successor words.
- The additional register output’s meaning and calling convention.
- The operation delegated for object field +0x18.
- Allocation, ownership, destruction, and retained-reference rules.
- Failure propagation, concurrency, reentrancy, and cleanup.
- Conditions sufficient for termination and successful reconstruction.

The container-rebuild interpretation remains inferred. Element preservation, stable ordering, successful migration, and reclamation are not established guarantees.

## Evidence

The supplied static representation establishes the comparison branch, four-byte field accesses, preparation order, captured traversal boundary, equality-controlled traversal, metadata processing, later head reread, successor read, head replacement, decrement, node-handling call, working-word update, and final field-operation order.

It establishes two distinct mutation-sensitive observations: the successor is read after both pre-removal dependencies, and the next slot head is read after F-65514a6f4064. Neither an initially zero successor nor an earlier zero slot write alone establishes subsequent loop termination.

Dependency implementations, authoritative calling-convention information, allocation contracts, and runtime observations are not supplied. Their behavior remains unresolved rather than replaced with conventional container assumptions.

No runtime tests have been performed.
