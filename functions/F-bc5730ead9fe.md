<!-- Trusted provenance: F-bc5730ead9fe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc5730ead9fe

## Purpose

F-bc5730ead9fe creates a node, attaches it at a caller-selected position in a mutable tree-like structure, performs color-based balancing, and returns the created node through an output location.

The visible field accesses and balancing cases are consistent with insertion into a red-black tree. That interpretation is **inferred intent**, not a complete guarantee: node initialization, comparison semantics, and rotation behavior belong to opaque dependencies. The function does not itself search from the root, reject duplicate values, or validate the proposed parent.

The established contract is therefore an ordered sequence of node construction, count adjustment, attachment, conditional balancing, root-color assignment, and output publication. Full tree correctness requires additional dependency and input invariants.

## Interface

The function has five inputs and no explicit return value:

1. **Output location.** A writable location receiving one 32-bit node address. The function writes this location only after balancing and the final root-color write.
2. **Container address.** A 32-bit address identifying mutable container state.
3. **Attachment override.** A 32-bit integer. Zero permits a comparison-dependent attachment decision; any nonzero value forces the left-side attachment path.
4. **Proposed parent address.** A 32-bit address identifying either an existing node or the container’s header.
5. **Value address.** An address supplying at least two 32-bit words for construction and, conditionally, an input to the comparison dependency.

The supplied representation uses 32-bit pointer-sized fields and 32-bit integer operations. Colors are accessed as individual bytes. No evidence establishes byte order or alignment requirements beyond those implied by the word accesses.

### Container layout

Offsets are byte offsets from the container address.

| Offset | Width | Established use |
|---|---:|---|
| +16 | 32 bits | Address of a header-like object; repeatedly reread |
| +20 | 32 bits | Element-count-like field, incremented once |
| +25 | Address only | Passed as the first comparison argument; its layout is unknown |

The count interpretation is inferred from its increment during insertion. The function does not otherwise inspect it.

### Header and node layout

The header and ordinary nodes share the following link offsets:

| Offset | Width | Established use |
|---|---:|---|
| +0 | 8 bits | Color-like field, used on ordinary nodes |
| +4 | 32 bits | Node parent; on the header, root link |
| +8 | 32 bits | Node left child; on the header, left-extreme-like link |
| +12 | 32 bits | Node right child; on the header, right-extreme-like link |
| +16 | 32 bits | First constructed value word in a new node |
| +20 | Dependency-defined | Destination supplied to the value-construction dependency |

“Root,” “parent,” “left,” “right,” and “extreme” describe the structural interpretation supported by these accesses. The actual fields and their updates are established; their higher-level container meaning remains partly inferred.

### Preconditions

For a normally completing invocation:

- The output location must accept a 32-bit write.
- The container, header, proposed parent, and all links traversed during balancing must support the accesses described below.
- The value address must support the two initial 32-bit reads and any later comparison-dependent reads.
- The node-creation dependency must provide storage suitable for the subsequent payload, link, and color accesses.
- Existing structural relationships must make the parent and grandparent accesses valid whenever balancing reaches them.
- Dependency calls must return normally if completion is expected.

The function performs no explicit validation of these conditions. It has no visible error result or rollback interface.

## Behavior

### 1. Obtain node storage and construct its value

The function first calls F-489ea509a003 with the container address and retains the returned address as the new node.

It tests whether the address obtained by adding 16 bytes to that result is nonzero. This is not a direct test that the returned node address is nonzero.

When the tested address is nonzero, the function:

1. Reads the first 32-bit word from the value address.
2. Writes that word to the new node at offset +16.
3. Reads the second 32-bit word from the value address.
4. Calls F-4113700b9ff3 with the new node’s address at offset +20 and that second word.

The second word is passed as a value, not as the address of its source slot. Its interpretation is unknown.

If the tested address is zero, both payload operations are skipped, but insertion continues. There is no separate failure branch. In a wrapping 32-bit address model, a node result of `0xFFFFFFF0` would make the tested address zero; a null node result would not. Whether such address arithmetic represents a valid execution is not established.

The function does not visibly initialize the new node’s color or child links. Those initial conditions must come from the node-creation dependency or another external invariant.

### 2. Increment the container field

After the construction step, the function reads the 32-bit field at container offset +20, adds one, and writes it back.

This occurs before attachment and balancing. There is no count-overflow check. The source does not establish a language-independent overflow policy; a machine-level 32-bit modular interpretation would wrap an all-ones value to zero, but that boundary behavior is not independently verified.

No restoration of this field is visible if a later operation fails.

### 3. Select and perform attachment

The function reads the header address at container offset +16.

It chooses left-side attachment without calling the comparator if either:

- The proposed parent equals that header address; or
- The attachment override is nonzero.

Only when both conditions are false does it call F-3a00e7887b9d, with these arguments in order:

1. Container address plus 25 bytes.
2. The original value address.
3. Proposed parent address plus 16 bytes.

Any nonzero result selects left-side attachment. Zero selects right-side attachment. The result is used as a zero/nonzero decision, not as a signed ordering value.

#### Left-side attachment

The function first writes the new node address to the proposed parent’s +8 field. It then rereads the header address.

If the proposed parent is not that reread header:

- It reads the header’s +8 field.
- If that field equals the proposed parent, it replaces it with the new node address.
- Otherwise, it leaves that header field unchanged.

If the proposed parent is that reread header:

- It writes the new node address to the header’s +4 field.
- It rereads the container’s header address and writes the new node address to that header’s +12 field.

Consequently, with a stable header and a header-as-parent insertion, the writes occur in this order: header +8, header +4, header +12.

#### Right-side attachment

The function writes the new node address to the proposed parent’s +12 field. It then rereads the header address and checks the header’s +12 field.

If that field equals the proposed parent, it replaces it with the new node address. Otherwise, it leaves it unchanged.

#### Parent-link assignment

After either attachment path, the function writes the proposed parent address to the new node’s +4 field.

There is no visible guard against replacing an existing nonempty child link. Correct placement is a caller/dependency precondition, not a condition enforced here.

### 4. Perform color-based balancing

The balancing cursor initially refers to the new node.

At the start of each iteration, the function rereads the header and its +4 root link. If the cursor equals that root link, balancing stops. Otherwise, it reads the cursor’s parent and the parent’s color byte.

A nonzero parent color stops balancing. A zero parent color enters the balancing cases. Thus zero is the distinguished “red-like” value; one is the value explicitly written for “black-like” nodes. Other nonzero byte values are treated as black-like in these tests.

The function then obtains the grandparent through the parent’s +4 link and reads the grandparent’s +8 link.

#### Parent is the grandparent’s left child

The uncle is read from the grandparent’s +12 link.

If the uncle is nonnull and has color zero, the function performs recoloring in this order:

1. Parent color becomes one.
2. Uncle color becomes one.
3. The cursor’s currently linked grandparent color becomes zero.
4. The cursor advances to the grandparent obtained through the current parent chain.

Balancing then repeats.

If the uncle is null or has a nonzero color:

- If the cursor equals the parent’s +12 child, the function calls F-7dc0dcd36c28 with the container and parent, then moves the cursor to the previously retained parent address.
- It writes one to the color of the cursor’s currently linked parent.
- It writes zero to the color of the cursor’s currently linked grandparent.
- It calls F-7b527193a60e with the container and that currently linked grandparent.

The parent and grandparent used after the dependency call are obtained from the links then present; they are not assumed to retain their earlier relationships.

#### Parent is not the grandparent’s left child

This is the mirror-side branch, although the function does not separately verify that the parent is the grandparent’s right child.

The previously read grandparent +8 link is the uncle candidate. If it is nonnull and its color is zero, the same parent/uncle/grandparent recoloring and cursor advancement occur.

Otherwise:

- If the cursor equals the parent’s +8 child, the function calls F-7b527193a60e and then moves the cursor to the previously retained parent address.
- It colors the cursor’s currently linked parent one.
- It colors the cursor’s currently linked grandparent zero.
- It calls F-7dc0dcd36c28 with the container and that currently linked grandparent.

The first call in this case visibly supplies additional argument expressions beyond the container and parent; their status is described under Dependencies.

### 5. Finalize and publish

After the balancing loop ends, the function:

1. Rereads the header address.
2. Reads its +4 root link.
3. Writes color byte one to that root.
4. Writes the original new-node address to the output location.

The output is the newly created node, not the final balancing cursor or necessarily the root.

## Side effects

Directly visible writes may affect:

- The new node’s first payload word and parent link.
- Payload storage handled by F-4113700b9ff3.
- The container’s count-like field.
- The selected child link of the proposed parent.
- Header root and extreme-like links, conditionally.
- Color bytes on parents, uncles, grandparents, and the final root.
- The caller’s output location, last.

Dependencies may have additional effects. Their global-state behavior, allocation behavior, ownership transfers, and failure behavior are not established.

No explicit deallocation or cleanup operation appears. This does not establish that dependencies never allocate or release memory. The function itself does not visibly free the input value or the new node.

Aliasing can affect observable behavior. In particular, the comparison receives the original value address after construction and count adjustment, and the output write occurs after all other visible work. No independent snapshot or alias protection is provided.

The function provides no synchronization, atomicity, transactional guarantee, or recovery path. If execution stops in a dependency or encounters an invalid memory access, earlier mutations may already be visible.

## Dependencies

| Opaque ID | Established use | Unknown contract |
|---|---|---|
| F-489ea509a003 | Called first with the container; returns the retained new-node address | Allocation, initialization, color, children, ownership, failure behavior |
| F-4113700b9ff3 | Called with new node +20 and the second input word, after the first payload write | Constructed type, storage extent, ownership, global effects |
| F-3a00e7887b9d | Conditionally called with container +25, input value address, and parent +16; zero selects right, nonzero left | Comparison ordering, argument layouts, side effects |
| F-7dc0dcd36c28 | Called in the left-parent inner case and mirror-side final case | Exact link transformations and broader effects |
| F-7b527193a60e | Called in the left-parent final case and mirror-side inner case | Exact link transformations, calling convention, broader effects |

The two balancing dependencies are consistent with opposite-direction tree rotations. That interpretation is inferred from their placement, not independently established.

One invocation of F-7b527193a60e visibly supplies six argument expressions: container, parent, uncle candidate, a 32-bit branch-dependent value, and two saved values.

For the fourth expression:

- If the uncle candidate is null, the retained value is the grandparent address.
- Otherwise, on the path reaching this call, it is the uncle’s nonzero color byte widened to 32 bits.

The last two values begin as the attachment override and proposed parent address. Both become zero if the comparison path was taken, including when that comparison selected left attachment.

Whether these extra expressions are effective formal arguments or reflect an unresolved calling-convention representation is **documented_only**. They must not be silently assigned additional semantics.

## Acceptance examples

These examples describe conditional expectations from the static evidence. They are not runtime results.

1. **Stable header, header-as-parent insertion.**  
   Given a usable created node and normally returning dependencies, the comparator is not called. The count-like field is incremented, then the header receives the node at +8, +4, and +12 in that order. The node’s parent becomes the header. The root-equality check ends balancing immediately, the node’s color becomes one, and the output receives its address.

2. **Forced left attachment.**  
   With an ordinary proposed parent and any nonzero override, no comparator call occurs. The parent’s +8 link is replaced. The header’s +8 link changes only if it equals that parent. This remains true even if the comparator would have selected right.

3. **Comparison-selected right attachment.**  
   With zero override, a non-header parent, and comparator result zero, the parent’s +12 link is replaced. The header’s +12 link changes only if it equals that parent.

4. **Nonzero comparator result.**  
   Under the same conditions, any nonzero result—including a negative 32-bit value—selects left attachment.

5. **Nonzero parent color.**  
   If the new node is not the root and its parent color is nonzero, no balancing dependency is called during that iteration. Finalization still writes one to the current root’s color.

6. **Zero-colored parent and uncle.**  
   With a valid grandparent and a nonnull zero-colored uncle, the parent and uncle become one, the grandparent becomes zero, and balancing continues from the grandparent. No rotation dependency is called in that iteration.

7. **Invalid created-node address.**  
   A null result from F-489ea509a003 does not select an explicit error return. The subsequent address test does not constitute null handling. Safe failure behavior is not established.

## Uncertainties

The following remain **documented_only**:

- The complete container, header, and payload types.
- Node allocation and initialization guarantees.
- Comparator meaning beyond zero/nonzero selection.
- Rotation semantics and the effective signature of F-7b527193a60e.
- Ownership, lifetime, and release obligations.
- Exception, allocation-failure, and invalid-memory behavior.
- Portable overflow and address-wrap behavior.
- Whether all valid callers satisfy red-black-tree insertion invariants.
- Whether dependencies mutate the header, payload input, links, or other shared state.

Accordingly, balanced-tree correctness and successful insertion are conditional on contracts not supplied here.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes access widths and offsets, explicit call arguments, branch predicates, direct write ordering, repeated header and link reads, color tests, and final output publication.

The red-black-tree interpretation, header extreme-link meanings, and rotation intent are inferences. No runtime tests have been performed.
