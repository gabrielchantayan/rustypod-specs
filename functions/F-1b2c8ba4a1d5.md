<!-- Trusted provenance: F-1b2c8ba4a1d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b2c8ba4a1d5

## Purpose

F-1b2c8ba4a1d5 conditionally removes a node from a linked tree, performs color-dependent structural repair, passes the removed node’s payload address to a dependency, and places the removed node on a container-local chain. It completes by writing a four-byte result to caller-provided storage.

The representation and repair operations are consistent with red-black-tree deletion. That is inferred intent. The contract below describes the supplied behavior without assuming textbook deletion semantics, successful balancing, or unavailable dependency implementations.

In particular, the repair cursor need not be a replacement child. It can be the removed leaf’s parent or a relocated successor whose replacement child is absent.

An initial dependency can bypass removal. On that path, the output is the container’s header reference reread after the dependency returns.

## Interface

### Arguments and output

| Argument | Representation | Established use |
|---|---|---|
| Result storage | Address of writable four-byte storage | Receives the final result word |
| Container reference | Four-byte address value | Locates tree metadata, chain head, and count |
| Input storage | Address of readable four-byte storage | Supplies the initial dependency operand and, on the removal path, the subsequently reread target |

There is no explicit return value. The interface’s direct output is the final four-byte write, together with memory changes and dependency effects.

All accessed pointer-bearing fields are four bytes. Color accesses are one byte. The interface is not established as portable to a different pointer width.

### Container layout

Offsets are measured in bytes from the container reference.

| Offset | Width | Established role |
|---|---:|---|
| +4 | 4 bytes | Local chain head |
| +16 | 4 bytes | Tree header reference |
| +20 | 4 bytes | Count decremented after removal |

No other container fields are directly accessed. Dependencies may access additional state.

### Node and header layout

| Node offset | Width | Established role |
|---|---:|---|
| +0 | 1 byte | Color or repair-state marker |
| +4 | 4 bytes | Parent reference |
| +8 | 4 bytes | Left-child reference |
| +12 | 4 bytes | Right-child reference; later reused as a chain link |
| +16 | Unknown extent | Start of payload passed to F-8a5aadac202f |

The header uses +4 as the root reference, +8 as the cached leftmost reference, and +12 as the cached rightmost reference.

The empty-tree representation directly written by this function is a zero root with both cached extrema referring to the header.

Color zero is red-like and color one is black-like. Other byte values are not validated; their branch behavior is described below rather than normalized to either color.

### Preconditions and validity limits

Supported removal requires accessible container, header, node, and output storage; valid parent/child relationships for the path taken; and terminating child traversals. Storage must permit the indicated widths and subsequent accesses.

The function does not validate target membership, allocation extent, tree integrity, or traversal acyclicity. A null target or invalid non-null reference is not converted into a reported error.

Dependencies can change externally reachable state. Consequently, structural preconditions must hold when structural removal begins, after both initial dependencies and the input reread—not merely when the function is entered. Later accesses must remain valid after subsequent dependency calls.

No general aliasing contract is established. Result storage is written last, but aliases with node or container fields can still affect the meaning of earlier and later operations. The input word is read twice on the removal path, with dependency calls between those reads.

## Behavior

### Initial dependency sequence

The function reads the input word into a private four-byte result temporary, then reads the container’s header reference into a second private four-byte temporary. F-227cdabeb472 receives the addresses of these two temporaries, not the original input location or the container’s header field.

If F-227cdabeb472 returns nonzero, the function rereads container +16, places that header reference in the result temporary, writes it to result storage, and returns. It does not itself perform structural removal, repair, payload processing, chain insertion, or count decrement on this path. The dependency’s own effects remain unrestricted by the available evidence.

If F-227cdabeb472 returns zero, F-315d952a53b9 receives the address of the result temporary. Its return value, if any, is not used here.

Only after F-315d952a53b9 returns does the function reread input storage to obtain the removal target. Changes made through other references during either initial dependency can therefore change which node is removed.

The final output on the removal path is the word in the result temporary at the final write. Its interpretation as an iterator, successor, status, or other value is unknown.

### Structural removal

The original target remains the node eventually supplied to payload processing and placed on the local chain. Relocating another node into its tree position does not change that identity.

#### Target with no children

The target’s parent becomes the repair cursor.

If that parent equals the current header reference, the function writes the empty-tree fields in this order:

1. Header rightmost becomes the header.
2. Header leftmost becomes the header.
3. Header root becomes zero.

Otherwise, the target is compared with its parent’s left link. A match selects that left link; a mismatch selects the right link without checking that it actually references the target.

The selected four-byte link is cleared through four successive one-byte zero writes, in increasing byte-offset order.

The corresponding cached extremum is then examined. The left-link case examines cached leftmost; the right-link case examines cached rightmost. If that field equals the target, it is replaced with the target’s parent.

#### Target with exactly one child

The existing child becomes both the replacement and repair cursor.

First, the child’s parent is assigned the target’s parent. The incoming target link is then replaced, with this precedence:

- Replace the header root if it equals the target.
- Otherwise replace the parent’s left link if it equals the target.
- Otherwise replace the parent’s right link without an additional membership check.

Cached extrema are considered afterward:

- If cached leftmost equals the target, it becomes the target’s parent when the target has no right child; otherwise it becomes the leftmost node reached from the replacement.
- If cached rightmost equals the target, it becomes the target’s parent when the target has no left child; otherwise it becomes the rightmost node reached from the replacement.

Extremum traversal follows the appropriate child links until zero. There is no cycle detection.

#### Target with two children

The structural successor is the leftmost node in the target’s right subtree. Its right child, possibly zero, is the initial replacement.

The successor moves into the target’s structural position. This function does not copy payload bytes between the two nodes.

The relevant mutation order is:

1. Change the target’s left child’s parent to the successor.
2. Assign the successor’s left link from the target’s left link.
3. Adjust the successor’s former position and right-subtree links as described below.
4. Redirect the target’s incoming root or parent link to the successor.
5. Assign the successor’s parent from the target’s parent.
6. Exchange the successor’s and original target’s one-byte colors.

If the successor is the target’s immediate right child, an existing replacement receives the successor as parent. The successor’s right link is retained.

If the successor lies deeper in the right subtree, an existing replacement first receives the successor’s former parent. That former parent’s left link becomes the replacement, including zero when absent. The successor’s right link then becomes the target’s right subtree, and that subtree root’s parent becomes the successor.

Incoming-link selection uses the same root-first, then parent-left, otherwise parent-right precedence described for one-child removal.

A nonzero replacement becomes the repair cursor. If the replacement is zero, the relocated successor itself becomes the cursor. The original target remains the removed node.

This branch performs no direct cached-extremum update.

### Repair gate and relationships

After unlinking, the function rereads the container’s header reference. Repair is entered only when:

- The repair cursor differs from that header.
- The removed node’s color, after any exchange, is nonzero.

A zero removed-node color suppresses repair. Any nonzero removed-node color can admit repair, but loop continuation requires the cursor’s color to equal exactly one.

At each iteration, the current header root is read. If the cursor equals that root, or if its color is not exactly one, the function leaves the loop through the completion point.

Otherwise, it reads the cursor’s parent and compares the cursor with that parent’s left link. A match selects the left-side case; a mismatch selects the right-side case.

| Cursor side | Sibling | Near child | Far child |
|---|---|---|---|
| Left | Parent’s right child | Sibling’s left child | Sibling’s right child |
| Right | Parent’s left child | Sibling’s right child | Sibling’s left child |

### Established repair calls

The following are the visible call-site inputs. A parent argument is a four-byte word freshly read from the cursor’s +4 field at that call; it is not simply a parent value retained from the start of the iteration.

| Cursor side | Transformation | Dependency | Visible inputs |
|---|---|---|---|
| Left | Zero-colored sibling | F-bff5bbe8ab77 | Container reference, freshly read cursor-parent word |
| Left | Near-child preparation | F-3008804f5ed8 | Container reference only |
| Left | Final transformation | F-bff5bbe8ab77 | Container reference, freshly read cursor-parent word |
| Right | Zero-colored sibling | F-3008804f5ed8 | Container reference, freshly read cursor-parent word |
| Right | Near-child preparation | F-bff5bbe8ab77 | Container reference only |
| Right | Final transformation | F-3008804f5ed8 | Container reference, freshly read cursor-parent word |

These call-site inputs are established even though the dependencies’ true signatures and full effects are unknown. In particular, a second explicit argument must not be invented for either near-child call.

### Repair transitions and call placement

**Absent sibling.** The cursor’s color is written as zero. The cursor is promoted to the parent, and the loop condition is reconsidered.

**Zero-colored sibling.** The sibling is set to one, then the cursor’s parent is set to zero. The corresponding zero-colored-sibling dependency from the table is called. After return, the cursor’s parent and sibling relationship are reread. If the new sibling is absent, the absent-sibling transition applies.

**Both sibling children absent or exactly one-colored.** The near child is checked first; the far child is checked when the near-child condition permits it. When both satisfy “absent or color exactly one,” the sibling is set to zero, the cursor becomes its parent, and the loop continues.

**Near-child preparation.** If the previous case did not apply and the far child is absent or exactly one-colored, an existing near child is set to one and the sibling is set to zero. The corresponding near-child dependency is then called with only the container visibly supplied. After return, the sibling is reread through the cursor’s current parent. If that sibling is zero, repair proceeds directly to completion rather than dereferencing it.

**Final transformation.** The current parent’s color is copied to the sibling. The parent is set to one, and an existing far child is set to one. Only then is the final-transformation dependency called, using the freshly read cursor-parent word specified in the table. Repair subsequently proceeds to completion.

At repair completion, the current cursor’s color is written as one, including after a final dependency returns. If the initial repair gate was false, this completion write does not occur.

The structural dependencies resemble rotations, but their names and surrounding operations do not establish their complete mutation contracts or a balancing postcondition.

### Payload processing and publication

After structural processing and any repair, the function performs these operations in order:

1. Read the current container chain head and write it to the removed node’s +12 field.
2. Call F-8a5aadac202f with the removed node’s payload address, at node +16.
3. Write the removed node reference to container +4 as the new chain head.
4. Read the four-byte count at container +20, subtract one, and store the result there.
5. Write the result temporary to caller-provided result storage.

The chain-link write precedes payload processing. Chain-head publication and the count operation follow it.

The count’s input is read after the payload dependency, not saved before the call. A dependency mutation to the count can therefore affect the value decremented.

The supplied representation expresses a signed integer decrement in four-byte storage. For values whose subtraction is representable, the stored result is one less. A portable wraparound guarantee at the signed boundary is not established.

## Side effects

Direct writes can affect root and child links, parent links, cached extrema, one-byte colors, the removed node’s chain-link field, the container chain head and count, and result storage.

Not every field of the removed node is cleared. Its parent, left link, and color may retain values produced or retained during removal.

No direct allocation or deallocation operation is established. The local chain suggests node reuse or deferred reclamation, but that is inferred intent rather than an ownership guarantee.

F-8a5aadac202f receives an interior payload address. Whether it destroys an object, releases resources, allocates memory, or performs some other operation is unknown. This function subsequently publishes the removed node reference, but does not directly dereference that node after the payload call. Its continued allocation validity and the chain’s ownership rules cannot be established from this function alone.

There is no direct rollback, synchronization, or atomic-publication mechanism. Intermediate states are observable to dependencies at their call points. If an operation fails to return normally, later writes—including the result write—are not guaranteed.

## Dependencies

| Opaque function ID | Established interaction | Unresolved contract |
|---|---|---|
| F-227cdabeb472 | Receives addresses of the input-copy and header-copy temporaries; nonzero bypasses removal | Predicate meaning, temporary mutations, external effects |
| F-315d952a53b9 | Receives the result temporary’s address before input is reread | Transformation semantics, retained references, external effects |
| F-bff5bbe8ab77 | Receives the call-site inputs specified in the repair table | True signature, structural effects, global effects |
| F-3008804f5ed8 | Receives the call-site inputs specified in the repair table | True signature, structural effects, global effects |
| F-8a5aadac202f | Receives removed-node address plus 16 after the chain-link write | Payload extent, ownership, lifetime effects, failure behavior |

No dependency is assumed to preserve container fields, global state, nodes, or other reachable memory. Statements about direct writes and visible call placement do not imply that dependencies leave those writes intact.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

### Bypass with a changed header

If F-227cdabeb472 returns nonzero after changing the container’s header reference, output must receive the newly reread header reference, not the original header copy or an arbitrary mutation of the result temporary.

This function must not itself perform removal, invoke F-315d952a53b9 or F-8a5aadac202f, publish a chain entry, or decrement the count.

### Removing the sole node

At the structural-removal stage—after both initial dependencies have returned and input storage has been reread—suppose:

- The reread target is the sole node.
- Its parent is the current header.
- Both child links are zero.
- The header root and extrema consistently describe that single-node tree.
- Required storage remains accessible.

The direct structural writes must set header rightmost to the header, then leftmost to the header, then root to zero. Repair must be skipped because the cursor is the header.

Before payload processing, the removed node’s +12 field must receive the chain head read at that time. On normal return from payload processing, the function must publish the removed node as chain head and decrement the count value read afterward.

Final-state claims about the empty tree and retained chain-link contents additionally require that the payload dependency not overwrite those fields. A final count equal to the pre-payload count minus one requires that dependency not change the count. No such preservation follows merely from normal return.

### One right child

At structural removal, suppose the target has no left child and has one valid right child. The child must receive the target’s parent before the incoming target link is redirected.

If cached leftmost equals the target, it must be replaced with the leftmost node reachable through that child. Subsequent dependency mutations are outside this direct structural obligation.

### Two children with no successor replacement

For a valid two-child target whose successor has no right child, the successor must move into the target’s structural position and become the repair cursor. The original target remains the payload-processing and chain-insertion node.

The colors must be exchanged before the repair gate examines the removed node’s color.

### Left-side zero-colored sibling

When repair reaches a left-side cursor with a zero-colored sibling, the sibling must be set to one and the parent to zero before F-bff5bbe8ab77 is called. Its visible inputs must be the container and a freshly read cursor-parent word. Parent and sibling relationships must be reread afterward.

The mirrored right-side case must use F-3008804f5ed8 with those two visible inputs.

### Near-child preparation

When a left-side cursor reaches near-child preparation, the visible dependency call must be F-3008804f5ed8 with only the container. The right-side counterpart must call F-bff5bbe8ab77 with only the container.

If the subsequently reread sibling is zero, the function must proceed to the cursor’s completion recoloring without attempting the final sibling-based transformation.

### Payload-call ordering

On entry to F-8a5aadac202f, the direct chain-link assignment has already occurred. This function has not yet published the removed node as chain head or performed the final count decrement. Any different state observed because of aliasing or earlier dependency mutations must not be mistaken for a different direct operation order.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the initial predicate and result transformation.
- True dependency signatures beyond the visible call-site inputs.
- Dependency effects, retained references, ownership rules, and failure behavior.
- Payload extent and destruction or resource-release semantics.
- Whether the local chain is a free list, retirement list, or another structure.
- Permitted argument and structure aliasing.
- Defined behavior for inaccessible storage, corrupt links, cycles, and invalid colors.
- Count-domain constraints and behavior outside representable signed subtraction.
- Pointer-offset overflow, alignment requirements, and byte order.
- Thread safety and reentrancy.
- A general ordering or red-black-invariant preservation guarantee after removal.

## Evidence

Established static evidence consists of the supplied function representation: access widths and offsets, branch conditions, traversal directions, direct write order, dependency-call placement, visible call-site inputs, and final publication sequence.

The tree-header interpretation, red/black terminology, structural-successor terminology, likely rotation roles, and possible node-reuse intent are inferences. They do not establish unavailable dependency contracts.

No runtime tests have been performed. The acceptance examples specify conditional obligations from static evidence, not observed executions.
