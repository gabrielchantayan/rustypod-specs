<!-- Trusted provenance: F-11aa8eec50ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-11aa8eec50ad

## Purpose

F-11aa8eec50ad conditionally detaches a record from a doubly linked structure, places that record at the head of a separate singly linked chain, and writes a continuation address to a caller-provided output slot.

That description is **inferred intent**. The established behavior is narrower: the function compares two 32-bit address values and, when they differ, performs fixed-offset link updates, decrements a 32-bit field, and prepends the selected record to a chain rooted in another field. No allocation, deallocation, payload processing, or external function call is visible.

When the selected address equals the distinguished address stored in the structure, the function skips all link and count changes and writes that distinguished address to the output slot.

## Interface

The function has three inputs and no direct return value:

| Input | Role | Required representation |
|---|---|---|
| Result slot | Destination for the continuation address | Address of a writable 4-byte slot |
| Structure address | Base address of the state being modified | A 32-bit address-like value identifying storage with the fields described below |
| Selection slot | Storage containing the selected record address | Address of a readable 4-byte slot |

The supplied representation uses 4-byte words for the addresses manipulated by this function. This contract does not establish a corresponding 64-bit layout.

### Structure layout

Only the following fields are accessed directly relative to the structure address:

| Byte offset | Width | Established use | Inferred role |
|---|---:|---|---|
| +4 | 4 bytes | Read as a chain head, then replaced with the selected record address | Head of a reusable-record chain |
| +16 | 4 bytes | Compared with the selected address; read again on the equality path | Sentinel or distinguished record address |
| +20 | 4 bytes | Read, decremented by one, and written back | Active-record count |

Other structure fields are not directly accessed.

### Record layout

The selected record and the records reached through its links use the following two-word layout:

| Byte offset | Width | Established use | Inferred role |
|---|---:|---|---|
| +0 | 4 bytes | Address used as a continuation and as a link | Successor link |
| +4 | 4 bytes | Address used to locate another record and repair a link | Predecessor link |

On the modification path, the selected record’s first word is ultimately repurposed as a link to the former head of the separate chain. Its second word is not explicitly cleared or reset.

### Preconditions

For ordinary, non-aliased use:

- The selection slot must be readable.
- The result slot must be writable.
- The structure’s distinguished-address field must be readable.
- If the selected address differs from the distinguished address, the selected record’s first two words must be readable, and its first word must be writable.
- On that modification path, the address in the selected record’s second word must identify a writable first word.
- The address in the selected record’s first word must identify a writable second word.
- The structure’s chain-head and count fields must be accessible for their respective reads and writes.

There are no visible checks for null pointers, inaccessible storage, malformed links, count consistency, or overlapping objects.

A coherent doubly linked structure is a precondition for the inferred removal semantics, not a condition validated by the function. The raw operations can target storage even when the links do not describe a valid list.

## Behavior

### Distinguished-address case

The function reads the selected address from the selection slot and compares it with the structure’s field at offset +16.

If the values are equal:

- No record link is traversed.
- No link repair occurs.
- The count field is not accessed.
- The chain-head field is not accessed.
- The distinguished-address field is read again.
- That second reading is written to the result slot.

Under stable, non-overlapping storage, the output is the distinguished address that matched the selected address.

Equality does not require dereferencing the distinguished address. Consequently, a distinguished value of zero does not itself cause a record access on this path. The selection slot, structure field, and result slot must still be valid for their accesses.

The result is obtained from a fresh read of the distinguished-address field, rather than from a retained copy of the value used for comparison. Behavior under concurrent modification is not established.

### Record-removal case

When the selected address differs from the distinguished address, the function first retains the selected record’s first word as the eventual output value.

For stable, non-overlapping storage, this is the selected record’s original successor address.

The function then repairs the two neighboring links:

- The first word of the record identified by the selected record’s second word is replaced with the selected record’s first word.
- The second word of the record identified by the selected record’s first word is replaced with the selected record’s second word.

In an ordinary doubly linked structure, these changes connect the predecessor directly to the successor and the successor directly to the predecessor.

After the neighboring links have been updated, the function decrements the structure’s 32-bit field at offset +20 by one.

It then reads the selection slot again to obtain the record that will be prepended to the separate chain. That record’s first word is overwritten with the current value of the structure’s field at offset +4. The structure’s field at offset +4 is then overwritten with the record address.

Finally, the retained continuation value is written to the result slot.

The output is therefore not taken from the selected record’s first word after chain insertion. Under ordinary conditions, it is the original successor address, while the selected record’s first word now points to the former chain head.

### Observable ordering

The supplied representation establishes the following order between completed operations:

1. The selected address and distinguished address are read for comparison.
2. On the modification path, the continuation value is captured before any structural write.
3. The predecessor-side link update occurs.
4. The successor-side link update occurs.
5. The count update occurs.
6. The selection slot is read again to identify the record to prepend.
7. That record’s first word receives the current chain-head value.
8. The chain-head field receives that record’s address.
9. The result slot receives the retained continuation value.

On the equality path, the second read of the distinguished-address field precedes the result-slot write, and there are no intervening structural writes.

This ordering does not establish an exact machine-instruction sequence or the relative evaluation order of all reads within a single link-update expression. In particular, the comparison does not establish which of its two operands is read first.

### Repeated reads and aliasing

The implementation does not retain one immutable selection address for the entire operation. It rereads the selection slot during link processing and again before chain insertion. It also rereads link words rather than capturing both neighboring addresses before making changes.

Accordingly, the ordinary removal description depends on storage relationships:

- An earlier link write must not change a selection or link value that is read later, unless that effect is intentionally part of the caller’s layout.
- The count field and chain-head field should not overlap unrelated record links.
- Concurrent mutation is outside the established contract.
- Output aliasing can overwrite state after the structural operations have finished.

For example, if the result slot is the selection slot, the final write advances that slot to the retained continuation address under otherwise ordinary conditions. The function does not otherwise explicitly update the selection slot.

If the result slot overlaps the structure’s chain-head field, the final output write replaces the newly installed head. If it overlaps the count field, it replaces the decremented count. Such overlaps are not rejected, but they do not satisfy the ordinary removal-and-recycling postconditions.

### Integer width and decrement boundary

The count is accessed as a 32-bit integer and reduced by one. No saturation, lower-bound check, or consistency check is visible.

For an initial count within the signed 32-bit range and greater than −2,147,483,648, the represented decrement has an unambiguous result. In particular, an initial count of zero becomes −1; zero does not prevent removal.

At an initial count of −2,147,483,648, a mathematical decrement is outside the signed 32-bit range. The supplied representation alone does not establish whether the executable wraps to 2,147,483,647 or has another relevant behavior at that boundary. That case is **documented_only**, not an asserted portable outcome.

The fixed byte offsets describe storage addressing, not checked arithmetic. Address overflow and invalid resulting addresses are not handled explicitly.

## Side effects

Under ordinary, non-overlapping conditions, the modification path writes:

- The predecessor’s first word.
- The successor’s second word.
- The structure’s 32-bit count field.
- The selected record’s first word.
- The structure’s chain-head field.
- The result slot.

The equality path writes only the result slot.

The function does not visibly:

- Allocate or release memory.
- Call a payload destructor.
- Clear the removed record’s predecessor word.
- Inspect or modify payload fields beyond the two link words.
- Validate reciprocal links.
- Verify that the selected record belongs to the supplied structure.
- Report an error or return a success flag.
- Acquire a lock or provide transactional rollback.

Moving the record onto the separate chain is a structural state change, not evidence that its backing memory has been freed. Storage lifetime remains a caller obligation.

If an access fails partway through execution, preceding writes may already have occurred. No rollback behavior is visible. The exact fault or exception mechanism is unknown.

## Dependencies

No external function dependency is visible for F-11aa8eec50ad.

Its operational dependencies are the caller-supplied memory layout, accessible storage at the traversed addresses, and stable values where later reads depend on earlier state.

There is no basis for attributing memory ownership, global-state preservation, locking, validation, or cleanup to an unknown helper: no helper call appears in the supplied representation.

## Acceptance examples

These examples assume stable memory, valid accesses, and no unintended overlap unless stated otherwise.

### Distinguished record selected

The selection slot contains the same address as the structure’s distinguished-address field.

Expected behavior:

- The result slot receives the distinguished address.
- The count remains unchanged.
- The separate chain head remains unchanged.
- No record links are read or written.

The distinguished address need not identify readable record storage for this path.

### Ordinary interior record

A selected record has a predecessor and successor, and the neighboring records link back to it. The structure’s count is 7, and its separate chain has an existing head.

Expected behavior:

- The predecessor’s successor link becomes the original successor.
- The successor’s predecessor link becomes the original predecessor.
- The count becomes 6.
- The selected record’s first word becomes the former separate-chain head.
- The separate-chain head becomes the selected record.
- The selected record’s second word remains unchanged by an explicit write to that field.
- The result slot receives the original successor address.

### Last active record before the distinguished record

The selected record’s successor is the distinguished record.

Expected behavior:

- Removal still takes the modification path because the selected record itself is not the distinguished record.
- The distinguished record’s predecessor word is updated as part of link repair.
- The output is the distinguished address.
- The selected record is prepended to the separate chain.

### Single-record circular structure with a distinguished record

The selected record’s predecessor and successor both identify the distinguished record, whose two links identify the selected record.

Expected behavior:

- The distinguished record’s first and second words both become its own address.
- A count of 1 becomes 0.
- The selected record becomes the separate-chain head.
- The result is the distinguished address.

### Empty separate chain

The chain-head field initially contains zero, and the selected record differs from the distinguished record.

Expected behavior:

- The selected record’s first word becomes zero.
- The chain-head field becomes the selected record’s address.
- The function does not dereference the old chain-head value.

### Zero count with a removable record

The links are accessible, the selected record differs from the distinguished record, and the count is zero.

Expected behavior:

- Link repair and chain insertion still occur.
- The count becomes −1.
- No count-related rejection or error result occurs.

### Result slot also serves as the selection slot

The output and selection slots are the same 4-byte location, with no overlap with structural fields or record links.

Expected behavior:

- Removal and chain insertion use the selected record address before the final output write.
- The shared slot ends containing the original successor address.

## Uncertainties

The following are **documented_only**:

- Whether the structure is formally a list, queue, pool, or another container.
- Whether the field at +16 is a permanent sentinel.
- Whether the separate chain is specifically a free list.
- The true ownership and lifetime rules for records and payloads.
- The intended permitted alias relationships.
- Exact behavior at signed count underflow.
- Alignment requirements, byte order, and fault handling.
- Concurrency guarantees and atomicity.
- The exact instruction-level ordering of reads within compound expressions.

The list-removal and record-recycling interpretation is supported by the link updates, count decrement, and chain insertion, but semantic names do not replace the established memory effects.

## Evidence

The established behavior comes solely from the supplied static function representation: 4-byte accesses, fixed field offsets, the equality branch, ordered writes, repeated selection and link reads, and the absence of visible external calls.

No runtime tests have been performed. The acceptance examples are conditional consequences of that representation, not observed executions. No independent evidence establishes caller invariants, memory ownership, or behavior beyond the accesses described above.
