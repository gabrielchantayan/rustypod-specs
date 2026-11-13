<!-- Trusted provenance: F-c63d7eb9eac2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c63d7eb9eac2

## Purpose

F-c63d7eb9eac2 conditionally removes a selected node from a pointer-linked tree, performs color-dependent structural repair, and places the removed node on a container-managed linked list. It writes a one-word result through a caller-supplied output address.

The apparent intent is an iterator-based erase operation on a red-black-tree-like container, returning an advanced position and recycling the erased node. That interpretation is inferred from the links, color operations, rotations, and final list insertion. The established behavior is narrower: a dependency gates entry, another dependency updates a saved position, and the function then performs the specific link and byte mutations described below.

No key comparison, payload copy, explicit allocation, explicit deallocation, or payload destruction appears in the supplied body. Dependencies may have additional effects; their implementations are not supplied.

## Interface

The function has three arguments and no explicit return value:

| Argument | Established role |
|---|---|
| Output address | Writable address receiving one 32-bit word at normal completion. |
| Container address | Address of container state, including a list head, a tree-header pointer, and a count. |
| Position address | Readable address containing a 32-bit node or position representation. It is read initially and read again after dependency calls on the removal path. |

The representation uses 32-bit words and pointers. Color accesses are one byte wide. The supplied body does not establish byte order, alignment requirements, or the calling convention.

### Container layout

Offsets are measured in bytes from the container address.

| Offset | Width | Observed use |
|---|---:|---|
| +4 | 32 bits | Head of the list onto which the removed node is placed. |
| +16 | 32 bits | Pointer to the tree header. |
| +20 | 32 bits | Count, decremented once after removal and repair. |

Other container fields are not directly accessed here.

### Node and header layouts

Ordinary nodes expose:

| Offset | Width | Observed use |
|---|---:|---|
| +0 | 8 bits | Color or repair-state byte. |
| +4 | 32 bits | Parent pointer. |
| +8 | 32 bits | Left-child pointer. |
| +12 | 32 bits | Right-child pointer while in the tree; reused as a next pointer on the removed-node list. |

Bytes +1 through +3 and any payload beyond this prefix are not directly modified.

The header uses the same pointer offsets differently:

- +4: root pointer.
- +8: cached leftmost node.
- +12: cached rightmost node.

An empty tree is represented by a null root and both cached extrema pointing to the header itself.

### Preconditions

The caller must supply accessible storage for the arguments and every address traversed on the executed path. On the removal path, the selected node must have coherent parent and child links and belong to a structure compatible with the header.

The body does not validate membership, detect cycles, check the count, or reject malformed pointers. It has no explicit error return for these conditions. Invalid structure can cause invalid memory access, nontermination, or partial mutation.

No non-aliasing guarantee is established. In particular, output storage is written last, so aliasing it with container or node storage can overwrite state already updated by the operation.

## Behavior

### 1. Gate the operation and prepare the result

The observable order begins as follows:

1. Read the 32-bit value at the position address into a temporary result slot.
2. Read the header pointer at container offset +16 into a separate temporary slot.
3. Call F-c7e5bb3c4e39 with the addresses of those two slots.
4. Test its returned integer for zero versus nonzero.

Both temporary slots are passed by address. Their contents after the call cannot be assumed unchanged.

If the return is nonzero, the function:

- Rereads the current header pointer from container offset +16.
- Replaces the temporary result with that pointer.
- Writes that result to the output address.
- Performs no direct tree mutation, list insertion, or count decrement.

This branch is consistent with rejecting an end/header position, but the dependency’s actual predicate is unknown. Its own side effects are not excluded.

If the return is zero, the function calls F-350c3f9b3126 with the address of the temporary result slot. The resulting slot value is retained for the final output. It is plausibly an advanced iterator, but that meaning is not established by this body alone.

Only after these calls does the function reread the position address to obtain the node to remove. Consequently, the removal target is not necessarily the value initially captured if a dependency changes reachable state.

### 2. Select the structural removal case

The function first reads the selected node’s left child.

- If the left child is null, it reads the right child.
  - A nonnull right child becomes the replacement.
  - If both children are null, it takes the leaf-removal path.
- If the left child is nonnull, it reads the right child.
  - If the right child is null, the left child becomes the replacement.
  - If both children exist, it follows left links from the right child until reaching the leftmost node of that subtree. This node is the structural successor; its right child is the initial replacement.

These traversals have no cycle or depth guard.

### 3. Remove a leaf

For a leaf, the repair position is initially the selected node’s parent.

If that parent equals the current header pointer, the function treats the selected node as the only root:

1. Set the header’s rightmost link to the header.
2. Set the header’s leftmost link to the header.
3. Clear the header’s root link.

Otherwise, the function reads the parent’s left-child link to decide which child slot to clear:

- If it equals the selected node, clear the left-child slot.
- Otherwise, clear the right-child slot.

Each such child-slot clear is expressed as four separate zero-byte writes covering that 32-bit field. A replacement contract requiring only final pointer equality would omit this observable store granularity.

The function then examines the corresponding cached extremum:

- Clearing a left-child slot checks the header’s leftmost cache.
- Clearing a right-child slot checks the header’s rightmost cache.

If that cache points to the selected node, it is replaced with the parent. The opposite cache is not updated on this non-root leaf path.

### 4. Replace a node having one child

When the selected node itself is the structural removal node and has a nonnull replacement:

1. Write the selected node’s parent into the replacement’s parent field.
2. Replace the header’s root link if it names the selected node.
3. Otherwise, replace the appropriate child link in the selected node’s parent, using equality with the parent’s left child to choose the side.

The function then updates extrema if necessary:

- If the selected node was the cached leftmost:
  - When its right child is null, use its parent.
  - Otherwise, follow left links from the replacement to obtain the new leftmost.
- If the selected node was the cached rightmost:
  - When its left child is null, use its parent.
  - Otherwise, follow right links from the replacement to obtain the new rightmost.

The removed node’s former links are not generally cleared.

### 5. Transplant a successor for a node having two children

For a two-child target, the function moves the structural successor into the target’s tree position. It does not copy payload data into the target.

The structural writes occur in this order:

1. Make the target’s left child name the successor as its parent.
2. Install that left child as the successor’s left child.
3. Handle the successor’s old position:
   - If the successor is the target’s immediate right child, update its nonnull right child’s parent to the successor.
   - Otherwise:
     - Update the successor’s nonnull right child’s parent to the successor’s former parent.
     - Replace that former parent’s left-child link with the successor’s right child.
     - Install the target’s right subtree as the successor’s right child.
     - Update that subtree root’s parent to the successor.
4. Replace the target in the header’s root link or in its parent’s child link.
5. Install the target’s former parent as the successor’s parent.
6. Exchange the successor’s and target’s one-byte color values.

If the successor’s original right child was null, the repair position is changed to the transplanted successor itself. Otherwise, that right child remains the repair position.

The original target remains the node ultimately placed on the container’s removed-node list. Its color after the exchange is the successor’s original color.

### 6. Perform conditional color repair

Repair is entered only when:

- The repair position is not the current header; and
- The removed node’s current color byte is nonzero.

For a two-child removal, this byte is the successor’s original byte because of the preceding exchange.

The repair loop continues only while the repair position is not the current root and its own byte equals exactly 1. Thus, the entry test treats every nonzero removed-node byte alike, but the loop distinguishes exactly 1 from other values.

The conventional interpretation is 0 = red and 1 = black. That naming is inferred; the byte comparisons and assignments are established.

At each iteration, the function obtains the repair position’s parent and determines whether the position is the parent’s left child.

For a left-side position, the sibling is the parent’s right child. For a right-side position, the sibling is the parent’s left child. The two cases are broadly mirrored:

- **Missing sibling:** write 0 to the current repair position, move the position to its parent, and reconsider the loop.
- **Sibling byte is 0:** write 1 to the sibling and 0 to the parent; invoke the direction-appropriate dependency; then reload the parent and sibling through the repair position.
- **Both sibling children absent or byte 1:** write 0 to the sibling, move the repair position to its parent, and reconsider the loop.
- **Outer sibling child absent or byte 1, with the earlier condition not satisfied:** write 1 to the nonnull inner child, write 0 to the sibling, invoke the opposite-direction dependency, and reload the sibling.
- **Final repair:** copy the parent’s byte into the sibling, write 1 to the parent, write 1 to the nonnull outer sibling child, invoke the direction-appropriate dependency, and finish repair.

Here, “outer” means the sibling’s right child for a left-side repair position and its left child for a right-side position.

If a sibling reload after the inner-child adjustment yields null, the function finishes repair without the final sibling recoloring and final rotation call.

On every exit from an entered repair block, the current repair position’s byte is set to 1. If repair was skipped entirely, this final byte write does not occur.

These details must not be silently replaced with a generic textbook deletion algorithm. In particular, null-child handling and the selected repair position are specific to the supplied body.

### 7. Link the removed node and publish the result

After structural changes and any repair:

1. Read the current list head at container offset +4.
2. Store it in the removed node’s +12 field.
3. Store the removed node’s address as the new list head.
4. Read the 32-bit count at container offset +20 and decrement it by one.
5. Write the temporary result slot to the output address.

The count operation uses a 32-bit field and includes no lower-bound check. No saturation, count-validity error, or wider arithmetic is present. Under modulo-32-bit machine arithmetic, decrementing zero produces all one bits; portable signed-overflow semantics are not established by the supplied representation.

## Side effects

Direct effects may include:

- Changing the root, cached extrema, parent links, and child links.
- Changing multiple one-byte color fields.
- Reusing the removed node’s right-link field as a list-next field.
- Prepending exactly the selected target node to the container list.
- Decrementing the count once on the removal path.
- Writing one 32-bit output value at normal completion.

The removed node is not explicitly released to an allocator. Its list insertion suggests retention for reuse, but the list’s ownership and eventual disposal policy are unknown.

The function offers no rollback. Tree mutation precedes list insertion, count decrement, and output publication. Failure during a dependency or invalid access may leave a partially changed structure.

Header and parent links are reread at several points rather than universally cached. Dependencies must not be assumed to preserve global state or reachable memory. Concurrent access, atomicity, synchronization, and reentrancy guarantees are not established.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-c7e5bb3c4e39 | Called first with addresses of two 32-bit temporary slots. Nonzero selects the no-direct-removal branch. | Predicate meaning, slot mutation, other effects, failure behavior. |
| F-350c3f9b3126 | Called only after a zero gate result, with the result-slot address. | Whether it advances an iterator, traversal rules, memory ownership, other effects. |
| F-6e50c907f5e4 | Used during repair in positions consistent with leftward rotation. Several calls visibly supply the container and a parent value; one supplies only the container. | Exact signature, omitted-argument interpretation, full mutation set. |
| F-58326c7630ef | Used during repair in positions consistent with rightward rotation. Several calls visibly supply the container and a parent value; one supplies only the container. | Exact signature, omitted-argument interpretation, full mutation set. |

The inconsistent visible argument counts prevent an unconditional specification of the rotation helpers’ interfaces. Symmetry alone does not justify inventing a missing operand.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Gate returns nonzero.**  
   If F-c7e5bb3c4e39 returns nonzero, output must receive the header pointer reread after that call. F-350c3f9b3126 is not called. This function performs no direct list or count mutation.

2. **Remove the sole node.**  
   Given a leaf whose parent is the header, a zero gate result, and dependencies that leave the relevant structure unchanged, completion leaves the root null and both extrema equal to the header. The node becomes the list head and the count decreases once.

3. **Remove a leftmost non-root leaf.**  
   Given a coherent left-child leaf that is the cached leftmost, unlinking clears its parent’s left slot and replaces the leftmost cache with the parent. Later repair may additionally alter links and colors.

4. **Remove a root with only a right child.**  
   Before repair, the right child has the header as parent and becomes the root. If the target was the cached leftmost, that cache is recomputed by following the replacement’s left links. The original target is the node subsequently linked onto the removed-node list.

5. **Remove a two-child node with a deeper successor.**  
   The successor is detached from its old parent’s left slot and transplanted into the target’s position. Its nonnull former right child receives its former parent. The target, not the successor, enters the removed-node list. No direct payload copy occurs.

6. **Missing sibling during repair.**  
   If an iteration reaches a null sibling, the current repair position’s byte is set to 0 before repair moves upward. An implementation that merely skips a null sibling without this write does not match the supplied behavior.

7. **Result helper changes its temporary slot.**  
   If F-350c3f9b3126 writes a value into the result slot and no later dependency changes that slot indirectly, normal completion publishes that value. The output is not recomputed from the final root or list head.

## Uncertainties

The following are **documented_only**:

- The exact abstract container type and key ordering.
- The gate predicate and advancement semantics.
- Whether the selected representation is always a raw node pointer.
- Rotation-helper signatures and behavior at the visibly shorter calls.
- Allocation, payload destruction, and ultimate ownership of listed nodes.
- Valid color values beyond the observed 0 and 1.
- Guarantees for corrupted structures, aliasing, concurrency, or exceptional exits.
- The precise overflow model beyond the visible 32-bit decrement.

The evidence supports tree-erasure intent, not an unconditional proof of preservation of red-black invariants. Such a proof would require dependency contracts and valid-entry invariants.

## Evidence

Established static evidence consists solely of the supplied function body: access widths and offsets, branch conditions, dependency calls, pointer traversals, mutation order, color-byte operations, count decrement, and final output store.

Names such as “header,” “successor,” “rotation,” “iterator,” and “recycling list” describe structural interpretations. They do not establish unprovided callee behavior or ownership guarantees.

No runtime tests have been performed.
