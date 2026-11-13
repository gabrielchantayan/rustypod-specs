<!-- Trusted provenance: F-aab94c7c2aad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aab94c7c2aad

## Purpose

F-aab94c7c2aad conditionally removes a node from a linked, color-marked tree, updates structural links and boundary caches, performs conditional color repair using opaque dependencies, and prepends the detached node to a container-maintained chain. On normal completion, it writes one four-byte value to caller-provided result storage.

**Established static behavior:** the function invokes a decision dependency, either completes early or performs removal, and writes the current contents of a local result cell at completion.

**Inferred intent:** an iterator-based erase operation on a red-black-tree-like container, with detached nodes retained for reuse. The result appears intended to represent the next position. These interpretations do not establish public API semantics, allocator ownership, or correctness for every tree topology.

No runtime tests have been performed.

## Interface

The supplied representation exposes three arguments and no explicit return value.

| Argument | Observable role |
|---|---|
| Result-storage address | Receives one four-byte value on normal completion. |
| Container address | Provides access to a chain head, a header pointer, and a count-like field. |
| Position-storage address | Supplies an initial four-byte value and, on the removal path, is read again to select the node removed. |

Pointer-bearing values and links occupy four bytes in this representation. Color accesses occupy one byte. This contract does not establish a wider-pointer representation, alignment requirements, or a public calling convention.

### Container layout

| Byte offset | Width | Established use |
|---|---:|---|
| +4 | 4 bytes | Detached-node chain head. |
| +16 | 4 bytes | Header pointer, reloaded at multiple points. |
| +20 | 4 bytes | Count-like field, decremented on removal. |

No other container fields are directly accessed.

### Header and node layout

| Byte offset | Width | Ordinary-node use | Header use |
|---|---:|---|---|
| +0 | 1 byte | Color or state byte. | No dedicated header-color assignment is established. |
| +4 | 4 bytes | Parent link. | Root link. |
| +8 | 4 bytes | Left-child link. | Cached left boundary. |
| +12 | 4 bytes | Right-child link; later reused as detached-chain link. | Cached right boundary. |

“Left,” “right,” and “boundary” describe the traversal and replacement patterns. The function does not inspect ordering keys or payloads.

### Preconditions and failure boundaries

Normal completion requires readable position storage, writable result storage, and valid memory for every container, header, node, and link actually accessed. Successful tree removal additionally requires compatible structural relationships and dependency behavior.

The function does not validate membership, reciprocal links, color range, count consistency, or address validity. Traversals require terminating child chains; repair requires eventual termination through its conditions or dependency effects.

Null child links are explicitly handled. Null container, header, or selected-node addresses are not generally supported. There is no explicit error result, rollback, diagnostic, allocation-failure handling, or recovery from an invalid access.

Aliasing is not generally excluded by the visible interface, but no alias-safety guarantee follows. Acceptance examples below assume storage does not overlap in ways that invalidate their stated relationships.

## Behavior

### 1. Initial decision and result-cell lifetime

The function first reads four bytes from position storage into a local result cell. It then reads the container’s +16 header pointer into a separate four-byte local cell.

F-ca7a1f20809e receives the addresses of both cells.

If that dependency returns a nonzero integer, the function:

1. Reloads the container’s +16 field.
2. Assigns that newly read value to the local result cell.
3. Copies the cell’s contents to result storage.
4. Completes without executing its own removal, detached-chain insertion, or count decrement.

This path is not established as side-effect-free: the dependency may change externally visible state.

If the dependency returns zero, the function invokes F-017ac946138e with the address of the local result cell. After that call returns, it rereads position storage and uses that newly read value as the removal target. Changes to position storage during either dependency can therefore affect which node is removed.

**Result guarantee:** on the removal path, this function makes no further direct assignment to the local result cell after F-017ac946138e returns. At completion, it copies the cell’s then-current contents to result storage.

That is not an unconditional preservation guarantee for the value present when F-017ac946138e returned. Both initial dependencies receive the result cell’s address and may retain or expose it. Later repair dependencies may consequently modify that cell indirectly. Equality with the value left by F-017ac946138e is guaranteed only if no subsequent indirect mutation occurs.

### 2. Selecting the structural case

The selected node’s left link is read first.

- With no left child, its right link determines whether this is a leaf or a right-only replacement.
- With a left child and no right child, the left child is the replacement.
- With both children present, the function starts at the right child and follows left links to the leftmost node. That node is the structural successor; its right child is the replacement at its former position.

The originally selected node is ultimately the detached node in every removal case.

### 3. Leaf removal

For a leaf, the function reads its parent and compares it with the container’s current header pointer.

If they match, the header fields are written in this order:

1. +12 receives the header pointer.
2. +8 receives the header pointer.
3. +4 receives zero.

The container’s header pointer is reloaded for the destination accesses; a single immutable header snapshot is not established throughout the function.

Otherwise, the parent’s +8 link is compared with the selected node:

- If equal, parent +8 is cleared.
- Otherwise, parent +12 is cleared, without requiring that it previously identified the selected node.

The chosen four-byte link is cleared by four consecutive one-byte writes at increasing offsets. A single atomic four-byte clear is not established.

The function then selects the matching header boundary field: +8 for the left-link case, +12 otherwise. If that field identifies the removed node, it is replaced with the parent.

The parent becomes the repair cursor.

### 4. Replacement by a sole child

The replacement child first receives the selected node’s parent in its +4 field.

The selected node is then replaced in the surrounding structure:

- If header +4 identifies the selected node, header +4 receives the replacement.
- Otherwise, the selected node’s parent is consulted. Its +8 link is replaced if it identifies the selected node; its +12 link is replaced otherwise.

Boundary updates occur afterward, with +8 checked before +12:

- If header +8 identifies the selected node, the new value is the selected node’s parent when the selected node has no right child. Otherwise, it is the leftmost node reached from the replacement.
- If header +12 identifies the selected node, the new value is the selected node’s parent when the selected node has no left child. Otherwise, it is the rightmost node reached from the replacement.

These are the actual branch rules; no separate consistency check establishes that a boundary cache was correct before removal.

The replacement child becomes the repair cursor.

### 5. Successor relocation for two children

The successor is moved into the selected node’s structural position. No whole-node or payload copy is performed.

The link changes occur in the following order:

1. The original left child’s parent becomes the successor.
2. The successor’s left link receives the selected node’s left link.
3. The successor’s former position is handled:
   - If the successor is the immediate right child, its nonnull right child receives the successor as parent.
   - Otherwise, its nonnull right child receives the successor’s former parent; that former parent’s left link receives the child, including zero when absent; the successor receives the selected node’s right subtree; and that subtree’s parent becomes the successor.
4. Header +4 or the selected node’s parent link is changed to identify the successor, using the same root/left/otherwise-right selection pattern.
5. The successor’s parent receives the selected node’s parent.
6. The successor’s old color is read. If its former right child is absent, the successor itself is selected as repair cursor.
7. The successor receives the selected node’s color, and the selected node receives the successor’s saved color.

When the successor’s former right child exists, that child is the repair cursor. When absent, the relocated successor is the cursor. This explicit substitution must not be replaced in the contract with an assumed missing-child repair model.

The original selected node remains the node destined for the detached chain. This branch performs no direct boundary-cache update.

### 6. Repair eligibility

After structural changes, the function reloads the container’s header pointer.

Repair is entered only if:

- The repair cursor differs from that header; and
- The detached node’s color byte is nonzero.

The detached color is not read for this eligibility test when the cursor equals the header. In the two-child case, the detached node already holds the successor’s former color.

If repair is not entered, no final cursor-color write from the repair region occurs.

**Inferred convention:** zero represents red and one represents black. Eligibility accepts every nonzero detached color, whereas loop and child-color conditions specifically compare against one. Values outside zero and one are not normalized or rejected.

### 7. Repair transitions

At each iteration, the root is read through the current container header. The body is bypassed if the cursor equals that root or its color is not exactly one. Root equality is checked before reading the cursor color.

Otherwise, the cursor’s parent is read. The cursor is treated as left-side if the parent’s +8 link identifies it, and right-side otherwise. The sibling is the opposite child.

“Near” denotes the sibling child toward the cursor; “far” denotes the other child. The two sides implement mirrored transitions:

| Condition | Ordered effects |
|---|---|
| Sibling absent | Write zero to the cursor, move the cursor to its parent, and reconsider the loop conditions. |
| Sibling color zero | Write one to the sibling, write zero to the parent, invoke the principal structural dependency, then reload the parent and sibling. If the sibling is absent afterward, take the absent-sibling transition. |
| Both sibling children are absent or have color exactly one | Write zero to the sibling, move the cursor to its parent, and reconsider the loop conditions. |
| Far child absent or color exactly one, after excluding the preceding case | Write one to the nonnull near child, write zero to the sibling, invoke the complementary structural dependency, and reload the sibling through the cursor’s parent. If absent, finish the repair region. |
| Remaining final-repair case | Copy the current parent color to the sibling, write one to the parent, write one to the nonnull far child, invoke the principal structural dependency, and finish the repair region. |

For a left-side cursor, the principal dependency is F-c3b5aab03a9c and the complementary dependency is F-6372ad19c9b6. For a right-side cursor, the roles reverse.

In the all-black-child decision, the near child is inspected first. The far child is inspected only if the near child is absent or color one. The far child is subsequently inspected for the preparation decision.

On leaving an entered repair region, the function writes one to the current cursor’s color. Thus, a cursor whose non-one color stopped the loop is still assigned one.

The dependencies suggest rotations, but their transformations are unknown. Some calls visibly provide the container and a parent value; complementary preparation calls visibly provide only the container. A uniform complete signature is not established.

### 8. Chain insertion, count update, and output

After structural changes and any repair, the function performs these operations in order:

1. Reads the container’s +4 chain head.
2. Writes that value to the detached node’s +12 field.
3. Writes the detached node’s address to container +4.
4. Reads container +20, subtracts one, and writes the four-byte result back.
5. Copies the current local result cell to result storage.

The detached node’s parent and left links are not explicitly cleared. Its color may have changed. Its right-link storage is repurposed as the chain link.

There is no underflow check. Under a wrapping 32-bit machine interpretation, decrementing zero produces the all-ones bit pattern. The static representation does not independently establish language-level signed-overflow semantics or whether zero is an admissible pre-removal count.

## Side effects

Direct writes may affect header fields, node links, node colors, the detached-node chain, the count-like field, and result storage.

The function contains no explicit allocation, deallocation, payload destruction, or allocator invocation. Prepending a node to a chain does not establish transfer of ownership, destruction of its contents, or permission for callers to release it.

Dependency effects are not bounded by the direct writes listed here. In particular:

- Initial dependencies can mutate the local cells whose addresses they receive.
- They may retain or expose those addresses during this invocation.
- Later repair dependencies may indirectly change the result cell or other reachable state.
- No dependency is assumed to preserve globals, container fields, links, allocation state, or memory ownership.

The final output write occurs after the chain and count changes. If result storage overlaps container, position, or node storage, it can overwrite state established earlier.

No synchronization is shown. Intermediate structural states and bytewise link clearing do not support an atomicity or thread-safety guarantee.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-ca7a1f20809e | Receives addresses of the four-byte result and header-value cells. A nonzero integer selects early completion. | Predicate meaning, cell mutation, retained addresses, external effects, ownership. |
| F-017ac946138e | Receives the result-cell address before position storage is reread. | Result selection, possible position advancement, retained addresses, external effects. |
| F-c3b5aab03a9c | Invoked during specified repair transitions after color writes. | Exact transformation, complete calling convention, indirect access to escaped local cells, other effects. |
| F-6372ad19c9b6 | Invoked in complementary repair transitions after color writes. | Exact transformation, complete calling convention, indirect access to escaped local cells, other effects. |

No dependency body is available. Familiar tree-repair patterns are not substitutes for dependency contracts.

## Acceptance examples

These are conditional acceptance criteria from static evidence, not executed tests. Structural examples assume valid, nonoverlapping storage and dependencies that do not disrupt the stated structure.

### Early completion

If F-ca7a1f20809e returns nonzero, result storage receives the header value reloaded after that call. The function itself does not invoke F-017ac946138e, remove a node, prepend a detached node, or decrement the count. Effects performed inside the decision dependency remain possible.

### Removal target changes during preparation

If the initial position value identifies one node but position storage identifies another when F-017ac946138e returns, the second value selects the removal target. The initial copy does not fix the target.

### Removing the sole node

For a leaf whose parent is the header:

- Header +12 becomes the header pointer, followed by +8 becoming that pointer, then +4 becoming zero.
- The node is prepended to the detached chain.
- The count-like field is decremented.
- Result storage receives the result cell’s contents at completion.

Under an unchanged header relationship, the repair cursor equals the header and repair is skipped.

### Removing a nonroot boundary leaf

For a leaf identified by its parent’s +8 link, that link is cleared through four ordered byte writes. If header +8 identifies the leaf, it becomes the parent. The corresponding right-side case uses parent +12 and header +12. Repair may subsequently change links or colors.

### Replacing a root with one child

The child first acquires the selected node’s parent, then replaces the selected node in header +4. Applicable boundary-cache updates follow. Repair remains conditional on cursor/header identity and detached color.

### Relocating a deeper successor

For a successor below the immediate right child, its former parent’s left link receives its right child, including zero if absent. The successor adopts the original subtrees and parent position. The original selected node, not the successor, is chained.

If the successor’s former right child is absent, the relocated successor is the repair cursor.

### Helper-produced result without later mutation

If F-017ac946138e leaves a particular four-byte value in the result cell and no subsequent indirect mutation changes that cell, completion writes that value to result storage. This function makes no intervening direct assignment to reconstruct or replace it.

### Helper-produced result with later mutation

If an initial dependency retains or exposes the result-cell address and a later repair dependency validly changes that cell, completion writes the changed contents. Returning the earlier value left by F-017ac946138e is not required. The source establishes the final copy, not unconditional preservation across opaque calls.

### Count boundary

For a zero count, no rejection is shown before decrement. Under wrapping 32-bit arithmetic, the stored result is all ones. Whether this input is a valid container state remains unspecified.

## Uncertainties

- **documented_only:** public types, exported calling convention, alignment, and admissible aliasing.
- **documented_only:** exact dependency behavior, including retention and use of local-cell addresses.
- **documented_only:** whether the initial decision denotes an end position, equality, or another predicate.
- **documented_only:** whether the result is an iterator, successor address, or another four-byte token.
- **documented_only:** repair-dependency transformations and their complete argument contracts.
- **documented_only:** behavior for malformed trees, cycles, invalid colors, nonmember positions, and inconsistent counts.
- **documented_only:** fault, exception, nonlocal-exit, and overflow semantics beyond the visible operations.
- **Inferred intent:** red-black deletion, cached extrema, retained-node storage, and a size counter.
- **Not established:** correctness for every valid red-black topology or preservation of the helper-produced result across later indirect mutation.

## Evidence

All established claims derive from the supplied static function body:

- Field accesses establish four-byte links and one-byte colors.
- Branches establish early completion, target rereading, and structural cases.
- Traversals and assignments establish replacement, relocation, and boundary updates.
- Color tests, writes, and opaque calls establish the repair transitions.
- Final operations establish chain insertion, decrement, and output ordering.
- Passing the result-cell address to both initial dependencies, combined with the absence of later direct assignments on the removal path, establishes a final-current-contents guarantee rather than unconditional helper-result preservation.

No dependency bodies, external API documentation, execution traces, or runtime tests were provided. No runtime behavior is claimed as observed.
