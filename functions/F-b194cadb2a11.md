<!-- Trusted provenance: F-b194cadb2a11; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b194cadb2a11

## Purpose

F-b194cadb2a11 conditionally detaches a node from a linked tree, performs color-dependent structural repair, places the detached node at the head of a container-associated chain, decrements a 32-bit count, and publishes a 32-bit result through caller-provided storage.

The structure is consistent with an ordered red-black tree. Tree erasure, iterator advancement, and node recycling are inferred intent, not independently established dependency contracts. The behavior below retains unusual repair choices rather than replacing them with a conventional deletion algorithm.

A preliminary dependency can bypass removal. On that path, this function publishes the container’s header reference as reread after the dependency returns and performs no direct tree, chain, or count updates.

On the removal path, the final output is the then-current contents of a private result word. Two early dependencies receive that word’s address. Preservation of a value they produce is not guaranteed across later dependency calls.

This contract is based solely on supplied static evidence. No runtime tests have been performed.

## Interface

### Arguments and output

| Argument | Representation | Observed use |
|---|---|---|
| Result storage | Address of a writable 32-bit word | Receives the final result word. |
| Container reference | 32-bit address representation | Locates the chain head, header reference, and count. |
| Input storage | Address of a readable 32-bit word | Supplies the initial helper input and, on the removal path, is reread to select the removal target. |

No explicit return value is established. The caller-visible result is a 32-bit store through result storage.

Pointer-bearing fields and ordinary word transfers are 32 bits wide. Color accesses are one byte wide. Byte order, alignment requirements, and invalid-address behavior are not established.

### Container layout

Offsets are byte offsets from the container reference.

| Offset | Width | Observed role |
|---|---:|---|
| +4 | 32 bits | Head of the chain receiving detached nodes |
| +16 | 32 bits | Header reference |
| +20 | 32 bits | Count decremented on the removal path |

No other container fields are directly accessed by this function. Dependencies may access additional state.

### Header and node layout

| Offset | Width | Ordinary node | Header |
|---|---:|---|---|
| +0 | 8 bits | Color marker | No color access on the ordinary sole-root removal path |
| +4 | 32 bits | Parent reference | Root reference |
| +8 | 32 bits | Left-child reference | Cached leftmost reference |
| +12 | 32 bits | Right-child reference; reused as chain link after detachment | Cached rightmost reference |

A zero child reference denotes absence. The header is a separate structural marker; it is not interchangeable with a zero child.

Color 0 has the role conventionally associated with red, and color 1 with black. Some branches test zero versus nonzero; others test exactly 1. Values other than 0 and 1 therefore cannot be treated uniformly as black.

### Preconditions and validity limits

Successful completion requires accessible argument storage and all reached fields, using the widths above. References and traversal chains must support the selected operations and terminate. Dependencies must return normally and leave sufficient validity for subsequent accesses.

For the ordinary tree-removal interpretation, the selected target must belong to the current container, and parent, child, root, and extremum references must be consistent. The function does not validate those conditions.

No explicit error result covers null arguments, inaccessible memory, foreign nodes, malformed links, invalid colors, or an inconsistent count.

The interface does not explicitly prohibit aliasing. However, straightforward structural postconditions require distinct storage where overlap would interfere with those operations. In particular:

- Input storage is read again after the early dependencies.
- Structural and count writes precede the final result write.
- Result storage overlapping a field can overwrite that field at publication.
- Dependencies may retain an address passed to them and subsequently mutate the addressed private word while it remains live.

Behavior under arbitrary overlapping storage is **documented_only**; it is not a general alias-safe erase contract.

## Behavior

### Preliminary decision and result preparation

The function first copies the input word into a private 32-bit result word. It next copies the container’s header reference into a second private 32-bit word. F-dac28b48a0fc receives the addresses of both private words.

If that dependency returns nonzero:

1. The container’s header reference is reread.
2. That value replaces the private result word.
3. The private result word is written to result storage.
4. The function returns.

There are no intervening dependency calls on this bypass path. Any effects of F-dac28b48a0fc itself remain outside this function’s direct-update guarantees.

If the dependency returns zero, F-047d5ca4034d receives the address of the private result word. After it returns, the function rereads input storage and uses that newly read word as the removal target.

The output source and target source are therefore distinct: removal uses the reread input word, while publication later reads the private result word.

This function does not explicitly recompute that private result word after F-047d5ca4034d. Nevertheless, its value is not necessarily fixed at that point. F-dac28b48a0fc and F-047d5ca4034d have received its address and may retain it. Later repair dependencies cannot be assumed to preserve it or to lack access to it through retained or shared state. The final store uses the word’s value at publication, after repair, chain insertion, and count update.

### Selecting the structural case

The target’s left-child field is read first.

- With no left child, the right-child field distinguishes a leaf from replacement by the right child.
- With a left child, the right-child field distinguishes replacement by the left child from the two-child case.
- With two children, traversal starts at the right child and follows left-child references until reaching a node without a left child. That node is the successor used for structural replacement.

The original target is the object eventually linked into the container-associated chain. In the two-child case, the successor is relocated structurally; no payload copy is visible.

### Leaf removal

The target’s parent becomes the repair cursor.

If that parent equals the container’s current header reference, the writes occur in this order:

1. Header rightmost becomes the parent reference, which equals the header.
2. Header leftmost becomes that same reference.
3. Header root becomes zero.

Otherwise, the parent’s left-child field is compared with the target.

- If equal, the parent’s left link is cleared.
- Otherwise, the parent’s right link is cleared without a separate equality check.

Each such link clear consists of four one-byte zero stores in increasing address order, rather than a single 32-bit store.

The corresponding extremum cache is then checked: leftmost for the left-link case, rightmost otherwise. If it equals the target, it is replaced with the parent.

These are ordered, intermediate writes; atomic link clearing or atomic removal is not established.

### Replacement by one child

The replacement child first receives the target’s parent reference.

Next, the function checks the current header’s root field. If it identifies the target, it is replaced with the child. Otherwise, the target’s parent’s left link is compared with the target: equality selects replacement of the left link; inequality selects replacement of the right link.

Extremum caches are considered afterward, leftmost before rightmost. Each is changed only if it currently equals the target.

For the leftmost cache:

- If the target’s right child is absent, the new cache is the target’s parent.
- Otherwise, the function follows left links from the replacement child and uses the terminal node.

For the rightmost cache:

- If the target’s left child is absent, the new cache is the target’s parent.
- Otherwise, the function follows right links from the replacement child and uses the terminal node.

These are the visible branch rules, including cases that ordinary valid-tree extrema may make unusual or unreachable.

The replacement child becomes the repair cursor.

### Successor relocation for two children

The successor is the leftmost node in the target’s right subtree. The successor’s right child, possibly zero, is the replacement at its former position.

The update order is:

1. Set the target’s left child’s parent to the successor.
2. Set the successor’s left link to the target’s left child.
3. Resolve the successor’s former location:
   - If the successor is the target’s immediate right child, set its right child’s parent to the successor when that child exists.
   - Otherwise, set its right child’s parent to the successor’s old parent when that child exists; replace the old parent’s left link with that child; give the successor the target’s right subtree; then set that subtree root’s parent to the successor.
4. Replace the target in the header’s root field or in its parent’s selected child field. The parent’s left link is preferred when it equals the target; otherwise the right link is written.
5. Set the successor’s parent to the target’s parent.
6. Exchange the successor’s and target’s one-byte colors.

There are no direct extremum-cache updates in this branch.

If the successor originally had a right child, that child becomes the repair cursor. Otherwise, the relocated successor itself becomes the cursor. The function does not substitute a null-child cursor or automatically use the successor’s old parent.

The original target remains the detached object. Its color after the exchange is the successor’s former color and controls repair entry.

### Color-dependent repair

Repair is entered only if the cursor differs from the container’s current header and the detached object’s current color is nonzero. If the cursor equals the header, the detached object’s color need not be read for this decision.

At the start of each iteration, the cursor is compared with the current root. A root cursor finishes repair without needing its color read for the loop condition. Otherwise, a cursor whose color is not exactly 1 also finishes. These exit paths write color 1 to the cursor.

For an exactly-1, non-root cursor, the function reads its parent and compares the parent’s left link with the cursor:

- Equality selects the parent’s right child as sibling.
- Inequality selects the parent’s left child as sibling.

“Near child” below means the sibling child toward the cursor; “far child” means the other child.

**Absent sibling.** The cursor is colored 0, the parent becomes the cursor, and the loop is reevaluated. This color write is part of the visible behavior.

**Sibling color 0.** The sibling is colored 1, the parent is colored 0, and a structural dependency is called: F-bf05f6f01092 for a left-side cursor, F-030a3110641a for a right-side cursor. The cursor’s parent and sibling are then reloaded. If the sibling is absent, processing takes the absent-sibling case using the reloaded parent.

**Both sibling children absent or exactly 1.** Child references are examined near before far for this decision. Missing children are accepted without dereferencing them. If both qualify, the sibling is colored 0, the cursor advances to its parent, and the loop repeats.

**Preliminary transformation.** Otherwise, if the far child is absent or exactly 1:

1. Color the near child 1 when present.
2. Color the sibling 0.
3. Call F-030a3110641a for a left-side cursor, or F-bf05f6f01092 for a right-side cursor.
4. Reload the sibling through the cursor’s parent.

If the reloaded sibling is absent, repair ends by coloring the cursor 1.

**Final transformation.** The sibling receives the parent’s color, the parent is colored 1, and the sibling’s far child is colored 1 when present. The orientation-appropriate dependency is then called: F-bf05f6f01092 for a left-side cursor, F-030a3110641a for a right-side cursor. Finally, the cursor is colored 1 and repair ends.

Structural dependency effects are unknown. Their placement and subsequent reloads are established, but successful rotation semantics are not.

### Chain insertion, count update, and publication

After structural processing and any repair:

1. Read the container’s current chain head.
2. Write it into the detached original target’s +12 field.
3. Write the detached original target into the container’s +4 chain-head field.
4. Read the container’s 32-bit count, subtract one, and write the 32-bit result back.
5. Read the private result word’s then-current value and write it to result storage.

No underflow check is visible. The representation shows a signed 32-bit count operation. Behavior outside the valid signed range is not resolved: guaranteed modular wraparound and guaranteed checked failure are both unsupported.

The detached target’s parent and left links are not explicitly cleared. Its color may have been exchanged, and its +12 field is repurposed as the chain link.

## Side effects

Direct effects include structural reference writes, byte-wide color writes, detached-node chain insertion, count decrement, and final output publication. Header references and other fields are reread at multiple points rather than being uniformly fixed before all dependency calls.

No direct allocation, deallocation, payload access, payload destruction, or storage-release operation is visible. Chain membership alone does not establish recycling ownership or reclamation.

Dependencies may modify globals, container fields, reachable memory, or retained private-word addresses. Their implementations do not establish preservation or ownership guarantees.

No synchronization, rollback, exception translation, or recovery is visible. Failure or interruption after intermediate writes is not shown to restore prior state.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-dac28b48a0fc | Receives addresses of the private result word and private header-copy word; zero versus nonzero selects removal | Comparison meaning, mutations, address retention, ownership, and wider effects |
| F-047d5ca4034d | Receives the private result word’s address before input storage is reread | Advancement meaning, resulting value, address retention, ownership, and wider effects |
| F-bf05f6f01092 | Used in left-oriented main transformations and mirrored preliminary transformations | Internal structural effects and complete argument requirements |
| F-030a3110641a | Used in right-oriented main transformations and mirrored preliminary transformations | Internal structural effects and complete argument requirements |

Main structural calls show the container and a parent reference. Preliminary structural calls show only the container. This discrepancy is **documented_only**; missing arguments cannot safely be invented.

“Comparison,” “advancement,” and “rotation” describe inferred roles, not dependency guarantees. In particular, a repair dependency must not be presumed unable to modify the result word merely because that call does not explicitly receive its address.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Structural examples assume accessible, consistent, noninterfering storage and dependencies compatible with the stated scenario.

1. **Bypass after header change.** If F-dac28b48a0fc changes the container header reference and returns nonzero, output must use the header reread after that call. This function must not call F-047d5ca4034d or perform its own structural removal, chain insertion, or count decrement.

2. **Distinct result and target.** If F-047d5ca4034d leaves the private result word identifying B, while the subsequent input reread identifies A, removal targets A. Output is B only if the private result word still contains B at publication.

3. **Later mutation through a retained address.** If an early dependency retains the private result word’s address and an invoked repair dependency changes that word from B to C, with execution otherwise completing normally, final output must be C. No explicit result recomputation by F-b194cadb2a11 is required for that change to be visible.

4. **Sole-root leaf.** With the target’s parent equal to the header, removal writes header rightmost to the header, then leftmost to the header, then root to zero. The target enters the chain with its +12 field holding the previous chain head. Count 1 becomes 0.

5. **Root with one child.** The child receives the target’s parent before the header root is changed to that child. Repair then depends on the detached target’s color and the cursor’s state.

6. **Immediate successor.** With two children and an immediate-right-child successor, the successor takes the target’s left subtree and structural position. Its existing right child, if any, has its parent written to the successor. The original target enters the chain.

7. **Deeper successor without a right child.** The successor’s old parent’s left link becomes zero. After relocation and color exchange, the moved successor is the repair cursor, not a synthesized null position or automatically its old parent.

8. **Absent sibling.** For an exactly-1, non-root cursor with no sibling, the cursor is first colored 0; only then does repair advance to its parent.

9. **Count boundary.** Initial count zero does not trigger visible validation before removal. For an ordinarily representable signed decrement, the stored value becomes −1; the function does not reject it merely because it is negative. A decrement below the signed minimum remains unresolved.

## Uncertainties

- **Inferred intent:** ordered-tree erasure, red-black color meanings, iterator advancement, and node recycling.
- **documented_only:** dependency implementations, complete structural-call signatures, ownership, payload layout, and external invariants.
- **documented_only:** retention and later use of private-word addresses. The helper-produced result is preserved only if dependency behavior preserves it until publication.
- **documented_only:** malformed references, arbitrary aliasing, dependency failures, nontermination, and arithmetic outside the valid signed range.
- **Not established:** atomicity, concurrency safety, diagnostics, reclamation, or preservation of globals.
- **Not established:** reachability of every unusual repair or extremum-update path under intended valid-tree invariants.

## Evidence

The supplied static representation establishes argument use, field widths and offsets, branch conditions, dependency-call placement, structural and color writes, chain insertion, count decrement, and final publication from the private result word.

The passed addresses establish potential dependency access to that word; unavailable dependency bodies do not establish that such access ends when an early call returns. Accordingly, final output is specified by the word’s value at the final read, not unconditionally by its value immediately after F-047d5ca4034d.

Tree terminology and helper roles remain interpretations. No dependency bodies, executable observations, or runtime tests were available. All acceptance examples are conditional static criteria.
