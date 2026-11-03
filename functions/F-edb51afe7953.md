<!-- Trusted provenance: F-edb51afe7953; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-edb51afe7953

## Purpose

F-edb51afe7953 compares a selected address with a container-held sentinel address and publishes one 32-bit result word through caller-provided storage.

If the addresses compare equal, it reads the sentinel field again and publishes that value. Otherwise, it retains a selected entry’s first word, performs two link-related writes, decrements a container counter, invokes an indirectly resolved operation, and prepends the entry selected after that invocation to a container-local chain. It then publishes the retained word.

“Entry,” “sentinel,” “unlinking,” and “chain” describe the apparent roles of the memory accesses. Conventional linked-list removal is an inferred interpretation, not an established representation invariant. The indirect operation is not established to be a destructor, and the final chain is not established to be a free list.

A central behavioral constraint is that the selection slot is read repeatedly before the indirect invocation. The address used for one access need not equal the address used for another if an earlier write changes that slot through aliasing.

## Interface

### Arguments and result

| Argument | Observed role |
|---|---|
| Output address | Destination for one final 32-bit write. |
| Container address | Base address for fields at byte offsets 4, 16, and 20. |
| Selection-slot address | Address of a 32-bit slot whose contents are interpreted as an entry address. |

No scalar return value is exposed. Observable results consist of memory accesses, memory changes, the indirect invocation, and the final output write when execution reaches it.

Stored addresses, link values, the counter, and the published result occupy 32 bits in the supplied representation. This contract does not generalize them to native-width pointers on a wider-address platform.

All offsets are byte offsets.

### Container fields

| Offset | Width | Use |
|---|---:|---|
| 4 | 32 bits | Chain head read and replaced after the indirect invocation. |
| 16 | 32 bits | Sentinel address used for comparison and, on the equal branch, read again for the result. |
| 20 | 32 bits | Signed counter read, decremented, and written on the unequal branch. |

No other container fields are directly accessed.

### Entry fields

| Offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Retained result, link-related source or destination base, and eventual chain-link destination. |
| 4 | 32 bits | Link-related destination address or source word. |
| 8 | 32 bits | Address of a record whose first word supplies an indirect call target. |

These are access requirements, not a complete entry-size declaration.

### Preconditions for normal completion

The caller must provide storage valid for every access actually reached:

- The output address must permit a 32-bit write.
- The selection slot must permit each repeated 32-bit read.
- Container fields must permit their path-specific reads and writes.
- On the unequal branch, every address obtained for a link-related access must permit that access.
- The indirect target record and target must permit resolution and invocation.
- After the indirect invocation returns, the then-selected address must permit an offset-0 write.

These requirements apply to values observed when each access occurs, not merely to values present at entry. Aliasing or indirect-call side effects may change which addresses require validity.

No visible check enforces non-null addresses, membership, link consistency, a positive counter, a valid target, or a live post-call entry.

## Behavior

### Branch selection

The function reads the selection slot and the container’s offset-16 field and compares the resulting addresses.

Equality is address equality. Two distinct addresses with identical entry contents take the unequal branch.

The supplied representation does not establish an additional observable order between the two operand reads within this comparison. It does establish that branch selection precedes the selected branch’s subsequent accesses.

### Equal branch: sentinel result

If the compared addresses are equal:

1. The container’s offset-16 field is read again.
2. That second read supplies the result word.
3. The result is written to the output address.

There are no direct entry-field accesses, link-related writes, counter updates, indirect invocations, or chain-head updates on this branch.

The result is not necessarily the sentinel value observed during comparison. With stable ordinary memory, the two sentinel reads agree. No separate stability guarantee is established.

The output write still occurs. If output storage overlaps a container field or the selection slot, it can modify that storage even on this branch.

### Unequal branch: retain the result

The function reads the selection slot again and reads the first 32-bit word at the resulting address. It retains that word until final publication.

Under stable, conventional linked-list conditions, this is the initially selected entry’s successor address. Without stability assumptions, it is precisely the first word reached by this result-capture read; the earlier comparison does not pin the selection for later accesses.

Later selection changes and link updates do not cause the retained result to be recomputed.

### First link-related write

The function obtains:

- A destination address from offset 4 of an entry reached through a read of the selection slot.
- A source word from offset 0 of an entry reached through another read of that slot.

It writes the source word to offset 0 at the destination address.

With a stable selection and conventional links, this redirects the predecessor’s forward link to the successor. The source word is read again; it is not expressly reused from the retained result.

The selection reads used for the destination and source are distinct. Their relative evaluation order is not additionally established here. They must not be treated as one permanently captured entry address.

### Second link-related write

After the first write, the function obtains:

- A destination base from offset 0 of an entry reached through a read of the selection slot.
- A source word from offset 4 of an entry reached through another read of that slot.

It writes the source word to offset 4 at the destination base.

With stable selection and conventional links, this redirects the successor’s backward link to the predecessor. Its source and destination computations again involve separate selection-slot reads; no additional relative ordering between those computations is established.

The first write precedes the reads and write of this phase. If it changes the selection slot, an entry field, or another subsequently consulted location through aliasing, this phase uses the changed state. Consequently, the entry that supplied the retained result is not unconditionally the entry supplying either or both unlinking operations.

### Counter decrement

After both link-related writes, the function reads the signed 32-bit counter at container offset 20, subtracts one, and writes the result back.

The relevant counter value is the value present when this read occurs. A value present at function entry is insufficient if either earlier write can overwrite the counter.

For values whose decrement is representable as a signed 32-bit integer, the stored numeric result is exactly one less. In particular:

- A read value of 3 becomes 2.
- A read value of 0 becomes −1, represented by the all-ones 32-bit word.
- No check rejects zero or a negative value.

Subtracting one from the minimum signed 32-bit value exceeds the represented signed range. A particular wraparound result at that boundary is not established by the supplied source representation and remains documented_only.

### Indirect invocation

After the counter write, the function resolves an indirect target by:

1. Reading the selection slot for this phase.
2. Reading the address at offset 8 of the entry thereby selected.
3. Reading the first word at that address as the call target.

It invokes that target with no explicit arguments visible in the supplied representation. No return value is visibly consumed.

The target can therefore be associated with a different entry from the one used during result capture or either link-related operation if prior writes changed the selection slot.

The target’s full calling convention and side effects are unknown. The function provides no established guarantee that the target preserves the selection slot, container fields, entry storage, globals, or object lifetimes.

If the invocation does not return normally, the subsequent chain insertion and output write are not reached. There is no visible rollback of preceding writes.

### Post-call chain insertion

After a normal return from the indirect invocation, the function reads the selection slot again and retains that address for the insertion.

It then performs these ordered operations:

1. Read the current container chain head at offset 4.
2. Write that head word to offset 0 of the retained post-call entry.
3. Write the retained entry address to container offset 4.

The insertion uses the head read after the indirect invocation. It does not restore a pre-call head.

Both writes use the same retained post-call entry address. Even if the entry’s offset-0 write aliases and changes the selection slot, the subsequent container-head write still uses the retained address.

There is no second sentinel comparison. If the post-call selection names the sentinel, insertion proceeds with that address, provided the resulting accesses are valid.

The function does not directly update the post-call entry’s offset-4 or offset-8 fields during insertion.

### Final publication

Finally, the function writes the retained result word to the output address.

On the unequal branch, this is the word captured before the link-related writes, counter decrement, and indirect invocation. The output write follows both chain-insertion writes.

If output storage overlaps a previously modified location, this final write can replace the value established there. It does not alter the already retained result itself.

## Side effects

### Direct effects

With ordinary valid storage and normal completion, the unequal branch performs:

- Two ordered link-related 32-bit writes.
- One 32-bit counter write.
- One indirect invocation.
- One 32-bit write to the post-call selected entry’s first word.
- One 32-bit container chain-head write.
- One final 32-bit output write.

Aliasing may cause multiple writes to affect the same physical storage. These operations are not an assertion that seven distinct locations are modified.

The function has no designated direct write to the selection slot. Nevertheless, any destination may overlap it, and the indirect target may change it.

### Ownership and lifetime

No ownership transfer or memory-release contract is established. No explicit allocation or deallocation operation appears outside the indirect invocation.

That absence does not constrain the indirect target. It may allocate, release, or otherwise invalidate storage.

For normal post-call insertion, the reloaded entry must be writable. If the target invalidates the selected entry without arranging a valid replacement selection, a later write may access invalid storage. Conversely, the original entry need not remain selected: the selection is expressly reloaded.

### Errors, interruption, and concurrency

No structured error status, validation failure path, or rollback mechanism is visible.

An invalid access or a nonreturning indirect invocation can leave only a prefix of the mutations completed, with no final output publication. Platform-specific fault behavior is not established.

No locking, atomic access protocol, or synchronization mechanism is visible. Concurrent access is not established as supported. Repeated reads and alias-sensitive behavior do not themselves constitute a concurrency guarantee.

## Dependencies

### F-indirect-unknown

This opaque documentation-only identifier denotes the dynamically resolved target. Its stable identity is unavailable.

Established properties:

- Reached only after an unequal initial comparison.
- Resolved through the selected entry’s offset-8 field and the addressed record’s first word.
- Invoked after both link-related writes and the counter decrement.
- Invoked before the post-call selection reload and insertion.
- Receives no explicit arguments visible in the supplied representation.
- Has no visibly consumed return value.

Its identity, calling convention, implicit inputs, ownership effects, global-state effects, error behavior, and normal-return guarantees remain documented_only.

No other function dependency is visible.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, assume valid ordinary storage, no concurrent mutation, representable arithmetic, and no unintended overlap among output, selection, container fields, entries, or indirect-target storage.

### Stable sentinel selection

Assume the selection equals the sentinel address and the sentinel field remains stable through both reads.

Expected:

- Output receives the sentinel address.
- No link-related write, counter decrement, indirect invocation, or chain insertion occurs.
- Separate container and selection storage remain unchanged.

### Ordinary removal and insertion

Assume a stable selected entry lies between a predecessor and successor, the counter reads as 3 during decrement, and the indirect target returns without modifying relevant state.

Expected:

- The predecessor’s forward link becomes the successor address.
- The successor’s backward link becomes the predecessor address.
- The counter becomes 2.
- The indirect invocation occurs after those writes.
- The selected entry’s first word receives the current chain head.
- The container chain head becomes that entry’s address.
- Output receives the successor address captured before mutation.

### Indirect target changes the selection

Assume all pre-call writes leave the selection slot and subsequent unlinking inputs unchanged, including through aliasing. The initial selection therefore supplies the retained result and both conventional unlinking operations. Assume the indirect target then changes the slot to a different valid writable entry and returns.

Expected:

- The original entry is unlinked under the conventional interpretation.
- The replacement entry receives the post-call chain-head word.
- The container chain head becomes the replacement entry’s address.
- Output receives the result retained before the call.

Without the pre-call stability assumption, the original entry is not guaranteed to supply all unlinking inputs.

### A link write changes the selection before the call

For this case, permit intentional overlap: the selection slot is also the first link-related write’s destination. Assume it initially contains the address of a first entry, that entry’s offset-4 field identifies the slot, and its offset-0 word contains a second entry’s address. Assume all resulting accesses are valid and no other mutation interferes.

Expected:

- Result capture retains the second entry’s address.
- The first link-related write changes the selection slot to that address.
- The second link-related operation reads the now-selected second entry rather than relying on the first entry’s saved identity.
- Absent another intervening change, indirect-target resolution also uses the second entry.

This behavior does not require any side effect from the indirect target.

### Counter is zero when decremented

Assume the counter is still zero when the decrement reads it, after both link-related writes. The earlier writes either do not overlap the counter or otherwise leave it zero.

Expected:

- The counter receives the 32-bit representation of −1.
- Zero does not prevent the indirect invocation or subsequent insertion.
- Processing continues unless another failure intervenes.

An initially zero counter alone is insufficient to establish this outcome.

### Indirect target changes the head

Assume the target writes a different valid chain-head value and returns normally.

Expected:

- Insertion reads and stores the changed head in the post-call entry.
- The container head then becomes the post-call entry’s address.

### Indirect invocation does not return

Assume execution reaches the invocation, which does not return normally.

Expected:

- Both link-related writes and the counter write have already occurred.
- Post-call insertion and final output publication are not reached.
- No rollback is established.

## Uncertainties

The following remain documented_only:

- Concrete types, complete layouts, and externally enforced invariants.
- Whether conventional linked-list terminology describes every supported representation.
- Whether the final chain serves reuse, reclamation, or another purpose.
- The identity and full contract of F-indirect-unknown.
- Ownership and lifetime rules across the invocation.
- Which aliasing arrangements callers are intended to permit.
- Concurrent-access and synchronization requirements.
- Signed-overflow behavior at the counter’s minimum value.
- Platform-specific alignment, address-arithmetic limits, and fault behavior.
- Any finer operand-evaluation ordering not established by the supplied representation.

These unknowns do not erase the established repeated selection reads, mutation order, post-call reload, or final publication order.

## Evidence

This contract is based solely on static examination of the supplied function body.

Established evidence includes the 32-bit accesses, byte offsets, address comparison, repeated selection-slot reads, retained pre-mutation result, two ordered link-related writes, signed counter decrement, indirect-target lookup, post-call address capture, ordered insertion writes, and final output write.

Conventional removal and reuse terminology expresses inferred intent only. No assumption that the indirect target preserves state or releases memory is established.

No runtime tests, implementation execution, dependency inspection, or platform verification have been performed.
