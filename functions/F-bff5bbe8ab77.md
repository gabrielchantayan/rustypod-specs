<!-- Trusted provenance: F-bff5bbe8ab77; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bff5bbe8ab77

## Purpose

F-bff5bbe8ab77 performs a local, in-place pointer transformation on a linked structure. Under the conventional interpretation of the observed fields as parent, left-child, and right-child links, the transformation is a **left rotation** around the node supplied by the second argument.

The second argument identifies the node being demoted. Its current right child becomes the promoted node. The promoted node’s former left subtree becomes the demoted node’s right subtree. The incoming link that formerly selected the demoted node is redirected to the promoted node.

This interpretation is inferred from the field relationships. The established behavior is the ordered sequence of 32-bit reads, comparisons, and writes described below. The function does not establish that the surrounding structure is a valid tree, and it does not enforce ordering, balancing, or ownership rules.

## Interface

**Inputs**

The function receives two 32-bit arguments used as addresses:

- **Context address:** identifies an object containing a 32-bit pointer field at byte offset `+0x10`.
- **Pivot address:** identifies the node around which the link transformation occurs.

The object selected by the context’s `+0x10` field is called the **anchor** in this contract. Its field at byte offset `+0x04` is compared with the pivot address and may be replaced.

**Observed node layout**

| Byte offset | Width | Observed use | Structural interpretation |
|---|---:|---|---|
| `+0x04` | 32 bits | Receives and supplies links to surrounding nodes | Parent link |
| `+0x08` | 32 bits | Supplies the transferred link and receives the pivot address | Left-child link |
| `+0x0C` | 32 bits | Supplies the promoted address and receives the transferred link | Right-child link |

The function does not access a node field at offset zero. It provides no evidence about payload layout, complete object size, allocation metadata, or other fields.

**Output**

There is no explicit return value. Results are communicated through memory writes.

**Preconditions for an ordinary successful rotation**

The source contains no checks establishing these conditions:

- The context and pivot addresses support the accesses described here.
- The pivot’s right-child field selects a valid, non-null promoted node.
- The promoted node’s left-child field is either zero or selects a node whose parent field is writable.
- The context selects a valid anchor with a readable root-link field and, when needed, a writable one.
- If the anchor’s root link does not equal the pivot address, the pivot’s parent link selects an object with the required readable and writable child fields.
- The participating fields do not overlap in ways that change the intended operation.
- No concurrent mutation invalidates the links while the function executes.

These are structural preconditions, not validated input constraints. Violating them may cause invalid accesses or a transformation that does not preserve a tree.

## Behavior

### Entry selection and terminology

The function first reads the 32-bit value at the pivot’s `+0x0C` field. That value is retained as the promoted address for the remainder of the operation.

For the ordinary, non-overlapping tree case:

- **Pivot** is the node supplied by the second argument.
- **Promoted node** is the pivot’s original right child.
- **Transferred subtree** is the promoted node’s original left child.
- **Former parent** is the pivot’s original parent.
- **Root link** is the anchor’s field at `+0x04`.

These terms describe the intended topology. The implementation does not independently verify any of those relationships.

### Ordered observable accesses

The following order is established by the supplied source. Reads mentioned more than once are distinct source-level reads; the source does not consistently capture every link in a single snapshot.

1. Read the pivot’s right-child field and retain it as the promoted address.

2. Read the promoted node’s left-child field. Write that 32-bit value into the pivot’s right-child field.

3. Read the promoted node’s left-child field again and compare it with zero.

   If that read is nonzero, read the same field once more to obtain an address, then write the pivot address to the parent field at `+0x04` of that addressed object.

   If the tested value is zero, omit this parent-field write.

4. Read the pivot’s parent field. Write the resulting 32-bit value to the promoted node’s parent field.

5. Read the pointer at context offset `+0x10`. Use that pointer to identify the anchor’s `+0x04` root-link field, then read that field.

6. Select one incoming-link update:

   - If the root-link value equals the pivot address, write the promoted address to that root-link field.
   - Otherwise, read the pivot’s parent field again. Read that parent object’s left-child field at `+0x08`.
     - If the left-child value equals the pivot address, write the promoted address to that left-child field.
     - Otherwise, write the promoted address to the parent object’s right-child field at `+0x0C`.

7. Write the pivot address to the promoted node’s left-child field.

8. Write the promoted address to the pivot’s parent field.

The function then returns.

The source does not separately establish evaluation order between independent address calculations and value reads within a single assignment. The sequence above does not impose an additional ordering where the source supplies none.

### Result for a valid, stable tree

When the structural preconditions hold and the fields remain stable except for this function’s own writes:

- The pivot’s right child becomes the transferred subtree.
- If the transferred subtree is non-null, its parent becomes the pivot.
- The promoted node’s parent becomes the pivot’s former parent.
- The root link or former parent’s selected child link becomes the promoted node.
- The promoted node’s left child becomes the pivot.
- The pivot’s parent becomes the promoted node.

The pivot’s left-child field and the promoted node’s right-child field are not written. Other fields are unchanged except where storage aliases a field that is written.

### Branch precedence and malformed relationships

The root-link comparison takes precedence over parent-child inspection. When the anchor’s root link equals the pivot address, the function does not inspect the former parent’s child fields. It still reads the pivot’s parent field earlier and copies it to the promoted node.

On the non-root path, only the former parent’s left-child field is checked. A mismatch causes an unconditional write to the former parent’s right-child field. The function does **not** verify that the right-child field previously selected the pivot.

Consequently, an inconsistent incoming relationship is not rejected. The function may overwrite an unrelated right-child link.

The promoted address is not tested for zero before its fields are accessed. A pivot with no right child is therefore outside the ordinary successful-rotation preconditions.

### Widths and address calculations

The observed link values, comparisons, and stores are 32 bits wide. Zero is tested only for the promoted node’s left-child link.

All field displacements are byte offsets. There are no counters, size calculations, payload arithmetic, or signed ordering comparisons. The function performs address calculations using the supplied or loaded values and fixed offsets.

The source does not establish portable behavior for overflowing address calculations, invalid addresses, or misaligned accesses. A contract for well-formed use requires each accessed field to be valid on the target platform; it does not rely on address wrapping.

## Side effects

The function mutates existing storage. It makes no visible allocation, deallocation, ownership-transfer, callback, logging, or external-service operation.

For distinct, well-formed objects, it performs five unconditional 32-bit writes:

- Pivot right-child field.
- Promoted node parent field.
- One incoming-link field.
- Promoted node left-child field.
- Pivot parent field.

It performs one additional 32-bit write when the promoted node’s left-child link is nonzero: that node’s parent field is changed to the pivot.

There is no rollback. Intermediate states are structurally incomplete: the pivot’s right link changes before the incoming link is redirected, and the promoted node’s left link changes before the pivot’s parent link is finalized. The source provides no locking, atomic publication, or synchronization protocol.

If execution fails partway through, earlier writes may already have occurred. No explicit failure result or recovery path is present.

No payload is copied. Ownership and lifetime obligations for all addressed objects remain outside the established contract.

## Dependencies

No function calls are visible in F-bff5bbe8ab77. There are therefore no observed callable dependencies requiring additional opaque function IDs.

Its direct dependencies are the caller-provided addresses and the memory fields reached through them. The function relies on those fields being accessible and structurally suitable; it does not validate that dependency.

No behavior of an unknown callee is assumed.

## Acceptance examples

These examples are conditional deductions from the source, not runtime observations. They assume distinct valid objects, stable memory, and no interfering access.

### Root pivot with a non-null transferred subtree

Initially:

- The anchor’s root link selects the pivot.
- The pivot’s right child is the promoted node.
- The promoted node’s left child is a third node.
- The pivot’s parent field contains the structure’s root-parent representation.

After return:

- The anchor’s root link selects the promoted node.
- The promoted node’s parent contains the pivot’s former parent value.
- The promoted node’s left child is the pivot.
- The pivot’s parent is the promoted node.
- The pivot’s right child is the third node.
- The third node’s parent is the pivot.

The root-parent representation is copied, not normalized or validated.

### Non-root pivot selected by its parent’s left link

Initially, the anchor’s root link does not select the pivot, and the pivot’s parent selects it through its left-child field.

After return, that parent’s left-child field selects the promoted node. The parent’s right-child field is not written. The remaining link changes follow the rotation described above.

### Non-root pivot selected by its parent’s right link

Initially, the anchor’s root link does not select the pivot, the parent’s left-child field differs from the pivot, and its right-child field selects the pivot.

After return, the parent’s right-child field selects the promoted node. The parent’s left-child field is unchanged.

### Empty transferred subtree

Initially, the promoted node’s left-child field is zero.

After return, the pivot’s right-child field is zero. No transferred-subtree parent-field write occurs. The other five writes still occur.

### Inconsistent parent links

Initially, the anchor’s root link differs from the pivot, and neither child field of the pivot’s recorded parent selects the pivot.

The function does not signal this inconsistency. Provided the accesses remain valid, it overwrites that parent’s right-child field with the promoted address.

### Missing promoted node

Initially, the pivot’s right-child field is zero.

There is no null-child early return. The function attempts to read the field at offset `+0x08` from the zero promoted address. Successful completion is not promised.

## Uncertainties

- **Inferred intent:** The operation is a left rotation. Tree membership, search ordering, balancing rules, and the larger data structure’s identity are not established.
- **documented_only — Anchor semantics:** The anchor’s `+0x04` field behaves as a root link here. Whether the anchor is a header, sentinel, or another structure is unknown.
- **documented_only — Root-parent representation:** The function copies this value without establishing whether zero, a sentinel, or another representation is required.
- **documented_only — Aliasing:** No rejection of coincident or overlapping objects is visible. Such overlap can affect later reads and final contents; the ordinary rotation guarantees require non-overlapping participating fields.
- **documented_only — Platform behavior:** Endianness, alignment requirements, memory-fault handling, and address-overflow behavior are not established.
- **documented_only — Concurrency:** No synchronization contract is supplied. Repeated reads cannot be assumed to observe a single coherent snapshot.
- **documented_only — Ownership:** Allocation origin, lifetime guarantees, and responsibility for eventual release are unknown.

## Evidence

Established static evidence consists solely of the supplied body of F-bff5bbe8ab77: fixed-offset 32-bit accesses, zero and address-equality tests, conditional incoming-link selection, and the ordered writes described above.

The left-rotation interpretation follows from those link relationships; it is not supported by an independent type definition or surrounding API contract.

No implementation access beyond the supplied source, external dependency analysis, compilation, runtime execution, or tests were performed. The acceptance examples are conditional static predictions, not verified runtime results.
