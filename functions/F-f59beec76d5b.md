<!-- Trusted provenance: F-f59beec76d5b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f59beec76d5b

## Purpose

F-f59beec76d5b conditionally removes a node from a linked, colored binary-tree structure, places the detached node on a container-associated chain, decrements a 32-bit count, and writes a 32-bit result to caller-provided storage.

The supplied static representation establishes link manipulation, color-byte updates, dependency calls, and the final chain insertion. A red-black-tree erase operation with node recycling is the inferred intent. That interpretation does not establish the contracts of the dependencies, the meaning of node payloads, or ownership transfer.

A preliminary dependency call can bypass all tree-removal work. The output on that path is a freshly read container header reference. On the removal path, the output is a local 32-bit value previously passed by address to two dependencies; it is not necessarily the address of the detached node.

## Interface

### Arguments and result

The function has three arguments:

1. **Output slot:** address of writable storage for one 32-bit value.
2. **Container address:** a 32-bit address identifying the container record.
3. **Input slot:** address of readable storage containing one 32-bit value, ordinarily interpreted as a node reference.

There is no explicit return value. Normal completion writes exactly one 32-bit value to the output slot.

The input slot is read before the preliminary dependency call and, on the removal path, read again after the second dependency call. Those reads need not yield the same value if either dependency changes accessible state.

### Established record layout

All offsets below are byte offsets. References and link fields occupy 32 bits. Zero is used as the absent-child value.

| Record | Offset | Width | Observed use |
|---|---:|---:|---|
| Container | 4 | 32 bits | Head of a detached-node chain |
| Container | 16 | 32 bits | Header reference |
| Container | 20 | 32 bits | Count decremented after removal |
| Header | 4 | 32 bits | Root reference |
| Header | 8 | 32 bits | Cached lower endpoint |
| Header | 12 | 32 bits | Cached upper endpoint |
| Node | 0 | 8 bits | Color or balancing state |
| Node | 4 | 32 bits | Parent reference |
| Node | 8 | 32 bits | Left-child reference |
| Node | 12 | 32 bits | Right-child reference; reused for detached-chain linkage |

The endpoint interpretation follows the observed minimum/maximum traversals. Calling those endpoints the smallest and largest elements additionally assumes that the tree satisfies its ordering invariant.

The function tests color values against zero and one and writes both values. The balancing pattern is consistent with zero meaning red and one meaning black. Other byte values are not validated and do not behave uniformly across the tests.

No payload layout, complete record size, alignment rule, or byte order is established here. The accesses require the referenced storage to support the indicated widths.

### Preconditions

For defined, useful removal behavior:

- The output slot and container fields used by the selected path must be writable.
- The input slot must be readable at both reads performed on the removal path.
- The header and all traversed nodes must remain accessible.
- Parent/child links must be coherent, and traversed child chains must terminate.
- The selected node must belong to the structure described by the container.
- The color and topology invariants required by the balancing process must hold.
- Dependency calls must satisfy whatever additional preconditions their implementations impose.

These conditions are not checked locally. In particular, there is no null-container check, membership check, cycle detection, or count-underflow guard.

Aliasing between arguments and structure fields is not rejected. Because the output write occurs last, aliasing can overwrite an otherwise completed structural update. Ordinary erase semantics therefore require noninterfering argument storage.

## Behavior

### Preliminary decision and output preparation

The function first reads the input slot into a local 32-bit result value. It next reads the container’s header reference into a separate local 32-bit value.

It calls **F-2f569f4b2914**, passing the addresses of those two local values.

If that dependency returns nonzero:

- The function rereads the container’s header reference.
- It selects that fresh value as the result.
- It writes the result to the output slot and returns.

This branch performs no direct tree edits, detached-chain insertion, or count decrement. Dependency-induced changes remain possible.

If the dependency returns zero:

- The function calls **F-2d9569c0b133** with the address of the local result value.
- It then rereads the input slot to obtain the node to remove.
- It proceeds with structural removal.

The local result value is not subsequently reassigned by the visible removal logic. Thus, on normal completion of removal, the output is the value left there by the dependencies.

### Selecting the structural removal case

The function reads the selected node’s left child first.

#### No left child and no right child

The selected node is a leaf. Its parent becomes the balancing reference.

If that parent equals the currently read header reference, the function performs the empty-tree transition in this order:

1. Set the header’s upper endpoint to the header reference.
2. Set the header’s lower endpoint to the header reference.
3. Set the header’s root to zero.

Otherwise, it reads the parent’s left-child field and compares it with the selected node:

- If equal, it clears the parent’s left-child field.
- Otherwise, it clears the parent’s right-child field without separately verifying that field’s previous value.

In this branch, clearing a child link is represented as four consecutive zero-byte stores in increasing address order.

It then examines the corresponding cached endpoint: lower endpoint for a left-child removal, upper endpoint otherwise. If that endpoint names the selected node, it changes the endpoint to the parent.

The detached leaf retains its own parent field. Its right-child field is overwritten later during chain insertion.

#### Exactly one child

The existing child replaces the selected node.

The order is:

1. Copy the selected node’s parent reference into the replacement child’s parent field.
2. If the header’s root names the selected node, replace that root reference.
3. Otherwise, replace the parent’s left link if it names the selected node; replace the parent’s right link otherwise.
4. Repair the lower endpoint if it names the selected node.
5. Repair the upper endpoint if it names the selected node.

For a lower-endpoint repair, the replacement is:

- The selected node’s parent when the selected node has no right child.
- Otherwise, the leftmost node reachable from the replacement child.

For an upper-endpoint repair, the replacement is:

- The selected node’s parent when the selected node has no left child.
- Otherwise, the rightmost node reachable from the replacement child.

The replacement child is the balancing reference.

#### Two children

The function finds the leftmost node in the selected node’s right subtree. This node is the structural successor. Its right child, possibly zero, is the replacement for its former position.

The successor is moved into the selected node’s structural position; the selected node itself is eventually chained as detached storage. No payload copy is visible.

The updates occur in this order:

1. Change the selected node’s left child’s parent to the successor.
2. Assign the selected node’s left child to the successor’s left-child field.
3. Handle the successor’s old position:
   - If the successor is the selected node’s immediate right child, assign the successor as the replacement child’s parent when that child exists.
   - Otherwise, assign the successor’s former parent as the replacement child’s parent when the child exists; replace that former parent’s left link with the replacement child; attach the selected node’s right subtree to the successor; then change that subtree root’s parent to the successor.
4. Replace the selected node in the header root or its parent’s child link.
5. Copy the selected node’s parent into the successor’s parent field.
6. Save the successor’s original color.
7. Copy the selected node’s color to the successor.
8. Copy the saved successor color to the selected node.

If the successor’s former right child exists, it becomes the balancing reference. If it does not exist, the moved successor becomes the balancing reference.

That last rule is explicitly present. It must not be replaced with a conventional null-child deletion algorithm or an assumed saved-parent rule.

This branch does not directly update either cached endpoint.

### Balancing

The balancing phase is skipped if either:

- The balancing reference equals the currently read header reference.
- The detached node’s color byte is zero.

Otherwise, the function enters a repair process. At each iteration it first checks whether the balancing reference is the current root or has a color other than one. Either condition leads to the final write of one to that reference’s color byte.

For continued repair, it reads the balancing reference’s parent and determines whether the reference is that parent’s left child. The opposite child is treated as its sibling. The two sides use mirrored child selections.

The visible cases are:

- **Absent sibling:** Set the current reference’s color to zero, move the reference to its parent, and repeat.
- **Sibling color zero:** Set the sibling to one, set the parent to zero, invoke the corresponding structural helper, and reread the parent and sibling.
- **Both sibling children absent or color one:** Set the sibling to zero, move the balancing reference to its parent, and repeat.
- **Far sibling child absent or color one, with another case still requiring repair:** Set the near child to one when present, set the sibling to zero, invoke the opposite structural helper, and reread the sibling.
- **Final repair case:** Copy the parent’s color to the sibling, set the parent to one, set the far child to one when present, and invoke the corresponding structural helper.

If the reread sibling after the near-child transformation is absent, the function exits to the final color write rather than continuing the ordinary sibling sequence.

Whenever this repair process reaches its normal exit, it writes one to the current balancing reference’s color byte.

The structural helpers are consistent with opposite-direction rotations, but their implementations and full calling contracts are unavailable. The visible repair logic alone does not prove that every valid input produces a valid red-black tree.

### Detached-chain insertion and count update

After structural removal and any balancing, the function:

1. Reads the container’s detached-chain head.
2. Writes that value into the detached node’s offset-12 field.
3. Writes the detached node reference into the container’s detached-chain head.
4. Reads the 32-bit count at offset 20, subtracts one, and stores the result.
5. Writes the saved local result value to the output slot.

In the two-child case, the detached node is the originally selected node, not the successor moved into the tree.

The visible operation does not clear the detached node’s parent or left-child fields.

The count operation is a 32-bit decrement with no explicit saturation or validity check. The supplied representation uses a signed integer expression; behavior at signed overflow cannot be resolved from that representation alone. A portable contract must not silently promise wraparound for an invalid boundary count.

## Side effects

Direct writes may affect:

- Header root and endpoint fields.
- Parent and child links in multiple nodes.
- Node color bytes.
- The detached node’s offset-12 field.
- The container’s detached-chain head and count.
- The output slot.

The function does not visibly allocate memory or directly release a node. Chain insertion suggests retention for later reuse, but it does not establish an ownership policy.

In particular, **F-2d9569c0b133 must not be assumed to be a destructor, deallocator, iterator increment, or ownership-neutral helper** without its contract. The same restriction applies to all dependencies: they may have effects outside the local values or nodes apparent at the call site.

The implementation repeatedly rereads container and node fields. Its behavior is not equivalent to taking one immutable snapshot at entry. No synchronization or atomic transaction is visible; intermediate inconsistent links can exist between writes.

There is no explicit error return, rollback, or local recovery from invalid memory, malformed topology, or dependency failure.

## Dependencies

| Opaque ID | Established interaction | Interpretation and limits |
|---|---|---|
| **F-2f569f4b2914** | Receives addresses of two initialized 32-bit local values; its zero/nonzero return selects removal versus bypass. | Comparison-like intent is inferred. Mutation and other effects are unknown. |
| **F-2d9569c0b133** | Called only after a zero preliminary result; receives the address of the local value eventually written as output. | Preparing a continuation result is plausible but unproven. |
| **F-5166d0f0eac3** | Called during balancing with the container and, at several sites, an explicit parent reference. One supplied call site shows only the container argument. | Left-rotation-like role is inferred. The inconsistent visible argument counts prevent a complete interface contract. |
| **F-3890c860a6ae** | Called during mirrored balancing with the container and, at several sites, an explicit parent reference. One supplied call site shows only the container argument. | Right-rotation-like role is inferred. The same argument uncertainty applies. |

The helpers’ effects on globals, allocation, ownership, and unrelated container fields are **documented_only unknowns**. Correct helper behavior cannot be inferred merely because later code uses the structure.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Preliminary bypass

Given that F-2f569f4b2914 returns nonzero, the output must equal the container’s header reference reread after that call. F-2d9569c0b133 must not be called, and this function must not directly decrement the count or prepend a detached node.

This remains true if the preliminary dependency changes the header reference.

### Removing the only node

Given a single-node tree whose selected node has no children and whose parent is the header, and assuming dependencies do not invalidate those facts:

- The header’s root becomes zero.
- Both cached endpoints become the header reference.
- The selected node is prepended to the detached chain.
- The count is decremented once.
- The final output is the dependency-prepared local result, not automatically the header.

### Removing a non-root leaf

Given a selected leaf that is its parent’s left child, its parent’s left field is cleared. If the header’s lower endpoint names the leaf, that endpoint becomes the parent.

Balancing depends on the detached leaf’s color and subsequent topology. The unconditional structural expectation does not promise that balancing leaves the parent’s final child fields unchanged.

### Removing a node with one right child

Given no left child and one right child, the right child receives the selected node’s parent and replaces the selected node in the root or parent link.

If the selected node was the lower endpoint, the endpoint becomes the leftmost node reachable from that right child before balancing.

### Removing a node with two children

Given a finite, coherent right subtree, the leftmost node in it occupies the selected node’s former position. The originally selected node, rather than that successor, is prepended to the detached chain.

The two node color bytes are exchanged before repair eligibility is evaluated.

### Successor without a right child

Given the two-child case with no successor right child, the balancing reference is the moved successor. Acceptance must follow that observed choice; substituting an assumed conventional replacement-parent reference would change behavior.

### Result mutation by a dependency

Given a zero preliminary result and F-2d9569c0b133 changing the local result to another 32-bit value, normal completion writes that changed value to the output slot. Structural removal still uses the later reread of the input slot.

## Uncertainties

The following remain **documented_only**:

- Exact dependency signatures, particularly the balancing helpers’ inconsistent visible argument counts.
- Whether the preliminary operation is equality testing.
- Whether the result-preparation operation advances an iterator.
- Whether helpers preserve any global, container, or node state.
- Allocation, destruction, reference counting, and ownership rules.
- Payload layout and ordering semantics.
- Formal validity requirements for the header’s color and unused fields.
- Exact arithmetic behavior at an overflowing count boundary.
- Supported aliasing, concurrency, and reentrancy contracts.
- Behavior for malformed links, invalid colors, cycles, or inaccessible storage.

The structural pattern supports a red-black-tree interpretation, but it does not justify silently correcting unusual repair choices or assuming undocumented helper effects.

## Evidence

This contract is based solely on the supplied static representation of F-f59beec76d5b.

Established evidence consists of field offsets and widths, direct reads and writes, branch conditions, traversal directions, dependency-call placement, color-byte operations, detached-chain insertion, count decrement, and final output assignment.

Inferred intent consists of the tree/header terminology, red-black color interpretation, rotation-like helper roles, and recycling interpretation of the detached chain.

No dependency implementation, runtime trace, executable verification, or runtime test was available or performed.
