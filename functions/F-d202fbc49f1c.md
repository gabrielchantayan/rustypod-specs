<!-- Trusted provenance: F-d202fbc49f1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d202fbc49f1c

## Purpose

F-d202fbc49f1c rearranges links among existing records. Its established behavior is consistent with a **right rotation in a parent-linked binary tree**: it promotes the node identified by the supplied pivot’s offset-8 link, transfers the promoted node’s offset-12 link to the pivot, reconnects the incoming root or parent link, and attaches the pivot beneath the promoted node.

The link accesses and assignments are established by the supplied static body. The terms **tree**, **root**, **parent**, **left child**, and **right child** describe inferred structural roles, not independently established record types.

The function has two address-bearing inputs and no return value. It performs no explicit argument validation, allocates or releases no storage, and calls no other function. It does not inspect keys or payloads or explicitly update balancing metadata.

A valid-tree interpretation requires caller-maintained structural and storage preconditions. For malformed or overlapping records, the source establishes an access sequence, not a guarantee that the resulting structure is a tree.

## Interface

### Inputs

The first input identifies a **context record**. Its four-byte field at byte offset 16 supplies the address of an **anchor record**. The anchor’s four-byte field at byte offset 4 is the **recorded root slot**.

The second input identifies the **pivot node**.

The **promoted node** is selected by the first read of the pivot’s offset-8 field. Its address is retained throughout the operation.

The **transferred subtree** ordinarily means the node identified by the promoted node’s initial offset-12 field. That link is read repeatedly rather than retained once for all subsequent uses.

The function modifies records reached through the supplied addresses. It does not modify the caller’s argument-storage locations merely by receiving those values.

### Referenced layout

All offsets are byte offsets. Every listed field access is four bytes wide in the supplied representation.

| Record | Offset | Established use |
|---|---:|---|
| Context | 16 | Address used to locate the anchor |
| Anchor | 4 | Recorded root, compared with the pivot and conditionally replaced |
| Node | 4 | Parent-like link |
| Node | 8 | Left-child-like link |
| Node | 12 | Right-child-like link |

No explicit access targets offset 0 or other unlisted fields. Preservation of those fields is conditional on the absence of overlapping storage or other aliases that make them coincide with an accessed location.

Complete record sizes and formal type definitions are not supplied.

### Widths and arithmetic

The representation uses four-byte link fields and integer-valued addresses. The contract therefore concerns **32-bit stored address values**, not host-native pointers of an arbitrary width.

Address formation uses fixed byte displacements of 4, 8, 12, and 16. There is no multiplication, element-count calculation, key arithmetic, or explicit adjustment of a stored link value.

The body compares address-bearing values for equality and compares the transferred link with zero. It does not perform signed ordering comparisons on them.

Exact address-conversion rules, alignment requirements, and overflow behavior are not established. In particular, the contract does not specify wrapping address arithmetic or a defined result when adding a field displacement exceeds the supported address range.

### Output and ownership

There is no return value, status code, or explicit error result. Successful completion is reflected in the changed links.

The function neither creates nor destroys records. It performs no ownership acquisition or release. Callers remain responsible for storage lifetime and for making every accessed location valid for the required read or write.

## Behavior

### Preconditions for a conventional rotation

The following conditions support the inferred tree operation:

1. The context and pivot identify accessible records.
2. The pivot’s offset-8 link identifies an accessible promoted node. A zero promoted-node address is not handled as an empty-child case.
3. The context’s offset-16 field identifies an accessible anchor. The anchor’s offset-4 field is readable and, if selected for replacement, writable.
4. Every field written by the selected path is writable.
5. A nonzero transferred-subtree link identifies a record whose offset-4 field can be written.
6. If the recorded root differs from the pivot, the pivot’s parent-like link identifies a record with a readable offset-12 field and a writable selected child field.
7. The relevant links are initially consistent with the intended local tree topology.
8. Storage does not overlap in a way that changes the intended field relationships, and the relevant contents remain stable during execution.

These are caller obligations for the structural interpretation, not checks performed by the function.

### Source-level access sequence

The sequence below describes the accesses expressed by the supplied body. It does not prescribe machine instructions, compiler scheduling, atomicity, visibility to other execution contexts, or a synchronization protocol.

1. **Select the promoted node.**  
   Read the pivot’s offset-8 field. Retain that address as the promoted node for all later uses of the promoted-node identity.

2. **Replace the pivot’s offset-8 link.**  
   Read the promoted node’s offset-12 field and write that four-byte value into the pivot’s offset-8 field.

3. **Conditionally repair a transferred node’s parent-like link.**  
   Read the promoted node’s offset-12 field again and test it against zero.

   If the tested value is nonzero, read that field once more to obtain a destination address, then write the pivot address into offset 4 of that destination.

   The destination read is separate from the read used for the zero test. The body does not retain the tested value and use that retained value as the destination.

4. **Assign the promoted node’s parent-like link.**  
   Read the pivot’s offset-4 field and write that value into the promoted node’s offset-4 field.

5. **Locate the recorded root slot.**  
   Read the context’s offset-16 field and form the address of the anchor’s offset-4 field. Read that slot and compare its value with the pivot address.

6. **Reconnect the incoming link.**
   - If the recorded root equals the pivot, write the promoted-node address into the located root slot.
   - Otherwise, read the pivot’s offset-4 field again and retain that value as the parent address for this branch. Read the parent’s offset-12 field.
     - If that field equals the pivot address, replace it with the promoted-node address.
     - Otherwise, write the promoted-node address into the parent’s offset-8 field.

   The final alternative does not read or verify the parent’s offset-8 value before overwriting it.

7. **Attach the pivot below the promoted node.**  
   Write the pivot address into the promoted node’s offset-12 field.

8. **Update the pivot’s parent-like link.**  
   Write the promoted-node address into the pivot’s offset-4 field.

The function then returns without a result.

### Result under valid, stable tree preconditions

For valid, non-overlapping records with stable contents:

- The promoted node takes the pivot’s former incoming root or parent position.
- The promoted node’s offset-4 field receives the pivot’s original parent-like value.
- The promoted node’s offset-12 field identifies the pivot.
- The pivot’s offset-4 field identifies the promoted node.
- The pivot’s offset-8 field receives the promoted node’s former offset-12 link.
- If that transferred link is nonzero, the transferred node’s offset-4 field identifies the pivot.
- Exactly one incoming-link location is replaced: the recorded root slot or one child field of the former parent.

The pivot’s offset-12 field and the promoted node’s offset-8 field are not explicitly modified. Under the inferred tree interpretation, they retain the subtrees outside the transfer.

The root case does not normalize the promoted node’s parent-like field. It copies the pivot’s original value, whether that value is zero, a sentinel address, or another marker. The required root-parent convention must therefore already be represented by the pivot’s original offset-4 field.

### Branch boundaries and malformed structures

The root decision depends only on equality between the recorded root and the pivot. A zero parent-like link does not independently select the root branch.

In the non-root branch, an unequal parent offset-12 link always selects a write to parent offset 8. There is no consistency check that either parent child link originally identifies the pivot.

The zero check applies only to the promoted node’s offset-12 link. There is no zero check on the promoted-node address itself.

Aliasing can change later reads through earlier writes. For example, if the pivot and promoted node coincide, replacing the pivot’s offset-8 field also changes that same node’s offset-8 field; the function contains no special handling for this relationship. The valid-tree postconditions do not extend to such inputs.

Repeated reads are also material: the link installed into pivot offset 8, the value tested against zero, and the address used for the transferred-node parent write are separate reads. Without stable contents and suitable non-overlap, they need not describe the same node. This observation does not establish support for concurrent mutation.

## Side effects

For valid, distinct, stable records, the explicit writes target:

- Pivot offset 8.
- Transferred-subtree offset 4, only when the separately read link passes the zero test.
- Promoted-node offset 4.
- One incoming-link location: anchor offset 4, former-parent offset 12, or former-parent offset 8.
- Promoted-node offset 12.
- Pivot offset 4.

A completing path contains five four-byte writes when the transferred-link test is zero, or six when it is nonzero. These counts describe write operations, not necessarily distinct storage locations if records overlap.

Within the source-level sequence, the pivot’s offset-8 link is replaced before the incoming root or parent link is updated. The incoming link is updated before the final writes establish the promoted node’s offset-12 link and the pivot’s offset-4 link.

These intermediate source-level states are **not an external visibility guarantee**. The supplied evidence does not establish which intermediate values, if any, a concurrent observer could observe. Compiler transformations, access atomicity, memory ordering, and the execution environment are unspecified; unsynchronized concurrent access may lack defined behavior.

There is no explicit rollback or recovery path. If an access fails after earlier assignments, the body contains no operation that restores the earlier values. Exact fault, exception, or termination behavior remains environment-dependent and is not established.

There is no explicit I/O, allocation, deallocation, callback, or direct global-variable access. Records reached through the inputs may nevertheless reside in globally shared storage.

## Dependencies

No function calls appear in the supplied body. There are no dependency function IDs to enumerate and no unknown callee behavior to assume.

Execution depends on caller-provided storage and the surrounding memory model:

- Four-byte address-bearing fields at the stated offsets.
- Valid address representations and field-address calculations.
- Accessible storage with the required read and write permissions.
- Sufficient storage lifetime.
- Any alignment and synchronization obligations imposed by the execution environment.

The function supplies no locking, atomic publication, retry, or interference detection.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where stated otherwise, each assumes accessible, distinct, non-overlapping records, stable contents, valid address arithmetic, and writable destination fields. Unmentioned fields remain unchanged only under those assumptions.

### Root rotation with a transferred subtree

Initially:

- The anchor’s recorded root identifies the pivot.
- Pivot offset 8 identifies the promoted node.
- Promoted-node offset 12 identifies a nonzero transferred subtree.
- Pivot offset 4 contains a root-parent marker.

After completion:

- The recorded root identifies the promoted node.
- Promoted-node offset 4 contains the original root-parent marker.
- Promoted-node offset 12 identifies the pivot.
- Pivot offset 4 identifies the promoted node.
- Pivot offset 8 identifies the transferred subtree.
- Transferred-subtree offset 4 identifies the pivot.

The marker is copied without interpretation or normalization. The former parent’s child fields are not consulted through the non-root branch.

### Non-root pivot in its parent’s offset-12 position

Initially, the recorded root differs from the pivot, pivot offset 4 identifies a parent, and parent offset 12 identifies the pivot.

After completion, parent offset 12 identifies the promoted node. Parent offset 8 is not explicitly written. The transferred link and local parent links change as described above.

### Non-root pivot in its parent’s offset-8 position

Initially, the recorded root differs from the pivot, parent offset 12 differs from the pivot, and parent offset 8 identifies the pivot.

After completion, parent offset 8 identifies the promoted node. Parent offset 12 remains unchanged. The function selects this branch because of the offset-12 comparison, not because it verifies offset 8.

### Empty transferred subtree

Initially, the promoted node’s offset-12 link is zero.

The function writes zero into pivot offset 8 and skips the transferred-node parent write. It later writes the pivot address into promoted-node offset 12.

This case is supported by an explicit zero check. It is not equivalent to a zero promoted-node address.

### Inconsistent non-root parent links

Initially, the recorded root differs from the pivot, pivot offset 4 identifies an accessible parent, and neither parent child field identifies the pivot.

The function overwrites parent offset 8 with the promoted-node address. It reports no consistency error and returns no status.

This is established branch behavior, not acceptance of the resulting structure as a valid tree.

### Missing promoted node

Initially, pivot offset 8 is zero.

The next field read is addressed relative to that zero promoted-node value, at displacement 12. There is no early return or explicit error result. Whether that access faults or encounters accessible storage is not established; a successful tree rotation is not promised.

## Uncertainties

### Inferred intent

The link topology and replacement pattern support the interpretation of a right rotation in a parent-linked binary tree. Tree ordering, balancing discipline, and surrounding container semantics are not established.

### Documented-only unknowns

The following remain **documented_only**:

- Formal record types, complete sizes, and required alignment.
- The meaning of unaccessed fields.
- Whether the anchor is a sentinel, header, or another object.
- The required root-parent marker.
- Caller guarantees concerning structural consistency and aliasing.
- Platform endianness and exact integer-to-address interpretation.
- Overflow, invalid-address, and misalignment behavior.
- Compiler transformations and the mapping from source-level accesses to machine accesses.
- Atomicity, memory ordering, concurrent visibility, and the validity of unsynchronized access.
- External invariants involving counts, colors, heights, iterators, payloads, or ownership.
- Fault handling outside the supplied body.

These unknowns do not justify adding validation, metadata maintenance, synchronization, memory management, or recovery behavior to this contract.

## Evidence

The established claims derive solely from the supplied static body for F-d202fbc49f1c:

- Four-byte field accesses at the listed byte offsets.
- Retention of the initially selected promoted-node address.
- Repeated reads of the promoted node’s offset-12 link.
- A zero test controlling one parent-like-field write.
- A recorded-root equality branch followed, when necessary, by a parent offset-12 equality branch.
- The stated source-level assignment sequence.
- No function calls, explicit status result, allocation, release, validation path, or rollback.

The tree terminology and valid-tree postconditions are structural inferences conditional on the stated preconditions. Source-level ordering does not establish external observation order or concurrent-use semantics.

No runtime tests have been performed.
