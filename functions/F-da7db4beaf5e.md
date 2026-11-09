<!-- Trusted provenance: F-da7db4beaf5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da7db4beaf5e

## Purpose

F-da7db4beaf5e creates a node, attaches it to a tree-like container, performs color-based insertion repair, and returns the new node’s address through an output slot.

The established behavior includes a four-byte payload copy, an early count increment, conditional attachment on either side of a supplied parent, updates to header-held links, repair through two structural dependencies, and a final write of color value 1 to the current root.

The structure and repair pattern strongly suggest red-black tree insertion. That interpretation is inferred intent, not an independently established guarantee. In particular, this function does not visibly initialize the new node’s color or child links, validate ordering, search for an insertion position, or enforce uniqueness.

## Interface

### Arguments and result

The function has five argument positions:

| Position | Role | Established representation |
|---|---|---|
| 1 | Output slot | Address receiving the new node’s address as one four-byte write |
| 2 | Container | Address used for header, count, and dependency-state access |
| 3 | Attachment override | 32-bit value tested only for zero versus nonzero |
| 4 | Proposed parent | Address of a node or header used as the attachment point |
| 5 | Payload source | Address from which four bytes are copied; also passed to the comparison dependency when needed |

There is no explicit return value. The observable result is delivered through argument 1 after attachment and repair finish.

The supplied representation uses four-byte addresses and four-byte integer fields. It does not establish a portable interface for systems with wider pointers.

### Container layout

Offsets are byte offsets from the container address.

| Offset | Width | Observed use |
|---|---:|---|
| +0x10 | 4 bytes | Header address, loaded repeatedly |
| +0x14 | 4 bytes | Count-like field, read, incremented, and written |
| +0x19 | Not established | Address passed as the first argument to F-257a89ed2c43 |

The function does not directly read the contents at +0x19. The size and interpretation of the dependency state beginning there remain unknown.

### Node and header layout

| Offset | Width | Observed use |
|---|---:|---|
| +0x00 | 1 byte | Color or repair-state marker |
| +0x04 | 4 bytes | Parent link for ordinary nodes; root link for the header |
| +0x08 | 4 bytes | Left child link for ordinary nodes; apparent minimum link for the header |
| +0x0c | 4 bytes | Right child link for ordinary nodes; apparent maximum link for the header |
| +0x10 | 4 bytes | Payload storage |

The header’s minimum and maximum interpretations follow from the update pattern. They are not separately verified container invariants.

Marker value 0 triggers red-parent repair. Any nonzero marker suppresses that repair or qualifies an uncle for the non-red case. Writes performed here use exactly 0 and 1.

### Preconditions

Successful operation requires valid, suitably aligned storage for every reached access, including the output slot, container fields, new node, selected parent link, header links, and repair ancestors.

For the inferred tree-insertion behavior, the caller and dependencies must additionally provide:

- A coherent header and root representation.
- A valid proposed insertion location.
- A new node with appropriate initial color and child links.
- Structural dependencies that preserve or restore the expected parent/child relationships.
- Ordering and duplicate-handling rules consistent with the caller’s chosen position.

These are prerequisites, not checks performed by this function.

## Behavior

### 1. Obtain a node and copy its payload

The first external action is a call to F-84e8874bcd0c with the container address. Its result becomes the new-node address retained for the eventual output.

The function then tests whether that address plus 0x10 is nonzero. If so, it reads four bytes from the payload source and writes those four bytes at new-node offset +0x10.

This is not a check that the returned node address itself is nonnull. A null result would ordinarily produce a nonzero destination address of 0x10, so it would still lead toward an invalid payload write. At the 32-bit address-calculation level, wraparound could instead make the offset address zero. The exact language-level validity of such address arithmetic is not established.

Even when the payload copy is skipped by that condition, the function continues into count modification and structural accesses. There is no visible allocation-failure return path.

The copied value is exactly four bytes. No payload constructor, deep copy, reference increment, or ownership transfer is visible.

### 2. Increment the count-like field

After the payload-copy attempt, the function reads the four-byte field at container offset +0x14, adds one, and writes it back.

This occurs before:

- Selecting the attachment side through comparison.
- Linking the new node into the tree.
- Performing insertion repair.
- Publishing the result through the output slot.

There is no overflow guard. The source presents the addition as signed integer arithmetic, so behavior at the signed limit is not established by this material; a portable wraparound guarantee must not be inferred.

If any later operation fails, no rollback of this increment is visible.

### 3. Select and attach to a side

The function first compares the current container header address with the proposed parent.

It selects left-side attachment without calling F-257a89ed2c43 if either:

- The proposed parent equals the current header; or
- The attachment override is nonzero.

Otherwise, it calls F-257a89ed2c43 with, in order:

1. Container address plus 0x19.
2. The original payload-source address.
3. Proposed-parent address plus 0x10.

A nonzero result selects left attachment. Zero selects right attachment. The dependency’s meaning beyond this zero/nonzero decision is unknown; it is not established as a three-way comparison or a particular ordering relation.

The dependency receives the original payload source, not the copied payload in the new node. Consequently, aliasing or changes made by the node-acquisition dependency can affect what is copied and what is subsequently compared.

The function does not inspect whether the selected child slot is empty. It overwrites that slot.

### 4. Update parent and header links

For left attachment, the first structural write places the new-node address at proposed-parent offset +0x08.

The function then reloads the container’s header address:

- If it differs from the proposed parent, the header’s +0x08 link is changed to the new node only if its current value equals the proposed parent.
- If it equals the proposed parent, the function writes the new node to that header’s +0x04 root link, reloads the header address from the container again, and writes the new node to the reloaded header’s +0x0c link.

With a stable header, the latter path establishes root, minimum, and maximum links for a first-node insertion. The actual reads and writes remain significant if header storage aliases another written location.

For right attachment, the function first writes the new-node address at proposed-parent offset +0x0c. It then reloads the container’s header address and changes the header’s +0x0c link to the new node only if that link currently equals the proposed parent.

After these link and header updates, the function writes the proposed-parent address into new-node offset +0x04.

No node parent link is written before the initial child/header updates.

### 5. Repair color and structure

Repair starts with the new node as the current repair node.

At each loop entry, the function reloads the container’s header and reads its +0x04 root link. If the current repair node is that root, repair stops. Otherwise, it reads the current repair node’s parent and that parent’s one-byte marker. A nonzero parent marker also stops repair.

When the parent marker is zero, the function obtains the grandparent and reads the grandparent’s left link to determine the repair orientation.

#### Parent is the grandparent’s left child

The uncle is taken from the grandparent’s right link.

If the uncle is nonnull and its marker is zero:

1. Write 1 to the parent’s marker.
2. Write 1 to the uncle’s marker.
3. Re-read the current repair node’s parent and that parent’s parent, then write 0 to the resulting grandparent’s marker.
4. Again derive the grandparent through those links and make it the current repair node.
5. Continue at the loop condition.

If the uncle is null or has a nonzero marker:

- When the current repair node is the parent’s right child, call F-da6a2f17419a with the container and that parent, then use the saved old parent as the current repair node.
- Write 1 to the parent reached through the current repair node.
- Re-read the relevant parent/grandparent links and write 0 to the reached grandparent.
- Call F-3a8a0224b9df with the container and the grandparent reached through the then-current links.

#### Parent is not the grandparent’s left child

This branch uses the grandparent’s previously read left link as the uncle. The function does not separately verify that the parent is the grandparent’s right child.

If that uncle is nonnull and has marker 0, the same three-marker recoloring and upward continuation occur.

Otherwise:

- When the current repair node is the parent’s left child, call F-3a8a0224b9df with the container and that parent, then use the saved old parent as the current repair node.
- Write 1 to the parent reached through the current repair node.
- Re-read the relevant parent/grandparent links and write 0 to the reached grandparent.
- Call F-da6a2f17419a with the container and the grandparent reached through the then-current links.

The conditional F-3a8a0224b9df invocation is displayed with six arguments rather than two. Beyond the container and parent addresses, those displayed arguments include the uncle address, a value derived from the grandparent address or uncle marker, and two retained scalar values. The retained scalars begin as the override and proposed-parent values, but are set to zero before taking the comparison path. Whether these extra arguments are meaningful to the dependency is unresolved.

Repair does not explicitly terminate after a structural dependency call. It returns to the loop condition.

### 6. Force the current root marker and publish the result

After repair stops, the function reloads the container’s header, reads its root link at +0x04, and writes marker value 1 to that root.

Only afterward does it write the originally acquired new-node address to the output slot.

Thus the output identifies the inserted node, not necessarily the root or the node last used during repair.

## Side effects

Directly visible writes can affect:

- Four bytes of the new node’s payload.
- The container’s four-byte count.
- A selected parent child link.
- Header root and apparent extremum links.
- The new node’s parent link.
- One-byte markers on parents, uncles, grandparents, and the final root.
- The four-byte output slot.

Additional effects of all dependencies are unknown. Their calls may affect memory or external state beyond the fields described here; no preservation guarantee is supplied.

No deallocation, payload destruction, explicit error reporting, or cleanup path is visible. The function retains no separately visible ownership record. Acquiring a node and linking it into the container suggests container ownership, but neither ownership transfer nor eventual release is established.

There is no visible synchronization. Atomicity, concurrent-reader safety, and reentrancy are not established.

Aliased arguments are not rejected. Because the output write occurs last, an output slot overlapping a container or node field can overwrite completed structure. Earlier aliases can likewise affect later reads. Ordinary successful-use expectations therefore require suitably noninterfering storage unless the precise access ordering is intentionally relied upon.

## Dependencies

| Opaque ID | Established interaction | Unestablished properties |
|---|---|---|
| F-84e8874bcd0c | Called first with the container; returns the address used as the new node | Allocation strategy, initialization, failure convention, ownership, other state changes |
| F-257a89ed2c43 | Conditionally receives container +0x19, payload source, and parent payload address; zero selects right, nonzero selects left | Ordering semantics, argument mutation, preservation of container state |
| F-da6a2f17419a | Called during repair with container and a selected node | Exact transformation and all ancillary effects |
| F-3a8a0224b9df | Called during repair with container and a selected node; one displayed call includes extra arguments | Exact signature, extra-argument significance, transformation, ancillary effects |

The repair pattern suggests F-da6a2f17419a and F-3a8a0224b9df perform opposite-direction rotations. This is inferred intent only. No assumption that either dependency preserves unrelated state is part of the established contract.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### First-node insertion

Given a stable header equal to the proposed parent, valid node initialization, and no interfering dependency effects:

- The comparison dependency is not called.
- The count is incremented once.
- The header’s left, root, and right links become the new-node address.
- The new node’s parent becomes the header.
- Repair stops at the root check.
- The root marker becomes 1.
- The output slot receives the new-node address.

### Forced left attachment

Given a nonheader parent and any nonzero override, comparison is skipped and the parent’s left link is overwritten.

If the header’s apparent minimum link equals that parent, it becomes the new node. Otherwise, that header link is left unchanged by this attachment step. No minimum search occurs.

### Comparison-selected right attachment

Given a nonheader parent, zero override, and a zero comparison result, the parent’s right link is overwritten.

If the header’s apparent maximum link equals that parent when read, it becomes the new node. The comparison result does not itself validate that this is a legal insertion position.

### Nonzero parent marker

Given that the new node is not the root and its parent marker is any nonzero byte, repair stops without an uncle inspection or structural dependency call. The final root-marker write still occurs.

### Zero-marker parent and uncle

Given valid ancestry, a zero-marker parent, and a nonnull zero-marker uncle, the parent and uncle receive marker 1 and the reached grandparent receives marker 0. Repair continues from that grandparent.

### Inner-child repair

Given a left-side parent, a null or nonzero-marker uncle, and the current node as that parent’s right child, the function first calls F-da6a2f17419a on the parent. It then recolors through the resulting links and calls F-3a8a0224b9df on the reached grandparent.

The mirrored orientation reverses the dependency order.

### Invalid acquisition result

A null result from F-84e8874bcd0c is not converted into an explicit failure result. The function continues toward invalid memory accesses. Acceptance must not require graceful allocation-failure handling from this function alone.

## Uncertainties

The following remain **documented_only**:

- The allocator-like dependency’s initialization and failure behavior.
- Whether the structure satisfies red-black invariants before or after this operation.
- Payload meaning, comparison semantics, and duplicate policy.
- Exact signatures and transformations of the structural dependencies.
- The significance of the extra arguments displayed at one repair call.
- Signed count-overflow behavior and maximum supported size.
- Exception, fault, or dependency-failure propagation.
- Ownership and eventual release of node and payload resources.
- Behavior under malformed trees, cycles, invalid pointers, hostile aliasing, or concurrent mutation.
- Termination when dependencies do not maintain the structural relationships required by repair.

None of these unknowns should be replaced with conventional container-library assumptions.

## Evidence

Established claims above come from static inspection of the supplied function body: access widths and offsets, branch conditions, call sites, explicit writes, repeated header and ancestry reads, and output timing.

The red-black-tree interpretation, header-extremum roles, and rotation roles are inferred from the structural pattern. They are separated from directly established operations.

No dependency implementations, runtime traces, or independent type definitions were supplied. No runtime tests have been performed.
