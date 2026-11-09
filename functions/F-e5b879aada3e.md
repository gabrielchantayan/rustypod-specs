<!-- Trusted provenance: F-e5b879aada3e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5b879aada3e

## Purpose

F-e5b879aada3e creates and links a node into a tree-like container, performs color-based structural repair, forces the resulting root’s color byte to 1, and returns the created node through caller-provided storage.

The supplied body establishes the field accesses, insertion decisions, recoloring, and dependency-call order described below. The structure strongly suggests red-black-tree insertion with a header holding the root and extremal nodes. That interpretation is inferred intent: node initialization, comparison semantics, and structural transformations performed by dependencies are not supplied.

This operation is not established as a general key-searching insertion API. The caller supplies an attachment node and a flag that can force one attachment direction. The function itself performs at most one comparison.

## Interface

The interface has five arguments, in this order:

1. **Result storage.** An address receiving one 32-bit node-address value on normal completion.
2. **Container address.** An address represented by a 32-bit argument.
3. **Attachment flag.** A 32-bit integer. Zero and nonzero are distinguished; no narrower Boolean restriction is visible.
4. **Attachment-node address.** A 32-bit address identifying either an existing node or the container header.
5. **Incoming record address.** An address from which an 8-bit field and a 32-bit field are copied, and which can also be passed to the comparison dependency.

There is no explicit return value. The result is delivered only through result storage.

### Observed container layout

Offsets are byte offsets from the container address:

| Offset | Observed use |
|---|---|
| +16 | 32-bit address of a header-like object |
| +20 | 32-bit count-like field, incremented once |
| +25 | Address passed as the first comparison argument |

The extent and representation of the comparison state beginning at +25 are unknown.

### Observed node and header layout

| Offset | Width | Observed use |
|---|---:|---|
| +0 | 8 bits | Color or state byte |
| +4 | 32 bits | Parent address for ordinary nodes; root address in the header |
| +8 | 32 bits | First child address for ordinary nodes; inferred minimum-node address in the header |
| +12 | 32 bits | Second child address for ordinary nodes; inferred maximum-node address in the header |
| +16 | 8 bits | First copied record field |
| +20 | 32 bits | Second copied record field |

The first and second child fields are referred to below as **left** and **right**, respectively, to express the inferred tree interpretation. Their actual offsets, rather than those names, define the established accesses.

The copied record has a byte at offset 0 and a four-byte field at offset 4. Bytes at record offsets 1–3 are not explicitly copied. Likewise, the body does not explicitly initialize node bytes +17 through +19.

### Preconditions

Normal completion requires accessible storage for every address actually dereferenced, including addresses reached through parent, child, header, and root links. Four-byte accesses must be supported at the supplied addresses under the execution environment’s alignment rules.

The attachment node, flag, existing topology, and dependency behavior must be mutually compatible with the repair procedure. The function does not validate those conditions.

The visible body does not establish a valid failure return from node creation. A null returned node is not handled as allocation failure.

## Behavior

### Node creation and record copying

The first observable dependency call is to F-872bff2f3329, with the container address. Its returned address is retained as the created-node address and is the address eventually written to result storage.

After that call, the function tests whether the address obtained by adding 16 to the created-node address is nonzero. If so, it:

1. Reads the incoming record’s byte at offset 0 and writes it to the created node at +16.
2. Reads the incoming record’s four-byte field at offset 4 and writes it to the created node at +20.

The byte copy precedes the four-byte copy. This ordering can matter if source and destination storage overlap. No whole-record snapshot is established before the first write.

The guard is not a conventional null-node check. For a returned address of zero, the tested address is 16 and the payload stores are attempted. Under a wrapping 32-bit address interpretation, the exceptional returned address `0xfffffff0` makes the tested address zero and skips the payload copies. Neither case is an established supported input: later node accesses still occur.

The body then reads the container’s four-byte count-like field at +20, adds one, and writes it back. This happens before the attachment comparison and before linking the node.

No overflow check is visible. The operation is a 32-bit update; behavior at signed overflow cannot be settled from the supplied representation alone. A portable contract should not promise either successful wraparound or a reported overflow error.

### Choosing and recording the attachment

The function compares the current header address from container +16 with the supplied attachment-node address.

It selects left attachment without calling the comparator when either:

- The attachment node equals the header; or
- The attachment flag is nonzero.

Otherwise it calls F-b5e582db3504 with:

- The address container +25;
- The incoming record address;
- The address attachment node +16.

A nonzero result selects left attachment. Zero selects right attachment. No particular ordering convention, equality policy, or duplicate-key policy is established for that dependency.

#### Left attachment

The created-node address is first written to attachment node +8.

The header address is then read again from the container.

- If this header differs from the attachment node, its +8 field is read. That field is replaced by the created-node address only if its previous value equals the attachment node.
- If this header equals the attachment node, the created-node address is written to its +4 root field. The header address is read again, and that header’s +12 field is set to the created-node address unconditionally.

With a stable conventional header, insertion through the header therefore establishes the same new node as root, minimum, and maximum. Repeated header reads remain significant if earlier writes alias container storage.

#### Right attachment

The created-node address is first written to attachment node +12.

The header address is then read from the container. Its +12 field is replaced by the created-node address only if that field currently equals the attachment node.

There is no corresponding root-field assignment on this branch.

After either branch, the supplied attachment-node address is written to the created node’s +4 parent field. Thus attachment and extremal-header updates precede this parent assignment.

### Color-based repair

The repair cursor initially refers to the created node. The originally created address is retained separately for the final result, even when the cursor changes.

At each iteration, the function reads the current header from the container and then its +4 root field.

- If the cursor equals that root, repair stops.
- Otherwise, it reads the cursor’s parent address and that parent’s color byte.
- If the parent’s color byte is nonzero, repair stops.
- If the parent’s color byte is zero, repair continues using the parent’s parent as the grandparent.

For descriptive purposes, color 0 is called **red** and color 1 **black**. The actual tests are broader: any nonzero parent or uncle color is treated as not red, while recoloring stores use exactly 0 or 1.

#### Parent is the grandparent’s left child

The right child of the grandparent is treated as the uncle.

If that uncle is nonnull and its color byte is zero:

1. The parent’s color is set to 1.
2. The uncle’s color is set to 1.
3. The grandparent reached through the cursor’s current parent chain is set to color 0.
4. The cursor moves to the grandparent reached through that chain.

If the uncle is null or has nonzero color:

- When the cursor is the parent’s right child, F-b82c2922b45c is called with the container and the saved parent address. The cursor then becomes that saved parent.
- The function reads the cursor’s current parent and sets its color to 1.
- It reads the current parent chain and sets the resulting grandparent’s color to 0.
- F-9966ca29b392 is called with the container and the grandparent obtained from the current parent chain.

The chain is reread around writes and calls; these steps do not establish that dependencies leave its addresses unchanged.

#### Parent is not the grandparent’s left child

This branch treats the grandparent’s left child as the uncle. It does not separately verify that the parent is the grandparent’s right child.

If this uncle is nonnull and has color zero, the same parent/uncle recoloring and cursor promotion occur.

Otherwise:

- When the cursor is the parent’s left child, F-9966ca29b392 is called with the container and saved parent, plus additional arguments described under Dependencies. The cursor then becomes that saved parent.
- The cursor’s current parent is set to color 1.
- Its current grandparent is set to color 0.
- F-b82c2922b45c is called with the container and the grandparent read from the current chain.

There is no explicit iteration bound, cycle detection, or rollback.

### Completion

After repair stops, the function rereads the header from the container and the root from header +4, then writes color byte 1 at that root address.

Only after this final root-color write does it store the originally created-node address into result storage. If result storage aliases tree or container storage, this last store can itself alter that storage.

## Side effects

Directly established writes include:

- Up to two payload-field writes into the created node;
- One increment of the container’s count-like field;
- One attachment-child write;
- Conditional header extremum writes and, on header attachment, a root write;
- One created-node parent write;
- Zero or more repair recolorings;
- One final root-color write;
- One final result-address write.

Dependency calls may have additional effects. Their effects on globals, allocation state, memory ownership, tree links, and error handling are not supplied and must not be assumed absent.

There is no visible explicit release operation, incoming-record write, rollback, error code, or exception handler in this body. This does not establish that dependencies cannot release memory or terminate execution.

Under the inferred container model, the created node becomes part of the container and the result is a handle to it. Actual allocation ownership, disposal responsibility, and handle lifetime remain unknown.

## Dependencies

| Opaque ID | Established use | Inferred role or limitation |
|---|---|---|
| F-872bff2f3329 | Called once first, with the container; returns the node address | Likely node allocation or preparation. Initialization and failure semantics are unknown. |
| F-b5e582db3504 | Called only for nonheader attachment with a zero flag; receives container +25, incoming record, and attachment node +16 | Likely comparison. Only zero versus nonzero is consumed. |
| F-b82c2922b45c | Called with the container and a node address during repair | Consistent with a left rotation; implementation is not supplied. |
| F-9966ca29b392 | Called during complementary repair steps | Consistent with a right rotation; implementation and full calling contract are not supplied. |

One F-9966ca29b392 call site displays six arguments, unlike the two-argument call in the opposite repair branch. Beyond the container and saved-parent address, the displayed arguments carry:

- The uncle address;
- The grandparent address if the uncle is null, otherwise the uncle’s nonzero color byte promoted to a 32-bit value;
- The original attachment flag on a direct-attachment path, or zero after the comparison path;
- The original attachment-node address on a direct-attachment path, or zero after the comparison path.

Whether those additional arguments are meaningful parameters, unused machine-level values, or a recovered-signature artifact is unknown. A faithful behavioral account must retain the call-site distinction without inventing a dependency contract.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests.

### Header attachment

Given a stable header, a valid prepared node, and the header as attachment node:

- No comparison call occurs, regardless of the flag.
- The incoming byte and four-byte field are copied.
- The count-like field is incremented.
- Header +8, +4, and +12 become the created-node address in that write order.
- The new node’s parent becomes the header.
- Repair stops immediately because the cursor is the root.
- The root color becomes 1 before result storage receives the new-node address.

This example assumes the dependency and payload writes do not unexpectedly alter the header configuration.

### Forced left attachment

Given an ordinary attachment node and any nonzero flag, no comparison occurs. Its +8 field is replaced by the new-node address. The header’s +8 field changes only if it currently names that attachment node.

The function does not first search for an empty child slot or reject an occupied one.

### Comparison-selected right attachment

Given a zero flag, a nonheader attachment node, and a zero comparison result, the new node is written to attachment node +12. Header +12 changes only when its current value equals the attachment node.

This example does not imply that zero means “greater than or equal”; the comparator’s meaning is unknown.

### Red parent and red uncle

Given a nonroot cursor, a zero-colored parent, and a nonnull zero-colored uncle, the parent and uncle become color 1, the grandparent becomes color 0, and repair continues from that grandparent. No rotation dependency is called for that iteration.

### Null uncle

A null uncle is not dereferenced for its color. It selects the structural-repair case, not the recolor-and-promote case. Other required links must still be valid.

### Noncanonical nonzero color

A parent color of 2 stops repair just as a parent color of 1 does. An uncle color of 2 takes the same branch as an uncle color of 1. The function does not normalize all encountered nonzero colors.

### Failure after count increment

If comparison, linking, repair, or a dependency fails after the count increment, the body supplies no rollback path. A transactional insertion guarantee is therefore not supported.

## Uncertainties

- **documented_only — Node preparation:** Initial color, child links, allocation source, allocation size, and failure convention are unavailable.
- **documented_only — Dependency effects:** Global preservation, ownership changes, release behavior, and mutation of container fields are unknown.
- **documented_only — Structural transformations:** Rotation names are inferred from call placement, not verified dependency behavior.
- **documented_only — Ordering policy:** Comparator semantics and duplicate handling are unknown.
- **documented_only — Integer boundary behavior:** A four-byte increment is visible, but signed-overflow semantics are not established.
- **documented_only — Address model:** The representation uses 32-bit address-bearing values. Endianness, alignment guarantees, and behavior of exceptional address arithmetic are not supplied.
- **documented_only — Malformed topology:** Cycles, invalid parent chains, null roots, inconsistent header fields, and occupied attachment slots are not validated. Their outcomes are not specified as supported behavior.
- **documented_only — Aliasing:** No nonaliasing declaration is provided. The stated access order matters; simpler tree-level postconditions require noninterfering storage and compatible dependencies.
- **documented_only — Concurrency and reentrancy:** No synchronization or concurrent-modification contract is established.
- **documented_only — Additional call arguments:** The significance of the expanded F-9966ca29b392 call is unresolved.

## Evidence

Established static evidence consists of the supplied function body: byte and four-byte accesses, offsets, branch conditions, repeated link reads, direct writes, dependency-call sites, and final result assignment.

The red-black-tree interpretation, header extremum meanings, and rotation directions are inferred from those operations. They are not independently established dependency specifications.

No runtime tests have been performed. No dependency implementation, allocator contract, comparator definition, or external type declaration was supplied.
