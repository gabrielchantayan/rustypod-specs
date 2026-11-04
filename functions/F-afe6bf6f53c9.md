<!-- Trusted provenance: F-afe6bf6f53c9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afe6bf6f53c9

## Purpose

F-afe6bf6f53c9 performs a local pointer-link transformation consistent with a left rotation in a parent-linked binary tree. It promotes the designated node’s right child, transfers that child’s left subtree to the designated node’s right side, reconnects the promoted node to the former incoming link, and places the designated node beneath the promoted node as its left child.

The link transformation is established by the supplied static representation. The interpretation as a tree rotation is inferred intent: no type declarations, tree invariants, or caller contracts are available.

The operation does not establish whether the structure is a valid tree before or after the transformation. In particular, it does not check that the promoted node exists, that parent links agree with child links, or that the nodes are distinct.

## Interface

### Inputs

The function takes two values represented as 32-bit integers in the supplied material:

| Input | Required interpretation |
|---|---|
| Control address | Address of a control structure whose field at byte offset 16 contains another address. |
| Pivot address | Address of the node being rotated downward. |

The implementation uses these integer values as addresses. The intended address-bearing fields are four bytes wide. Pointer signedness, native pointer width, address-space rules, and the original declared types are not established.

### Referenced layout

The following names describe roles, not recovered type definitions.

| Object | Byte offset | Width | Role in this operation |
|---|---:|---:|---|
| Control structure | 16 | 4 bytes | Address of a root-link holder |
| Root-link holder | 4 | 4 bytes | Root link compared with the pivot |
| Node | 4 | 4 bytes | Parent link |
| Node | 8 | 4 bytes | Left-child link |
| Node | 12 | 4 bytes | Right-child link |

No object size beyond the accessed fields is established. No meaning is established for other offsets.

The **promoted node** is selected by the initial read of the pivot’s right-child field. The **transferred subtree** is the promoted node’s left child under ordinary, non-overlapping, stable-memory conditions.

Zero is explicitly treated as an absent link only when examining the promoted node’s left child. There is no corresponding check for a zero pivot, promoted-node address, control address, root-link-holder address, or required parent address.

### Output

There is no explicit return value or status result. Successful completion is represented by the resulting memory updates.

No error code, exception translation, rollback, or diagnostic output appears in the supplied representation.

## Behavior

### Normal structural result

For a valid, non-overlapping tree representation without concurrent modification:

- The promoted node replaces the pivot at its former incoming link.
- The promoted node receives the pivot’s former parent.
- The pivot’s right child becomes the promoted node’s former left child.
- If that transferred child is nonzero, its parent becomes the pivot.
- The promoted node’s left child becomes the pivot.
- The pivot’s parent becomes the promoted node.

The incoming link is selected using the root-link-holder comparison first. If that comparison does not identify the pivot as the root, the function examines the pivot parent’s left-child link. It uses the parent’s right-child link as the fallback destination without verifying that the right-child link previously pointed to the pivot.

The normal tree interpretation also implies that the pivot’s left subtree and the promoted node’s right subtree retain their existing links. This preservation follows from the absence of direct writes to those fields; it depends on the fields not overlapping other accessed storage.

### Ordered access requirements

The supplied representation establishes the following logical access order. This is a source-level account, not a claim about processor instructions or memory-ordering guarantees.

**Selection and transfer.** The function first reads the pivot’s right-child field and retains that value as the promoted-node address. It then reads the promoted node’s left-child field and writes that value into the pivot’s right-child field.

After that write, it reads the promoted node’s left-child field again to test whether it is nonzero. If the test is nonzero, it reads that field once more to obtain the address whose parent field will be written. That parent field receives the pivot address.

These are separate reads in the supplied representation. The transferred-child parent update does not explicitly use a saved copy of the value already written into the pivot’s right-child field.

**Parent-link preparation.** Next, the function reads the pivot’s parent field and copies it into the promoted node’s parent field.

**Incoming-link selection.** Only after the preceding mutations does the function read the control structure’s field at offset 16. The resulting address identifies the root-link holder. It then reads the holder’s field at offset 4 and compares that value with the pivot address.

- If equal, the holder’s root-link field is overwritten with the promoted-node address.
- If unequal, the function reads the pivot’s parent field again. It reads that parent’s left-child field and compares it with the pivot address.
  - If equal, the parent’s left-child field receives the promoted-node address.
  - Otherwise, the parent’s right-child field receives the promoted-node address, regardless of its previous value.

The later read of the pivot’s parent field is distinct from the earlier read used to populate the promoted node’s parent field.

**Final connection.** After replacing the incoming link, the function writes the pivot address into the promoted node’s left-child field. Its final write places the promoted-node address in the pivot’s parent field.

### Preconditions for the intended tree transformation

The following are necessary assumptions for the ordinary rotation interpretation, not conditions checked by the function:

- All addresses used by the selected path refer to readable or writable storage as required.
- Each accessed field supports the four-byte operation shown in the representation.
- The pivot has a valid, nonzero right child.
- A nonzero promoted-node left child has a writable parent field.
- The control structure identifies a valid root-link holder.
- For a non-root pivot, its parent is valid and has the pivot as one of its children.
- Nodes, control storage, and root-link storage do not overlap in ways that alter the intended field relationships.
- Relevant links remain stable while the operation runs.

For a root pivot, the function still reads the pivot’s parent and copies it into the promoted node’s parent before selecting the root-update branch. It does not force the new root’s parent to zero or to a sentinel.

### Aliasing and changing memory

The normal before-and-after description is insufficient if fields overlap or another actor changes them.

For example, the promoted node’s left-child value is read both before and after the pivot’s right-child field is written. Aliasing could make those reads differ even without concurrency. Likewise, the root-link-holder address is obtained after several writes; it is not captured at function entry.

The ordered accesses above therefore take precedence over the simplified tree interpretation for unusual layouts. No snapshot consistency or atomic rotation is established.

## Side effects

The function writes four-byte address values to a path-dependent set of fields:

| Destination | Write condition |
|---|---|
| Pivot right-child field | Always reached before incoming-link selection |
| Parent field reached through a fresh promoted-node left-child read | When the preceding left-child test is nonzero |
| Promoted-node parent field | Always reached after the optional transferred-child update |
| Root-link-holder root field | When its current value equals the pivot |
| Pivot parent’s left-child field | On the non-root branch when that field equals the pivot |
| Pivot parent’s right-child field | On the remaining non-root branch |
| Promoted-node left-child field | After the incoming-link replacement |
| Pivot parent field | Final write |

On a completing path, there are five writes when the transferred-child update is skipped and six when it is taken. These counts describe write operations, not necessarily distinct memory locations.

The operation mutates existing storage. It contains no visible allocation, deallocation, ownership-transfer API, or callback. Reparenting a node does not by itself establish a change in allocation ownership or lifetime responsibility.

No visible synchronization, explicit I/O, or global-symbol access occurs. Writes through the supplied addresses may nevertheless affect shared or globally reachable storage.

An invalid access may occur after earlier writes have completed. Nothing in the supplied representation restores those earlier values.

## Dependencies

No function calls are visible. No additional opaque dependency ID is required.

The operation depends directly on externally supplied storage and link invariants. The allocation, initialization, lifetime, and ownership rules for that storage are unknown.

There is no basis for assuming external synchronization or a surrounding transaction merely because the function resembles a standard tree primitive.

## Acceptance examples

These examples are conditional acceptance criteria derived from the static representation. They have not been executed. Unless stated otherwise, assume distinct valid objects, stable memory, and sufficient access permissions.

### Root rotation with a transferred subtree

Initially:

- The root link identifies the pivot.
- The pivot’s right child is the promoted node.
- The promoted node’s left child is a third node.
- The pivot’s parent is a designated sentinel.

On completion:

- The root link identifies the promoted node.
- The promoted node’s parent is the sentinel.
- The pivot’s right child is the third node.
- The third node’s parent is the pivot.
- The promoted node’s left child is the pivot.
- The pivot’s parent is the promoted node.

The sentinel value is copied, not interpreted or normalized.

### Non-root pivot on its parent’s left

Initially, the root link does not identify the pivot, and the pivot parent’s left-child link identifies the pivot.

On completion, that left-child link identifies the promoted node. The parent’s right-child field is not directly written. The subtree-transfer and final parent-child relationships follow the normal structural result.

### Non-root pivot on its parent’s right

Initially, the root link does not identify the pivot, the parent’s left-child link differs from the pivot, and the parent’s right-child link identifies the pivot.

On completion, the parent’s right-child link identifies the promoted node. Its left-child field is not directly written.

### Empty transferred subtree

Initially, the promoted node’s left-child field is zero and remains stable through its repeated reads.

On completion, the pivot’s right-child field is zero. No transferred child’s parent field is written. The promoted node still replaces the pivot and receives the pivot as its left child.

### Inconsistent incoming relationship

Initially, the root link differs from the pivot, and neither child field of the recorded parent identifies the pivot.

The function nevertheless overwrites that parent’s right-child field with the promoted-node address. It does not report the inconsistency. A contract requiring rejection of this malformed relationship would not match the supplied behavior.

### Root with a nonstandard parent value

Initially, the root link identifies the pivot, but the pivot’s parent contains a nonzero value that is not a valid node address.

Provided the accessed storage is otherwise valid and non-overlapping, the root branch copies that value into the promoted node’s parent without dereferencing it as a parent node. The final write replaces the pivot’s parent with the promoted-node address.

### Missing promoted node

Initially, the pivot’s right-child value is zero.

The function attempts an access at byte offset 8 from that value before any explicit rejection. No portable completion, diagnostic, or failure result is specified for this input.

## Uncertainties

### Inferred intent

- The transformation is a left rotation.
- The field at offset 4 represents a parent link.
- The control-indirected field represents a root link.

These interpretations explain the access pattern but are not substitutes for the concrete behavior.

### documented_only unknowns

- Original parameter and field types, pointer signedness, and native pointer width.
- Alignment requirements and behavior for unaligned accesses.
- Whether zero-relative addresses can designate accessible storage.
- Address-arithmetic rules near representation boundaries.
- Actual fault, signal, or exception behavior for invalid accesses.
- Compiler treatment of overlapping objects and data races.
- Whether callers guarantee a valid tree, sentinel conventions, or mutual exclusion.
- Ownership and lifetime obligations for all referenced objects.
- Any higher-level balancing, ordering, coloring, or metadata invariants.

Offsets are byte displacements of 4, 8, 12, and 16. No counters or sizes are computed. The four-byte access evidence does not establish that overflowing address arithmetic wraps safely.

## Evidence

The sole basis is the supplied static function representation. It shows two address-like integer inputs, four-byte field accesses, a conditional transferred-child parent update, root-versus-parent incoming-link selection, and the final reciprocal pivot/promoted-node connection.

No runtime tests, compilation, caller inspection, binary verification, or independent type recovery have been performed. Acceptance examples are deductions under their stated assumptions, not observed outcomes.
