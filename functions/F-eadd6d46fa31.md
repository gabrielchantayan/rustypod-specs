<!-- Trusted provenance: F-eadd6d46fa31; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eadd6d46fa31

## Purpose

F-eadd6d46fa31 obtains a node from a dependency, conditionally copies an eight-byte payload into it, attaches it at a caller-supplied tree position, performs color-based balancing, and publishes the originally obtained node address through an output slot.

The established behavior consists of 32-bit link operations, one-byte color operations, conditional dependency calls, and ordered memory writes. The structure and mirrored balancing cases suggest red-black-tree insertion. That interpretation is **inferred intent**, not proof that the dependencies preserve any particular tree invariant.

The function does not search for an insertion parent. It does not visibly reject duplicate values, check that the selected child link is empty, or validate the supplied structure.

## Interface

### Arguments and result

| Argument | Established role |
|---|---|
| Output slot | Address receiving the originally obtained node’s 32-bit address representation after balancing and the final root-color write. |
| Tree address | 32-bit address used to locate management fields and passed to dependencies. |
| Attachment selector | 32-bit value tested for zero versus nonzero during attachment selection; also retained for a possible dependency argument. |
| Attachment parent | 32-bit address identifying the supplied parent or header; also retained for a possible dependency argument. |
| Value source | Address of two consecutive 32-bit words supplying the payload. Also passed directly to the comparison-like dependency when that call is required. |

No explicit return value is established. The result is the final write to the output slot.

No general address, alignment, size, consistency, or output-slot validation is visible. Successful use requires accessible storage for every read and write actually reached.

### Management-object layout

Offsets are byte offsets from the tree address.

| Offset | Width or use | Established behavior |
|---|---|---|
| `0x10` | 32-bit address | Locates the header. This field is read repeatedly rather than captured once for the whole operation. |
| `0x14` | 32-bit integer | Read, incremented by one, and written after payload handling. |
| `0x19` | Address passed to a dependency | First argument of the comparison-like call. No object size or representation is established. |

The field at `0x14` appears to be a node count. Its established behavior is an increment, not a validated count invariant.

### Node and header layout

| Offset | Width | Ordinary-node use |
|---|---|---|
| `0x00` | One byte | Color or balancing state. |
| `0x04` | 32 bits | Parent address. |
| `0x08` | 32 bits | Left-child address. |
| `0x0c` | 32 bits | Right-child address. |
| `0x10` | 32 bits | First payload word. |
| `0x14` | 32 bits | Second payload word. |

“Left” and “right” name the two observed child-link positions. The header uses offsets `0x04`, `0x08`, and `0x0c` as apparent root, left-extreme, and right-extreme links.

A zero parent-color byte permits entry into balancing. Any nonzero parent-color byte stops that condition. An uncle-color byte is likewise distinguished as zero versus nonzero. Direct recoloring writes exactly `0` or `1`; existing nonzero colors need not equal `1`.

### Preconditions for successful completion

The caller and dependencies must collectively provide:

- A created node whose subsequently accessed storage is valid.
- A compatible attachment parent and header.
- Valid parent and ancestor chains wherever balancing traverses them.
- A writable root color at finalization.
- Accessible source words whenever payload copying occurs.
- A writable output slot at publication.

The function does not establish these conditions. It also does not establish safety for arbitrary overlap among node storage, source words, management fields, header fields, or the output slot.

## Behavior

### Creation and payload handling

The first dependency call is F-7f10dc6e7fcb with the tree address. Its result becomes the created-node address. The function retains that original address for final publication, independently of later changes to its balancing position.

The payload guard tests whether the effective address at created-node offset `0x10` is nonzero. It does **not** directly test whether the created-node address is nonzero.

If that effective address is nonzero, the observable word-operation order is:

1. Read the first 32-bit source word.
2. Write it at created-node offset `0x10`.
3. Read the second 32-bit source word.
4. Write it at created-node offset `0x14`.

This is not an established atomic eight-byte transfer. In particular, if the first destination overlaps the second source word, the first write can change the value subsequently read.

If the effective address is zero, neither payload word is copied. Execution still proceeds to the increment and attachment logic. This branch is not an established allocation-failure return.

No direct initialization of the created node’s child links or initial color is visible. Any required initialization must already hold, potentially because of F-7f10dc6e7fcb.

### Management-field increment

After payload handling, the function reads the 32-bit integer at tree offset `0x14`, adds one, and writes the result back.

This precedes comparison, attachment, balancing, and output publication. No overflow check or rollback is visible. The supplied representation uses a signed 32-bit integer operation; exact overflow behavior at its representational boundary is **documented_only** rather than specified as guaranteed wraparound.

### Attachment-side selection

The function first reads the current header address from tree offset `0x10` and compares it with the supplied parent.

It selects left attachment without comparison if:

- That header address equals the supplied parent; or
- The attachment selector is nonzero.

Otherwise, it prepares two retained values as zero and calls F-bcefa5957e76 with:

1. The address at tree offset `0x19`.
2. The original value-source address.
3. The address at supplied-parent offset `0x10`.

A nonzero result selects left attachment; zero selects right attachment. Positive and negative nonzero results have the same selection effect.

The comparison-like dependency receives the original source, not the created node’s payload. Their contents may differ because of aliasing or intervening dependency effects.

The two retained values mentioned above initially contain the original selector and supplied parent. They remain those original values when comparison is bypassed, but both become zero whenever the comparison path is entered. They are later relevant only to the six-argument occurrence of F-57d09289448f.

### Attachment writes

For **left attachment**, the first structural write places the created-node address in the supplied parent’s left link at `0x08`. The function then reads the tree’s header address again.

- If that header differs from the supplied parent, its left-extreme link is read and replaced only when it equals the supplied parent.
- If that header equals the supplied parent, its root link at `0x04` is written with the created node. The tree’s header address is then read again, and the right-extreme link of that newly obtained header is written unconditionally with the created node.

With a stable header equal to the supplied parent, these writes establish all three apparent header links as the created node. The repeated header reads remain part of the behavior; this conclusion is conditional on header stability.

For **right attachment**, the created-node address is first written to the supplied parent’s right link at `0x0c`. The function then obtains the current header and replaces its right-extreme link only if that link equals the supplied parent.

After either attachment path, the created node’s parent link at `0x04` is written with the originally supplied parent address.

The selected child link is overwritten without an occupancy check. No direct preservation or release of its former target is visible.

### Balancing entry and common conditions

The balancing position initially equals the created node.

At each loop check, the function reads the current header and its root link. If the root equals the balancing position, the loop ends without reading that position’s parent color.

Otherwise, it reads the position’s parent address and then that parent’s color byte. A nonzero color ends balancing. A zero color causes the parent’s parent address to be read as the grandparent, followed by the grandparent’s left-child link.

The parent is classified solely by equality with that left-child link. Failure of this equality selects the mirrored case; there is no separate validation that the parent equals the grandparent’s right child.

### Parent equals the grandparent’s left child

The grandparent’s right child is captured as the uncle.

If the uncle is nonnull and its color is zero, the recoloring path described below is taken.

Otherwise:

- If the balancing position equals the parent’s right child, F-3ed578214a76 is called with the tree and the captured parent. After that call, the balancing position is set to that captured parent.
- The current position’s parent link is read, and the addressed parent is colored `1`.
- The current position’s parent chain is read again to locate a grandparent, which is colored `0`.
- The parent chain is read again to obtain the node passed with the tree to the two-argument occurrence of F-57d09289448f.

The loop then rechecks its root and parent-color conditions.

The post-call links are read from memory. They are not assumed to retain the values observed before F-3ed578214a76.

### Parent does not equal the grandparent’s left child

The already captured grandparent-left link is the uncle.

If the uncle is nonnull, its color byte is loaded and zero-extended to a 32-bit value. A zero value takes the recoloring path and cannot reach the six-argument call in that iteration.

For a null uncle or a nonnull uncle with nonzero color:

- If the balancing position equals the captured parent’s left child, the six-argument occurrence of F-57d09289448f is called.
- After that call, the balancing position is set to the captured parent.
- Whether or not that conditional call occurred, the current position’s parent is located and colored `1`.
- Its parent chain is read again to locate a grandparent, which is colored `0`.
- The chain is read again to obtain the node passed with the tree to F-3ed578214a76.

The six supplied argument positions are:

| Position | Established value |
|---|---|
| 1 | Tree address. |
| 2 | Captured parent address. |
| 3 | Captured uncle address, possibly null. |
| 4 | For a nonnull uncle: its loaded, zero-extended, **nonzero** color byte, from `1` through `255`. For a null uncle: the previously loaded grandparent address retained as a 32-bit value. |
| 5 | Original attachment selector if comparison was bypassed; otherwise zero. |
| 6 | Original supplied parent if comparison was bypassed; otherwise zero. |

Thus a nonnull zero-colored uncle never supplies zero as the fourth argument to this call: that color selects recoloring instead. Even with a null or nonzero-colored uncle, this call additionally requires the balancing position to equal the parent’s left child.

The actual dependency signature and meaning of the extra positions remain unresolved.

### Shared recoloring path

When either orientation finds a nonnull zero-colored uncle:

1. The captured parent’s color is written as `1`.
2. The captured uncle’s color is written as `1`.
3. The current position’s parent chain is read to locate a grandparent, whose color is written as `0`.
4. The parent chain is read again, and that resulting grandparent becomes the balancing position.

No balancing dependency is called on this path during that iteration.

For a well-formed, nonoverlapping structure, these traversals identify the previously observed grandparent. The actual operations use fresh link reads; aliasing must not be silently replaced with that conventional interpretation.

### Finalization

Once balancing ends, the function reads the current header and its root link, then writes `1` to that root’s color byte.

Only afterward does it write the originally created node’s 32-bit address representation into the output slot.

The published address is not necessarily the final root or the final balancing position. If the output slot overlaps previously modified storage, its final write can overwrite that storage; no alias protection is visible.

## Side effects

Direct modifications comprise:

- Up to two payload words.
- The 32-bit management field at `0x14`.
- The selected supplied-parent child link.
- Conditional header links.
- The created node’s parent link.
- Balancing color bytes.
- The final root color.
- The output slot.

Dependencies may additionally mutate memory, global state, links, colors, or ownership. Their lack of such effects is not established.

No explicit release, rollback, error result, exception handler, or synchronization operation is visible. If a call or memory access prevents completion, earlier writes are not visibly undone, and output publication may not occur.

The apparent result is a node linked into a tree and exposed to the caller. Whether that exposed address is borrowed, owned, or subject to another lifetime convention is **documented_only**. No assumption that a dependency allocates, preserves, transfers, or releases ownership is justified by this function alone.

Concurrency, reentrancy, and transactional guarantees are not established.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-7f10dc6e7fcb | Called first with the tree; its returned address is used as the created node. | Allocation, initialization, failure signaling, ownership, and other effects. |
| F-bcefa5957e76 | Conditionally receives management-state address, original source address, and parent-payload address. Zero selects right; nonzero selects left. | Ordering relation, equality handling, mutations, and failure behavior. |
| F-3ed578214a76 | Receives the tree and a node in the balancing cases specified above. | Exact structural operation, memory effects, and failure behavior. |
| F-57d09289448f | Appears in both two-argument and six-argument forms, under the distinct conditions specified above. | True signature, interpretation of additional arguments, structural effects, and failure behavior. |

The balancing calls suggest opposite rotation directions. That is inferred intent only. No dependency is assumed to preserve globals, header fields, node links, colors, or memory lifetime.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Header insertion.** With a stable header supplied as parent, accessible storage, and a suitably initialized created node, comparison is bypassed. Header left-extreme, root, and right-extreme links become the created node. Its parent becomes the header. Root equality ends balancing; the node is colored `1` before publication.

2. **Selector bypass.** With a non-header parent and any nonzero 32-bit selector, including a negative value, left attachment is selected without comparison. The retained selector and parent remain their original values for any later six-argument call.

3. **Comparison entered.** With a non-header parent and zero selector, comparison is called and both retained values become zero. A zero result selects right; either positive or negative nonzero selects left.

4. **Nonzero-colored parent.** If the created node is not the current root and its parent color is nonzero, balancing ends at that check. The final root-color write still precedes publication.

5. **Zero-colored uncle.** Given a zero-colored parent and a nonnull zero-colored uncle, the captured parent and uncle become `1`, the freshly located grandparent becomes `0`, and balancing moves upward. Neither balancing dependency is called during that iteration.

6. **Six-argument call with colored uncle.** In the mirrored case, suppose the uncle is nonnull with color `255`, the parent is zero-colored, and the balancing position equals the parent’s left child. F-57d09289448f receives that uncle address in position three and the 32-bit value `255` in position four—not zero and not a sign-extended negative value.

7. **Six-argument call with null uncle.** Under the same positional condition but with a null uncle, argument three is zero and argument four retains the previously loaded grandparent address. Arguments five and six depend on whether comparison was entered earlier.

8. **Positional condition absent.** In the mirrored case with a null or nonzero-colored uncle, if the position does not equal the parent’s left child, the six-argument call is skipped. The subsequent parent/grandparent recoloring and F-3ed578214a76 call still occur.

9. **Payload overlap.** If the first payload destination coincides with the second source word, the second read observes the value after the first write, assuming otherwise accessible storage.

10. **Occupied child link.** An occupied selected child link is overwritten. No visible occupancy error or release accompanies that direct write; structural validity afterward is not guaranteed.

## Uncertainties

The following remain **documented_only**:

- Creation dependency behavior, initialization, and allocation-failure handling.
- Comparator semantics beyond zero/nonzero selection.
- Balancing dependencies’ implementations and true signatures.
- Meaning and consumption of the six-argument call’s extra positions.
- Signed-increment overflow and exceptional pointer-arithmetic semantics.
- Supported alignment and aliasing beyond the explicit ordered accesses.
- Fault handling, exception propagation, and partial-state guarantees.
- Ownership, destruction, and dependency effects on memory lifetime.
- Concurrency and reentrancy.
- Termination or consequences for malformed structures.

Red-black-tree intent, header-extreme meanings, and rotation roles remain inferences. They do not override the established branches, access widths, argument values, or repeated memory reads.

## Evidence

The evidence is the supplied static function representation and opaque dependency mapping. The prior narrative and reviewer feedback are not independent evidence of behavior.

The representation establishes the field offsets and widths, ordered payload operations, increment placement, attachment conditions, repeated header and parent-chain reads, balancing branches, recoloring values, dependency-call positions, and publication order.

For the six-argument call specifically, the nonnull-uncle path loads an unsigned byte into a 32-bit value. Zero branches to recoloring before the call site; only values `1`–`255` can reach that call with a nonnull uncle. The null-uncle path skips that load and retains the earlier grandparent address. The parent-left-child equality condition independently gates the call.

No dependency implementation or independent structure definition was supplied. No runtime tests have been performed.
