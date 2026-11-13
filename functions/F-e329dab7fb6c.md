<!-- Trusted provenance: F-e329dab7fb6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e329dab7fb6c

## Purpose

F-e329dab7fb6c removes a selected node from a linked structure, conditionally repairs one-byte node states, invokes a removal dependency, decrements a stored count, and writes a separately prepared position word to an output location.

The structure is consistent with a tree having parent links, left and right children, a header, and cached extrema. Red–black-tree deletion and return of the following iterator position are **inferred intent**. The established contract is the visible sequence of accesses, mutations, branches, and dependency calls. Correct balancing and iterator semantics are not established independently.

Two repair-focus choices are material: removal of an ordinary leaf initially focuses on its parent, and a two-child transplant whose successor has no right child focuses on the promoted successor. Neither choice may be replaced with a conventional null-leaf model.

## Interface

### Arguments and result

| Argument | Representation | Established role |
|---|---|---|
| Output location | Address of writable 32-bit storage | Receives one result word on normal completion. |
| Container address | Address carried in a 32-bit integer-sized representation | Provides access to the header pointer and count. |
| Position location | Address of readable 32-bit storage | Supplies the initial position and, on the removal path, the selected node through a later reread. |

There is no explicit scalar result. Completion is reported only by the output write; no status or error code is produced.

The visible representation uses 32-bit pointers and words and 8-bit state fields. Host byte order, alignment requirements, and behavior on inaccessible storage are not specified.

### Container and node layout

| Object | Byte offset | Access width | Role |
|---|---:|---:|---|
| Container | 16 | 32 bits | Header address |
| Container | 20 | 32 bits | Count, accessed as a signed integer for decrement |
| Node | 0 | 8 bits | State |
| Node | 4 | 32 bits | Parent address |
| Node | 8 | 32 bits | Left-child address |
| Node | 12 | 32 bits | Right-child address |
| Header | 4 | 32 bits | Root address |
| Header | 8 | 32 bits | Cached minimum address |
| Header | 12 | 32 bits | Cached maximum address |

The header uses the same link offsets as ordinary nodes. Its state byte has no established role here. Bytes 1–3 and node payload outside these fields are not directly accessed.

State 0 has a red-like role and state 1 a black-like role. These names are interpretive: repair eligibility tests for a nonzero state, whereas several later decisions require exactly 1. Other byte values are not uniformly equivalent to state 1.

### Preconditions and limits

Each address actually dereferenced must remain accessible for the required width. Traversals must terminate for the function to complete. The selected branch also requires sufficiently consistent links to make its accesses meaningful.

No visible checks establish those conditions. In particular, the function does not validate null arguments, node membership, reciprocal links, acyclicity, count range, or state-byte validity.

There is no established aliasing, concurrency, reentrancy, or dependency-preservation guarantee. Descriptions of ordinary tree outcomes below are conditional on storage remaining valid and dependencies not disrupting the required structure.

## Behavior

### Initial comparison and preparation

The function first reads the position word, then the container’s header word. It passes addresses of separate local copies of those words to F-50be1c0375de.

If that dependency returns nonzero:

1. The container’s header word is reread.
2. That reread value becomes the result word.
3. The output location receives the result.

This branch performs no direct structural mutation, repair, removal-dependency call, or count decrement. Effects of F-50be1c0375de itself remain unknown.

If the dependency returns zero, F-ee5c701f2f6f receives the address of the local position copy. That local word supplies the eventual output. The function then rereads the caller’s position location to select the node for removal.

Thus, the prepared result and selected node are distinct values. A transformation of the local position copy does not itself change the selected node. Conversely, a dependency-induced change to the caller’s position storage can affect the later selection.

The header is not retained as an immutable snapshot. Later operations reload it from the container.

### Structural removal

The selected node’s left link is inspected first. Its right link is then inspected to distinguish leaf removal, replacement by one child, and replacement by a successor.

Some branch conditions and traversal termination checks visibly reread link fields. The description below does not grant atomic snapshots of those fields.

### Leaf removal

When both child links are zero, the selected node’s parent becomes the repair focus.

If that parent equals the header currently obtained from the container, the direct writes occur in this order:

1. Header maximum becomes the parent address, which equals the header at the comparison.
2. Header minimum becomes that same address.
3. Header root becomes zero.

With stable storage, this produces a null root and self-referential extrema.

Otherwise, the parent’s left link is read and compared with the selected node:

- A match selects the parent’s left-link storage for clearing.
- A mismatch selects the right-link storage, without checking whether that link actually names the selected node.

The selected link is cleared through four separate zero-byte writes at ascending addresses. These are not represented as a single 32-bit store.

The same left-versus-right decision determines which cached extremum is examined: minimum for a left detachment, maximum for a right detachment. If that cache equals the selected node, it is replaced by the parent. The other cache is not directly examined on this branch.

### Replacement by one child

If exactly one child exists, that child becomes the repair focus and replaces the selected node.

The direct mutation order is:

1. Write the selected node’s parent word into the child’s parent field.
2. Examine the current header root. If it names the selected node, replace it with the child. Otherwise, inspect the selected node’s parent’s left link; replace that link if it matches, or replace the parent’s right link otherwise.
3. Check and, when necessary, update the cached minimum.
4. Check and, when necessary, update the cached maximum.

The nonmatching-parent fallback does not validate the right link.

If the cached minimum names the selected node:

- When the selected node’s right link is zero, the minimum becomes the selected node’s parent.
- Otherwise, the replacement subtree is followed through left links, and its leftmost node becomes the minimum.

If the cached maximum names the selected node:

- When the selected node’s left link is zero, the maximum becomes the selected node’s parent.
- Otherwise, the replacement subtree is followed through right links, and its rightmost node becomes the maximum.

These conditions follow the visible link tests; they are not unconditional updates based solely on which child exists.

### Replacement by a successor

When both children exist, the function follows left links from the selected node’s right child until it finds a node with no left child. This node is the structural successor. Its right child is the initial replacement for its old position.

The transplant has the following ordered effects:

1. The selected node’s left child receives the successor as parent.
2. The successor’s left link receives the selected node’s left-child word.
3. The successor’s old position is resolved:
   - If the successor is the selected node’s immediate right child, an existing successor right child receives the successor as parent.
   - Otherwise, an existing successor right child receives the successor’s old parent; that old parent’s left link is replaced by the right child, including zero when absent; the successor’s right link receives the selected node’s right subtree; and that subtree root receives the successor as parent.
4. The current root or selected parent link is replaced with the successor. A non-root replacement uses the same left-match, otherwise-right rule.
5. The successor’s parent receives the selected node’s parent word.
6. The successor and selected node exchange their state bytes.

The original selected node remains the object passed to the removal dependency. No payload replacement or payload copying is visible.

If the successor’s original right child exists, it becomes the repair focus. Otherwise, the promoted successor becomes the repair focus.

No direct cached-minimum or cached-maximum update occurs in this branch. After the state exchange, the selected node holds the successor’s former state, which controls repair eligibility.

### Repair eligibility and loop checks

After structural mutation, repair is entered only if:

- The repair focus differs from the currently loaded header; and
- The node designated for removal has a nonzero state byte.

The header comparison short-circuits the state read: when focus equals header, that eligibility state byte is not read.

At each repair-loop check:

1. The current header and its root are read.
2. If focus equals root, the loop exits without testing the focus state for that decision.
3. Otherwise, the focus state is tested. Any value other than exactly 1 exits the loop.

Both exits write state 1 to the focus. If repair was never entered, no such final repair write occurs.

A continuing iteration loads the focus parent and that parent’s left link. Equality with the focus selects the right-sibling orientation; any mismatch selects the left-sibling orientation.

### Repair with an absent sibling

When the selected sibling is absent, repair:

1. Writes state 0 to the current focus.
2. Adopts the loaded parent address as the next focus.
3. Repeats the root and exact-state checks.

This behavior applies both when the sibling is initially absent and when it becomes absent on the reload following a zero-state-sibling dependency call. In the latter case, the parent address has first been reloaded from the focus’s parent field.

Although temporary storage elsewhere holds state-byte values, every incoming path to this absent-sibling handling supplies a parent address. No state-byte-to-address propagation into this handling is established.

Address validity and dependency effects remain unresolved. In particular, this upward transition does not prove that the loaded parent remains accessible.

### Repair with a present sibling

The orientations are mirrored. “Inner” means the sibling child nearer the focus; “outer” means the farther child.

| Focus orientation | Sibling | Inner child | Outer child | Outward dependency | Inner dependency |
|---|---|---|---|---|---|
| Focus matches parent’s left link | Right | Left | Right | F-9a4773035256 | F-960cfae00cac |
| Otherwise | Left | Right | Left | F-960cfae00cac | F-9a4773035256 |

The following rules apply in order.

**Zero-state sibling.** The sibling receives state 1, the parent receives state 0, and the outward dependency is called with the container and parent word. The focus parent and corresponding sibling are then reloaded. A now-absent sibling takes the absent-sibling transition above.

**Both children absent or exactly state 1.** The sibling receives state 0. The focus’s parent field is read to obtain the next focus, and the loop checks repeat. Unlike absent-sibling handling, this branch does not first write state 0 to the old focus.

**Outer child absent or exactly state 1, without the preceding both-children condition.** When present, the inner child receives state 1. The sibling receives state 0. The inner dependency is invoked, visibly with only the container argument. The corresponding sibling is reloaded through the focus parent. If absent, repair exits by writing state 1 to the focus.

**Terminal repair.** With a sibling available, it receives the parent’s state; the parent receives state 1; and an existing outer child receives state 1. The outward dependency is called with the container and parent word. Repair then writes state 1 to the focus and ends.

The exact-one tests are significant. A present child whose state is neither 0 nor 1 does not satisfy the “absent or exactly state 1” condition.

### Removal, count, and output

After structural mutation and any repair, the function:

1. Calls F-5da13b8c65b8 with the container address, original selected node address, and integer 1.
2. Reads the container’s 32-bit count and writes that value minus one.
3. Writes the prepared local position word to the output location.

The count is read after the dependency returns. A count change made by that dependency therefore affects the decrement.

The visible arithmetic is signed 32-bit subtraction by one. Representable cases decrease by one, including zero becoming −1. There is no nonzero-count guard. Behavior at the signed minimum is **documented_only**; wraparound is not established.

## Side effects

Direct effects comprise link rewrites, cached-extremum updates, state-byte writes, the post-dependency count decrement, and the final output-word write.

No direct allocation, payload comparison, payload copy, or memory release is visible. F-5da13b8c65b8 may destroy or release storage, but neither responsibility nor ownership transfer is established.

After that dependency returns, the function does not directly access the selected node again through its retained node address. Count and output accesses still follow and could reach overlapping storage if arguments alias. No safe-retention guarantee for other references follows.

The position location is not explicitly assigned as an input. Aliasing or dependency effects can nevertheless change it. The final output write can overwrite a position, count, header field, or node field if their storage overlaps.

Writes are incremental; no rollback is present. A fault or nonreturning dependency can prevent later repair, count adjustment, or output assignment after earlier mutations have occurred. Exception propagation and recovery are unknown.

## Dependencies

| Opaque ID | Established invocation and use | Unresolved contract |
|---|---|---|
| F-50be1c0375de | Receives addresses of local position and header copies; nonzero selects the header-result path | Comparison meaning, argument mutation, external effects |
| F-ee5c701f2f6f | Receives the local position-copy address on the zero-comparison path | Transformation, traversal, retained access, external effects |
| F-9a4773035256 | Outward right-sibling repair receives container and parent; mirrored inner repair visibly supplies only container | Exact interface, structural effects, preservation |
| F-960cfae00cac | Outward left-sibling repair receives container and parent; mirrored inner repair visibly supplies only container | Exact interface, structural effects, preservation |
| F-5da13b8c65b8 | Receives container, original selected node, and 1 before count decrement | Destruction, deallocation, ownership, count changes, failure behavior |

Directional rotation roles are inferred from surrounding operations. Calls showing only one argument do not establish a hidden node argument or a conventional two-argument interface.

No dependency is assumed to preserve globals, container fields, reachable nodes, local words whose addresses it receives, or memory ownership.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations. Unless stated otherwise, dependencies return normally, required storage remains valid, and unrelated fields remain unchanged.

### Header-result path

If F-50be1c0375de returns nonzero, output receives the header word reread after that call, even if it differs from the initial header copy. Neither F-ee5c701f2f6f nor F-5da13b8c65b8 is called by this path, and no direct count decrement occurs.

### Sole-node removal

For a selected leaf whose parent equals the header, direct header writes set maximum, minimum, then root, producing header-valued extrema and a zero root. Repair is skipped because focus equals header, independently of the selected node’s state. Removal, count decrement, and prepared-result output follow.

### One-child root replacement

For a root with one child, with removed-node state 1, the child becomes root and receives the old parent word. Repair encounters focus equal to root and writes state 1 to the child. Cache updates occur only when the relevant cache names the selected node.

### Successor identity and state

For a two-child selection, the successor is structurally promoted, but F-5da13b8c65b8 receives the original selected address. The exchanged state on that selected node determines repair eligibility. If the successor originally had no right child, repair initially focuses on the promoted successor.

### Initially absent sibling

Suppose a repair iteration has a non-root focus in state 1, a valid parent, and an absent selected sibling. The focus receives state 0, and the next loop check uses that parent as focus. If that parent is root, it then receives state 1.

### Sibling absent after dependency

Suppose a zero-state sibling causes an outward dependency call, after which the focus parent field names a valid parent whose corresponding sibling link is zero. Repair writes state 0 to the focus and adopts the reloaded parent. It does not adopt a sibling state value.

### Present sibling with black-like children

If the sibling exists and both children are absent or exactly state 1, the sibling receives state 0 and repair moves to the reread focus parent. This branch does not perform the absent-sibling write to the old focus.

### Late count and distinct result

If F-5da13b8c65b8 leaves count 9, the subsequent direct count write is 8. If F-ee5c701f2f6f previously changed only the local position copy, output receives that changed word while removal uses the node reread from the caller’s position location.

## Uncertainties

The following remain **documented_only**:

- Formal tree invariants and correctness of balancing for all supported inputs.
- Exact comparison and position-transformation semantics.
- Complete directional-dependency interfaces and effects.
- Validity and termination of all repair paths.
- Intended rationale for the leaf and successor repair-focus choices.
- Supported state values beyond 0 and 1.
- Ownership, destruction, allocation, and deallocation obligations.
- Alignment, byte order, signed-overflow behavior, exceptions, and supported aliasing.
- Dependency changes to globals, header fields, counts, links, or storage lifetime.

The absent-sibling transition itself is established as parent-directed. What remains unknown is whether its parent address and subsequent accesses are valid under the surrounding contracts.

## Evidence

Established behavior derives solely from the supplied static function body: argument accesses, field widths and offsets, branch conditions, traversals, write order, dependency invocations, and final output assignment.

In particular, both initial absent-sibling paths use the loaded focus parent, and both post-dependency absent-sibling paths first reload that parent. No intervening state-byte assignment reaches the upward transition.

Tree terminology, color names, successor-position intent, and rotation roles are inferences, not evidence of dependency guarantees. No dependency implementations or external invariant specifications were supplied. No runtime tests have been performed.
