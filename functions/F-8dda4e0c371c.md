<!-- Trusted provenance: F-8dda4e0c371c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8dda4e0c371c

## Purpose

F-8dda4e0c371c modifies linked records in place. Under the inferred interpretation of their fields as parent, left-child, and right-child links, it performs a **right rotation around a designated pivot**: the pivot’s left child replaces the pivot in its incoming link, and the pivot becomes that child’s right child.

The established behavior is narrower than the tree interpretation: fixed-offset, four-byte memory accesses, conditional link replacement, and ordered writes. The function does not validate that the records form a tree or that the recorded parent references the pivot.

This contract distinguishes:

- **Established static behavior:** accesses and control flow directly supported by the supplied representation.
- **Inferred intent:** interpretation of those accesses as a tree rotation.
- **documented_only:** requirements or behavior not resolved by the supplied evidence.

No runtime tests have been performed.

## Interface

The function accepts two address-bearing arguments and has no explicit return value.

| Argument | Role |
|---|---|
| Context address | Provides access to an indirect root-reference slot. |
| Pivot address | Identifies the record whose offset-8 link supplies the record to promote. |

The representation uses 32-bit integer address values and four-byte link fields. It does not establish a portable interface for wider pointers or define the behavior of invalid address arithmetic.

The following semantic names describe the accessed storage:

| Name | Definition |
|---|---|
| Pivot | Record identified by the second argument. |
| Promoted node | Record identified by the initial value read from the pivot at offset 8. |
| Transferred subtree | In ordinary stable storage, the record identified by the promoted node’s offset-12 field before final attachment. That field is read repeatedly, so this name does not imply a single retained value. |
| Header | Record identified by the value read from the context at offset 16. |
| Root slot | Four-byte location at header offset 4. |
| Recorded parent | In the non-root branch, the record identified by a later read of the pivot at offset 4. |

The partial layouts are:

| Storage | Byte offset | Width | Inferred meaning |
|---|---:|---:|---|
| Context | 16 | 4 bytes | Header address |
| Header | 4 | 4 bytes | Root address |
| Node | 4 | 4 bytes | Parent address |
| Node | 8 | 4 bytes | Left-child address |
| Node | 12 | 4 bytes | Right-child address |

These are accessed fields, not complete record definitions. Payload layout, allocation boundaries, and additional metadata are unknown.

### Access and permission preconditions

Every read requires readable storage at the address actually used. Every write requires writable storage at its actual destination. The function does **not** require every accessed location to be writable.

| Location | Required access on the selected path |
|---|---|
| Context offset 16 | Read only. |
| Root slot | Read on every completed path; write only when its value equals the pivot address. |
| Pivot offset 8 | Read initially, then write. |
| Pivot offset 4 | Read during parent inheritance, read again in the non-root branch, and write at final attachment. |
| Promoted node offset 12 | Repeated reads during transfer, then a final-attachment write. |
| Promoted node offset 4 | Write during parent inheritance. |
| Transferred-subtree parent field | Write only if the transfer condition is nonzero; the destination comes from a subsequent read. |
| Recorded parent offset 12 | Read in the non-root branch; also written if it equals the pivot address. |
| Recorded parent offset 8 | Written only when the non-root right-child comparison fails. No confirming read is required. |

Accordingly, read-only context storage is compatible with the function. On the non-root branch, a read-only root slot is also compatible. These statements assume those locations do not overlap another actual write destination.

The promoted-node address is dereferenced without a zero check. A zero value in its right-child field is explicitly handled by skipping the transferred-subtree parent update.

A zero or otherwise non-dereferenceable value in the pivot’s parent field can be compatible with the root branch: the function copies that value but does not dereference it as a parent address there. The non-root branch requires the recorded parent’s selected fields to be accessible.

There is no status result or explicit error result. The observable output is the changed memory state.

## Behavior

### Transfer phase

The function first reads the pivot’s offset-8 field and retains that value as the promoted-node address.

It then reads the promoted node’s offset-12 field and copies its four-byte value into the pivot’s offset-8 field.

After that write, it reads the promoted node’s offset-12 field again and tests the value against zero:

- If zero, it skips the transferred-subtree parent update.
- If nonzero, it reads that same field once more. It uses this third value as an address and writes the pivot address to that record’s offset-4 field.

Thus, this phase performs two reads of the promoted node’s right-child field when the condition is zero and three when it is nonzero. The copied value, tested value, and destination address are not one retained snapshot.

With stable, non-overlapping storage, those reads agree. If an earlier write overlaps the repeatedly read field, they may differ even without outside mutation. The representation supplies no consistency check between them. In particular, the third value is not separately checked for zero or validity.

### Parent inheritance

The function next reads the pivot’s offset-4 field and writes that value into the promoted node’s offset-4 field.

This is the pivot’s parent value at this phase, after the transfer writes. It is not necessarily the value present at function entry if storage overlaps.

### Incoming-link replacement

The function reads the context’s offset-16 field, derives the root-slot address by adding four bytes, and reads that slot. The slot address is retained for this decision; the context field is not reread.

If the root-slot value equals the pivot address, the function writes the promoted-node address into that same root slot. This branch does not dereference the pivot’s recorded parent.

Otherwise, the function:

- Reads the pivot’s offset-4 field again and retains it as the recorded-parent address.
- Reads that parent’s offset-12 field and compares it with the pivot address.
- Replaces the parent’s offset-12 field if the comparison succeeds.
- Replaces the parent’s offset-8 field if the comparison fails.

Both replacement branches write the promoted-node address.

The left-child case is selected solely by failure of the right-child comparison. The function does not read the left-child field to verify that it contains the pivot.

These decisions occur after the transfer and parent-inheritance writes. Any overlap between earlier write destinations and subsequently read control fields can affect branch selection.

### Final attachment

After the incoming-link replacement, the function performs two writes in this order:

1. The pivot address replaces the promoted node’s offset-12 field.
2. The promoted-node address replaces the pivot’s offset-4 field.

The pivot’s parent field is therefore the last direct write destination.

### Ordinary tree result

Assuming stable storage, distinct non-overlapping records and control fields, valid links, and a correctly represented incoming reference, normal completion yields:

| Relationship | Result |
|---|---|
| Pivot’s left child | Promoted node’s former right child |
| Nonzero transferred subtree’s parent | Pivot |
| Promoted node’s parent | Pivot’s former parent |
| Root or parent link previously identifying pivot | Promoted node |
| Promoted node’s right child | Pivot |
| Pivot’s parent | Promoted node |

Under these assumptions, the pivot’s right-child field and the promoted node’s left-child field remain unchanged because neither is a direct write destination.

The function does not traverse the structure or inspect keys or payloads. It does not itself verify or update ordering, height, balance, color, subtree counts, or other metadata.

### Malformed structures and arithmetic boundaries

There is no explicit rejection of null promoted-node addresses, invalid addresses, cycles, self-links, overlapping records, or inconsistent parent-child relationships.

For a non-root pivot whose recorded parent references neither child as the pivot, the function still overwrites that parent’s left-child field if the preceding accesses complete.

Address formation uses only fixed byte offsets: 4, 8, 12, and 16. No counts, products, variable strides, or allocation-size calculations occur. The address-bearing values and stored links are represented as 32-bit quantities, but the supplied representation does not establish safe wraparound semantics, alignment rules, or fault behavior. Successful execution requires each selected access to resolve to usable storage; modular address wrapping is not a supported assumption.

## Side effects

The function directly writes:

- The pivot’s left-child field.
- A transferred-subtree parent field, conditionally.
- The promoted node’s parent field.
- One incoming-link field: the root slot, the recorded parent’s right-child field, or the recorded parent’s left-child field.
- The promoted node’s right-child field.
- The pivot’s parent field.

A normally completed path performs five four-byte writes when the transfer condition is zero and six when it is nonzero. These are write-operation counts, not counts of distinct destinations when addresses overlap.

The context field is not a direct write target. The root slot is not a write target on the non-root branch. Neither requires writable storage merely because it is read. Overlap with another write destination can independently impose a write requirement on the same physical bytes.

No allocation, release, callback, logging, I/O, or synchronization operation is visible. The function rearranges stored references; it does not visibly create or destroy records. Application-level ownership and lifetime rules are **documented_only**.

There is no visible rollback. If an access fails after earlier writes, the representation provides no restoration of the previous state. The exact failure mechanism and resulting execution state remain unknown.

The writes are not an atomic publication of a completed rotation. In particular, the incoming link changes before the final two attachments. No guarantee of safe unsynchronized concurrent observation follows from this function.

## Dependencies

No calls to other functions are visible. There are no identified callable dependencies requiring opaque function IDs.

Execution depends on caller-provided address values and successful four-byte accesses to the selected locations. Permissions are path-specific: all reads need readable storage, while only actual write destinations need writable storage.

The absence of visible calls establishes no guarantees about how an enclosing caller manages ownership, synchronization, or recovery. No behavior of an unknown callee is assumed.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed test results. Unless an example says otherwise, assume valid addresses, stable values, distinct non-overlapping records and control fields, and the permissions required for the selected accesses.

### Root rotation with a transferred subtree

Initially:

- The root slot identifies A.
- A’s left child is B.
- B’s right child is nonzero C.
- A’s parent field contains S.

After normal completion:

- The root slot identifies B.
- A’s left child is C.
- C’s parent is A.
- B’s parent contains S.
- B’s right child is A.
- A’s parent is B.

The context field may be read-only. The root slot must be writable because this branch replaces it. S need not identify a usable parent record because this branch copies S without dereferencing it.

### Root rotation with no transferred subtree

Initially, the root slot identifies A, A’s left child is B, and B’s right-child field is zero.

After normal completion:

- A’s left-child field is zero.
- No transferred-subtree parent write occurs.
- The root slot identifies B.
- B’s parent contains the value read from A’s parent during parent inheritance.
- B’s right child is A.
- A’s parent is B.

A zero transferred subtree is supported. A zero promoted-node address is not covered by this case.

### Non-root rotation with read-only context and root slot

Initially:

- The context’s header-address field is readable but not writable.
- The root slot is readable but not writable and does not identify A.
- A’s left child is B.
- A’s parent is D.
- D’s right child is A.
- All actual write destinations are writable.

The read-only context field and root slot do not prevent the represented operation from completing. D’s right-child field becomes B; the remaining links change as in the ordinary tree result. The root slot is read but not written.

### Pivot is the recorded parent’s left child

Initially, the root slot does not identify A, A’s parent is D, D’s right child is not A, and D’s left child is A.

After normal completion, D’s left-child field contains B, where B was A’s left child. Selection depends on the failed right-child comparison, not on a successful left-child comparison.

D’s right-child field need only be readable for this selection. D’s left-child field must be writable.

### Recorded parent references neither child as the pivot

Initially, the root slot does not identify A, A’s parent is D, and neither child field of D identifies A.

If all accesses complete, D’s left-child field is nevertheless replaced with B. Requiring validation, rejection, or preservation of that field would contradict the represented behavior.

### Root slot lacks write permission on the root branch

If the root slot identifies A but is not writable, the successful-execution preconditions are not satisfied. The function contains no permission check or explicit error result. Earlier transfer and parent-inheritance writes precede the attempted root-slot replacement, so unchanged memory cannot be required.

The particular fault, recovery action, or termination behavior is unresolved.

### Unusable promoted-node address

If A’s left-child field supplies an unusable address, the function still attempts to read that address at offset 12. It does not return a validation status. No specific exception or successful result is established.

## Uncertainties

The following remain **documented_only**:

- Actual record types, full layouts, allocation sizes, and whether the header is a sentinel.
- Exact correspondence between the represented 32-bit address values and the execution environment.
- Byte order, alignment constraints, invalid-address behavior, and overflow handling.
- Ownership and lifetime policies for the context, header, and nodes.
- External synchronization and recovery mechanisms.
- Application-level meaning or acceptability of aliases, cycles, and self-links.

Some alias effects follow directly from the stated access order, but the function does not establish that such structures are intended or valid inputs.

The tree-rotation interpretation is structural inference. It does not identify a particular balancing algorithm or establish a search-tree invariant.

The access sequence is supported at the supplied representation’s level. Actual machine instructions, compiler transformations, and concurrent memory semantics have not been independently established.

## Evidence

The supplied representation supports the four-byte field widths, fixed offsets, repeated transfer-field reads, branch comparisons, retained addresses, and write order described here. It also supports the distinction between read-only access to the context field, conditional write access to the root slot, and writes to the selected node fields.

The rotation interpretation follows from the link pattern: promotion through offset 8, transfer through offset 12, incoming-link replacement, and attachment of the pivot through offset 12.

No external type definitions, binary behavior, or runtime traces were examined. No runtime tests have been performed. All acceptance examples are static consequences under their stated preconditions, not observed executions.
