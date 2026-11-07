<!-- Trusted provenance: F-66a77f4c4cbe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-66a77f4c4cbe

## Purpose

F-66a77f4c4cbe prepares a record associated with an input object, checks whether that record is already active, and conditionally inserts an inactive record into a globally rooted singly linked list. After processing an inactive record, it performs a separate global-state check that may invoke an indirect callback.

The established behavior comprises preparation calls, an unconditional preparation-field update, an early return for an active record, optional arithmetic and list insertion, and conditional dispatch. Dispatch does not require insertion.

A scheduling-related purpose is inferred from the interval-like field, multiplication by 1000, a result-producing dependency, and comparison-controlled insertion. Neither time units nor scheduling semantics are established.

## Interface

### Arguments and result

The visible interface contains four argument positions:

| Position | Width | Established use |
|---|---|---|
| First | 32 bits | Address-like value identifying an object with a record-pointer field at byte offset 180. |
| Second | 32 bits | Opaque value with no explicit use in this function. |
| Third | 32 bits | Forwarded to each traversal comparison. |
| Fourth | 32 bits | Forwarded to each traversal comparison. |

No explicit return value or status result is provided.

Each comparison also receives an incoming register-carried 32-bit word. Its origin, initialization requirements, and status under the external calling convention are unresolved. It is not established as a fifth public argument.

### Object and record layout

The object’s field at byte offset 180 contains a 32-bit record pointer. The function reads this field three times during preparation. Dependencies occur between those reads, so the three values need not identify the same record.

The third value selects the record retained for subsequent direct accesses:

| Record byte offset | Width | Direct role |
|---|---|---|
| 0 | 32 bits | Next-record pointer. |
| 4 | 32 bits | Interval-like arithmetic operand; zero skips insertion. |
| 8 | 32 bits | Computed deadline-like value; its address is passed to the comparison dependency. |
| 24 | 32 bits | Value copied from a global source on the insertion path. |
| 28 | 32 bits | Active marker; initially tested for zero, later set to 1 during insertion. |
| 32 | 32 bits | Value copied from a global source during preparation. |

The function does not directly access bytes 12 through 23. Dependency access to those bytes is unknown.

For its own direct accesses, the function requires the retained record to provide storage through byte offset 35. Traversed records require a readable next-pointer field and a valid comparison operand at offset 8. The comparison dependency’s access extent is not established.

### Global-state roles

The following labels distinguish global locations without assigning application semantics:

- **Preparation-value global:** supplies the word written at record offset 32.
- **Shared-state anchor global:** supplies the argument to F-338d16f9ce72. Later, the same global location is reread to obtain a pointer to the slot containing the final dispatch-gating pointer.
- **List-anchor global:** supplies a pointer to the list-head slot.
- **Insertion-value global:** supplies the word written at record offset 24.
- **Final-call argument global:** supplies the argument to F-5bcae2eff9a7.
- **Callback-target global:** contains the indirect function pointer loaded only when both dispatch-gating tests succeed.

The shared-state anchor’s two uses are established uses of the same global location, not merely similarly shaped accesses. Whether other global locations alias storage is unresolved.

### Preconditions

All memory accessed on the selected path must remain valid. There are no explicit validity checks for the input object, its selected record, the list-head slot, or the slot addressed through the shared-state anchor.

In particular, the final null check applies to the pointer obtained from the shared-state slot. It does not validate the anchor value before that slot is dereferenced.

Normal traversal requires eventual arrival at a null link or a nonzero comparison result. A cycle with comparisons that continue returning zero has no visible termination mechanism.

If dispatch is selected, the value loaded from the callback-target global must designate a callable target under the actual calling convention. The function does not explicitly validate that target.

## Behavior

### Preparation and early return

Preparation has the following observable order:

1. Read the object’s record pointer at offset 180 and pass that value to F-936a8728139d.
2. Read the same object field again and pass its current value, together with 4000, to F-6904f7699031.
3. Read the object field a third time and retain that record pointer.
4. Invoke F-936a8728139d again.
5. Read the preparation-value global and write its 32-bit value to retained-record offset 32.
6. Read retained-record offset 28. Return immediately if it is nonzero.

The second F-936a8728139d occurrence has no explicit argument expression in the supplied representation. Its effective argument state is unresolved; it must not be specified as receiving the retained record.

The retained record does not change merely because a dependency later changes the object’s pointer field. The function performs no subsequent explicit reload of that field to select another record.

The active-marker early return occurs only after all preparation calls and the offset-32 write. Any nonzero marker qualifies, not just 1. This return bypasses all remaining dependencies, insertion processing, and dispatch-state reads.

### Processing an inactive record

When the active marker is zero:

1. Read the shared-state anchor global and pass its current value to F-338d16f9ce72.
2. Pass the retained record to F-add44d4e9aaa.
3. Read the retained record’s 32-bit field at offset 4.

Both dependencies precede the interval read. Their effects can therefore change the value used to decide whether insertion occurs.

A zero interval skips the result-producing dependency, deadline calculation, offset-24 write, traversal, and insertion writes. Execution nevertheless continues to F-5bcae2eff9a7 and the final dispatch check.

There is no second active-marker check. A dependency’s change to offset 28 does not cause the visible body to reconsider the earlier decision to process the record.

### Arithmetic and insertion setup

For a nonzero interval, the function calls an unidentified dependency that supplies a 32-bit result. It then reads the list-anchor global and retains the resulting pointer to the list-head slot.

The function computes a value by multiplying the previously read interval by 1000 and adding the dependency’s result. It writes the result to retained-record offset 8.

The interval is not reread after the result-producing call. If that dependency changes offset 4, the already captured operand remains the operand used for this calculation.

The supplied representation expresses the multiplication and addition using signed 32-bit integer values. No explicit widening, range check, saturation, overflow notification, or alternative branch is present. Exact behavior outside the signed 32-bit representable range is documented_only; neither portable signed overflow nor modulo wrapping is asserted as the contract.

After writing offset 8, the function reads the insertion-value global and writes it to record offset 24. It then reads the list head through the previously retained head-slot pointer.

Thus the list-anchor global is read before the offset-8 and offset-24 writes, while the head stored in the selected slot is read after those writes.

### Traversal and publication

Traversal begins at the captured head with no predecessor.

For every nonnull current record, F-0d70d6f9010c receives, in order:

1. The address of the current record’s field at byte offset 8.
2. The address of the retained record’s field at byte offset 8.
3. The third public argument.
4. The fourth public argument.
5. The unresolved incoming register-carried word.

A zero comparison result continues traversal. Only after that result does the function read the current record’s next pointer, advance to that pointer, and retain the previous current record as predecessor.

A nonzero result stops traversal before reading that current record’s next link for advancement. A null current pointer also stops traversal, without a comparison call for the null position.

Insertion places the retained record immediately before the stopping position. The direct writes occur in this order:

1. Publish the retained record pointer through the captured head slot if there is no predecessor, otherwise through the predecessor’s next field.
2. Write exactly 1 to retained-record offset 28.
3. Write the stopping-position pointer to retained-record offset 0.

The record is therefore published before its marker and next pointer receive their insertion values. No atomic-publication or concurrent-observer guarantee is established.

The comparison’s sign has no distinct treatment: every nonzero result stops traversal. Ascending order, descending order, equality handling, and wraparound-aware ordering depend on the unknown comparison contract.

No explicit duplicate-membership check occurs. If the retained record is already reachable despite a zero marker, the function still follows these operations; a well-formed resulting list is not guaranteed by the supplied evidence.

### Final call, shared-state reread, and callback

Both the insertion path and the zero-interval path reach this sequence:

1. Read the final-call argument global and pass its value to F-5bcae2eff9a7.
2. After that call returns, reread the **same shared-state anchor global** whose earlier value was passed to F-338d16f9ce72.
3. Dereference the anchor’s current value to obtain the pointer used to gate dispatch.
4. If the obtained pointer is nonnull, read its first 32-bit word.
5. Only if the pointer and its first word are both nonzero, read the callback-target global and invoke the function pointer obtained from it.

The final lookup does not reuse the earlier anchor value. Any intervening dependency may change the shared-state anchor global, and the final lookup uses the value present after F-5bcae2eff9a7 returns.

The pointer and first word are dispatch gates only. Neither is the callback target. The target comes from its separate global function-pointer location, and that location is not read on a path where either gate fails.

A null gating pointer prevents the first-word read as well as the callback-target read. A nonnull pointer with a zero first word prevents the target read and invocation.

The indirect invocation has no explicit arguments in the supplied representation, and no returned value is consumed. Its effective calling convention remains unresolved.

## Side effects

### Direct writes

Depending on the path, the function directly writes:

- Retained-record offset 32 during preparation.
- Retained-record offsets 8 and 24 for a nonzero interval.
- The captured head slot or a predecessor’s next field during insertion.
- Retained-record offset 28, setting it to 1.
- Retained-record offset 0, setting its next link.

On the already-active path, offset 32 is the only direct record write. On the zero-interval path, the function does not directly change the active marker or link fields.

These statements concern direct operations only. Dependencies may cause additional effects.

### Ownership, lifetime, and synchronization

No explicit allocation, deallocation, reference-count operation, or ownership transfer appears. Publishing a pointer into the list does not establish whether ownership transfers.

Dependency contracts do not establish preservation of records, globals, anchors, or allocation lifetimes. Continued use of a captured pointer is not proof that a dependency preserves its referent.

Locking, unlocking, exclusion, and reentrancy semantics are unknown. In particular, the calls surrounding processing cannot be relied upon as a synchronization pair merely from their placement.

### Failure behavior

There is no explicit error status, validation-failure branch, rollback, or recovery mechanism.

Invalid memory, an invalid indirect target, dependency failure, or nonterminating traversal can prevent normal completion. Failure reporting and partial-state recovery are not established. Once publication begins, the visible body does not undo an earlier insertion write if a later operation fails.

## Dependencies

| Dependency | Established interaction |
|---|---|
| F-936a8728139d | Called twice during preparation. Only the first occurrence has an explicit record-pointer argument. |
| F-6904f7699031 | Receives a freshly read record pointer and 4000. No result is consumed. |
| F-338d16f9ce72 | Receives the current value of the shared-state anchor global after the active check. That same global location is reread during final dispatch processing. |
| F-add44d4e9aaa | Receives the retained record before the interval is read. |
| F-0d70d6f9010c | Receives two field addresses and three additional words. Zero continues traversal; nonzero stops it. |
| F-5bcae2eff9a7 | Receives the final-call argument global’s value before the shared-state anchor is reread. |
| Unidentified result-producing dependency | Called only for a nonzero interval; supplies the additional 32-bit arithmetic operand. No opaque ID was supplied. |
| Unidentified indirect callback | Target is loaded from the callback-target global only after both dispatch gates succeed. No opaque ID was supplied. |

No dependency is assumed to preserve globals or own, retain, or release memory.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless a case states otherwise, dependencies return normally and preserve the memory and values needed for the stated outcome.

### Already-active record

If the retained record’s marker is 7 when tested, the function first completes preparation and writes offset 32. It then returns without inactive-record processing, list insertion, F-5bcae2eff9a7, or final dispatch reads.

### Zero interval

If the marker is zero and the interval read after F-add44d4e9aaa is zero, no deadline calculation or direct insertion occurs. F-5bcae2eff9a7 and final dispatch processing still occur.

### Empty-list insertion

If the interval is 3, the result-producing dependency returns 10000, and the captured head is null:

- Offset 8 becomes 13000.
- No comparison occurs.
- The head slot receives the retained record pointer.
- The marker becomes 1.
- The next pointer becomes null.

The three insertion writes retain that order. This arithmetic example remains within signed 32-bit range.

### Insertion after a prefix

If comparisons return zero for the first two records and nonzero for the third, the second record’s next field receives the retained record pointer. The retained record is then marked active, and its next field receives the third record’s pointer. The function does not advance through the third record’s next link.

### Shared-state anchor replacement

Suppose the shared-state anchor contains one slot address when its value is passed to F-338d16f9ce72. An intervening dependency changes that global to another valid slot address.

The final lookup, after F-5bcae2eff9a7, reads the replacement slot. Dispatch is gated by the pointer obtained from that slot and its first word, not by the earlier anchor value or the earlier slot’s contents.

### Conditional callback-target read

If the final slot contains null, neither the first-word read nor the callback-target read occurs.

If the slot contains a valid nonnull pointer whose first word is zero, that word is read, but the callback-target global is not read.

If both gates are nonzero, the function then reads the callback-target global and invokes its value. A zero interval does not prevent this dispatch. A nonzero gate does not establish that the target itself is valid.

## Uncertainties

The following remain documented_only:

- Application meanings and units of the interval, computed field, and constant 4000.
- Effective arguments for the second F-936a8728139d call.
- External calling-convention status of the comparison’s extra register-carried word.
- Contracts and identities of the result-producing dependency and indirect callback.
- Arithmetic behavior outside the signed 32-bit range.
- Comparison ordering, equality treatment, and access extent.
- Dependency effects on globals, records, ownership, and lifetimes.
- Synchronization and reentrancy guarantees.
- Aliasing beyond the established reuse of the shared-state anchor global.
- Resulting list integrity when the retained record is already reachable or dependencies mutate traversal state.

These unknowns do not obscure the established final-state relationship: the anchor global is reread after F-5bcae2eff9a7, and the separate callback target is loaded only after both gating tests succeed.

## Evidence

Established facts derive exclusively from the supplied static function representation: access widths and offsets, explicit call arguments, global-location identity, branch conditions, arithmetic expressions, traversal behavior, and read/write order.

“Record,” “active marker,” “interval,” “deadline,” and the global-role labels are explanatory terms. The scheduling interpretation is inferred intent, not an established interface promise.

Dependency implementations, authoritative calling-convention information, machine-level overflow behavior, and runtime observations were not supplied. No runtime tests have been performed.
