<!-- Trusted provenance: F-f76ef3e16f1e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f76ef3e16f1e

## Purpose

F-f76ef3e16f1e performs an in-place pointer rearrangement consistent with a **right rotation in a parent-linked binary tree**.

For an ordinary, well-formed tree, the supplied node is the pivot. Its left child is promoted into its former position. The promoted node’s former right subtree becomes the pivot’s left subtree, and the pivot becomes the promoted node’s right child. Parent links and either the root slot or an incoming child link are updated accordingly.

The tree interpretation is inferred intent. The established behavior is a fixed sequence of 32-bit field reads, conditional writes, and link replacements. The routine does not validate that its inputs form a tree or that their relationships are mutually consistent.

There are no dependency calls, allocation operations, deallocation operations, or explicit error reports in the supplied body.

## Interface

The routine accepts two address-bearing arguments and has no explicit return value:

- **Context address:** identifies an object containing a link to a separate root-bearing object.
- **Pivot address:** identifies the node whose links are rearranged.

The supplied representation uses 32-bit integer arguments and 4-byte fields. Values stored in the relevant fields are used as addresses or as zero-valued absent links. Their representation as integers does not establish meaningful signed ordering.

### Accessed layout

Only the following fields are established. No interpretation is established for other bytes.

| Object | Byte offset | Width | Role |
|---|---:|---:|---|
| Context | +16 | 4 bytes | Address of root-bearing object |
| Root-bearing object | +4 | 4 bytes | Root link |
| Node | +4 | 4 bytes | Parent link |
| Node | +8 | 4 bytes | Left-child link |
| Node | +12 | 4 bytes | Right-child link |

The **promoted node** is the address obtained from the pivot’s left-child field on entry. That address is retained for the rest of the operation; subsequent changes to the pivot’s left-child field do not select a different promoted node.

The **transfer subtree** is the promoted node’s right-child link. In stable, nonoverlapping memory, it has one consistent value throughout the transfer phase. The implementation reads that field more than once, so this consistency is a precondition for the simple tree-level description, not a concurrency guarantee.

### Preconditions for ordinary tree behavior

The intended rotation requires:

- A readable context field at offset +16.
- A valid root-bearing object with a readable root slot, and a writable root slot if the pivot is the recorded root.
- A readable and writable pivot node.
- A nonzero pivot left-child link identifying a readable and writable promoted node.
- A writable parent field in the transfer subtree’s root, if that subtree link is nonzero.
- When the pivot is not the recorded root, a valid parent node with accessible child fields.
- Consistent parent/child relationships and no overlapping storage that changes the meaning of later accesses.
- No concurrent modification of the links involved.

These conditions are not checked. In particular, the routine does not test whether the pivot or its left child is zero before dereferencing them.

The source does not establish alignment requirements, byte order, address-space validity rules, or behavior on an invalid access.

## Behavior

### Tree-level result

For a well-formed, stable tree:

1. The pivot’s left child is replaced by the promoted node’s former right child.
2. If that transferred child exists, its parent becomes the pivot.
3. The promoted node inherits the pivot’s former parent link.
4. The link that previously selected the pivot—either the root slot or one child field in the pivot’s parent—is redirected to the promoted node.
5. The promoted node’s right child becomes the pivot.
6. The pivot’s parent becomes the promoted node.

The pivot’s right-child field and the promoted node’s left-child field are not directly written. The operation does not access keys, payloads, colors, heights, balance factors, counts, or allocation metadata.

Under the ordinary tree assumptions, the rearrangement preserves in-order node ordering. That property follows from the rotation structure; the routine does not inspect or compare any ordering keys.

### Observable access ordering

The operation is not an atomic replacement of one tree shape with another. Its writes occur in an order that temporarily leaves links inconsistent.

**Subtree transfer comes first.** The promoted node is selected by reading the pivot’s left-child field. The promoted node’s right-child field is then read, and that value is written into the pivot’s left-child field.

After this write, the promoted node’s right-child field is read again to determine whether a transfer subtree exists. If that read is nonzero, the field is read again to obtain the address whose parent field will be written. That parent field receives the pivot address.

Consequently, the value installed as the pivot’s left child and the address used for the transfer-subtree parent update need not agree if the field changes between reads. The routine provides no synchronization or snapshot semantics.

**The promoted node’s parent is established next.** The pivot’s parent field is read and copied into the promoted node’s parent field. This precedes the root decision.

**The incoming link is then selected and replaced.** Only after those writes does the routine read the context’s root-bearing-object address and then read that object’s root slot.

- If the root slot equals the pivot address, the root slot is overwritten with the promoted-node address.
- Otherwise, the pivot’s parent field is read again. The right-child field of that parent is read and compared with the pivot address:
  - Equality selects the parent’s right-child field for replacement.
  - Inequality selects the parent’s left-child field for replacement.

There is no additional check that the selected left-child field actually contains the pivot. Root status is determined by root-slot equality, not by whether the pivot’s parent is zero.

**The pivot/promoted relationship is completed last.** The promoted node’s right-child field is overwritten with the pivot address. The final write sets the pivot’s parent field to the promoted-node address.

For stable, distinct objects, this ordering has the following consequences:

- The transfer subtree’s parent is repaired before the promoted node is linked into the pivot’s former position.
- The root or former parent may point to the promoted node before the promoted node points back to the pivot as its right child.
- The pivot’s parent field is not updated to the promoted node until the last write.

### Selection behavior on inconsistent inputs

The branch rules are mechanical rather than validating:

- Root-slot equality always selects the root-update branch, even if the pivot’s parent field is nonzero.
- If the pivot is not the recorded root, its parent is dereferenced without a preceding zero check.
- In the non-root branch, a matching right-child link takes precedence.
- A nonmatching right-child link causes the left-child link to be overwritten without checking its previous value.

These rules describe which fields are accessed. They do not make malformed structures supported inputs.

## Side effects

All successful effects are in-place 4-byte field writes.

On an ordinary invocation, the routine writes:

- The pivot’s left-child field.
- The transfer subtree root’s parent field, only when the tested transfer link is nonzero.
- The promoted node’s parent field.
- Exactly one selected incoming-link field: the root slot, the former parent’s right-child field, or the former parent’s left-child field.
- The promoted node’s right-child field.
- The pivot’s parent field.

This amounts to five field writes when no transfer subtree is present, or six when one is present. Distinct write operations need not address distinct storage under malformed or overlapping layouts.

The context’s field at offset +16 is read but is not directly written. There are no direct accesses to global variables in the supplied body. Objects reached through the arguments could nevertheless be shared or globally reachable; isolation is not established.

No ownership transfer or lifetime change is explicitly performed. In particular:

- No node is created or released.
- No payload is copied.
- No destructor or callback is invoked.
- Unlinking a field does not cause memory reclamation.

There is no rollback mechanism. If execution is interrupted or an access fails, earlier writes may already have occurred. The source supplies no error-return channel and does not establish the runtime’s handling of such failures.

## Dependencies

There are **no called-function dependencies** in the supplied body. No additional opaque function IDs are needed.

Successful execution depends on the surrounding memory model and the validity of the objects reachable from the two arguments. The evidence does not establish locks, atomic operations, barriers, address validation, exception handling, or caller-side synchronization.

## Acceptance examples

These examples are conditional specifications, not reports of executed tests. Unless stated otherwise, assume valid, distinct objects, stable memory, and internally consistent tree links.

### Pivot is the root; transfer subtree exists

Initially:

- The root slot identifies the pivot.
- The pivot’s parent is zero.
- The pivot’s left child is the promoted node.
- The promoted node’s right child is a transfer subtree root.

After normal completion:

- The root slot identifies the promoted node.
- The promoted node’s parent is zero.
- The promoted node’s right child is the pivot.
- The pivot’s parent is the promoted node.
- The pivot’s left child is the transfer subtree root.
- The transfer subtree root’s parent is the pivot.
- The pivot’s former right child and the promoted node’s former left child are unchanged.

### Pivot is its parent’s right child; no transfer subtree

Initially, the root slot does not identify the pivot, the pivot’s parent identifies an ancestor, that ancestor’s right child is the pivot, and the promoted node’s right-child link is zero.

After normal completion:

- The ancestor’s right child is the promoted node.
- The promoted node’s parent is the ancestor.
- The pivot’s left-child link is zero.
- No transfer-subtree parent write occurs.
- The promoted node’s right child is the pivot.
- The pivot’s parent is the promoted node.
- The root slot is unchanged.

### Pivot is its parent’s left child

If the root slot differs from the pivot and the parent’s right child differs from the pivot, the parent’s left-child field is selected for replacement.

For a consistent tree where that left child is the pivot, the result is the ordinary non-root right rotation. The parent’s right-child field remains unchanged.

### Parent links do not identify the pivot

Suppose the root slot differs from the pivot and neither child of the pivot’s recorded parent identifies the pivot. Provided every accessed address remains valid, the routine still overwrites that parent’s left-child field with the promoted-node address. It does not reject the inconsistency.

### Recorded root has a nonzero parent

If the root slot identifies the pivot but the pivot’s parent is nonzero, the root branch is still taken. The promoted node receives that nonzero parent value, and no former-parent child slot is repaired through the non-root branch. The routine does not enforce a zero parent for roots.

### Missing promoted node

A zero pivot left-child link does not produce a defined no-op or a reported failure. The routine proceeds to access the promoted node’s right-child field through that zero-valued base. Whether this faults or accesses mapped storage is outside the established contract.

## Uncertainties

**documented_only — Representation details.** The supplied representation establishes 4-byte accesses and the listed byte offsets. It does not independently establish endianness, alignment rules, or a complete object layout.

**documented_only — Address arithmetic.** The routine forms field addresses using fixed byte displacements. No bounds or overflow checks are visible. Intended inputs must make those addresses valid. The evidence does not justify promising either portable signed-overflow behavior or a particular address-wraparound outcome.

**documented_only — Aliasing.** No overlap checks exist. If the context, root-bearing object, or nodes overlap, an early write can alter a later input. The ordinary rotation postconditions require noninterfering storage; the source provides no separate alias-safe behavior.

**documented_only — Concurrency and interruption.** Repeated field reads and staged writes are visible, but no synchronization contract is supplied. Concurrent mutation, observation of intermediate states, and interruption handling remain unspecified.

**documented_only — Higher-level invariants.** Binary-tree use and right-rotation intent are strongly supported by the link pattern. Search-tree ordering, balancing policy, ownership rules, and permitted callers are not established by this body.

## Evidence

The established static evidence is limited to the supplied body for F-f76ef3e16f1e:

- Two address-bearing arguments and no explicit return value.
- Four-byte field accesses at the documented offsets.
- Promotion of the pivot’s left-linked node.
- Transfer of that node’s right link to the pivot’s left link.
- Conditional repair of the transferred child’s parent.
- Root-slot equality followed, when necessary, by right-child equality to select the incoming link.
- Final writes connecting the promoted node and pivot.
- No function calls, explicit allocation, explicit release, validation path, or error-reporting path.

The right-rotation interpretation and preservation of in-order ordering are inferred from that structure under valid-tree preconditions. They are not runtime observations.

No runtime tests have been performed.
