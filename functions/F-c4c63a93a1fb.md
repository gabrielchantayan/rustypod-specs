<!-- Trusted provenance: F-c4c63a93a1fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4c63a93a1fb

## Purpose

F-c4c63a93a1fb conditionally removes a node from a linked, ordered tree representation, performs color-based structural repair, and places the detached node on a container-local chain. It writes one 32-bit result word to caller-provided storage.

The supplied static representation establishes the pointer updates, byte updates, dependency calls, and count decrement described below. Interpreting the structure as a red-black tree, the header links as root/minimum/maximum, and the detached-node chain as a reuse pool is inferred intent. Those interpretations do not establish the behavior of the dependencies.

Two paths are materially different:

- A nonzero result from an initial dependency skips removal and writes the container’s then-current header word as the result.
- A zero result proceeds through a second dependency, structural removal, possible repair, detached-node chaining, and count decrement. The final result is the local word left by the dependencies, not necessarily the node ultimately removed.

No runtime tests have been performed.

## Interface

The function has three inputs and no explicit return value:

| Input | Established use |
|---|---|
| Result storage | Address receiving one 32-bit word at the end of either normal path. |
| Container address | Address of an object whose fields at offsets +4, +16, and +20 are accessed. |
| Node-word storage | Address from which a 32-bit word is read before the first dependency call and read again after the second dependency call on the removal path. |

The representation uses 32-bit pointer-bearing words and 32-bit integer fields. Node colors or status values are accessed as individual bytes. Native pointer widths from another environment must not be substituted for these observed widths.

### Container layout

| Byte offset | Width | Established role |
|---|---:|---|
| +4 | 32 bits | Head of a detached-node chain; read near completion and then replaced by the detached node. |
| +16 | 32 bits | Pointer to the tree header. Repeatedly read throughout the operation. |
| +20 | 32 bits | Count-like field decremented once on the removal path. |

Other container bytes are not directly accessed by this function.

### Node and header layout

| Byte offset | Width | Established role |
|---|---:|---|
| +0 | 8 bits | Color/status byte, inspected and sometimes overwritten. |
| +4 | 32 bits | Parent link for ordinary nodes; root link for the header. |
| +8 | 32 bits | Left-child link for ordinary nodes; apparent minimum link for the header. |
| +12 | 32 bits | Right-child link for ordinary nodes; apparent maximum link for the header. Also reused as the detached-chain link after removal. |

No direct access to node payload is visible. Bytes +1 through +3 are not directly written.

Zero child words represent absent children in the structural logic. The header is a distinct structural object; it is not generally interchangeable with a zero child pointer.

### Preconditions

The function does not validate its inputs. Normal completion requires:

- Readable input-word storage and writable result storage.
- A valid container and header whenever their fields are accessed.
- Valid, writable nodes for every traversed or modified link.
- A structure in which parent and child relationships support the selected branches.
- Terminating leftward and rightward traversals.
- Dependencies that return normally and leave subsequent accesses valid.

These requirements apply after dependency calls as well as before them. The function does not establish them itself.

Aliasing is not rejected. In particular, the result write occurs last and can overwrite any aliased container, node, or input field. The observed sequence does not justify a general alias-safety guarantee.

## Behavior

### 1. Dependency gate and result preparation

The function first reads the word in node-word storage into a local 32-bit result word. It then reads the container’s header word into a separate local word.

It calls F-cb011f129d08 with the addresses of these two local words.

If that dependency returns nonzero:

1. The function reads the container’s header word again.
2. That newly read word replaces the local result.
3. The function writes the result word to result storage.
4. It returns without performing its own structural removal, detached-chain update, or count decrement.

This path does not reuse the header snapshot taken before the call. Dependency-induced changes to the actual container header can therefore affect the result.

If the dependency returns zero, the function calls F-a030ebc6bb58 with the address of the local result word. It then rereads node-word storage and treats that newly obtained word as the target node address.

The target is therefore not necessarily the original local result value. Neither dependency’s name or implementation is available as a behavioral guarantee. In particular, the second call cannot be specified unconditionally as computing a successor.

### 2. Structural removal

For description, “target” means the node obtained by rereading node-word storage. “Detached node” means the node later placed on the container chain. “Repair position” means the non-header position used by the repair logic.

#### Target with no children

The function reads the target’s parent.

If that parent equals the current header pointer, it performs the empty-tree updates in this order:

1. Header +12 receives the header pointer.
2. Header +8 receives the header pointer.
3. Header +4 receives zero.

Otherwise, it compares the parent’s left link with the target:

- If equal, the parent’s left link is cleared.
- Otherwise, the parent’s right link is cleared.

These clears are represented as four consecutive zero-byte stores, beginning at the selected link offset and proceeding upward through its four bytes.

The same left-versus-right decision selects one header extremum field: +8 for a left child, +12 otherwise. If that selected header field equals the target, it is replaced by the parent.

The detached node remains the target. The repair position is the parent, including the header in the sole-node case.

#### Target with exactly one child

The existing child replaces the target.

The function first writes the target’s parent into the child’s parent link. It then redirects either:

- The header’s root link, if it equals the target; or
- The target parent’s left link, if that link equals the target; or
- Otherwise, the target parent’s right link.

Extremum maintenance follows the structural replacement:

- If the header’s +8 link equals the target, it is updated to the target’s parent when the target has no right child; otherwise it becomes the leftmost node reachable from the replacement.
- If the header’s +12 link equals the target, it is updated to the target’s parent when the target has no left child; otherwise it becomes the rightmost node reachable from the replacement.

The detached node is the target. The repair position is the replacement child.

#### Target with two children

The function follows the target’s right link and then follows left links until reaching a node with no left child. Call this the successor candidate. Its right child, possibly zero, is the initial replacement at its old position.

The successor candidate is moved into the target’s structural position. Payload bytes are not copied.

The visible mutation order is:

1. The target’s left child receives the successor candidate as its parent.
2. The successor candidate receives the target’s left-child link.
3. The successor candidate’s former position is handled:
   - If it is the target’s immediate right child, any nonzero replacement receives the successor candidate as parent.
   - Otherwise, any nonzero replacement receives the successor candidate’s former parent; that former parent’s left link receives the replacement; the successor candidate receives the target’s right link; and that right subtree’s parent becomes the successor candidate.
4. The root or target-parent link formerly pointing to the target is redirected to the successor candidate.
5. The successor candidate receives the target’s parent link.
6. The successor candidate’s original status byte is saved.
7. If the old-position replacement is zero, the repair position is changed to the successor candidate itself.
8. The successor candidate receives the target’s status byte.
9. The target receives the successor candidate’s saved status byte.

The detached node is still the original target, not the successor candidate.

There are no direct header-extremum writes in this branch. Under the inferred ordered-tree interpretation, a node with two children cannot be an extremum.

### 3. Repair eligibility and stopping conditions

Repair is entered only when both of the following hold:

- The repair position differs from the then-current header pointer.
- The detached node’s current status byte is nonzero.

A zero detached-node status bypasses repair entirely.

Within repair, the current position is first compared with the header’s root link. If it is the root, repair stops. Otherwise its status byte is checked. A value other than exactly 1 also stops repair.

Every normal exit from the entered repair region writes 1 to the current repair position’s status byte. This includes exits caused by the root comparison or a status value other than 1.

The code uses 0 and 1 asymmetrically: some predicates require zero, some require exactly one, and the initial repair gate accepts any nonzero value. Behavior for other byte values must not be normalized into an assumed two-color model.

### 4. Repair cases

The repair logic selects a side by comparing the current position with its parent’s left link. Equality uses the parent’s right child as sibling. Inequality uses the parent’s left child as sibling, without first proving that the current position is the parent’s right child.

The two sides perform corresponding operations.

**Absent sibling.** The current position’s status is written as 0, and repair moves to its parent.

**Sibling with status 0.** The sibling is written as 1 and the parent as 0. A directional dependency is called on the container and parent. Parent and sibling links are then reread before continuing.

**Sibling whose relevant children are absent or have status 1.** The sibling is written as 0, and repair moves to the current position’s parent.

**Sibling requiring an inner adjustment.** When the outer child is absent or has status 1, the inner child, if present, is written as 1; the sibling is written as 0; and the opposite directional dependency is called. The sibling is then reread. If it is now absent, repair stops.

**Final adjustment.** The sibling receives the parent’s current status, the parent receives 1, and the sibling’s outer child receives 1 if present. A directional dependency is then called on the parent, and repair stops.

For a current position on the left, F-e4074d2ad6d9 is used for the parent-side directional operation and F-9678b51c49d5 for the inner adjustment. Their roles reverse for a current position on the right.

The supplied representation shows only the container argument at the inner-adjustment call sites. Other calls show both container and node. A complete second-argument contract for those inner calls is not established.

### 5. Detached-chain update and output

After structural work and any repair complete, the function performs these operations in order:

1. Read the container’s +4 chain-head word.
2. Write that word to the detached node’s +12 field.
3. Write the detached node address to container +4.
4. Read the 32-bit count-like field at container +20, subtract one, and write the result back.
5. Write the local result word to result storage.

The detached node’s +12 field no longer represents its former right child. Its +4 and +8 fields are not generally cleared.

The count operation uses a 32-bit integer field. Ordinary representable decrements are established. No explicit underflow handling, saturation, or wider intermediate is shown; overflow semantics at the signed boundary are documented_only.

## Side effects

Direct side effects include:

- Parent, child, root, and extremum link updates.
- Status-byte changes.
- Transfer of the detached node into the chain rooted at container +4.
- One count decrement on normal completion of the removal path.
- One final result-word write.

No direct allocation or release operation is visible. Linking the detached node into a container chain supports an inference of retention or reuse, but does not prove ownership transfer, lifetime extension, or eventual release policy.

No rollback, error status, exception handler, or concurrency control is visible. Partial mutations may precede any failure to complete. Dependency effects on memory, globals, ownership, and control flow remain unknown.

## Dependencies

| Opaque ID | Established interaction | Limits |
|---|---|---|
| F-cb011f129d08 | Receives addresses of two local 32-bit words. Its zero/nonzero return selects removal versus skip. | Comparison semantics, local-word mutations, global effects, and ownership behavior are unknown. |
| F-a030ebc6bb58 | Called only after a zero gate result; receives the local result-word address. | Its effect on the eventual output, input storage, container, and memory lifetime is unknown. |
| F-e4074d2ad6d9 | Called during directional repair after specified recolorings. | Rotation intent is inferred; full effects and the inner-call argument contract are unknown. |
| F-9678b51c49d5 | Called during the opposite directional repair cases. | The same limitations apply. |

No dependency may be assumed to preserve globals, maintain tree invariants, or own or release memory merely because the surrounding logic resembles a conventional tree operation.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each assumes dependencies return normally and leave all subsequent accesses valid.

1. **Nonzero gate result.** If F-cb011f129d08 returns nonzero, the output must equal the header word reread after that call. F-a030ebc6bb58 must not be called, and this function must not directly decrement the count or enqueue a node.

2. **Sole-node removal.** If the reread target has no children and its parent is the header, header +12 and +8 become the header address, followed by header +4 becoming zero. The target is prepended to the detached chain, and the count decreases once. Repair is skipped because its position is the header.

3. **Non-root leaf on the left.** If the parent’s left link equals the target, that link is cleared through four ascending-address byte stores. Header +8 changes to the parent only if it previously equaled the target. Header +12 is not selected by this branch.

4. **Root with only a right child.** The child’s parent becomes the header, and the header’s root link becomes the child. If the header’s apparent minimum equals the target, it is replaced by the leftmost node reachable from the child.

5. **Two children, deeper successor candidate.** The leftmost node in the right subtree replaces the target. Its former parent’s left link receives its former right child. The original target, not the moved candidate, is placed on the detached chain. The two status bytes are exchanged before repair eligibility is evaluated.

6. **Two children, zero old-position replacement.** The moved successor candidate becomes the repair position. An implementation that instead unconditionally dereferences a zero replacement does not match the supplied behavior.

7. **Zero detached-node status.** If the detached node’s status is zero at the repair gate, no repair call or repair-final status write occurs. Chain insertion and count decrement still occur.

8. **Output independent of target reread.** If the dependencies leave a particular local result word while node-word storage supplies a different valid target, normal removal writes that local result word to output. It must not substitute the detached node address merely because that address was used structurally.

9. **Existing detached chain.** If container +4 initially holds a chain head, the detached node’s +12 becomes that head before container +4 becomes the detached node.

## Uncertainties

- **Inferred intent:** ordered-tree erasure, red-black coloring, root/minimum/maximum header fields, directional rotations, and detached-node reuse.
- **documented_only — output meaning:** the local result may represent an iterator-like position, but dependency behavior is required to establish that.
- **documented_only — dependency ABI:** inner repair calls do not establish the complete argument list.
- **documented_only — ownership:** neither destruction nor retained lifetime is proven.
- **documented_only — arithmetic boundary:** count behavior outside representable signed subtraction is not established.
- **documented_only — malformed structures:** there is no defined recovery for invalid pointers, cycles, inconsistent parent links, or unexpected header relationships.
- **documented_only — asynchronous observation:** the store order is visible statically, but atomicity, synchronization, and concurrent-reader guarantees are absent.
- **documented_only — external effects:** dependency-induced mutations, failures, and nonlocal control transfers are unknown.

## Evidence

The supplied static representation establishes field widths and offsets, branch predicates, dependency-call placement, link and status updates, final chain insertion, count decrement, and output-write order.

The contract preserves several nonstandard details rather than replacing them with textbook tree-erasure behavior: dependency-writable local output, rereading node-word storage, bytewise leaf-link clearing, use of a moved successor candidate when the replacement is zero, exact status predicates, and incomplete inner-call argument evidence.

No implementation access, dependency bodies, runtime observations, or executed tests support stronger claims.
