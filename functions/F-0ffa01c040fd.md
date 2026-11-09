<!-- Trusted provenance: F-0ffa01c040fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ffa01c040fd

## Purpose

F-0ffa01c040fd obtains a node, attaches it at a caller-selected location, performs color- and relationship-dependent repair, and writes the obtained node’s address to an output slot.

**Established static behavior:** the function conditionally copies a 32-bit payload, increments a 32-bit container field, selects a child link, updates structural links, executes a repair loop, writes one to the current root’s color byte, and finally writes the obtained node address to the output slot.

**Inferred intent:** this is an insertion operation for a red-black tree. The container appears to reference a header holding root and boundary links. Zero appears to represent red, and nonzero appears to represent black. Two dependencies have call patterns consistent with opposing rotations. These interpretations do not establish dependency behavior or guarantee tree invariants.

The function does not search for an insertion position, check whether the selected child slot is empty, or reject duplicate payloads.

## Interface

The function has five inputs and no explicit return value.

| Input | Observed role |
|---|---|
| Result slot | Address receiving the obtained node address in a final 32-bit write. |
| Container | Address of mutable container state. |
| Attachment discriminator | 32-bit integer tested for zero versus nonzero when choosing attachment direction; also retained for one dependency call presentation. |
| Attachment parent | 32-bit address identifying the caller-selected parent or header. |
| Payload source | Address supplying an optional 32-bit payload copy and passed to the comparison dependency when required. |

Links, stored addresses, payload transfers, and the incremented field use 32-bit accesses in the supplied representation. Color accesses are one byte. Exact target ABI, alignment requirements, and endianness are not independently established.

### Container layout

Offsets are in bytes.

| Offset | Direct use |
|---|---|
| +16 | Read as the header address, including repeated reads during attachment, repair, and completion. |
| +20 | Read and written as a 32-bit signed integer, incremented once. |
| +25 | Its address is passed as the first argument to F-0423ab0b78da. No direct access to its contents is shown. |

The size and representation of the comparison state beginning at +25 are unknown.

### Node and header layout

| Offset | Width | Observed use |
|---|---:|---|
| +0 | 1 byte | Node color. |
| +4 | 32 bits | Node parent link; header root link. |
| +8 | 32 bits | Node left-child link; apparent header left-boundary link. |
| +12 | 32 bits | Node right-child link; apparent header right-boundary link. |
| +16 | 32 bits for the copy | Payload destination; its address is also supplied for parent-payload comparison. |

The header and boundary interpretations are inferred from access patterns. The function does not independently establish their abstract meaning.

### Preconditions and completion conditions

For a selected execution path to complete normally:

- Every directly accessed address must support the indicated read or write.
- Followed parent, grandparent, header, and root links must lead to suitable storage.
- The obtained node must support all accesses subsequently performed through its address.
- The payload source must be readable when the payload-copy guard succeeds.
- Invoked dependencies must return normally.
- The repair loop must eventually terminate.

There is no general pointer, membership, alignment, capacity, or structural validation.

Nonoverlap is not checked and is not an established universal precondition. However, conventional tree-level outcomes require suitable separation of container fields, header fields, node fields, payload storage, and the result slot. With overlap, the ordered accesses—not independent field-update promises—define the observable behavior.

## Behavior

### Node acquisition and payload copy

The first call is F-c97926c32b91 with the container address. Its returned address is retained as the node ultimately written to the result slot.

The function tests whether the address sixteen bytes beyond that returned address is nonzero. If so, it reads one 32-bit word from the payload source and writes it at node offset +16.

This guard does **not** test whether the returned node address itself is nonnull. A null return therefore does not select an explicit failure path. Exact behavior for exceptional address arithmetic remains dependent on the target semantics.

The function does not directly initialize the obtained node’s color or child links. Their initial values depend on the dependency and existing storage.

### Bookkeeping increment

After the optional payload copy, the function reads the container field at +20, adds one, and writes the result back.

This precedes comparison, attachment, and repair. No explicit rollback appears later.

The supplied representation expresses this as signed 32-bit addition. Defined behavior when incrementing the maximum signed value is not established; guaranteed modulo-\(2^{32}\) behavior must not be assumed.

### Attachment selection

The function reads the header address from container offset +16.

It selects left attachment without calling the comparison dependency if either:

- that header address equals the supplied attachment parent; or
- the attachment discriminator is nonzero.

Otherwise it calls F-0423ab0b78da with these addresses, in order:

1. Container +25.
2. Payload source.
3. Attachment parent +16.

A nonzero result selects left attachment. Zero selects right attachment. Positive and negative nonzero results are not distinguished.

The comparison occurs after the payload copy and bookkeeping increment. Consequently, an aliased payload source may expose a different value to comparison than the value copied earlier. The dependency’s own writes and other side effects are unknown.

The original selection is not restarted if comparison changes the header or other memory. Subsequent accesses use the rereads described below.

### Attachment and header updates

**Left attachment:**

1. Write the obtained node address to attachment parent +8.
2. Reread the header address from container +16.
3. If this header differs from the supplied parent, read header +8. Replace that link with the node address only when its current value equals the supplied parent.
4. If this header equals the supplied parent, write the node address to header +4. Then reread the header address from container +16 and unconditionally write the node address to that header’s +12 link.

The initial child write and later header writes can affect different storage when the header reference changes or fields overlap.

**Right attachment:**

1. Write the obtained node address to attachment parent +12.
2. Read the header address from container +16.
3. Read that header’s +12 link.
4. Replace it with the obtained node address only when its current value equals the supplied parent.

**Common step:** after either attachment path, write the supplied attachment-parent address to obtained node +4.

No occupied-child check or subtree preservation operation is shown. The selected child-link write can disconnect existing structure.

### Repair entry and termination checks

The repair cursor initially identifies the obtained node. Moving this cursor does not change the retained result address.

At every loop condition, the function rereads container +16 and the resulting header’s +4 root link. If the cursor equals that root, the loop ends before reading the cursor’s parent or its parent’s color.

Otherwise, the function reads the cursor’s parent and that parent’s color byte. A nonzero parent color ends the loop.

Only a cursor different from the root, followed by a zero-colored parent, enters a repair branch. The function then reads the parent’s parent as the grandparent and reads the grandparent’s +8 link to select the branch.

### Parent equals the grandparent’s left-child link

The opposite-side relative is read from grandparent +12.

If that address is nonnull and its color byte is zero:

1. Write one to the saved parent’s color.
2. Write one to the saved opposite relative’s color.
3. Follow the cursor’s current parent and grandparent links and write zero to that grandparent’s color.
4. Follow the cursor’s parent and grandparent links again and make the resulting address the cursor.

The loop condition is then reevaluated.

If the opposite relative is null or has a nonzero color:

- Read the saved parent’s +12 link.
- If it equals the cursor, call F-ba7d9afb55b3 with the container and saved parent, then set the cursor to that saved parent address.
- Follow the cursor’s current parent link and write one to that parent’s color.
- Follow the cursor’s parent and grandparent links again and write zero to that grandparent’s color.
- Traverse those links again and call F-cd5e6c93f84c with the container and resulting grandparent address.

The loop then reevaluates its condition.

### Parent does not equal the grandparent’s left-child link

This branch does not separately verify that the parent is the grandparent’s right child.

The opposite-side relative is the address already read from grandparent +8.

If it is nonnull and has color zero, the same ordered recoloring and cursor-promotion sequence applies.

Otherwise:

- Read the saved parent’s +8 link.
- If it equals the cursor, call F-cd5e6c93f84c using the extended argument presentation specified under Dependencies, then set the cursor to the saved parent address.
- Follow the cursor’s current parent link and write one to that parent’s color.
- Follow the cursor’s parent and grandparent links again and write zero to that grandparent’s color.
- Traverse those links again and call F-ba7d9afb55b3 with the container and resulting grandparent address.

Subsequent traversals use current memory. They do not assume that a dependency preserved previously observed relationships. Even without a dependency call, overlapping color and link storage can affect later traversals.

### Completion

After repair ends, the function:

1. Rereads the container’s header address.
2. Reads that header’s +4 root link.
3. Writes one to the root’s color byte.
4. Writes the retained obtained-node address to the result slot.

There is no null-root guard. The result write occurs after the final root-color write and can overwrite it or another structural field if storage overlaps.

## Side effects

Direct writes comprise:

- An optional 32-bit payload copy.
- One 32-bit bookkeeping increment.
- One selected child-link write.
- Conditional or unconditional header-link updates, depending on attachment path.
- One obtained-node parent-link write.
- Zero or more one-byte repair recolorings.
- One final one-byte root-color write.
- One final 32-bit result write.

These are ordered operations, not guarantees that independently named fields retain their written values. Later writes may overlap earlier ones.

For example, if the supplied parent is the header and the obtained node address equals header +4, the common node-parent write targets header +8. It replaces the node address previously written there with the header address. Valid writable storage and a stable header reference alone do not prevent this outcome.

The function contains no direct release or deallocation operation. This does not establish ownership transfer, allocation provenance, or whether dependencies retain or release memory.

There is no explicit error result, rollback mechanism, synchronization, or transaction boundary. Invalid accesses, dependency failures, or nontermination have no specified recovery path in the supplied body.

## Dependencies

| Opaque ID | Established invocation | Inferred role and limits |
|---|---|---|
| F-c97926c32b91 | Called once, first, with the container; its result is retained as the obtained node address. | Likely node acquisition or initialization. Allocation, failure signaling, ownership, and initialized fields are unknown. |
| F-0423ab0b78da | Conditionally called with container +25, payload source, and attachment parent +16. Return tested only for zero. | Likely comparison. Ordering semantics and side effects are unknown. |
| F-ba7d9afb55b3 | Called during repair with container and a saved parent or freshly traversed grandparent address. | Consistent with a left rotation; actual effects require its own contract. |
| F-cd5e6c93f84c | Called during repair with either two presented arguments or the extended presentation below. | Consistent with a right rotation; actual interface and effects remain unresolved. |

One F-cd5e6c93f84c invocation presents six arguments:

1. Container address.
2. Saved parent address.
3. Opposite-side relative address.
4. Grandparent address if the opposite relative is null; otherwise its nonzero color byte promoted to a 32-bit unsigned value.
5. Original attachment discriminator.
6. Original attachment-parent address.

For that invocation, the fifth and sixth presented values are instead zero if execution entered the comparison path, regardless of the comparison result.

Whether the additional arguments are actual consumed parameters is **documented_only**. No dependency may be assumed to preserve container state, globals, payload contents, structural links, or memory ownership without separate evidence.

## Acceptance examples

These are conditional static consequences, not runtime observations.

### Header attachment with separated storage

Assume:

- The header address is stable and equals the supplied parent.
- Container, header, obtained-node, payload-source, and result-slot storage are mutually nonoverlapping over all relevant accesses.
- All required storage is valid.
- F-c97926c32b91 returns the stated usable node and leaves the assumed container/header relationship intact.
- No external mutation occurs.

Then comparison is bypassed. Header +8, +4, and +12 receive the obtained node address; node +4 receives the header address. The first loop condition finds the node equal to the root, so no repair iteration is entered. The node’s color becomes one, and the result slot receives its address.

Without the separation assumptions, these final link outcomes are not guaranteed.

### Forced left attachment

For a nonheader parent and nonzero discriminator, comparison is bypassed and parent +8 is written with the node address.

The subsequently read header’s +8 link is replaced only if its value at that read equals the supplied parent. This describes the immediate operations; overlapping later writes can change their results.

### Comparison-selected right attachment

For a nonheader parent, zero discriminator, and a comparison result of zero, parent +12 receives the node address.

The header reread after that write supplies the boundary link to inspect. Its +12 link is replaced only if it currently equals the supplied parent. No duplicate rejection occurs within this function. A nonzero comparison result instead selects left attachment.

### Root equality takes precedence

If the cursor equals the root obtained at a loop condition, repair stops before reading its parent’s color. Even a zero-colored parent and zero-colored opposite relative elsewhere in the structure do not cause recoloring in that iteration.

### Entered repair with zero-colored opposite relative

Assume that, at the relevant loop reads:

- The cursor differs from the root.
- Its parent has color zero.
- The selected opposite relative is nonnull and has color zero.
- The involved nodes have distinct, nonoverlapping storage, with valid parent/grandparent links unchanged during these direct accesses.

Then the function writes one to the parent and opposite relative, writes zero to the grandparent reached through current cursor links, and promotes the cursor to that grandparent. The next loop condition may immediately terminate repair.

### Nonzero parent color

If the cursor differs from the current root but its parent’s color is nonzero when read, that iteration performs no repair recoloring or repair dependency call. Completion still writes one to the current root and then writes the result address.

### Null acquisition result

A null return from F-c97926c32b91 is not translated into a defined failure result. Neither the payload guard nor subsequent node accesses establish safe allocation-failure handling.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, byte order, alignment, and pointer-arithmetic semantics.
- Signed-overflow behavior of the bookkeeping increment.
- Node initialization, allocation provenance, and ownership.
- Dependency side effects and failure behavior.
- Exact rotation semantics and the consumed parameter list of F-cd5e6c93f84c.
- Comparator ordering and equivalence semantics.
- Full sentinel, acyclicity, and structural-invariant requirements.
- Supported aliasing and concurrent-mutation conditions.
- Recovery after invalid memory access or nonreturning dependencies.

The red-black-tree interpretation is an inference, not a proof of correctness or termination for arbitrary inputs.

## Evidence

Established claims derive from the supplied static function representation and opaque dependency mapping: access widths, offsets, branch conditions, call presentations, repeated reads, and write ordering.

Tree roles, boundary meanings, color meanings, and rotation directions are identified as inferred intent. Dependency implementations, callers, external layout definitions, and runtime traces were not supplied.

No runtime tests have been performed.
