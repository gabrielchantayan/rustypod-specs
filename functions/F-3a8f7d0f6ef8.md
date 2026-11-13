<!-- Trusted provenance: F-3a8f7d0f6ef8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a8f7d0f6ef8

## Purpose

F-3a8f7d0f6ef8 conditionally removes a selected node from a parent-linked binary structure, updates links and selected endpoint metadata, performs color-based repair, and prepends the removed node to a container-associated chain. It also writes one four-byte output value.

The structure resembles a red-black tree with a header and a retained-node chain. That is **inferred intent**. The contract below follows the supplied static behavior, including its treatment of absent replacements and incompletely specified dependency calls, rather than substituting conventional tree-deletion rules.

On the removal path, the output comes from a live four-byte temporary whose address is passed to two dependencies. Its value is read at the final output store. It is not captured immediately after the second dependency returns; intervening indirect modifications can affect the output.

No runtime tests have been performed.

## Interface

### Inputs and result

| Input | Established use |
|---|---|
| Output destination | Receives a four-byte value at normal completion. |
| Container address | Provides the chain head, header reference, and count. |
| Selection address | Holds a four-byte node reference; read initially and reread after output preparation. |

There is no explicit return value, status code, or structured error result.

### Representation

References and integer fields occupy four bytes in the supplied representation. Color fields occupy one byte. Offsets below are in bytes.

**Container**

| Offset | Width | Role |
|---|---:|---|
| +4 | 4 bytes | Head of the chain receiving the removed node |
| +16 | 4 bytes | Header reference |
| +20 | 4 bytes | Count decremented on the removal path |

**Header**

| Offset | Width | Role |
|---|---:|---|
| +4 | 4 bytes | Root reference |
| +8 | 4 bytes | Cached left endpoint |
| +12 | 4 bytes | Cached right endpoint |

**Node**

| Offset | Width | Role |
|---|---:|---|
| +0 | 1 byte | Color or repair-state byte |
| +4 | 4 bytes | Parent reference |
| +8 | 4 bytes | Left-child reference |
| +12 | 4 bytes | Right-child reference; reused as a chain link after removal |

Zero references indicate absent children in the tested branches. Color values 0 and 1 have distinct meanings; their interpretation as red and black is inferred, not required terminology.

The function does not directly inspect or copy node payloads beyond these fields. It does not directly write the bytes between the color byte and parent reference.

### Preconditions and aliasing

There is no general validation of pointers, membership, counts, colors, or structural consistency. Normal completion requires valid storage for every executed access, sufficient alignment for four-byte accesses, terminating traversals, and dependencies that return with enough state valid for subsequent operations.

A selection that passes the initial dependency-controlled check is not thereby proven to belong to the structure. Invalid non-header references can be dereferenced.

No explicit alias check is present. The final output store can overwrite a container or node field modified earlier if the output destination aliases that field. The selection storage can also change before its second read. Postconditions therefore require appropriate non-aliasing and dependency-preservation assumptions where stated.

## Behavior

### Initial decision and live output storage

The function first reads the selection’s four-byte value into one temporary, then reads the container’s header reference into a second temporary. It passes both temporary addresses to F-10d8d7d1cfaa.

A nonzero result selects the early-completion path:

1. The container’s header reference is reread after the dependency returns.
2. That reference overwrites the first temporary.
3. The final output store reads that temporary and writes the output destination.

This path skips the function’s own unlinking, repair, chain insertion, and count decrement. It does not establish that the dependency made no changes.

A zero result selects removal. The function calls F-5c9931762599 with the address of the first temporary. After that call returns, it rereads the selection address to obtain the node to remove.

The first temporary remains live until final completion. Both initial dependencies have received its address and could retain it. Later repair dependencies could consequently modify it indirectly. The function does not save a separate output snapshot after F-5c9931762599.

Thus:

- The removed node is determined by the selection storage’s second read.
- The eventual output is determined by the temporary’s value at the final output store.
- Equality with the temporary’s value immediately after F-5c9931762599 requires preservation of that value through the remaining execution.

An iterator-advance interpretation of F-5c9931762599 is plausible but unestablished.

### Removal shape

The selected node’s left child is inspected first. Its right child determines the remaining shape:

- With neither child, the node is detached as a leaf.
- With exactly one child, that child replaces it.
- With two children, the leftmost node in the right subtree is moved into its position.

There are no key comparisons or payload exchanges. The two-child case rewires links and exchanges color bytes.

The **removed node** below is the selected node eventually placed on the chain. The **repair position** is a separate reference whose initial value depends on the removal shape.

### Leaf removal

The repair position becomes the selected node’s parent.

If that parent equals the currently loaded header reference, writes occur in this order:

1. The header’s right endpoint becomes the parent/header reference.
2. The header’s left endpoint becomes that reference.
3. The header’s root becomes zero.

Otherwise, the parent’s left-child reference is compared with the selected node. Equality selects the left link; inequality selects the right link without separately verifying its contents.

The chosen link is cleared by four successive one-byte zero stores in increasing address order, not by one four-byte store.

The corresponding cached endpoint is then checked: left endpoint for a left-link removal, right endpoint otherwise. If it equals the selected node, it becomes the parent. The other endpoint is not independently recomputed.

### Removal with one child

The replacement child’s parent is first set to the selected node’s parent.

The selected node’s incoming link is then replaced:

- The header’s root field is used if it currently equals the selected node.
- Otherwise, the parent’s left link is used if it equals the selected node.
- Otherwise, the parent’s right link is used without a separate equality check.

Endpoint processing follows the incoming-link replacement.

If the left endpoint equals the selected node:

- When the selected node’s right child is absent, the endpoint becomes the selected node’s parent.
- Otherwise, the function follows left links from the replacement and stores the leftmost node reached.

If the right endpoint equals the selected node:

- When the selected node’s left child is absent, the endpoint becomes the selected node’s parent.
- Otherwise, the function follows right links from the replacement and stores the rightmost node reached.

The replacement child is the initial repair position. The selected node remains the node destined for the chain.

### Removal with two children

The successor is found by starting at the selected node’s right child and following left links until the next left link is absent. Its former right child is initially the repair position.

The established mutation order is:

1. The selected node’s left child receives the successor as parent.
2. The successor receives the selected node’s left subtree.
3. The successor’s former location is repaired.
4. The selected node’s incoming root or parent link is redirected to the successor.
5. The successor receives the selected node’s former parent.
6. The successor and selected node exchange color bytes.

Former-location handling has two cases.

**Immediate right-child successor.** Its right child, if present, receives the successor as parent. The successor’s existing right-child link is retained.

**Deeper successor.** Its right child, if present, receives the successor’s former parent. That former parent’s left link becomes the successor’s former right child. The successor then receives the selected node’s right subtree, and that subtree’s root receives the successor as parent.

If the successor’s former right child is absent, the repair position becomes the successor itself. This is an established behavior, not an assumed null-leaf placeholder.

The selected node is ultimately chained. Its color after the exchange is the successor’s former color. Cached endpoints are not directly updated on this path.

### Color-based repair

Repair is entered only if the repair position differs from the current header reference and the removed node’s color byte is nonzero. When repair is skipped, the final repair-position color write is also skipped.

At each loop entry, the root is obtained through the container’s then-current header reference. Iteration stops if the repair position is that root or its color is not exactly 1. Otherwise, its parent and sibling are examined.

The orientation depends on whether the repair position equals its parent’s left child:

| Position | Sibling | Near sibling child | Far sibling child | Primary dependency |
|---|---|---|---|---|
| Left child | Right child | Left | Right | F-da6a2f17419a |
| Otherwise | Left child | Right | Left | F-3a8a0224b9df |

The “otherwise” orientation does not separately verify that the repair position equals the parent’s right child.

The cases below apply in either orientation.

**Absent sibling.** The current repair position is written to color 0, and repair continues from the parent.

**Sibling of color 0.** The sibling becomes color 1, then the parent becomes color 0. The primary dependency is called with the container and parent. The parent and sibling are reread afterward. An absent reread sibling takes the absent-sibling case.

**Both sibling children absent or color 1.** The near-child condition is examined before the far-child condition. Null children satisfy the condition without a color dereference. The sibling becomes color 0, and repair continues from the current position’s parent.

**Far child absent or color 1, after the preceding combined condition failed.** The near child is written to color 1 when its link is nonnull. The sibling becomes color 0. The opposite orientation’s dependency is called; only the container argument is visible at this call site. The sibling is then reread through the repair position’s parent. If it is absent, execution proceeds to final coloring.

**Final sibling case.** The sibling receives the parent’s color, then the parent becomes color 1. The far child becomes color 1 if present. The primary dependency is called with the container and parent, and iteration ends.

Whenever an entered repair sequence finishes, the then-current repair position is written to color 1.

These predicates are exact byte comparisons. A nonzero removed-node color enables repair, but loop continuation requires the repair-position color to equal 1. Arbitrary other color values must not be silently normalized into conventional tree semantics.

### Chain insertion, count, and final output

After structural changes and any repair:

1. The container’s current chain head is read.
2. It is stored in the removed node’s field at +12.
3. The removed node becomes the container’s chain head.
4. The current four-byte count is read, decremented by one, and stored.
5. The live output temporary is read and its four-byte value is stored at the output destination.

The final output includes any intervening indirect modifications to that temporary, including modifications enabled by retained addresses passed to the initial dependencies. The function does not restore its post-F-5c9931762599 value.

There is no count-underflow check. At the observed four-byte arithmetic level, decrementing zero yields the all-ones bit pattern. The higher-level signed-overflow model is not established.

## Side effects

Direct writes can affect header fields, neighboring parent and child links, successor links, color bytes, the removed node’s chain-link field, the container’s chain head and count, and the output destination.

The function does not directly allocate memory, free memory, destroy payloads, or clear all detached-node metadata. In particular, it does not generally clear the removed node’s parent or left-child fields.

Prepending to a chain suggests retention for reuse or later disposal. Ownership transfer and eventual reclamation remain **documented_only**. Dependency calls may have additional memory, ownership, or global-state effects.

The operation has no visible transaction or rollback mechanism. Structural writes precede repair calls; chain insertion precedes the count update; the output store is last. Failures can therefore occur after partial mutation. No recovery result is specified.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-10d8d7d1cfaa | Receives addresses of the selection-derived and header-derived four-byte temporaries. Its zero/nonzero result selects the path. | Comparison semantics, temporary mutation or address retention, ownership and other side effects |
| F-5c9931762599 | Receives the address of the live output temporary before selection is reread. | Output meaning, address retention, iterator semantics, other side effects |
| F-da6a2f17419a | Used during repair; primary right-sibling calls visibly supply container and parent, while an opposite-orientation preparation call visibly supplies only container. | Exact signature, structural effects, interpretation of the differing visible arguments |
| F-3a8a0224b9df | Used in the mirrored repair orientation with the corresponding argument-visibility difference. | Exact signature, structural effects, interpretation of the differing visible arguments |

No dependency is assumed to preserve globals, selection storage, container fields, reachable links, or the live output temporary. Nor is any dependency assumed to own, retain, or release memory.

Rotation-like behavior is inferred from the repair pattern. It is not an established callee contract.

## Acceptance examples

These are conditional consequences of static paths, not executed tests. Each assumes valid accesses, normal dependency returns, and preservation of state not explicitly permitted to change in that example.

1. **Nonzero initial result.**  
   F-10d8d7d1cfaa returns nonzero. The output receives the container’s header reference reread after that call, even if the dependency changed the earlier header value. This function performs no removal, chain insertion, or decrement.

2. **Single-node structure.**  
   The selected node has no children and its parent is the header. The right endpoint becomes the header, then the left endpoint becomes the header, then the root becomes zero. Repair is skipped. The node is chained and the count decreases.

3. **Color-0 leaf under a non-header parent.**  
   The selected parent link is cleared by four ordered byte stores. A matching corresponding endpoint becomes the parent. Repair is skipped because the removed node’s color is zero.

4. **Root with one child.**  
   The child’s parent becomes the header before the root field becomes the child. Endpoint handling follows. If repair is entered, the root check ends iteration and final coloring sets the child to color 1.

5. **Immediate successor without a right child.**  
   The successor replaces the two-child selection, receives its left subtree, and exchanges colors with it. The repair position is the successor, not null. The selected node is chained.

6. **Deeper successor with a right child.**  
   That child replaces the successor in its former parent’s left link and receives that parent. The successor receives the selected node’s subtrees and former parent. The successor’s former right child remains the initial repair position.

7. **Intervening modification of the output temporary.**  
   An initial dependency retains the temporary’s address. F-5c9931762599 leaves one four-byte value there. A later repair dependency validly writes a different value through the retained address. With no subsequent modification, the final output is the later value.

8. **Preserved output temporary.**  
   If no operation changes the temporary after F-5c9931762599 returns, the final output equals its value immediately after that call. This equality depends on preservation, not on an internal snapshot.

9. **Zero count.**  
   Count zero alone does not reject removal. If execution reaches the decrement, the observed four-byte result is all ones.

10. **Output aliases the chain head.**  
    The node is first installed as chain head. The final output store then overwrites that same field. A lasting chain-head postcondition requires non-aliasing.

## Uncertainties

The following remain **documented_only**:

- Whether the complete operation preserves a red-black-tree invariant under the actual dependency contracts.
- The semantic meaning of the output value and whether any dependency retains its temporary address.
- Dependency signatures where only one argument is visible.
- Dependency effects on globals, links, counts, selection storage, ownership, or lifetimes.
- Ownership and eventual disposal of chained nodes.
- Target endianness and higher-level signed-overflow rules.
- Behavior with malformed links, cycles, invalid colors, unsafe aliases, or concurrent mutation.
- Dependency termination, nonlocal failure, and access-failure behavior.

No allocation bound, synchronization guarantee, rollback guarantee, or complexity guarantee is established. Traversal cost depends on the encountered structure and dependency behavior.

## Evidence

The supplied static representation establishes the listed field widths and offsets, explicit branch predicates, direct mutation order, dependency-call order, and final output store.

In particular, the same address-exposed four-byte temporary is used at final completion. There is no separate capture of its value after F-5c9931762599. Address retention and later indirect modification are possibilities left open by unavailable dependency bodies, not claims that such modification actually occurs.

Tree, header, endpoint, successor, rotation, and recycling terminology expresses inferred roles supported by the accesses. It does not establish hidden invariants or memory-management contracts.

The acceptance examples are conditional static deductions. No implementation was executed, no runtime tests were performed, and no external provenance identifier is asserted.
