<!-- Trusted provenance: F-c3b5aab03a9c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c3b5aab03a9c

## Purpose

F-c3b5aab03a9c rewires a pointer-linked structure around a supplied node. It promotes the node referenced at offset +12, moves that promoted node’s link at offset +8 into the supplied node’s +12 slot, reconnects the promoted node to the supplied node’s former incoming connection, and makes the supplied node the promoted node’s +8 child.

**Established static behavior:** the function performs a fixed sequence of four-byte link reads and writes, with branches for an absent intermediate child, a distinguished root connection, and the side of a non-root incoming connection.

**Inferred intent:** under a conventional binary-tree interpretation—parent at +4, left child at +8, right child at +12—the operation is a left rotation around the supplied node. That interpretation explains the rewiring but is not a validation performed by the function.

The function does not establish ordering, balance, acyclicity, or structural consistency. Such properties depend on the input structure and any surrounding algorithm.

## Interface

The function accepts two inputs and returns no value:

| Input | Role |
|---|---|
| Structure context | An address-like value identifying storage with a four-byte anchor reference at offset +16. |
| Rotation node | An address-like value identifying the node around which links are changed. |

The supplied representation declares both inputs as integer values and uses them as addresses. Link accesses are four bytes wide. The declaration does not independently establish the execution platform’s native pointer width, calling convention, byte order, or rules for converting integer values into addresses.

The following semantic names describe roles within this contract, not additional parameters or stored fields:

- **Pivot:** the rotation node supplied by the caller.
- **Promoted node:** the address-like value initially read from pivot offset +12.
- **Middle child:** the link read from promoted-node offset +8.
- **Former parent:** the value read from pivot offset +4 when needed.
- **Anchor:** the object referenced by the context’s field at offset +16.
- **Root slot:** the four-byte field at anchor offset +4.

The relevant layout is:

| Object | Offset | Accessed role |
|---|---:|---|
| Structure context | +16 | Anchor reference |
| Anchor | +4 | Distinguished root link |
| Node | +4 | Parent link |
| Node | +8 | Left-child link under the inferred tree interpretation |
| Node | +12 | Right-child link under the inferred tree interpretation |

No meaning is established for bytes outside these fields.

### Preconditions for defined, intended use

The function itself does not check these conditions:

- The context, pivot, promoted node, and anchor must support the accesses described below.
- The promoted node must exist. A zero pivot +12 link is not handled as an empty subtree or a no-op.
- If the middle-child link is nonzero when selected for a parent-link update, its target must support a four-byte write at +4.
- If the root slot does not identify the pivot, the pivot’s parent link must identify storage supporting the selected +8 or +12 access.
- Address calculations and memory accesses must be valid in the actual execution environment.
- For conventional rotation semantics, the structure must have consistent parent and child links, and the relevant fields must not overlap in ways that alter the intended operation.

There is no explicit ownership transfer, allocation, release, success result, or error result.

## Behavior

### 1. Select the node to promote

The function reads the four-byte link at pivot +12 and retains that value as the promoted-node address for subsequent accesses.

It does not test that value for zero before using it to access promoted-node +8.

### 2. Move the middle subtree beneath the pivot

The function reads promoted-node +8 and writes the resulting four-byte value into pivot +12.

It then reads promoted-node +8 again to determine whether the link is nonzero.

If that second read is nonzero, the function reads promoted-node +8 once more and writes the pivot’s address-like value into the selected target’s +4 parent slot.

Consequently, under stable, non-overlapping storage:

- The pivot’s new right-child link becomes the promoted node’s former left-child link.
- If that middle child is present, its parent link becomes the pivot.
- If that middle child is absent, no middle-child parent slot is written.

These are separate reads of promoted-node +8. The function does not capture one middle-child value and reuse it for all three purposes. With overlapping fields or externally changing memory, the value copied into pivot +12, the value tested, and the value used as a write destination need not agree.

### 3. Give the promoted node the pivot’s parent link

The function reads pivot +4 and writes that four-byte value into promoted-node +4.

This occurs after the middle-child processing and before the root-versus-non-root decision.

For conventional tree inputs, the promoted node thereby takes the pivot’s former parent link. The root case does not substitute a special parent value: it copies whatever is present in pivot +4 at this point.

### 4. Reconnect the incoming link

The function reads the anchor reference at context +16, forms the root-slot location at anchor +4, and reads that slot.

#### Root-slot match

If the root-slot value equals the pivot’s address-like value, the function writes the promoted-node address into that root slot.

This branch does not inspect the former parent’s child links.

#### Root-slot mismatch

If the root-slot value does not equal the pivot’s address-like value:

1. The function reads pivot +4 again to obtain a parent address.
2. It reads that parent’s +8 link.
3. If that link equals the pivot, it writes the promoted-node address into parent +8.
4. Otherwise, it writes the promoted-node address into parent +12.

The final alternative is unconditional once the +8 comparison fails. The function does **not** verify that parent +12 actually contains the pivot before overwriting it.

The parent value used here is obtained by a later read of pivot +4. It is not necessarily the same value previously copied into promoted-node +4 if aliasing or external modification has changed the field.

### 5. Complete the local parent-child exchange

After updating the incoming connection, the function:

1. Writes the pivot’s address-like value into promoted-node +8.
2. Writes the promoted-node address into pivot +4.

The function then returns without a value.

### Result under conventional tree preconditions

For stable, valid, consistently linked, non-overlapping nodes, the resulting relationships are:

- The promoted node replaces the pivot in the relevant incoming link: either the root slot or one child slot of the former parent.
- The promoted node retains the pivot’s former parent value.
- The pivot becomes the promoted node’s left child.
- The promoted node’s former left child becomes the pivot’s right child.
- A present middle child receives the pivot as its parent.

The pivot’s +8 child link and the promoted node’s +12 child link are not directly written. They remain unchanged only where they do not alias another write destination.

## Side effects

### Ordered memory effects

The source-level sequence of writes is:

| Order | Destination | Value |
|---:|---|---|
| 1 | Pivot +12 | Value read from promoted-node +8 |
| 2, conditional | Target selected through promoted-node +8, then +4 | Pivot address |
| 3 | Promoted node +4 | Value read from pivot +4 |
| 4 | Root slot, or selected parent child slot | Promoted-node address |
| 5 | Promoted node +8 | Pivot address |
| 6 | Pivot +4 | Promoted-node address |

There are five writes when the middle-child update is skipped and six when it is taken. These counts describe executed write operations, not necessarily distinct memory locations.

The ordered description concerns the supplied source representation. It does not establish machine-level atomicity, volatile access semantics, compiler barriers, or a concurrency memory model. Within an individual assignment, no unnecessary ordering is asserted between computing its destination address and obtaining its source value.

### Intermediate states

The transformation is not presented as an atomic transaction:

- Pivot +12 changes before the incoming link is redirected.
- The promoted node’s parent link changes before its +8 link is replaced by the pivot.
- The incoming link is redirected before the pivot receives its final parent link.

No rollback or recovery path is present. If an access fails after earlier writes, the function contains no logic to undo them. Whether the environment faults, traps, or behaves otherwise is not established.

### Ownership and other state

There are no explicit allocation, deallocation, callback, logging, or synchronization operations.

The function mutates existing storage. It does not return a replacement root to the caller; root replacement is communicated through the anchor’s +4 field.

No separate balancing metadata, counters, payloads, or context fields are directly updated. This is a statement about explicit access sites, not a guarantee against effects caused by overlapping storage.

## Dependencies

No called functions appear in the supplied body. Accordingly, there are no opaque function dependencies to list beyond F-c3b5aab03a9c itself.

The operation depends on a caller-provided memory layout and valid address-like values. Its full behavior also depends on execution-environment properties not established here, including address representation, alignment requirements, and the treatment of invalid memory accesses.

No unknown callee behavior is needed to explain the visible transformation.

## Acceptance examples

These examples are conditional acceptance cases derived from static inspection. They have not been executed.

Unless a case states otherwise, assume valid, stable, non-overlapping storage and consistent tree links.

### Root pivot with a present middle child

Initially:

- The root slot identifies the pivot.
- Pivot +12 identifies the promoted node.
- Promoted-node +8 identifies a middle child.
- Pivot +4 contains an external parent or sentinel value.

After return:

- The root slot identifies the promoted node.
- Promoted-node +4 contains that external parent or sentinel value.
- Promoted-node +8 identifies the pivot.
- Pivot +4 identifies the promoted node.
- Pivot +12 identifies the middle child.
- Middle-child +4 identifies the pivot.

The operation does not require the copied root parent value to be zero.

### Root pivot with no middle child

Initially, the root slot identifies the pivot and promoted-node +8 is zero.

After return:

- Pivot +12 is zero.
- No middle-child parent update occurs.
- The root slot identifies the promoted node.
- Promoted-node +8 identifies the pivot.
- Pivot +4 identifies the promoted node.

### Non-root pivot attached through its parent’s +8 slot

Initially, the root slot does not identify the pivot, pivot +4 identifies its parent, and parent +8 identifies the pivot.

After return:

- Parent +8 identifies the promoted node.
- Parent +12 remains unchanged.
- Promoted-node +4 identifies that parent.
- The local pivot, promoted-node, and middle-child relationships follow the transformation above.

### Non-root pivot attached through its parent’s +12 slot

Initially, the root slot does not identify the pivot, parent +8 does not identify the pivot, and parent +12 does identify it.

After return, parent +12 identifies the promoted node. Parent +8 remains unchanged.

### Inconsistent incoming links

Suppose the root slot does not identify the pivot and neither child slot of the parent identifies it.

Provided the required accesses remain valid, the function still chooses parent +12 for replacement because the parent +8 comparison fails. It does not report structural inconsistency.

A contract requiring rejection or preservation of such an input would not match the supplied behavior.

### Missing promoted node

Suppose pivot +12 is zero.

The function proceeds to an access at promoted-node offset +8 without a null check. No successful-return behavior, no-op guarantee, or controlled error result is established for this case.

### Overlapping fields

If one write destination overlaps a field read later, acceptance must follow the actual access sequence rather than the conventional tree diagram.

In particular, the middle-child value is read repeatedly, and the pivot’s parent value is read again on the non-root path. A contract that snapshots both values at entry would impose behavior not established by the source.

## Uncertainties

The following matters are **documented_only**:

- **Native address width and ABI:** four-byte link operations are visible, but the complete execution platform is unspecified.
- **Byte order and alignment:** neither is independently established.
- **Integer-to-address behavior:** the representation uses integer parameters as addresses; valid address ranges and overflow behavior are not supplied.
- **Anchor semantics:** the +4 anchor field behaves as a root connection here, but the anchor’s broader structure is unknown.
- **Structural invariants:** tree ordering, acyclicity, unique parentage, and sentinel conventions are not checked.
- **Concurrency:** there is no visible synchronization contract. The repeated reads do not establish support for concurrent mutation.
- **Invalid-access outcomes:** no error handling is shown, and platform failure behavior is unknown.
- **Aliasing policy:** conventional rotation semantics require suitable separation of fields, but no explicit rejection of overlapping inputs exists.
- **Higher-level balancing:** any responsibility for updating balancing information or coordinating this operation with other changes is outside the supplied evidence.

The binary-tree left-rotation interpretation is inferred intent. The ordered four-byte accesses and branch choices are the established static behavior.

## Evidence

This contract is based solely on static inspection of the supplied function body.

The body establishes:

- Two integer-valued inputs used in address calculations and no returned value.
- Four-byte link accesses at the listed offsets.
- Initial promotion selection through pivot +12.
- Repeated reads of promoted-node +8 during middle-child handling.
- Parent-link copying before root selection.
- Root detection by equality with the anchor’s +4 field.
- Non-root side selection by testing only the parent’s +8 field.
- Final writes making the pivot the promoted node’s +8 child and the promoted node the pivot’s parent.
- No visible calls, validation path, allocation, release, or recovery mechanism.

No runtime tests, execution traces, or independent layout documentation were provided or used.
