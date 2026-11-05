<!-- Trusted provenance: F-bcb8529decd8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bcb8529decd8

## Purpose

F-bcb8529decd8 conditionally grows a capacity value and appends one byte through caller-supplied storage references. Its directly established behavior consists of state-slot reads and writes, calls to opaque dependencies, and a final byte write when capacity exceeds logical length.

The apparent intent is to support a buffer that initially uses a primary storage reference and later uses auxiliary storage. Growth proceeds in increments of 1,024 capacity units. This buffer-management interpretation is inferred, not a guarantee of allocation, copying, preservation, or ownership transfer.

There is no return value or explicit success indicator. A normal return can follow either an append or a final capacity check that rejects the append. Dependency failure is not recognized through a dedicated local error path.

## Interface

### Inputs

| Input | Represented form | Use |
|---|---|---|
| Primary-reference slot | Pointer to a signed integer word | Holds a reference later usable as a destination address. Zero selects auxiliary storage on the append path. |
| Auxiliary-reference slot | Nullable pointer to a signed integer word | Holds another reference. A null slot pointer disables the growth loop. |
| Length slot | Pointer to an unsigned integer word | Holds the logical byte length. |
| Capacity slot | Pointer to an unsigned integer word | Holds the value compared against length and supplied to growth dependencies. |
| Appended value | One byte | Written at the final old-length offset if the append path is taken. |

A null auxiliary-slot pointer and a zero value stored in a valid auxiliary slot have different meanings. The former disables growth; the latter selects the initialization branch when growth is required.

### Widths and arithmetic

The supplied representation distinguishes signed integer reference words, unsigned length and capacity words, and a one-byte appended value. It does not independently establish the original ABI.

Under the apparent 32-bit integer model:

- Length and capacity range from 0 through 4,294,967,295.
- Capacity additions are unsigned and wrap modulo 4,294,967,296.
- Reference slots hold 32-bit integer representations.
- The final reference-plus-offset expression uses unsigned 32-bit arithmetic because the offset is unsigned.
- The byte write occupies exactly one byte.

Thus, under that model, a reference-plus-offset calculation can wrap before conversion to a destination address. Validity of the resulting address is a separate requirement. The original integer-to-pointer conversion rules and target fault behavior remain documented_only.

### Preconditions and observable results

For execution that completes without invalid accesses:

- The length and capacity slots must be readable whenever examined.
- Every slot written on the selected path must be writable.
- The primary slot must be readable on an append path and on an initialization path that invokes the migration-shaped dependency.
- The primary slot must be writable during auxiliary initialization, even if the length is zero.
- An enabled growth loop requires a valid auxiliary slot whenever its body executes.
- An append selecting auxiliary storage requires a readable auxiliary slot, even if its null pointer previously disabled growth.
- The selected destination address must permit the one-byte write.

Ordinary buffer use additionally requires sufficient accessible storage behind the selected reference. A numerical capacity value alone does not establish that storage exists.

Distinct, nonoverlapping state slots are the straightforward usage model. The function performs no alias checks. If slots overlap one another or the byte destination, later reads and final state can differ from ordinary buffer expectations.

There is no returned length, reference, status, or error code. Results are communicated through memory and dependency effects.

## Behavior

### Growth eligibility and loop condition

The growth loop is considered only when the auxiliary-slot pointer is non-null.

Each condition evaluation reads capacity into a retained snapshot and then reads length. Both are compared as unsigned values. An iteration begins when the capacity snapshot is less than or equal to the observed length.

Equality therefore requires growth processing. The loop seeks a comparison observing capacity strictly greater than length.

Each iteration next reads the auxiliary reference. A zero reference selects initialization; a nonzero reference selects enlargement.

### Initialization when the auxiliary reference is zero

If the iteration’s capacity snapshot is zero, the function first writes 1,024 to the capacity slot. Otherwise, this branch does not directly increase capacity.

It then reads capacity again and calls F-43da6b23bd44 with that value and two zero arguments. This fresh capacity read, rather than the earlier snapshot, supplies the request.

The returned integer reference is:

- Written to the auxiliary slot.
- Retained separately for a possible subsequent dependency call.

After storing the result, the function freshly reads length. If that read is nonzero, it calls F-unmapped-01 with the retained result and the primary reference read for that call. No length argument is visible.

The function then writes zero to the primary slot, whether or not F-unmapped-01 was called, and returns to the loop condition.

Consequences:

- A nonzero capacity is not enlarged by this branch. At length 1,024 and capacity 1,024, the initial request is for 1,024, assuming the state remains unchanged before that request.
- No local check rejects a zero initialization result.
- The primary reference is cleared even after a zero result, provided execution reaches the clearing operation.
- The decision to call F-unmapped-01 uses length after F-43da6b23bd44 returns, not the length observed by the loop condition.
- The first argument to F-unmapped-01 is the retained return value, not a reread of the auxiliary slot.
- Changes made by either dependency can affect subsequent loop conditions and branch selection. The function does not restore prior slot values.

### Enlargement when the auxiliary reference is nonzero

The function adds 1,024 to the iteration’s capacity snapshot using unsigned arithmetic and writes the result to the capacity slot.

It then reads the auxiliary reference and calls F-8b998a07b5e1 with:

1. That reference.
2. The calculated capacity.
3. Zero.
4. Zero.

The result overwrites the auxiliary slot, and execution returns to the loop condition.

The capacity write precedes the call. Its new value is therefore exposed before the dependency returns. The requested capacity comes from the retained iteration snapshot plus 1,024, not from a fresh capacity read after the write.

The reference supplied to the dependency is read after the capacity write. If those slots overlap, the supplied reference can differ from the nonzero value that selected this branch.

A zero result is stored without recovery. If growth remains necessary, a subsequent iteration can therefore select initialization rather than enlargement.

### Final capacity decision

After the loop terminates, or immediately when its auxiliary-slot pointer is null, the function freshly reads length into an old-length snapshot and then reads capacity.

If capacity is less than or equal to old length, it returns without a final length increment or destination-byte write. Any earlier growth-loop effects remain in place.

If capacity is greater than old length, it proceeds to append.

The final comparison is distinct from the loop comparison, but their relationship is constrained: when an enabled loop terminates normally, its last condition has observed capacity greater than length. No dependency call or direct state write occurs between that successful exit comparison and the final comparison. Consequently, absent concurrent or other intervening mutation, the final comparison also permits the append. Dependency-side mutations during earlier iterations do not invalidate this conclusion; the terminating loop comparison already observes the state after those calls.

When growth is disabled, there is no preceding successful loop-exit comparison. The final check alone decides whether to append.

### Reference selection and append ordering

Once the final comparison permits an append, the representation establishes this sequence:

1. Read the primary reference into a snapshot.
2. Write old length plus one to the length slot.
3. Select a destination reference using the primary snapshot:
   - If the snapshot is zero, read the auxiliary reference.
   - Otherwise, read the primary reference again.
4. Write the supplied byte at the selected reference plus old length.

The offset is the old length, not the incremented value.

With distinct slots and no intervening mutation, a nonzero primary reference takes precedence over auxiliary storage. A zero primary reference selects auxiliary storage.

The second primary read matters when slots overlap: the intervening length write can change the value actually used as the destination reference. Likewise, an overlapping destination-byte write can modify a state slot after the length increment.

The function updates length before attempting the byte write. There is no rollback operation if destination access fails. The host environment’s exact fault and post-fault visibility rules are not established.

## Side effects

### Direct state changes

Depending on the selected path, the function can:

- Set capacity to 1,024.
- Replace capacity with an iteration snapshot plus 1,024.
- Replace the auxiliary reference with a dependency result.
- Clear the primary reference.
- Set length to the final old length plus one.
- Store one byte at the selected destination offset.

It does not directly append a terminator, clear extra capacity, or explicitly release storage. Unknown dependencies may perform additional writes or lifetime operations.

### Ownership and preservation

No ownership transfer or release guarantee is established.

In particular:

- Clearing the primary slot does not prove that primary storage was freed.
- F-unmapped-01 is not established to copy any particular number of bytes.
- F-8b998a07b5e1 is not established to preserve contents or retain old storage on failure.
- Overwriting the auxiliary slot can discard the previously stored reference without a visible recovery path.
- A returned nonzero integer is not independently established to identify usable storage.
- No dependency is assumed to preserve state slots, globals, or memory unless an acceptance example explicitly imposes that condition.

### Errors and termination

There is no explicit null-result handling, overflow rejection, iteration bound, or local error return.

Under the 32-bit model, repeated capacity increments can wrap. Growth need not eventually reach a capacity greater than length.

For length 4,294,967,295, no unsigned 32-bit capacity can exceed it. If growth is disabled, the final check rejects the append. If growth is enabled and dependencies continue returning normally without changing length or otherwise enabling termination, the loop cannot exit through its capacity comparison.

Conversely, a permitted append cannot overflow the numerical length increment under that model: capacity greater than old length implies old length is at most 4,294,967,294. This does not prove that the destination address is valid or that the address addition avoids wraparound.

Invalid slots, unusable dependency results, and invalid destination addresses can cause invalid accesses. Their concrete runtime consequences are documented_only.

## Dependencies

### F-43da6b23bd44

Called in the zero-auxiliary initialization branch.

Its arguments are a fresh capacity read, zero, and zero. Its integer result is written to the auxiliary slot and retained for possible use by F-unmapped-01.

**Established:** argument relationships, call placement, result storage, and absence of a local failure check.

**Inferred intent:** obtain storage associated with the requested capacity.

**documented_only:** actual allocation behavior, alignment, initialization, ownership, failure semantics, and effects on accessible state.

### F-8b998a07b5e1

Called in the nonzero-auxiliary enlargement branch.

Its arguments are the auxiliary reference read after the capacity update, the unsigned snapshot-plus-1,024 result, zero, and zero. Its integer result replaces the auxiliary reference.

**Established:** capacity is written before the call; the requested capacity is derived from the iteration snapshot; the returned reference is unchecked.

**Inferred intent:** enlarge or replace existing auxiliary storage.

**documented_only:** movement, content preservation, disposition of old storage, failure semantics, and other state changes.

### F-unmapped-01

This contract-local opaque ID identifies the external dependency for which no mapped opaque ID was supplied.

It is called after initialization only when a fresh length read is nonzero. Its arguments are the retained initialization result and the then-read primary reference. No length argument is visible, and any return value is unused.

**Established:** conditional call placement and argument relationships.

**Inferred intent:** participate in migration from primary to auxiliary storage.

**documented_only:** operation performed, amount of memory accessed, ownership effects, and mutations of accessible state.

## Acceptance examples

These are conditional static reasoning cases, not executed tests.

Unless a case states otherwise, assume distinct, nonoverlapping slots; no concurrent or other intervening mutation; and dependencies that return normally without writing any of the four state slots. This explicitly includes preservation of primary reference, auxiliary reference, length, and capacity by every dependency involved. Assignments performed by F-bcb8529decd8 itself still occur.

A completed byte write additionally assumes that the selected destination remains writable after all dependency calls. These example assumptions do not establish general dependency guarantees.

### Existing primary storage with spare capacity

Given length 3, capacity 8, and a nonzero primary reference:

- No growth dependency is called, whether the auxiliary-slot pointer is null or valid.
- Length becomes 4.
- The supplied byte is written to primary storage at offset 3.
- Capacity and both stored references remain unchanged.

### Growth disabled and capacity exhausted

Given a null auxiliary-slot pointer, length 8, and capacity 8:

- Growth is skipped.
- The final comparison rejects the append.
- No primary-reference read, length increment, dependency call, or destination-byte write occurs.

### Growth disabled but auxiliary selection required

Given a null auxiliary-slot pointer, length 2, capacity 4, and a zero primary reference:

- The final capacity check permits the append.
- Length is updated to 3.
- Destination selection attempts to read the null auxiliary slot.

This case does not satisfy the access preconditions. Null disables growth; it does not make every append path safe.

### First initialization from zero capacity

Given a valid auxiliary slot containing zero, length zero, and capacity zero, suppose F-43da6b23bd44 returns usable nonzero storage and preserves all four slots during its call:

- Capacity is set to 1,024 before the call.
- The dependency receives 1,024, zero, and zero.
- Its result is stored in the auxiliary slot.
- F-unmapped-01 is skipped because length remains zero.
- The primary reference is cleared.
- The next loop condition observes capacity 1,024 and length zero and exits.
- The final comparison permits the append.
- Length becomes 1, and the byte is written at auxiliary offset zero.

If the dependency instead changes capacity or length, the next comparison must use those actual post-call values; this outcome no longer follows merely from a usable return value.

### Initialization at exact capacity

Given length 1,024, capacity 1,024, and a zero auxiliary reference, assume all invoked dependencies preserve all four state slots during their calls and return usable storage where required:

- F-43da6b23bd44 receives 1,024, zero, and zero.
- Its nonzero result is stored in the auxiliary slot.
- F-unmapped-01 receives that retained result and the current primary reference.
- The primary reference is cleared.
- The next iteration still observes length and capacity equal to 1,024 and a nonzero auxiliary reference.
- Capacity becomes 2,048, and F-8b998a07b5e1 receives that requested capacity.
- After a usable result, the loop exits and the append sets length to 1,025 and writes at offset 1,024.

Without preservation by both initialization-related dependencies, the 2,048 request is not guaranteed. The next iteration uses the actual capacity, length, and auxiliary reference then present.

### Zero initialization result

Given length zero, capacity zero, and a zero auxiliary reference, suppose F-43da6b23bd44 returns zero while preserving all four slots:

- Capacity becomes 1,024.
- The auxiliary reference is stored as zero.
- The primary reference is cleared.
- The loop exits and the final check permits the append.
- Length becomes 1 before a byte write is attempted using reference zero at offset zero.

No local allocation-failure report replaces this attempted write.

## Uncertainties

The following remain documented_only:

- Independently verified target ABI and integer widths.
- Exact integer-reference-to-address conversion and fault semantics.
- Dependency implementations and all state-preservation guarantees.
- Whether dependency results identify storage of the requested capacity.
- Preservation of existing bytes during initialization or enlargement.
- Ownership, lifetime, and release responsibilities.
- Caller guarantees about aliasing and overlapping storage.
- Behavior under concurrent mutation or unusual read-sensitive memory.

The static representation supports the stated control flow and data relationships. It does not establish machine-level atomicity, synchronization, or exact emitted load instructions.

## Evidence

Evidence consists solely of the supplied function representation and opaque dependency mapping.

The representation establishes the unsigned capacity comparisons; conditional 1,024 adjustment; 1,024-unit enlargement; separate initialization and enlargement branches; unchecked dependency results; conditional external call; primary-reference clearing; independent final capacity comparison; and length update before destination selection and byte storage.

It also establishes that no dependency call or direct state write separates a normal growth-loop exit comparison from the final comparison.

No dependency bodies, caller contracts, runtime traces, or executable verification were supplied. No runtime tests have been performed.
