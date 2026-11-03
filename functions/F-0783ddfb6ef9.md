<!-- Trusted provenance: F-0783ddfb6ef9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0783ddfb6ef9

## Purpose

F-0783ddfb6ef9 conditionally removes a selected record from a doubly linked structure, invokes a dependency on storage eight bytes into that record, and places a record onto a separate singly linked chain. It writes a four-byte result identifying either the selected record’s original forward neighbor or the structure’s distinguished boundary record.

The linked-structure interpretation is inferred from the observed field accesses. The established behavior is the sequence of memory reads, link updates, count adjustment, dependency invocation, chain insertion, and final result write described below.

The boundary case performs no unlinking, count adjustment, dependency invocation, or chain insertion. It still writes the result.

## Interface

The function has three logical inputs and no explicit return value:

| Input | Role | Required accessible storage |
|---|---|---|
| Result destination | Address receiving the result word | Four writable bytes |
| Context address | Base address for structure metadata | Fields at byte offsets 4, 16, and 20, as required by the selected branch |
| Selection-cell address | Address of a cell containing the selected record’s address | Four readable bytes, including after the dependency returns on the removal branch |

The result is delivered through the result destination, not through a function return value.

### Widths and representation

The supplied representation uses four-byte words for record links, metadata fields, the selection cell, and the output. Addresses carried in those words are consequently treated as 32-bit address values in this contract. This does not establish a portable interface for wider-pointer environments.

The relevant context layout is:

| Byte offset | Width | Observed use |
|---|---:|---|
| 4 | 4 bytes | Head of a separate singly linked chain |
| 16 | 4 bytes | Distinguished boundary-record address |
| 20 | 4 bytes | Count-like field decremented on removal |

No behavior is established for other context fields.

The relevant record layout is:

| Byte offset | Width or extent | Observed use |
|---|---:|---|
| 0 | 4 bytes | Forward link while linked; chain link when placed on the separate chain |
| 4 | 4 bytes | Backward link while linked |
| 8 | Unknown extent | Address passed to F-59c64e80b38d |

The function does not establish the size or representation of the storage beginning at offset 8.

### Preconditions

For the ordinary linked-record interpretation:

- The context and selection cell must be readable at the fields actually accessed.
- The result destination must be writable for four bytes.
- If the selected address differs from the boundary address, it must designate a record with readable forward and backward links.
- Its backward neighbor must permit a four-byte write at offset 0.
- Its forward neighbor must permit a four-byte write at offset 4.
- The context’s count-like field and separate-chain head must be writable.
- The address formed by adding eight bytes to the selected address must satisfy F-59c64e80b38d’s requirements.
- After that dependency returns, the address then held in the selection cell must designate storage suitable for the subsequent record-header write.

These requirements are not checked by this function. Alignment, mapping, and accessibility requirements remain those of the execution environment.

## Behavior

### 1. Boundary comparison

The function reads the selected address from the selection cell and compares it with the four-byte boundary address at context offset 16.

The supplied evidence does not establish an evaluation order between the two operand reads within that comparison. No explicit write precedes this decision.

### 2. Boundary case

If the selected address equals the boundary address:

1. The function reads the boundary field again.
2. It retains that value as the result.
3. It writes the retained four-byte value to the result destination.
4. It returns.

The result comes from the second boundary-field read, not necessarily from a retained copy of the comparison operand.

With stable memory, the result is the boundary address. If another actor changes that field between reads, the final result can differ from the address used in the comparison. No synchronization or concurrent-access guarantee is established.

This branch does not explicitly:

- Dereference the selected record.
- Change either linked-neighbor field.
- Read or update the count-like field.
- Invoke F-59c64e80b38d.
- Read or update the separate-chain head.

An overlapping result destination can nevertheless modify any of those locations through the final output write.

### 3. Removal result capture

If the addresses differ, the function reads the selected address again and reads the four-byte word at offset 0 of that record. It retains this word as the eventual result.

Under the linked-record interpretation, this is the selected record’s forward neighbor before unlinking.

The retained result is not recomputed after the dependency call or after insertion onto the separate chain.

### 4. Neighbor-link updates

The function performs two explicit link writes, in this order:

1. It writes the selected record’s forward-link value into offset 0 of the record addressed by its backward link.
2. It writes the selected record’s backward-link value into offset 4 of the record addressed by its forward link.

With stable, non-overlapping structure storage, these updates make the predecessor and successor refer directly to one another, bypassing the selected record.

The selection cell and record links are read again while these operations are performed. The function does not establish a single immutable snapshot of the selected record for the entire operation.

The first link write precedes the reads and write of the second link-update operation. Consequently, overlap between link fields, the selection cell, and context metadata can affect later accesses. The available evidence does not establish an order among all operand reads within each individual link-update expression.

### 5. Count-like field adjustment

After both neighbor-link writes, the function reads the four-byte field at context offset 20, subtracts one, and writes the result back to that field.

There is no explicit zero check, lower-bound check, or validation against actual list membership.

For ordinary in-range signed 32-bit values, the observed operation is a decrement by one. The supplied representation presents this field through signed integer operations; behavior at signed overflow boundaries is not established well enough to promise portable wraparound semantics.

### 6. Dependency invocation

After the count adjustment, the function reads the selection cell again and invokes F-59c64e80b38d with the address eight bytes beyond the selected address obtained at that point.

The dependency therefore runs after the two neighbor updates and count decrement, but before this function’s separate-chain insertion and final result write.

The function does not inspect a returned status from the dependency. Its meaning, completion conditions, and effects are not established by the supplied evidence.

In particular, this contract does not assume that the dependency:

- Preserves context fields or the selection cell.
- Leaves the selected record’s contents unchanged.
- Destroys a value.
- Releases or retains an allocation.
- Returns normally.
- Avoids callbacks or other externally observable effects.

### 7. Separate-chain insertion

If the dependency returns normally, the function reads the selection cell again. The address obtained by this post-call read is the record address used for the following two writes:

1. It reads the current head at context offset 4 and writes that head value into offset 0 of the post-call selected record.
2. It writes the post-call selected record’s address into context offset 4.

Under the ordinary interpretation, this pushes the record onto a singly linked reusable-record chain.

The context head is read after the dependency returns. A head change made by the dependency is therefore incorporated into this insertion.

The post-call selected address need not equal the address used for unlinking or for the dependency argument. If the dependency changes the selection cell, the final chain insertion uses the changed address.

No explicit write clears or replaces the record’s offset-4 field during this insertion.

### 8. Final result write

After the separate-chain insertion, the function writes the previously retained result word to the result destination and returns.

For a stable selection and ordinary non-overlapping storage, this result is the removed record’s original forward neighbor. It is not the separate-chain head and is not derived from the record after its offset-0 field has been reused.

## Side effects

On the removal branch, the explicit write order is:

1. Backward neighbor’s offset-0 field.
2. Forward neighbor’s offset-4 field.
3. Context offset 20.
4. Effects of F-59c64e80b38d, if any.
5. Post-call selected record’s offset-0 field.
6. Context offset 4.
7. Result destination.

On the boundary branch, the only explicit write is the final result write.

These operations are not presented as atomic or transactional. There is no explicit rollback path. If a memory access fails or the dependency does not return normally, earlier writes may already have occurred; the result write may not occur.

The function itself does not explicitly allocate or release memory. Moving an address onto a separate chain does not prove allocation ownership transfer. Any destruction, deallocation, or ownership effect of F-59c64e80b38d remains unknown.

There is no explicit store to the selection cell. It can nevertheless change through aliasing, through dependency effects, or through external mutation.

No non-aliasing requirement is checked. In particular:

- If the result destination overlaps the selection cell, the final output replaces the selected address.
- If it overlaps context metadata, the final output can overwrite a count, head, or boundary field.
- If it overlaps a neighbor link, it can overwrite an earlier link update.
- If structure fields overlap the selection cell, an earlier write can redirect a later read.

The simple unlink-and-recycle interpretation therefore requires suitable separation of storage, or explicit reasoning about the chosen overlaps.

## Dependencies

### F-59c64e80b38d

**Invocation condition:** The selected address differs from the boundary address during the initial comparison.

**Argument:** One address, formed by adding eight bytes to the selected address read immediately before invocation.

**Ordering:** Called after neighbor-link updates and count decrement; before the post-call selection read, separate-chain insertion, and final result write.

**Observed return use:** No return value is consumed.

**Unknown contract:** Required storage extent, ownership rules, mutation permissions, failure behavior, and normal-return guarantees are documented_only.

No additional dependency is established.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not executed tests.

### Boundary selection

Given stable, non-overlapping storage where both the selection cell and context offset 16 contain address **B**:

- The result destination receives **B**.
- No record-link dereference is required.
- The count and separate-chain head remain unchanged by explicit function writes.
- F-59c64e80b38d is not invoked.

### Ordinary interior removal

Given a selected record **N**, boundary **B**, predecessor **P**, successor **S**, and separate-chain head **H**, where:

- **N** differs from **B**.
- **N** has forward link **S** and backward link **P**.
- Storage is valid and suitably non-overlapping.
- The count is 7.
- The dependency returns without changing relevant storage.

The accepted outcome is:

- **P**’s forward link is **S**.
- **S**’s backward link is **P**.
- The count is 6.
- The dependency receives **N + 8**.
- **N**’s offset-0 word becomes **H**.
- The separate-chain head becomes **N**.
- The result is **S**.

### Successor is the boundary

If the ordinary removal preconditions hold and the selected record’s forward link is **B**, the result is **B**, and the function writes the selected record’s backward-link value to **B** at offset 4.

There is no special exclusion for a boundary record serving as a neighbor.

### Empty separate chain

If the separate-chain head contains zero and the dependency leaves it unchanged, the post-call selected record’s offset-0 word becomes zero before that record becomes the new head.

This does not establish that zero is universally the chain terminator; it establishes that the existing head word is copied without special handling.

### Dependency changes selection and head

Suppose unlinking captures successor **S** from record **N**. The dependency then changes the selection cell to valid record **M** and the separate-chain head to **K**.

On normal return:

- **M**’s offset-0 word becomes **K**.
- The separate-chain head becomes **M**.
- The final result remains **S**.

The function does not restore the original selected address.

### Zero count

If removal occurs with count zero, the function still performs the decrement. Zero is not treated as an empty-structure rejection condition. For signed 32-bit arithmetic, the resulting value is −1.

## Uncertainties

- **Inferred intent:** Doubly linked removal followed by recycling is consistent with the field accesses, but names and ownership semantics are not established.
- **Documented_only:** The boundary record’s broader role and complete layout.
- **Documented_only:** Whether the count tracks active records, and what invariants callers must maintain.
- **Documented_only:** F-59c64e80b38d’s implementation and memory-lifetime effects.
- **Documented_only:** Behavior at signed arithmetic overflow boundaries or invalid address arithmetic.
- **Documented_only:** Concurrency, reentrancy, memory-ordering guarantees, and invalid-access handling.
- **Established limitation:** There is no explicit validation, status output, or recovery path in the supplied function.

## Evidence

The supplied static body establishes the four-byte accesses, field offsets, branch condition, explicit write order, repeated selection reads, single conditional dependency call, and final output behavior.

The linked-list, boundary-record, count, and reusable-chain descriptions are interpretations of those accesses, not independently verified data-model declarations.

No runtime tests have been performed. No dependency implementation, caller behavior, or external ownership contract was supplied.
