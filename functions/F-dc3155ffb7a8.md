<!-- Trusted provenance: F-dc3155ffb7a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc3155ffb7a8

## Purpose

F-dc3155ffb7a8 exchanges two node-state bytes, conditionally repairs a linked structure through state changes and dependency calls, then places the explicitly supplied node on a container-held chain, decreases a 32-bit counter, and writes a supplied 32-bit value to an implicit destination.

**Inferred intent:** The node layout, mirrored repair cases, and state values resemble a red–black-tree deletion fix-up followed by recycling of a removed node. This interpretation is not an established ownership contract. The supplied body does not show the preceding unlink operation, define the container abstraction, or establish the behavior of either dependency.

The observable contract includes implicit entry state. The five explicit arguments alone are insufficient to invoke this operation correctly.

## Interface

### Explicit inputs

| Input | Width or representation | Observed use |
|---|---|---|
| Supplied node | Pointer | Its state byte is exchanged with another node’s state byte. Its link at offset 12 is later overwritten, and its address becomes the container’s chain head. |
| Link value | 32 bits | Written at offset 4 of the exchange node before any state-byte exchange. |
| Third argument | 32 bits | No direct use is visible in the supplied body. |
| Fourth argument | 32 bits | No direct use is visible in the supplied body. |
| Completion value | 32 bits | Written to an implicit destination during finalization. |

The lack of direct uses for the third and fourth arguments does not establish whether they are irrelevant to the surrounding calling convention or dependency execution.

### Implicit inputs

The body additionally requires:

- An **exchange node** pointer.
- An initial **repair node** pointer.
- A **container** address.
- A one-byte **alternate state**.
- A **completion destination** pointing to writable 32-bit storage.
- An incoming condition flag that selects whether the exchange node replaces the initial repair node.

The condition flag is consumed as entry state. It is not visibly calculated from the explicit arguments.

### Observed layouts

Node storage is accessed as follows:

| Byte offset | Access width | Role |
|---|---:|---|
| 0 | 1 byte | State |
| 4 | 4 bytes | Parent-like link, or the initial link-value destination |
| 8 | 4 bytes | First child link |
| 12 | 4 bytes | Second child link; reused as a chain link during finalization |

Container storage is accessed as follows:

| Byte offset | Access width | Role |
|---|---:|---|
| 4 | 4 bytes | Chain head |
| 16 | 4 bytes | Anchor pointer |
| 20 | 4 bytes | Counter |

The anchor’s field at offset 4 supplies the repair-loop stopping node. The body does not establish other anchor fields.

The represented target uses 32-bit link fields and 32-bit integer operations. A host implementation with wider native pointers cannot substitute its ordinary pointer layout without accounting for this representation.

### Outputs and preconditions

There is no explicit return value. Results are memory changes and dependency effects.

All directly accessed storage must be readable or writable as required by the selected path. In particular:

- The supplied node and exchange node must support their unconditional accesses.
- The container and completion destination must be valid for finalization.
- Entering repair requires a readable anchor and a usable repair node.
- Continuing repair requires valid parent and sibling-related fields on the paths actually traversed.

The body provides no null-argument validation, structured error result, or rollback. Recognized null child links are handled locally; that does not make arbitrary null nodes or malformed links safe.

## Behavior

### Initial write and byte exchange

The following operations precede the repair decision:

1. Write the 32-bit link value to the exchange node at offset 4.
2. Read and retain the exchange node’s state byte.
3. If the incoming condition flag is set, select the exchange node as the repair node. Otherwise, retain the incoming repair node.
4. Read the supplied node’s state byte and write it to the exchange node.
5. Write the retained exchange-node state byte to the supplied node.

This is a byte exchange only. It does not exchange links or payloads.

The order matters for overlapping storage. In particular, the offset-4 write occurs before either state is exchanged. If the supplied node and exchange node are the same address, the state exchange leaves that byte unchanged, while the offset-4 write still occurs.

Next, read the container’s anchor pointer.

Repair is bypassed if either:

- The anchor pointer equals the selected repair node; or
- Those pointers differ and the supplied node’s post-exchange state byte is zero.

The second condition is tested only after pointer inequality is established. Its state byte is the one just transferred from the exchange node.

Bypassing repair proceeds directly to finalization. It does not perform the repair path’s terminating state write.

### Repair-loop stopping conditions

On entering or repeating repair, the operation reads the current container anchor and the anchor’s offset-4 link.

Repair terminates when either:

- The repair node equals that anchor link; or
- The repair node’s state is not exactly 1.

The equality condition is evaluated first. If it holds, reading the repair node’s state for this test is unnecessary.

A terminating repair writes state 1 to the repair node, then finalizes.

These tests differ from the initial bypass: the initial comparison is against the anchor itself, whereas the loop comparison is against the anchor’s offset-4 link.

### Parent and sibling selection

For a repair node whose state is exactly 1 and which is not the stopping node:

- Read its offset-4 link as its parent.
- Read the parent’s offset-8 child link.
- If that child equals the repair node, use the parent’s offset-12 child as the sibling.
- Otherwise, use the parent’s offset-8 child as the sibling.

The second branch does not verify that the repair node equals the parent’s offset-12 child. A malformed relationship is therefore not rejected; it follows the second orientation.

In either orientation:

- The sibling child on the repair node’s side is the **near child**.
- The sibling child on the opposite side is the **far child**.

Thus:

| Repair-node orientation | Sibling | Near child | Far child |
|---|---|---|---|
| Parent’s offset-8 child | Parent’s offset-12 child | Sibling offset 8 | Sibling offset 12 |
| Otherwise | Parent’s offset-8 child | Sibling offset 12 | Sibling offset 8 |

### Missing sibling

If the selected sibling is null:

- Write the alternate state to the repair node.
- Promote the repair position to the parent.
- Repeat the loop’s stopping tests.

The same transition occurs if a sibling becomes null when reloaded after the initial dependency-assisted transformation described next. In that case, the parent used for promotion is the newly read parent.

### Sibling with state zero

For a non-null sibling, read its state. If it is exactly zero:

1. Write state 1 to the sibling.
2. Read the repair node’s current parent link and write the alternate state to that parent’s state byte.
3. Invoke the orientation-specific dependency:
   - F-1476b82dc9cd when the repair node is the parent’s offset-8 child.
   - F-cfe689dcfe89 in the opposite orientation.
4. Reload the repair node’s parent link.
5. Reload the sibling from that parent, using the same orientation.

If the reloaded sibling is null, take the missing-sibling transition. Otherwise, continue with child-state examination.

The operation does not restart the sibling-state-zero test for the reloaded sibling. A dependency’s exact structural or memory effects remain outside the established contract.

### Sibling whose children are absent or state 1

The next decision treats a child as satisfying the test if it is null or its state is exactly 1.

The near-child pointer is read first. Its state is read only if it is non-null. If the near child satisfies the test, the far-child pointer and, where applicable, state are examined.

If both children satisfy the test:

- Write the alternate state to the sibling.
- Read the repair node’s parent link and promote repair to that parent.
- Repeat the loop’s stopping tests.

This transition does not write the alternate state to the old repair node. That distinguishes it from the missing-sibling transition.

Values other than 1 fail the child-state test. The body does not normalize arbitrary state values to a two-state interpretation.

### Preparing a sibling with an absent or state-1 far child

If the both-children condition does not apply, the far child is examined again.

If the far child is null or has state exactly 1:

1. Re-read the sibling’s near-child link as a presence check.
2. If that link is nonzero, write state 1 through the previously retained near-child pointer.
3. Write the alternate state to the sibling.
4. Invoke:
   - F-cfe689dcfe89 when the repair node is the parent’s offset-8 child.
   - F-1476b82dc9cd in the opposite orientation.
5. Reload the sibling through the repair node’s current parent link.

If this reloaded sibling is null, terminate repair by writing state 1 to the repair node, then finalize. This is not the missing-sibling promotion behavior.

Under stable, non-overlapping ordinary node storage, the earlier child tests establish the near child needed for this preparation. Concurrent changes or unusual aliasing are not covered by that inference.

### Completing a repair

When a usable sibling remains:

1. Read the repair node’s current parent and its state.
2. Copy that state byte to the sibling.
3. Write state 1 to the repair node’s current parent.
4. Read the sibling’s far-child link and, if non-null, write state 1 to that child.
5. Invoke:
   - F-1476b82dc9cd when the repair node is the parent’s offset-8 child.
   - F-cfe689dcfe89 in the opposite orientation.
6. Terminate repair by writing state 1 to the repair node.
7. Finalize.

The post-call repair-node write is part of the visible sequence. Dependency behavior must not be assumed to make it redundant or harmless.

### Finalization

Every normally completing path reaches the following ordered sequence:

1. Read the container’s 32-bit chain-head field.
2. Write that value to the supplied node at offset 12.
3. Write the supplied node’s address into the container’s chain-head field.
4. Read the container’s 32-bit counter and write the result of decreasing it by one.
5. Write the completion value to the completion destination.
6. Return without an explicit result.

This establishes chain insertion, not allocation or deallocation. The supplied node’s offset-12 field is overwritten regardless of whether it previously served as a child link.

There is no counter-zero check. The counter operation is 32-bit. The supplied representation does not settle language-level signed-overflow semantics; a compatible reconstruction must resolve that separately rather than silently widening the counter or adding saturation.

## Side effects

Direct effects include:

- An unconditional 32-bit write to the exchange node at offset 4.
- State-byte writes to the supplied and exchange nodes.
- Conditional state-byte writes to repair nodes, parents, siblings, and sibling children.
- Dependency calls in orientation-dependent order.
- Overwriting the supplied node’s offset-12 link.
- Replacing the container’s chain head.
- Decreasing its counter.
- Writing the completion destination last on normal completion.

There is no visible allocator, deallocator, lock, atomic operation, logging operation, or error callback. Dependencies may have additional effects; none are ruled out by the body.

No ownership transfer beyond linking is established. In particular, the caller cannot infer that dependencies preserve globals, retain nodes, release nodes, or leave all previously referenced storage alive.

Aliasing can make the ordering significant. For example, if the completion destination overlaps the counter, the final completion write follows and can overwrite the counter update. Such overlap is not rejected, but it is not evidence of a supported container configuration.

## Dependencies

### F-1476b82dc9cd

Called for:

- A state-zero sibling in the offset-8 repair-node orientation.
- Near-child preparation in the opposite orientation.
- Final repair in the offset-8 orientation.

### F-cfe689dcfe89

Called for:

- A state-zero sibling in the opposite orientation.
- Near-child preparation in the offset-8 orientation.
- Final repair in the opposite orientation.

**Inferred intent:** These dependencies appear to implement complementary structural rotations.

**Established limitation:** No explicit argument transfer is shown at these calls. Their effective inputs, return effects, global mutations, storage-lifetime effects, and actual transformations are unknown. Reloads after calls establish which links this function reads again; they do not establish what a dependency changed.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, assume valid, stable, non-overlapping storage and normally returning dependencies.

1. **Anchor bypass.**  
   If the selected repair node equals the container’s anchor, the initial link write and byte exchange occur, repair is skipped, and finalization occurs. No dependency is called by this body.

2. **Zero exchanged state.**  
   If the selected repair node differs from the anchor and the exchange node initially has state 0, the supplied node receives 0. Repair is bypassed, irrespective of the repair node’s state.

3. **Incoming flag selects the exchange node.**  
   If the flag is set, subsequent repair decisions use the exchange node rather than the incoming repair pointer. Its offset-4 field has already received the explicit link value.

4. **Stopping-node termination.**  
   If repair is entered and the selected repair node equals the anchor’s offset-4 link, that node receives state 1 and finalization follows without a dependency call.

5. **Non-1 repair state.**  
   If repair is entered, the node is not the stopping node, and its state is 2, repair terminates by replacing 2 with 1. The implementation does not require that the prior value be 0.

6. **Missing sibling.**  
   For a state-1 repair node requiring another iteration, a null selected sibling causes the alternate state to be written to that repair node before repair moves to its parent.

7. **Both sibling children absent.**  
   With a nonzero-state sibling and two null child links, the sibling receives the alternate state and repair moves to the parent. The old repair node does not receive the missing-sibling write.

8. **State-zero sibling transformation.**  
   In the offset-8 orientation, a state-zero sibling receives 1, the parent receives the alternate state, and F-1476b82dc9cd is called before the parent and sibling are reloaded. Further expectations require the dependency’s effects.

9. **Finalization ordering.**  
   With an initial chain head pointing to another node, the supplied node’s offset-12 link receives that old head before the container points to the supplied node. The counter update precedes the completion write.

## Uncertainties

- **documented_only — Entry convention:** The origin and validity requirements of implicit pointers, the alternate state, and the incoming flag are unknown.
- **documented_only — Alternate-state meaning:** Conventional balancing behavior suggests a value of 0, but the body accepts an arbitrary byte and writes it unchanged.
- **documented_only — Structural invariants:** Tree ordering, sentinel organization, previous unlinking, and eventual loop termination are not established.
- **documented_only — Dependency contracts:** Rotation direction, effective arguments, global preservation, ownership, storage lifetime, and failure behavior are unknown.
- **documented_only — Arithmetic boundary:** The exact signed-overflow contract of the 32-bit decrement requires target-level evidence.
- **documented_only — Concurrency and aliasing:** No synchronization or comprehensive non-aliasing contract is provided.
- **documented_only — Abnormal completion:** Invalid accesses or dependency failures have no visible recovery path. Earlier writes are not rolled back.

## Evidence

Established behavior is derived solely from the supplied static body: access widths and offsets, branch conditions, direct write ordering, dependency-call placement, link reloads, and finalization sequence.

The balancing and recycling interpretation is inferred from those patterns, not independently verified. No dependency implementations, surrounding callers, binary-level arithmetic evidence, or runtime observations were supplied. No runtime tests have been performed.
