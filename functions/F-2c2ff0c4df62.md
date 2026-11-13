<!-- Trusted provenance: F-2c2ff0c4df62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c2ff0c4df62

## Purpose

F-2c2ff0c4df62 conditionally removes a node from a linked, ordered-tree-shaped structure. On the removal path, it redirects links, updates applicable cached boundaries, performs color-based repair, invokes a removal-related dependency, decrements a 32-bit count, and writes a 32-bit result to caller-provided storage.

The structure and repair rules suggest a red-black tree with a header node. That is **inferred intent**, not an independently established container contract. In particular, some repair paths use an existing node where a conventional deletion description might use a null replacement. Substituting a textbook algorithm requires separate equivalence evidence.

This contract distinguishes direct behavior established by the supplied representation from unknown dependency behavior. No runtime tests have been performed.

## Interface

### Arguments and output

| Argument | Established use |
|---|---|
| Result destination | Address receiving one 32-bit word on normal completion. |
| Container address | Base of storage containing a header address at byte offset 16 and a count at byte offset 20. |
| Input-position address | Address of a 32-bit word read initially and read again, on the removal path, after the first two dependency calls. |

There is no explicit return value. The caller-visible result is the final store through the result destination.

The represented address model uses 32-bit pointer-bearing words. Native 64-bit pointer layouts are not established.

The function retains an internal 32-bit result temporary throughout the operation. Its address is passed to dependencies. **Retaining this storage does not establish preservation of its contents:** a dependency could retain its address, and a later dependency could modify it. On the removal path, the output is the value read from this temporary after the removal-related dependency returns and after the count decrement.

### Directly accessed layout

| Object | Byte offset | Width | Operational role |
|---|---:|---:|---|
| Container | 16 | 32 bits | Header address |
| Container | 20 | 32 bits | Count |
| Node or header | 0 | 8 bits | Color/state byte, where accessed |
| Node or header | 4 | 32 bits | Parent link; header uses this as root link |
| Node or header | 8 | 32 bits | Left-child link; header uses this as cached first-node link |
| Node or header | 12 | 32 bits | Right-child link; header uses this as cached last-node link |

Color writes affect only the byte at offset 0. No meaning or explicit modification is established for bytes 1–3.

Zero denotes an absent child. A nonzero, accessible header address is required for the paths that dereference it.

### Preconditions and failure boundaries

Normal completion requires accessible argument storage, accessible traversed nodes, terminating traversal chains, and structural relationships sufficient for the selected relinking and repair path. The removal target must be suitable for those operations. Dependency preconditions are unavailable.

There are no explicit checks for null arguments, invalid node membership, inconsistent links, cycles, or count bounds. No error code, exception translation, or rollback is visible.

No non-aliasing contract is established. A result destination overlapping container or node storage receives its final store after the other direct work, potentially overwriting that storage.

## Behavior

### Initial decision and result storage

The function first reads the input-position word into the result temporary. It then reads the container’s header address into a separate 32-bit temporary. F-f69853ed0279 receives the addresses of both temporaries.

If that dependency returns nonzero:

- The function rereads the container’s header field.
- It assigns that header value to the result temporary.
- It writes the temporary’s value to the result destination.
- It performs no direct structural removal, removal-related call, or count decrement.

Dependency effects are not excluded. The nonzero result is not independently established as an error, equality test, or end-position test.

If the dependency returns zero, F-7d84ed08254e receives the address of the result temporary. After this call, the function rereads the input-position word to obtain the removal target. Consequently, the target need not equal the initially captured position if a dependency changed the input storage.

The result temporary remains the source of the eventual output, but its value immediately after F-7d84ed08254e is **not** guaranteed to be the final value. Both initial dependencies have received its address; their pointer-retention and later mutation behavior are unknown. The body does not take an immutable result snapshot at this point.

### Selecting the replacement

The function reads the target’s left-child link first.

- With no left child, it examines the right child. A nonzero right child is the replacement; otherwise the target is a leaf.
- With a left child but no right child, the left child is the replacement.
- With both children, it follows left-child links from the right subtree until reaching a node with no left child. That node is the structural successor under the inferred ordering interpretation. Its right child is the possible replacement at its former location.

There is no direct key or payload comparison.

### Leaf removal

The target’s parent becomes the repair position.

If that parent equals the current container header, the function writes, in order:

1. The header’s cached last-node link to the parent/header address.
2. The header’s cached first-node link to the same address.
3. The header’s root link to zero.

This is the empty-header shape established on this path.

Otherwise, the parent’s left link is inspected. If it equals the target, that link is cleared; otherwise the right link is cleared without a separate target-equality check. The selected link is cleared through four represented byte stores in increasing offset order.

The function then examines only the corresponding cached boundary: cached first for a left-side removal, cached last otherwise. If that boundary equals the target, it becomes the parent. The repair position remains the parent, not null.

### One-child replacement

The replacement child’s parent link is written before the target’s incoming link is redirected.

The incoming-link choice first checks whether the current header’s root equals the target. If so, the root becomes the replacement. Otherwise, the target parent’s left link is checked; a match redirects that link, and a nonmatch redirects the right link without separately verifying it.

Boundary maintenance follows, with cached first processed before cached last:

- If cached first equals the target, it becomes the target’s parent when the target has no right child; otherwise it becomes the leftmost node reachable from the replacement.
- If cached last equals the target, it becomes the target’s parent when the target has no left child; otherwise it becomes the rightmost node reachable from the replacement.

These are the established conditions, including their behavior on unusual structures. The function does not validate whether a combination is consistent with an ordered-tree invariant.

The replacement child is the repair position. The original target remains the node designated for the removal-related dependency.

### Two-child successor movement

The function moves the successor node into the target’s structural position. It does not directly copy the target’s payload.

The target’s left child first receives the successor as parent, and the successor receives the target’s left-child link.

If the successor is the target’s immediate right child, its right replacement, when nonzero, receives the successor as parent.

If the successor is deeper in the right subtree, the function performs these changes in order:

- A nonzero right replacement receives the successor’s former parent.
- The former parent’s left link becomes that replacement, including zero when absent.
- The successor’s right link becomes the target’s right child.
- That right child receives the successor as parent.

The target’s incoming link is then redirected to the successor using the root check and parent-left-check fallback described above. The successor receives the target’s parent link.

Finally, the successor’s original color byte is retained, the successor receives the target’s color byte, and the target receives the retained successor color. If the successor has no right replacement, the moved successor itself becomes the repair position; otherwise that replacement is the repair position.

The original target remains the node passed to F-3619430fef39. No cached-boundary update is directly performed in this two-child branch.

### Color-based repair

After relinking, the function rereads the container’s header. It enters repair only if the repair position differs from that header and the designated removed node’s color byte is nonzero. In the two-child case, the preceding exchange makes this byte the successor’s original color.

The numeric rules are established:

- Color 1 is tested for continuing upward repair.
- Color 0 is the contrasting value used by recoloring.
- Any nonzero removed-node color can admit repair, but a current repair-position color other than 1 stops the loop.

Interpreting 1 as black and 0 as red is inferred intent.

Each iteration first checks the current header’s root against the repair position. If they match, or if the position’s color is not 1, repair proceeds to its terminating color write. Otherwise, the position’s parent and its relationship to that parent determine orientation:

| Position relationship | Sibling | Near child | Far child | Final rotation-like dependency |
|---|---|---|---|---|
| Equals parent’s left child | Parent’s right child | Sibling’s left child | Sibling’s right child | F-7dc0dcd36c28 |
| Otherwise | Parent’s left child | Sibling’s right child | Sibling’s left child | F-7b527193a60e |

The right-side orientation is selected by failure of the left-child equality test; there is no separate right-child validation.

The repair cases are:

- **Absent sibling:** write color 0 to the current position, move the position to the retained parent, and repeat.
- **Sibling color 0:** write sibling color 1, write parent color 0, and invoke the orientation’s final rotation-like dependency. The parent is then reread through the current position’s parent-link field, and the sibling is reread. An absent refreshed sibling follows the absent-sibling case using the refreshed parent.
- **Both sibling children absent or color 1:** write sibling color 0, reread the current position’s parent as the next position, and repeat.
- **Far child absent or color 1, but the preceding case does not apply:** write near-child color 1 if present, write sibling color 0, invoke the opposite rotation-like dependency, and reread the sibling through the current position’s parent. If this sibling is absent, proceed to the terminating color write.
- **Final repair:** copy the current parent’s color to the sibling, write parent color 1, write far-child color 1 if present, invoke the orientation’s final dependency, and terminate repair.

Every exit through the repair block’s terminating point writes color 1 to the current repair position. If repair was never entered, this write does not occur.

Dependencies are not assumed to preserve the header, links, colors, globals, or escaped temporary storage. Subsequent explicit rereads observe whatever accessible values exist then.

### Removal-related call, count, and final output

After structural work and any repair, the function:

1. Calls F-3619430fef39 with the container address, original target node, and integer 1.
2. Reads the container’s count after that call returns.
3. Decreases that count by one and writes the 32-bit result back.
4. Reads the result temporary and writes its 32-bit value to the result destination.

The final output read is therefore later than both the removal-related call and the count update. It must not be replaced by an assumed snapshot from immediately after F-7d84ed08254e. A later dependency may have changed the temporary through a retained pointer.

The count operation is represented as signed integer subtraction in 32-bit storage. There is no count-zero check. Crossing the signed lower bound is not assigned portable wrapping semantics by this contract; that boundary remains documented_only.

## Side effects

Direct external writes affect the result destination, count, header links, affected node links, and affected color bytes. The function also maintains internal temporary storage whose addresses escape to dependencies. Dependency-mediated writes, including later writes to the result temporary, cannot be bounded from this body alone.

There is no direct allocation or explicit release operation in the visible body. F-3619430fef39 may concern destruction or disposal, but its ownership, deallocation, resource-release, and post-call lifetime rules are unknown. Passing the target to it does not independently establish ownership transfer.

The function does not clear the removed node’s entire link area. Nor is the output established to be the removed node or a successor: it is the final value read from the result temporary.

Mutations precede the removal-related call, count decrement, and final output. There is no rollback. If a dependency does not return normally, the later actions are not established to occur.

No synchronization, atomicity, or concurrent-access guarantee is visible. Represented byte-by-byte link clearing must not be described as an established atomic word update.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-f69853ed0279 | Receives addresses of the result and header temporaries. Zero permits removal; nonzero selects the skip path. | Comparison meaning, temporary mutation, retained pointers, global effects, abnormal exits. |
| F-7d84ed08254e | Receives the result temporary’s address before the removal target is reread. | Position transformation, validation, pointer retention, other storage effects. |
| F-7dc0dcd36c28 | Used for left-position final repair and the opposite orientation’s near-child adjustment. | Exact transformation, complete calling convention, memory and global preservation. |
| F-7b527193a60e | Used for right-position final repair and the opposite orientation’s near-child adjustment. | Exact transformation, complete calling convention, memory and global preservation. |
| F-3619430fef39 | Receives container, original target, and integer 1 after repair and before the count read. | Ownership, destruction, release, count mutation, retained-pointer effects, failure behavior. |

Several repair calls visibly supply the container and a parent value. Near-child adjustment calls visibly supply only the container. Their complete argument contracts remain documented_only; an expected pivot argument must not be invented.

Neither initial dependency’s receipt of a temporary address establishes exclusive, immediate-only access. Whether later dependencies can obtain and use retained pointers is unknown, not prohibited by the visible caller.

## Acceptance examples

These are conditional static criteria, not executed tests. Structural examples assume accessible, coherent storage and dependencies that preserve relevant state except where explicitly stated.

1. **Skip path.** If F-f69853ed0279 returns nonzero, the result receives the header value reread after that call. The body performs no structural removal, F-3619430fef39 call, or count decrement. Effects inside the dependency remain possible.

2. **Single-node removal.** For a leaf whose parent is the current header, cached last and cached first become header self-links, then root becomes zero. F-3619430fef39 receives the original leaf. Count processing precedes the final output read and store.

3. **Left boundary leaf.** For a non-root left leaf also recorded as cached first, the parent’s left link is cleared and cached first becomes the parent. If repair is entered, it begins from that parent.

4. **One-child root.** The child receives the target’s parent before becoming the header’s root. Applicable cached-first and cached-last changes occur before repair and the removal-related call.

5. **Deep successor with a right child.** That child is attached to the successor’s former parent. The successor takes the target’s structural position; colors are exchanged. The original target, not the successor, is passed to F-3619430fef39.

6. **Successor without a right child.** The moved successor becomes the repair position. Using null instead is not established as equivalent.

7. **Dependency-adjusted count.** If F-3619430fef39 leaves the count at 12 and returns, with no intervening mutation, the direct decrement writes 11.

8. **Later result mutation.** Suppose the result temporary contains word A immediately after F-7d84ed08254e. If an initial dependency retained its address and F-3619430fef39 subsequently changes that storage to word B, then normal completion outputs B, provided nothing changes it afterward. Outputting A merely because it was the earlier value would not match the final read.

9. **Unchanged result storage.** If the temporary contains A after F-7d84ed08254e and remains unchanged through the final read, the output is A. This outcome requires the preservation condition; preservation is not inferred from the caller.

10. **Result aliases count.** If the result destination is the count field and execution completes normally, the decrement is written first. The final store then overwrites that field with the value read from the result temporary.

## Uncertainties

The following remain **documented_only**:

- Complete dependency contracts and preservation guarantees.
- Retention, sharing, and later use of escaped temporary addresses.
- Whether any particular later dependency actually modifies the result temporary.
- Meaning of the initial decision and the final result word.
- Complete rotation-like dependency argument conventions.
- Node ownership, release behavior, and lifetime after F-3619430fef39.
- Signed count arithmetic at its lower-bound overflow.
- Behavior for invalid pointers, cycles, inconsistent relationships, or unsupported color states.
- Machine-level access granularity beyond the supplied representation.
- External restrictions on aliasing or concurrent mutation.

Balanced-tree validity and iterator-like output behavior are inferred intent, not guarantees established independently of the missing dependency contracts.

## Evidence

Established static evidence comprises the supplied function representation and opaque dependency mapping: access widths and offsets, traversal conditions, branch structure, ordered link and color writes, dependency invocations, post-call count access, and final result access.

In particular, the representation establishes that the result temporary’s address escapes before removal and that its value is read for output only after F-3619430fef39 and the count decrement. It supplies no dependency implementation proving that the temporary retains an earlier value.

No runtime trace, dependency implementation, ownership contract, or independent invariant proof was supplied. No runtime tests have been performed.
