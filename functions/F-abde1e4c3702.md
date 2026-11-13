<!-- Trusted provenance: F-abde1e4c3702; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-abde1e4c3702

## Purpose

F-abde1e4c3702 conditionally removes a selected node from a binary-tree-shaped structure, performs byte-color-dependent repair, and prepends the removed node to a container-maintained chain. It publishes a four-byte result through caller-provided storage.

The structure and repair resemble a red-black tree with a header containing the root and cached extrema. Those names describe inferred intent; the established behavior is the accesses, rewiring, dependency dispatch, and publication below.

An initial dependency can bypass removal. That path publishes a freshly read container header value without this function directly changing the tree, detached chain, or count.

## Interface

The function receives three arguments:

- **Result location:** storage receiving one four-byte value.
- **Container address:** address of the mutable container representation.
- **Selection location:** storage containing a four-byte value interpreted as a node address on the mutation path.

There is no explicit return value.

### Represented layout

Address-bearing fields and integer fields occupy four bytes. Color fields occupy one byte. These are representation widths, not a portability guarantee or a complete ABI description.

| Object | Offset | Width | Structural use |
|---|---:|---:|---|
| Container | +4 | 4 bytes | Detached-chain head |
| Container | +16 | 4 bytes | Header address |
| Container | +20 | 4 bytes | Count |
| Header | +4 | 4 bytes | Root address |
| Header | +8 | 4 bytes | Leftmost address |
| Header | +12 | 4 bytes | Rightmost address |
| Node | +0 | 1 byte | Color or repair state |
| Node | +4 | 4 bytes | Parent address |
| Node | +8 | 4 bytes | Left-child address |
| Node | +12 | 4 bytes | Right-child address, subsequently reused as a chain link |

Zero child values denote absent children in the tested branches. No payload layout, complete allocation size, key representation, or comparison rule is established.

### Preconditions and failure model

The selection location and container header field must initially be readable. The result location must be writable when publication occurs.

On the mutation path, the selected node and every accessed header, relative, and field must support the particular accesses performed. A null selected node is not rejected before dereference. Null children are supported only where explicitly checked.

Traversal requires terminating relationships. There is no cycle detection, structural validation, count validation, explicit error result, or rollback.

No non-aliasing contract is established. Storage shared among arguments, container fields, nodes, or dependency-accessible state can affect subsequent accesses. The sequencing below therefore does not replace rereads with entry-time snapshots. It describes source-visible access order, not hardware atomicity or a concurrency guarantee.

## Behavior

### Preliminary decision and result state

The function first reads four bytes from the selection location into a local result cell. It then reads the container’s header address into a separate four-byte local cell.

F-6183894e2484 receives the addresses of both cells.

If its integer result is nonzero, the function rereads the container header field, replaces the local result with that value, publishes it, and finishes. There is no direct removal, count decrement, or chain insertion on this path. Effects of F-6183894e2484 remain unrestricted.

If its result is zero, F-b65c251036f5 receives the address of the local result cell. After that call, the function rereads the selection location to obtain the node to remove.

Thus, the initial selection snapshot, the eventual output, and the node actually removed need not be equal. The body does not replace the mutation-path result with the removed node. Publication uses the result cell’s contents at publication time; dependency effects on exposed storage are not assumed away.

### Leaf removal

If both children are absent, the selected node’s parent becomes the repair cursor.

When that parent equals the current container header, the function performs the singleton updates in this order:

1. Writes the saved parent address to the current header’s rightmost field.
2. Writes that address to the current header’s leftmost field.
3. Clears the current header’s root field.

The header address is read again for these destinations; it is not one immutable entry-time header.

Otherwise, the parent’s left-child value is read and compared with the selected node. Equality selects the left link; every other value selects the right link. The chosen link is cleared with four one-byte zero writes in ascending address order.

That same saved comparison chooses which cached extreme to inspect: leftmost for the left-link case, rightmost otherwise. If the chosen extreme equals the selected node, it is replaced with the parent. The other extreme is not examined by this branch.

The originally selected node is the node ultimately chained as detached.

### Exactly one child

The existing child becomes the replacement and initial repair cursor. Its parent field is first assigned the selected node’s parent value.

The attachment is then changed:

- If the current header’s root equals the selected node, the root becomes the replacement.
- Otherwise, equality with the parent’s left child selects that link.
- Otherwise, the parent’s right link is replaced.

Cached extrema are handled afterward, leftmost before rightmost.

If the leftmost value equals the selected node, the selected node’s right-child field is checked again. An absent right child causes the leftmost value to become the selected node’s parent; otherwise, left-child traversal from the replacement supplies the new leftmost value.

If the rightmost value equals the selected node, the selected node’s left-child field is checked again. An absent left child causes the rightmost value to become the selected node’s parent; otherwise, right-child traversal from the replacement supplies the new rightmost value.

These decisions use fields read at their respective stages. The original selected node remains the detached node.

### Two children

The function finds the leftmost node in the selected node’s right subtree. This is called the **successor** here as structural terminology, without asserting a key-ordering contract. Its right child is saved as the prospective repair cursor.

The successor moves into the selected node’s structural position. The selected node itself is detached; no payload copying is visible.

The significant write order is:

1. Set the selected node’s left child’s parent to the successor.
2. Assign the selected node’s left-child value to the successor’s left link.
3. Repair the successor’s former position.
4. Replace the selected node at its root or parent attachment.
5. Assign the selected node’s parent value to the successor’s parent.
6. Exchange the successor’s and selected node’s color bytes.

For an immediate-right-child successor, its nonnull right child has its parent set to the successor.

For a deeper successor, its nonnull right child first receives the successor’s former parent. The former parent’s left link is then assigned that child, including null. The successor receives the selected node’s right subtree, whose parent is changed to the successor.

During the color exchange, the successor’s old byte is saved, the successor receives the selected node’s byte, and the selected node receives the saved byte. If the successor’s saved right child was null, the moved successor itself becomes the repair cursor.

No explicit cached-extreme update occurs in this branch.

### Repair entry and termination

After structural changes, the function rereads the container header. Repair is entered only when the cursor differs from that header and the detached node’s color byte is nonzero. The detached-color test accepts every nonzero byte, not only 1.

Within repair, a step proceeds only when the cursor is not the current root and its color is exactly 1. The root is obtained through the currently read container header.

Every exit from an entered repair region writes color 1 to the current cursor. Skipping repair entirely does not perform that write.

For a continuing step, the function reads the cursor’s parent and retains the address of the cursor’s parent field for later rereads. Equality between the cursor and that parent’s left child selects the **left orientation**; inequality selects the **opposite orientation**. The latter is not independently validated as a right-child relationship.

### Repair dispatch and access order

“Near” means the sibling child toward the cursor; “far” means the other sibling child.

| Orientation | Sibling | Near child | Far child | Red-sibling and terminal helper | Near-child adjustment helper |
|---|---|---|---|---|---|
| Cursor equals parent’s left child | Parent’s right child | Sibling’s left child | Sibling’s right child | F-bd7d3aa9e3fb | F-7e485a919ef1 |
| Cursor does not equal parent’s left child | Parent’s left child | Sibling’s right child | Sibling’s left child | F-7e485a919ef1 | F-bd7d3aa9e3fb |

The red-sibling and terminal calls visibly receive the container and a reread parent-field value. Near-child adjustment calls visibly receive only the container. No additional argument is established for those calls.

**Absent sibling.** If the initial sibling is null, the cursor’s color becomes 0 and processing continues with the previously read parent as cursor.

**Sibling with color 0.** The sibling is first colored 1. The cursor’s parent field is reread to find the node colored 0. The parent argument is then read through the saved cursor-parent-field address, and the orientation-specific red-sibling helper is called with that value and the container.

After the call, the parent is reread through that same saved field address. The sibling is then reloaded from the orientation-specific child field of that reread parent. Neither the pre-call parent nor the pre-call sibling is assumed preserved.

If this reloaded sibling is null, the old cursor is colored 0, and processing continues with the parent value obtained immediately after the call. That parent value is already saved before the cursor-color write.

**Both sibling children absent or color 1.** The near child is checked first; the far child is checked as needed. If each is absent or colored exactly 1, the sibling is colored 0. Only afterward is the cursor’s parent reread to obtain the next cursor. This differs from the absent-sibling case’s use of an already saved parent.

**Near-child adjustment.** Otherwise, the far child is examined again. If it is absent or colored exactly 1:

- The near-child field is checked again for nonzero. If present, the previously read near-child address is used for a color-1 write.
- The sibling is colored 0.
- The orientation-specific near-child helper is called with only the container visibly supplied.
- The cursor’s parent is reread, and the sibling is reloaded through that parent’s orientation-specific child field.

If this sibling reload yields null, repair exits to the final cursor-color-1 write. It does not take the absent-sibling propagation path.

**Terminal case.** For the remaining case, using the current sibling:

1. The cursor’s parent is reread, and its color is read into the sibling’s color.
2. The saved cursor-parent-field address is dereferenced again to identify the parent colored 1.
3. The sibling’s far-child field is checked; when nonnull, the pointed-to child is colored 1.
4. After those color writes, the cursor’s parent field is reread for the helper argument.
5. The orientation-specific terminal helper receives the container and that newly read value.

Repair then ends with the final color-1 write to the current cursor.

These rereads and helper selections are established behavior. Dependency internals are unknown, but that does not make dispatch or argument-read timing unknown. Aliasing can also make preceding color writes affect later field reads.

### Detached-chain insertion and publication

After skipped or completed repair, the function:

1. Reads the current detached-chain head.
2. Writes that four-byte value to the detached node’s field at +12.
3. Writes the detached node’s address to the container’s field at +4.
4. Reads the four-byte count and writes the result of subtracting one.
5. Publishes the local result cell to the result location.

The detached node’s right-child field is thereby reused as a chain link. Its parent, left-child, and color fields are not generally cleared.

The count is represented by a four-byte signed operation. A representable result is the prior value minus one: zero becomes −1, rather than being rejected or saturated. Exact behavior at the signed minimum is not established by the supplied representation.

## Side effects

Direct effects include header updates, parent/child rewiring, byte-color writes, chain insertion, count decrement, and result publication. The leaf-link clear is explicitly four byte writes, not an established atomic four-byte update.

There is no explicit allocation, deallocation, payload access, or destructor invocation in the visible body. Chain insertion does not establish that a node is freed, reusable by a particular allocator, or transferred under any complete ownership policy.

Dependencies are not assumed to preserve globals, container state, exposed storage, or memory ownership. They may affect values subsequently reread.

Mutation precedes publication. A dependency that does not return normally, an invalid access, or nonterminating traversal has no visible rollback mechanism. No locking or concurrent-reader safety is established.

## Dependencies

| Opaque ID | Established invocation | Unresolved semantics |
|---|---|---|
| F-6183894e2484 | First call; receives addresses of the selection/result cell and header-snapshot cell; zero permits mutation | Comparison meaning, storage effects, ownership effects |
| F-b65c251036f5 | Called after a zero decision; receives the result-cell address | Result transformation and other effects |
| F-bd7d3aa9e3fb | Left-orientation red-sibling and terminal calls receive container plus reread parent; opposite-orientation near adjustment visibly receives container only | Complete ABI and structural effects |
| F-7e485a919ef1 | Opposite-orientation red-sibling and terminal calls receive container plus reread parent; left-orientation near adjustment visibly receives container only | Complete ABI and structural effects |

The repair helpers appear rotation-like. That interpretation does not establish their implementations or justify supplying an unshown argument.

## Acceptance examples

These are conditional static consequences, not executed tests.

### Bypass and selection reread

If F-6183894e2484 returns nonzero and the subsequent header read produces H, the output receives H. This body does not decrement the count or insert a detached node.

If instead it returns zero, and the selection location contains A initially but B after F-b65c251036f5, structural removal uses B. The output need not be A or B.

### Singleton

With null children, parent H equal to the current header, and no intervening alias-induced changes:

- Rightmost and leftmost become H, then root becomes null.
- Repair is skipped.
- The selected node is prepended to the detached chain.
- Count 1 becomes 0.
- The result cell supplies the output.

### Deeper successor

For a two-child selection whose deeper successor has a nonnull right child, that child replaces the successor at its former parent’s left link. The successor takes the selected node’s position, their color bytes are exchanged, and the original selected node is chained. No payload-copy requirement follows.

### Left-orientation red sibling

Suppose repair reaches a cursor equal to its parent’s left child and a right sibling colored 0. The sibling becomes 1 and the reread parent becomes 0 before F-bd7d3aa9e3fb receives the container and a freshly read parent-field value.

If that helper changes the cursor’s parent field to Q, the subsequent parent reload observes Q and reloads the sibling from Q’s right-child field, assuming valid memory. It does not necessarily reuse the original parent.

### Opposite-orientation near adjustment

Suppose the cursor does not equal its parent’s left child, the sibling is present, the both-children condition fails, and the sibling’s left child is absent or color 1. The near adjustment calls F-bd7d3aa9e3fb with only the container visibly supplied.

Afterward, the sibling is reloaded from the left-child field of the cursor’s reread parent. A null reload exits repair and colors the current cursor 1.

### Terminal argument timing

In a left-orientation terminal case, sibling-color copying, parent coloring, and any far-child coloring precede the parent-argument read for F-bd7d3aa9e3fb. If permitted aliasing changes the parent field during those writes, the call uses its later value, not an earlier parent snapshot. The opposite orientation has the same timing with F-7e485a919ef1.

## Uncertainties

The following remain **documented_only**:

- Dependency internals, complete calling conventions, and abnormal-return behavior.
- Formal iterator or handle semantics and whether the output denotes a successor.
- Valid color domain; 0-as-red and 1-as-black are inferred names.
- Payload ordering, balance guarantees, allocation, and reclamation policy.
- Permitted aliases, lifetime guarantees for exposed storage, and concurrency rules.
- Exact signed-boundary overflow behavior and target memory-fault behavior.
- Defined behavior for malformed trees, cycles, invalid addresses, and inconsistent counts.

## Evidence

**Established static evidence:** four-byte snapshots and fields; dependency-controlled bypass; selection rereading; structural traversal and rewiring; byte-color conditions; orientation-specific helper dispatch, visible argument counts, and parent/sibling reread order; detached-chain insertion; count decrement; final publication.

**Inferred intent:** tree erasure, cached extrema, red-black-style repair, rotation-like helpers, and possible iterator advancement.

**Unknown:** dependency implementations, complete ABI, ownership policy, and runtime preservation of ordering or balance.

No runtime tests have been performed. These conclusions are limited to the supplied static representation.
