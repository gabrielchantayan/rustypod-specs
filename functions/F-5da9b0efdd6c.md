<!-- Trusted provenance: F-5da9b0efdd6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5da9b0efdd6c

## Purpose

F-5da9b0efdd6c replaces a node reference in a linked structure, conditionally updates two cached references, optionally performs state-and-link repair, and completes bookkeeping through a dependency call, a count decrement, and an output write.

**Established static behavior:** the function directly reads and writes node links and state bytes, invokes dependencies on specified branches, decrements a 32-bit field, and publishes an opaque 32-bit value. It exposes no explicit return value or status.

**Inferred intent:** the mirrored repair branches resemble red-black-tree removal repair. Under that interpretation, the state byte represents color, the cached references represent extrema, and two dependencies perform opposite rotations. Those interpretations do not establish dependency behavior or replace the exact byte comparisons described below.

The supplied representation has no declared parameters, but uses pre-existing machine state as inputs. Its complete calling convention and whether it is independently callable remain unresolved.

## Interface

### Logical inputs and output

These names identify behavioral roles, not established external parameter names.

| Role | Representation | Use |
|---|---|---|
| Container | Address of accessible storage | Provides the header reference and count field. |
| Removed node | Node address carried in a 32-bit reference | Identifies the reference to replace; supplies parent and child links. |
| Current node | Node address carried in a 32-bit reference | Initially replaces the removed node; may advance to an ancestor during repair. |
| Gate address | Address of a readable byte | Controls repair when the header-identity check does not already suppress it. |
| Alternate state | 8-bit value | Supplies state writes other than the constant `1`. |
| Output destination | Address of writable 32-bit storage | Receives the saved output value at normal completion. |
| Saved output value | Opaque 32-bit value | Published by the final direct write. |

No interpretation of the saved output value is established. In particular, it is not proven to be a node address, status, or ownership token.

### Observable storage layout

All offsets are byte offsets.

| Object | Offset | Width | Observed role |
|---|---:|---:|---|
| Container | `0x10` | 32 bits | Header reference |
| Container | `0x14` | 32 bits | Count |
| Header | `0x04` | 32 bits | Root reference |
| Header | `0x08` | 32 bits | First cached reference |
| Header | `0x0c` | 32 bits | Second cached reference |
| Node | `0x00` | 8 bits | State |
| Node | `0x04` | 32 bits | Parent reference |
| Node | `0x08` | 32 bits | Left-child reference |
| Node | `0x0c` | 32 bits | Right-child reference |

“Left” and “right” distinguish link positions; no key-ordering rule is established.

The header’s base address is itself compared with the initial current-node address. That comparison is separate from comparing the current node with the root reference stored inside the header.

The representation establishes these access widths, not byte order, alignment tolerance, payload layout, or compatibility with wider addresses.

### Preconditions, errors, and ownership

Every address actually dereferenced must remain valid for the required access, including after dependency calls. Every written location must be writable. There is no explicit validation of membership, link consistency, address validity, count range, or output capacity.

Specific obligations include:

- The removed node’s parent and child fields must be readable whenever selected branches use them.
- Repair requires a writable current-node state byte, including when the current node equals the root.
- A continuing non-root repair step requires a readable current-node parent reference and the parent’s relevant child links.
- Support for an absent sibling does not establish support for a null current node.
- Any dependency-specific requirements must also hold.

No allocation, ownership transfer, release, or lifetime guarantee follows from the visible direct operations. Dependencies may affect memory, globals, ownership, or validity; their preservation behavior is unknown.

There is no explicit error result, rollback, or recovery path.

## Behavior

### 1. Replace the structural reference

The function obtains the header from the container and reads its root reference.

- If the root names the removed node, the root field receives the initial current-node address.
- Otherwise, the function reads the removed node’s parent and that parent’s left-child reference. If the left-child reference names the removed node, that field receives the initial current-node address. Otherwise, the parent’s right-child field receives it.

The fallback branch does not verify that the right-child reference names the removed node.

This stage does not directly set the replacement node’s parent reference. Correct preparation of that reference, if required, must come from outside the visible operation.

### 2. Update cached references in order

The first cached reference is processed before the second. The container’s header reference is obtained for each cache decision.

For the first cache, at header offset `0x08`:

- If it does not name the removed node, there is no direct update.
- If it names the removed node and the removed node’s right-child reference is zero, it receives the removed node’s parent reference.
- Otherwise, F-91fe39e6f6dc is called. Its 32-bit result is stored at offset `0x08` of the header obtained from the container **after that call**.

For the second cache, at header offset `0x0c`:

- If it does not name the removed node, there is no direct update.
- If it names the removed node and the removed node’s left-child reference is zero, it receives the removed node’s parent reference.
- Otherwise, F-a4d0b2395344 is called. Its 32-bit result is stored at offset `0x0c` of the header obtained from the container **after that call**.

The second decision therefore observes storage after the first update and any preceding dependency effects. Neither cache decision nor its destination is specified as using an entry-time snapshot of the header.

### 3. Evaluate the repair gate

Only after both cached-reference stages does the function obtain the header address for the repair gate.

- If that header base address equals the initial current-node address, repair is skipped and the gate byte is not read.
- Otherwise, the byte at the gate address is read. Zero skips repair; every nonzero byte enters repair.

These conditions apply **when evaluated after all preceding cache updates and dependency effects**. Entry-time header identity or gate contents alone do not establish whether repair will run.

Skipping repair proceeds directly to finalization. It does not perform the repair path’s terminating state write.

### 4. Check whether repair should continue

Each repair iteration first obtains the current header’s root reference.

- If the current node equals that root, repair terminates.
- Otherwise, the current node’s state is read. If it is not exactly `1`, repair terminates.
- Only a non-root current node whose state is exactly `1` proceeds to sibling handling.

Root equality short-circuits the state read used by this test. Both termination cases nevertheless write state `1` to the current node before finalization.

No general “nonzero means state one” rule applies. Values other than `0` and `1` retain their distinct branch consequences.

### 5. Select the sibling and orientation

For a continuing repair step, the function reads the current node’s parent and the parent’s left-child reference.

| Observed relation | Selected sibling | Sibling’s near child | Sibling’s far child |
|---|---|---|---|
| Parent’s left child equals current node | Parent’s right child | Left child | Right child |
| Parent’s left child differs from current node | Parent’s left child | Right child | Left child |

The second orientation does not check that the current node is the parent’s right child.

For subsequent child tests, a child qualifies as “absent or state one” only if its reference is zero or its state byte equals exactly `1`.

### 6. Handle an absent sibling

If the selected sibling is absent:

1. The current node receives the alternate state.
2. The selected parent becomes the current node.
3. Repair repeats from the root/state checks.

No repair dependency is called for this step. This is upward propagation, not immediate termination.

The same propagation applies when a sibling becomes absent immediately after the state-zero conversion described next, using the parent reloaded after that dependency.

### 7. Convert a sibling whose state is zero

If the sibling exists and its state is exactly `0`, the direct actions occur in this order:

1. Write state `1` to the sibling.
2. Write the alternate state to the current node’s parent.
3. Call F-c470755d11e4 for the left-current-node orientation, or F-b951efde8a28 for the other orientation.
4. Reload the current node’s parent.
5. Reselect the sibling from that parent in the same orientation.

If the reselected sibling is absent, the preceding absent-sibling propagation applies.

An existing sibling with any nonzero state bypasses this conversion, including states other than `1`.

### 8. Propagate when both sibling children qualify

The near child is inspected first. Only if it is absent or state `1` does this decision inspect the far child.

If both qualify:

1. The sibling receives the alternate state.
2. The current node becomes the parent read through its parent field.
3. Repair repeats.

The parent read follows the sibling-state write. The description does not imply that a previously read parent address is always reused.

If propagation is not selected, the next decision reads the far-child reference and, when nonzero, its state. The far child can therefore be inspected more than once; the function does not take one immutable snapshot of both children.

### 9. Prepare the far-child case when necessary

After propagation has been rejected, if the far child is absent or state `1`:

1. The near-child link is checked again. If nonzero, state `1` is written to the near-child address selected earlier.
2. The sibling receives the alternate state.
3. F-b951efde8a28 is called for the left-current-node orientation, or F-c470755d11e4 for the other orientation.
4. The sibling is reloaded through the current node’s parent.

If this newly loaded sibling is absent, the function writes state `1` to the current node and finalizes. Unlike the earlier absent-sibling branches, this case does not advance repair to the parent.

The repeated near-link check and the earlier selected write target are distinct observations. Stable, noninterfering storage makes them agree. Concurrent mutation or unusual aliasing is not assigned a separate safety guarantee.

### 10. Complete the local repair

If a sibling remains, either directly or after the preparation dependency:

1. Read the current parent’s state and write it to the sibling.
2. Write state `1` to the parent reached through the current node’s parent field.
3. Inspect the sibling’s far-child reference. If nonzero, write state `1` through that link.
4. Call F-c470755d11e4 for the left-current-node orientation, or F-b951efde8a28 for the other orientation.
5. Write state `1` to the current node.
6. Proceed to finalization without another repair iteration.

The sibling-state write precedes the parent-state write. Parent and child references are accessed again where the visible operations require them; they are not all fixed at iteration entry. Aliasing can therefore affect later reads and destinations.

### 11. Finalize

Every visible normal completion path performs these actions in order:

1. Call F-1b2e0c144326.
2. Read the container’s 32-bit count, subtract one, and write the result back.
3. Write the saved 32-bit value to the output destination.
4. Return without an explicit result.

The count is read after the dependency returns. Dependency changes to that field are therefore relevant to the decrement.

There is no zero-count guard or saturation. For example, a signed count of zero becomes negative one. The count operation is represented as signed 32-bit arithmetic; subtraction is established where its result is representable. Behavior at the minimum signed 32-bit value is not resolved by the supplied evidence and must not be replaced with an assumed wraparound rule.

The output write is last. If its destination aliases the count or another modified field, it can overwrite the earlier result.

## Side effects

Direct writes can affect:

- The header’s root reference or one parent-child reference.
- Either or both cached references.
- State bytes of the current node, ancestors, siblings, and sibling children.
- The container’s count.
- The output destination.

Dependencies may have additional effects. The visible body does not establish that they preserve the header reference, gate byte, links, count, unrelated globals, or memory ownership.

No direct allocation, release, key comparison, or payload movement is shown. These absences do not constrain dependencies.

No locking, atomic publication, exception recovery, or rollback is established. Earlier writes are not visibly undone if a later access or dependency fails. Termination is not established for malformed or cyclic parent chains.

## Dependencies

| Opaque ID | Established call-site role | Unresolved behavior |
|---|---|---|
| F-91fe39e6f6dc | Returns a 32-bit value used for the first cache when that cache names the removed node and its right child is nonzero. | Arguments, traversal, preservation, ownership, and other effects. |
| F-a4d0b2395344 | Returns a 32-bit value used for the second cache when that cache names the removed node and its left child is nonzero. | Arguments, traversal, preservation, ownership, and other effects. |
| F-c470755d11e4 | Called in the specified directional repair cases after preceding state writes. | Exact transformation, inputs, preservation, and other effects. |
| F-b951efde8a28 | Called in the complementary directional repair cases. | Exact transformation, inputs, preservation, and other effects. |
| F-1b2e0c144326 | Called once on every visible normal completion path, before count decrement and output publication. | Whether it destroys, releases, detaches, or otherwise processes any object. |

Extremum traversal, opposite rotations, and destruction are plausible interpretations, not established dependency contracts.

## Acceptance examples

These are conditional expectations derived from static evidence, not runtime results. Each requires valid storage for the operations actually reached and normal return from dependencies. No dependency preservation is implicit.

1. **Root replacement followed by a zero gate at evaluation time.**  
   Suppose the initial structural comparison finds that the root names the removed node. The root field receives the initial current-node address. Suppose further that, **after both cache stages and all their dependency effects**, the header base differs from the initial current-node address and the gate byte is zero when read. Repair is skipped, with no direct repair-state write. Finalization still follows. An entry-time zero gate alone is insufficient for this expectation.

2. **Fallback structural link.**  
   At the structural decision, suppose the root differs from the removed node and the removed node’s parent does not name it as the left child. The parent’s right-child field receives the current-node address without a right-child membership check.

3. **First cache without a right child.**  
   When the first cache decision is reached, suppose it names the removed node and the removed node’s right-child reference is zero. The cache receives the removed node’s parent reference. F-91fe39e6f6dc is not called for this update.

4. **Header redirected by a cache dependency.**  
   Suppose F-91fe39e6f6dc is called and leaves the container pointing to a different valid header. Its result is written to offset `0x08` of that newly referenced header. The second cache decision then uses the header obtained for that subsequent stage.

5. **Header identity suppresses the gate read.**  
   Suppose the header base equals the initial current-node address when the post-cache gate comparison occurs. Repair is skipped without dereferencing the gate address, regardless of its contents.

6. **Repair begins at the root.**  
   Suppose repair is entered and the first loop root comparison equals the current node. The loop’s state-condition read, sibling inspection, and repair dependencies are skipped. State `1` is written to the current node before finalization.

7. **Absent sibling propagates upward.**  
   Suppose a continuing repair step has a non-root current node in state `1` and selects an absent sibling. The current node receives the alternate state, the selected parent becomes current, and the loop conditions are checked again.

8. **Both children qualify.**  
   Suppose sibling conversion is bypassed and both sibling children, when inspected, are absent or state `1`. The sibling receives the alternate state, and repair continues at the parent subsequently read through the current node.

9. **Sibling disappears after preparation.**  
   Suppose the far-child preparation dependency returns and sibling reload yields zero. The current node receives state `1` and finalization follows; repair does not propagate upward on this branch.

10. **Finalization changes the count.**  
    Suppose F-1b2e0c144326 leaves the count at `12`. The direct decrement writes `11`, regardless of the count before that call. The saved output write follows.

11. **Output aliases the count.**  
    Suppose finalization leaves a representable decrement and the output destination is the count field. The decrement occurs first, but the final field contents are the saved 32-bit output value because publication overwrites the decrement.

## Uncertainties

The following remain **documented_only**:

- Complete entry ABI and binding of ambient inputs.
- Whether this is an independent entry point or an internal continuation.
- Dependency arguments, preservation rules, failure behavior, and ownership effects.
- Whether valid callers always supply alternate state zero.
- Complete structure invariants, sentinel conventions, and payload layout.
- Supported aliasing among nodes, header, container, gate, and output storage.
- Signed-overflow behavior at the minimum 32-bit count.
- Termination with malformed or cyclic links.
- Concurrency and memory-ordering guarantees.
- Platform behavior for invalid or inadequately aligned accesses.

In particular, neither normal dependency return nor consistent links proves preservation of the repair gate or header identity across earlier calls.

No runtime tests have been performed.

## Evidence

**Established static evidence:** the supplied body supports the access widths and offsets, structural fallback behavior, ordered cache processing, post-call header reloads, post-cache gate evaluation, short-circuit root/state checks, mirrored sibling decisions, ordered direct writes and dependency calls, and common finalization sequence.

**Inferred intent:** tree-removal repair, color meanings, extrema, rotations, and destruction are interpretations of the visible pattern, not independently established contracts.

**Unknowns:** no dependency definitions, complete caller contract, or runtime observations establish the unresolved behavior listed above. Acceptance examples therefore state conditions at the relevant observation points rather than assuming entry-time values survive intervening effects.
