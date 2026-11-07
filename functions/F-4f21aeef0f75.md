<!-- Trusted provenance: F-4f21aeef0f75; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4f21aeef0f75

## Purpose

F-4f21aeef0f75 updates a record’s state byte, performs prerequisite lookups and notifications, and, if both lookups succeed, synchronizes a derived 32-bit mask with a nested object.

The operation is not transactional. The initial state-byte write precedes every dependency call and is not rolled back when a lookup returns zero.

Two separate reads of the primary-table global establish two roles:

- The first captures the table base used for a later notification’s offset-zero argument.
- The second supplies the base used to derive the selected record address for state, discriminator, and offset-12 accesses.

No dependency call occurs between those global reads. The evidence neither establishes that they differ nor guarantees global immutability. Their roles must therefore remain distinct.

With unchanged memory during the initial state reads, zero becomes one and every nonzero byte becomes zero. Later mask synchronization uses a fresh state-byte capture, not necessarily the value initially written or passed to the notification.

An option-toggle-and-propagate interpretation is inferred intent. The domain meanings of the records, constants, masks, and marker are not established.

## Interface

### Inputs and result

The function receives:

1. An object reference.
2. A signed 32-bit record index.

It has no explicit return value. Lookup failure produces a normal void return after the initial state write; there is no returned success or failure status.

The supplied representation uses 32-bit addresses, integers, and relevant dependency results. This contract describes that layout, not a portable interface for other pointer widths.

### Record addressing and layout

Primary and secondary records use a 20-byte stride.

For the primary table, the first captured base identifies the later notification record through the supplied index. The independently read second base identifies the selected record through the same index.

| Record offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Notification argument, read through the first captured primary-table base |
| 4 | 32 bits | Discriminator comparison, read through the selected record address |
| 12 | 32 bits | Argument to F-270c5b6d8d34, read through the selected record address |
| 16 | 8 bits | State reads and initial write through the selected record address |

If the two primary-table reads return the same base, the notification argument comes from the selected record. That equality is conditional, not a guaranteed invariant.

The secondary table also uses an 8-bit state field at offset 16. The evidence does not establish whether either table overlaps the other or any object storage.

### Object layouts

The input object exposes:

| Object offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Virtual-dispatch-table address |
| 24 | 32 bits | Index used for secondary-table addressing |
| 888 | 32 bits | Reference captured for final mask synchronization |

The object reached through offset 888 contains a reference at offset 168. The next object contains a reference at offset 3840. The final referenced object contains:

| Final-object offset | Width | Established use |
|---|---:|---|
| 2948 | 32 bits | Stored mask, compared and conditionally replaced |
| 2960 | 8 bits | Marker, conditionally written to one |

These offsets establish access patterns, not public type definitions or semantic ownership.

### Preconditions and arithmetic limits

No explicit bounds, null-reference, type, alignment, or lifetime checks protect the direct accesses.

All accessed storage must be valid when used, including after dependency calls. In particular:

- Both primary-table captures must support their respective indexed accesses.
- The secondary-table index must identify writable storage if its branch executes.
- Virtual receivers, dispatch tables, and nested references must remain usable at their access times.

Negative indices are not rejected. Table lengths and valid index ranges are unknown.

Index multiplication uses the supplied 32-bit representation and a stride of 20. The static evidence does not resolve signed-overflow or invalid-address behavior. This contract does not assume safe mathematical addressing or define wraparound as an accepted out-of-range behavior.

## Behavior

### 1. Capture both primary-table bases

The function first reads the primary-table global and retains that value for the later offset-zero notification access.

It then reads the same global again and uses this second value, together with the record index and 20-byte stride, to derive the selected record address.

The selected address is retained for subsequent state, discriminator, and offset-12 accesses. The first base remains the source for notification addressing. Neither is replaced by a fresh primary-table-global read later in the function.

No dependency call separates these captures. The function supplies no synchronization or immutability guarantee for the global.

### 2. Compute and write the initial state

The function performs two distinct unsigned-byte reads at offset 16 of the selected record:

1. The first supplies the byte subtracted from one to form a 32-bit computed value.
2. The second determines whether that computed value is replaced with zero: replacement occurs when this second byte is greater than one.

The function then writes the low eight bits of the resulting computed value to the selected state field. It retains the full 32-bit computed value for a later notification and the possible secondary-state write.

If both reads observe the same byte, the established results are:

| Observed initial byte | Retained computed value | Written byte |
|---:|---:|---:|
| 0 | 1 | 1 |
| 1 | 0 | 0 |
| 2–255 | 0 | 0 |

This table assumes agreement between the two reads. The representation does not collapse them into one read. If they observe different values, the subtraction and threshold decision use their respective observations.

The subtraction uses a zero-extended byte and 32-bit unsigned arithmetic. Before the threshold replacement, a first observation greater than one therefore produces a wrapped 32-bit result. If the second observation is at most one, that result is retained: the notification receives its full 32-bit representation, whereas state writes use only its low byte. Whether such differing observations can occur in a defined execution is not established.

The initial byte write occurs before any dependency call. Subsequent lookup failure does not undo it.

### 3. Resolve prerequisites

The function calls F-938c950e5046 without explicit arguments.

Its result is used as a virtual-dispatch receiver without an explicit null check. The virtual operation at dispatch-table byte offset 248 receives that receiver and one configured argument. This operation is labeled F-U01 below.

If F-U01 returns zero, the function returns immediately.

Otherwise, the function calls F-682adfd5a561 with:

- The nonzero result of F-U01.
- The fixed value 0x1700, equivalent to 5888 decimal.

If F-682adfd5a561 returns zero, the function returns immediately.

Neither zero-result path reaches the subsequent notifications, conditional secondary-state propagation, or mask synchronization. The initial state write remains performed, subject to any later mutations by dependencies.

### 4. Perform successful-path notifications

After both prerequisite results are nonzero, calls occur in this order:

1. F-U02 receives a 32-bit record field and the retained 32-bit computed state value.
2. F-U03, the input object’s virtual operation at dispatch-table byte offset 88, receives the input object and two configured arguments in their source-established order.
3. F-92bea7c30bd6 receives the successful result of F-682adfd5a561.

For F-U02, the 32-bit field is read at offset zero of the record addressed using the **first** primary-table capture and the supplied index. That field is read at this stage, after the prerequisite calls; its contents were not captured at entry.

Consequently:

- Earlier dependencies can affect the field’s value.
- The address still derives from the first captured base, even if the primary-table global has subsequently changed.
- The field does not necessarily belong to the selected record derived from the second capture.

F-U03’s receiver dispatch is resolved at this later stage. Earlier dependencies are not assumed to preserve the input object’s dispatch table or configured arguments.

The results of these three calls are unused. No ownership or cleanup semantics are inferred from their ordering.

### 5. Conditionally propagate the retained state

After the notifications, the function reads the selected record’s 32-bit field at offset 4 and compares it with a configured discriminator.

If they differ, it performs neither the secondary-state write nor the F-b002ce492389 call.

If they match, it:

1. Uses the input object’s 32-bit field at offset 24 as an index into the configured secondary table.
2. Writes the low eight bits of the retained computed state value at offset 16 of that secondary record.
3. Reads a 32-bit value through a configured reference and passes it to F-b002ce492389.

The secondary write precedes F-b002ce492389. No stronger relative ordering among the individual address-expression reads is established here.

The discriminator comparison occurs after the notifications, so their mutations may affect branch selection.

The propagated state is retained from the initial computation. It is not reread from the selected primary record. If the secondary destination aliases the selected state field, this write can overwrite an intervening state change.

Both branch outcomes continue to mask synchronization.

### 6. Capture mask inputs and compute the desired mask

The function captures the following values in order:

1. The selected record’s current state byte at offset 16.
2. Its current 32-bit field at offset 12.
3. The input object’s reference at offset 888.

It reads the captured referenced object’s field at offset 168 and passes that value to F-4fff691643db. The result is treated as a 32-bit starting mask.

It then passes the captured offset-12 record value to F-270c5b6d8d34. That result is treated as a 32-bit contribution mask.

The desired mask is determined by the captured state byte:

- Zero: retain starting-mask bits only where the contribution mask has zero bits.
- Nonzero: combine all bits set in either mask.

Complement, intersection, and union operate across all 32 bits. The contribution need not contain only one bit. Any nonzero captured state byte, including 2 or 255, selects union.

The state byte and offset-12 argument are captured before either mask-producing call. Those calls do not cause these inputs to be reread.

### 7. Compare, store, and mark

After both mask-producing calls, the function reuses the reference previously captured from input-object offset 888. It does not reread that input-object field.

Through the retained reference, it rereads offset 168. Thus, this intermediate reference may differ from the value supplied earlier to F-4fff691643db.

The function then follows offset 3840 of the resulting intermediate object and compares the final object’s stored 32-bit mask at offset 2948 with the desired mask.

If equal, it returns without writing the mask or marker. It does not clear, normalize, or otherwise update an existing marker.

If unequal, it:

1. Writes the desired 32-bit mask at offset 2948.
2. Rereads the offset-3840 reference from the same intermediate object.
3. Writes the byte value one at offset 2960 of the object reached by that reread.

The mask destination is not retained as the guaranteed marker destination. Aliasing or an otherwise possible intervening change to the reference-bearing storage can make those destinations differ. Whether concurrent changes constitute defined execution is unresolved.

No further explicit call or write follows the marker write.

## Side effects

The function’s explicit direct writes are:

- One initial selected-record state-byte write.
- One conditional secondary-record state-byte write.
- One conditional 32-bit stored-mask write.
- One subsequent marker-byte write of one.

These are logical destinations; non-aliasing is not established.

Dependency calls may produce additional side effects, mutate globals or reachable objects, or transfer control nonlocally. Their preservation and failure contracts are unknown.

There is no explicit allocation, deallocation, ownership transfer, retry, rollback, lock, or memory barrier in the supplied body.

Normal lookup failure permits partial completion. Faults or nonlocal exits after later writes have no specified recovery behavior.

References passed to dependencies must not be assumed to be retained, released, finalized, or preserved. In particular, F-92bea7c30bd6’s placement does not establish that it releases the successful lookup result.

## Dependencies

F-U01, F-U02, and F-U03 are document-local opaque dependency labels, not supplied mapped identities.

| Dependency | Established interaction |
|---|---|
| F-938c950e5046 | No explicit arguments; result supplies a virtual receiver. |
| F-U01 | Virtual operation at byte offset 248; receives that receiver and one configured argument; zero causes return. |
| F-682adfd5a561 | Receives F-U01’s nonzero result and 0x1700; zero causes return. |
| F-U02 | Unmapped notification; receives the offset-zero field addressed through the first primary-table capture and the retained computed state value. |
| F-U03 | Input-object virtual operation at byte offset 88; receives the receiver and two configured arguments. |
| F-92bea7c30bd6 | Receives F-682adfd5a561’s successful result. |
| F-b002ce492389 | Called after a secondary-state write on discriminator equality; receives a 32-bit value read through a configured reference. |
| F-4fff691643db | Receives the first offset-168 value read through the captured offset-888 reference; returns the starting mask. |
| F-270c5b6d8d34 | Receives the captured selected-record offset-12 field; returns the contribution mask. |

No callee preservation, ownership, exception, reentrancy, or thread-safety guarantees are established.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume valid storage, non-overflowing addressing, agreeing initial state reads, and no relevant dependency mutations.

### Initial update survives lookup failure

With an initial selected state of zero and a zero result from F-U01:

- The selected state is written to one.
- F-682adfd5a561 is not called.
- Successful-path notifications and mask synchronization do not occur.
- No status value is returned.

If F-U01 succeeds but F-682adfd5a561 returns zero, the same partial-update rule applies.

### Separate primary-table captures retain separate roles

If both initial global reads return the same base, F-U02’s offset-zero argument comes from the same indexed record used for state and mask inputs.

Conditionally, if valid execution permits the first read to return one base and the second another:

- State, discriminator, and offset-12 accesses use the second base’s selected record.
- F-U02’s offset-zero argument uses the first base’s indexed record.
- Neither role is redirected by a later change to the primary-table global.

This example does not assert that differing captures occur.

### Nonzero initial state clears contributed bits

With both initial state reads observing seven, the retained computed value and written byte are zero.

If the later state capture remains zero, a starting mask of 0x0000000F and contribution mask of 0x00000005 produce 0x0000000A.

### Enabling sets all contributed bits

With an initial state of zero that remains one after the update, a starting mask of 0x00000002 and contribution mask of 0x00000005 produce 0x00000007.

If the stored mask differs, it is replaced with 0x00000007, followed by a marker write of one through the reread final-object reference.

### Equal masks leave the marker untouched

If the desired mask already equals the stored mask, neither final write occurs. This preserves any existing marker byte, not only zero or one.

### Notification state and synchronization state may differ

With initial state zero, the retained notification value is one.

If a dependency later changes the selected state to zero, and no secondary aliasing write restores it before capture, F-U02 still receives one while final mask computation follows the clearing branch.

### Secondary propagation uses the retained value

On discriminator equality, the secondary state write precedes F-b002ce492389 and uses the retained computed value’s low byte.

On inequality, neither action occurs. Mask synchronization proceeds in either case.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of tables, fields, constants, discriminator values, and masks.
- Table sizes, valid indices, alignment requirements, and overflow behavior.
- Whether the two initial primary-table reads can differ in a defined execution.
- Whether the two initial state-byte reads can differ in a defined execution.
- Global immutability, aliasing relationships, and storage lifetimes.
- Callee mutation, preservation, ownership, and failure contracts.
- Mapped identities of F-U01, F-U02, and F-U03.
- Concurrency, reentrancy, visibility, and fault behavior.
- Whether the marker is consumed as a change indicator.

These unknowns do not imply validation, synchronization, ownership transfer, recovery, or immutability guarantees.

## Evidence

Established behavior derives solely from the supplied static representation and dependency mapping: two separate primary-table-global reads, their distinct subsequent uses, access widths and offsets, explicit call order, conditional returns, retained values, pointer rereads, and 32-bit mask operations.

The option-toggle interpretation and change-indicator interpretation are inferred intent. Dependency internals, actual runtime identities, valid allocation boundaries, and concurrent execution semantics are not established.

No runtime tests have been performed.
