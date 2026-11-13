<!-- Trusted provenance: F-8d7ff2efc6b8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8d7ff2efc6b8

## Purpose

F-8d7ff2efc6b8 conditionally removes a selected node from a pointer-linked, color-marked binary tree, prepends that node to a container-maintained chain, decrements a count, and writes a result word to caller-provided storage.

**Established behavior:** a preliminary dependency selects between a bypass path and a removal path. Removal can transplant another node, update cached extremes, change colors, and invoke structural dependencies. The removed node’s right-link field is subsequently reused as a chain link.

**Inferred intent:** balanced-tree erasure, advancement of a position-like result, and recycling of removed nodes. These interpretations do not establish ordering, balancing, ownership, or dependency guarantees beyond the visible operations.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

The function accepts three inputs and has no direct return value.

| Input | Representation | Role |
|---|---|---|
| Output address | Address of writable 32-bit storage | Receives the final result word |
| Container address | Address represented by a 32-bit value | Locates the header reference, chain head, and count |
| Selection address | Address of readable 32-bit storage | Supplies the initial candidate and, after dependency calls, the node to remove |

Stored links and result words are 32 bits in the supplied representation. Color accesses are one byte. No 64-bit pointer model is established.

### Container layout

Offsets are in bytes.

| Offset | Width | Established use |
|---|---:|---|
| +4 | 32 bits | Chain head |
| +16 | 32 bits | Header address |
| +20 | 32 bits | Count decremented on the removal path |

Other container bytes have no established meaning here.

### Header and node layout

| Offset | Width | Ordinary node | Header |
|---|---:|---|---|
| +0 | 8 bits | Color marker | No direct color interpretation established |
| +4 | 32 bits | Parent link | Root link |
| +8 | 32 bits | Left-child link | Cached left extreme |
| +12 | 32 bits | Right-child link | Cached right extreme |

Zero child links denote absence. The header is a distinct, non-null structural address under the ordinary valid-tree interpretation.

Removing a sole leaf whose parent is the header directly sets the root to zero and both cached extremes to the header.

The color predicates are not interchangeable:

- Some branches test for zero.
- The initial repair eligibility test accepts any nonzero removed-node color.
- Repair continuation and sibling-child tests distinguish exactly one from other byte values.

Calling zero “red” and one “black” is an interpretation. Other color values are not rejected.

### Preconditions and aliasing

Every accessed address must support the indicated read or write width. Traversals require terminating link chains. Structural removal and repair require sufficiently coherent links to make their dereferences valid.

There is no explicit validation of membership, tree ordering, cycles, count consistency, colors, alignment, or address bounds.

No general non-aliasing rule is established. The selection word is reread after dependencies execute; the output word is written last. Overlap can therefore affect observable results. Arbitrary overlap is not established as supported.

Descriptions of conventional tree outcomes below assume that earlier writes do not unexpectedly alias unrelated structural fields. The visible access ordering remains relevant when considering overlaps.

## Behavior

### Decision and result preparation

The function first reads the selection word into temporary result storage, then reads container +16 into separate temporary header storage. It passes writable addresses of both temporary words to **F-626327d2116a**.

A nonzero dependency result selects the bypass path. The function rereads container +16, replaces its temporary result with that fresh header value, writes the result to the output address, and returns.

On that path, this function performs no direct tree, chain, or count modifications. Effects of F-626327d2116a are not constrained by this observation.

A zero dependency result selects removal. The function calls **F-e6902067ce91** with the address of its temporary result word. Only after that call returns does it reread the selection word to obtain the removal target.

The target and output result are therefore distinct values with distinct histories:

- The removal target comes from the later selection read.
- The output comes from temporary result storage, potentially modified by dependencies.
- Removal does not recompute that result from the final tree.

### Structural case selection

The function examines the target’s left and right child links.

- With neither child present, it removes a leaf.
- With exactly one child present, that child replaces the target.
- With both children present, it follows left links starting from the right child until finding a node with no left child. That node is the structural successor.

No key comparison or payload inspection is shown. The successor interpretation depends on ordinary binary-tree ordering, which this function does not verify.

### Leaf removal

The target’s parent becomes the repair reference.

If that parent equals the current header, the function writes the header’s cached right extreme first, then its cached left extreme, then its root. The two extremes receive the header address; the root receives zero.

Otherwise, the parent’s left link is compared with the target:

- Equality selects the left link for clearing.
- Inequality selects the right link, without checking that the right link actually equals the target.

The selected link is cleared by four separate byte writes in increasing address order. These writes must not be described as an atomic 32-bit link replacement.

After clearing, the function examines the corresponding cached extreme: left for a cleared left link, right for a cleared right link. If that cache equals the target, it becomes the parent. The opposite cache is not updated by this branch.

The original target remains the node later prepended to the chain.

### One-child removal

The replacement child first receives the target’s parent link. The incoming link to the target is then redirected:

- If the current header’s root equals the target, the root becomes the replacement.
- Otherwise, the target’s parent’s left link is compared with the target. Equality selects that link; inequality selects the parent’s right link.

Cached-extreme updates follow incoming-link redirection and occur only when the respective cache equals the target.

For the cached left extreme:

- If the target’s right link is zero, the cache becomes the target’s parent.
- Otherwise, the function follows left links from the replacement to obtain the new cache.

For the cached right extreme:

- If the target’s left link is zero, the cache becomes the target’s parent.
- Otherwise, the function follows right links from the replacement to obtain the new cache.

These are the actual conditions, not a guarantee that both extremes are recomputed. Under a coherent ordinary tree, some combinations are unreachable; the body does not validate that coherence.

The replacement child becomes the repair reference. The original target remains the node selected for chain insertion.

### Two-child removal

The successor is moved into the target’s structural position. No payload copy is shown.

First, the target’s left child receives the successor as parent, and the successor receives the target’s left-child link.

The successor’s previous position is then handled:

- **Immediate right-child successor:** its right child, if present, receives the successor as parent.
- **Deeper successor:** its right child, if present, receives the successor’s former parent as parent. That former parent’s left link becomes the successor’s right child. The successor then receives the target’s right-child link, and that right child receives the successor as parent.

Next, the target’s incoming root or parent link is redirected to the successor. The successor then receives the target’s parent link.

Finally, the successor and target exchange their one-byte color values. The successor’s previous color is saved before the writes; the successor receives the target’s color before the target receives the saved color.

The repair reference is the successor’s former right child if that child is nonzero. Otherwise, the reference is the successor itself, including when the successor was deeper than the target’s immediate right child.

The original target, not the successor, remains selected for chain insertion. No direct cached-extreme update occurs in this branch.

### Repair eligibility and termination

After structural removal, the function rereads the header.

Repair is bypassed if the repair reference equals that header. Otherwise, repair is entered only when the removed target’s color is nonzero. Following two-child transplantation, this is the successor’s former color because of the exchange.

At each repair iteration, the function first compares the reference with the root obtained through the current header. If the reference is the root, or if its color is not exactly one, repair terminates by writing one to the reference’s color.

Consequently, a noncanonical nonzero color can permit entry into repair but immediately cause termination and normalization to one.

When iteration continues, the reference’s parent and its parent’s left link determine orientation. If the reference equals that left link, it takes the left-reference orientation. Otherwise, it takes the opposite orientation without independently verifying right-child membership.

| Orientation | Sibling | Near sibling child | Far sibling child | Parent restructuring dependency | Intermediate sibling dependency |
|---|---|---|---|---|---|
| Reference is left child | Parent’s right child | Sibling’s left child | Sibling’s right child | F-7265777c1e19 | F-726c5ad3d4e0 |
| Other orientation | Parent’s left child | Sibling’s right child | Sibling’s left child | F-726c5ad3d4e0 | F-7265777c1e19 |

### Repair transitions

**Sibling initially absent.** The function colors the current reference zero, moves the reference to the parent already obtained for that iteration, and repeats.

**Sibling color zero.** The sibling is colored one, the reference’s parent is colored zero, and the orientation’s parent restructuring dependency is called. The function then rereads the reference’s parent and obtains the sibling through that current parent.

If this reread sibling is absent, the function colors the reference zero, moves it to the reread parent, and repeats. This is an upward-propagation transition.

**Both sibling children absent or exactly one.** With a sibling present, the function checks whether the near child is absent or colored exactly one and, if so, whether the far child is absent or colored exactly one. If both conditions hold, the sibling is colored zero. The reference becomes its currently read parent, and iteration repeats.

**Intermediate sibling restructuring.** If upward propagation was not selected and the far child is absent or colored exactly one, the near child, when present, is colored one. The sibling is then colored zero, and the orientation’s intermediate sibling dependency is called.

After that call, the sibling is reread through the reference’s **current parent**:

- For a left-child reference, after **F-726c5ad3d4e0**, the function rereads the current parent’s right link.
- For the opposite orientation, after **F-7265777c1e19**, it rereads the current parent’s left link.

If this sibling is zero, repair **terminates immediately by coloring the current reference one**. It does not color the reference zero, move to the parent, or execute the final restructuring call.

This post-intermediate-call null transition is distinct from the two upward-propagating missing-sibling cases. Unknown dependency effects make it an express part of the contract.

**Final restructuring.** If the intermediate step is unnecessary, or it returns with a nonzero reread sibling, the sibling receives the reference’s current parent’s color. The parent is then colored one, followed by the far child when present. The orientation’s parent restructuring dependency is called. After it returns, the current reference is colored one and repair ends.

A helper call is not assumed to preserve the parent link, sibling links, header reference, or other externally reachable state.

### Chain insertion, count, and output

After structural processing and any repair, the function:

1. Reads container +4 and writes that word into the removed target’s +12 field.
2. Writes the removed target’s address to container +4.
3. Reads the 32-bit count at container +20, subtracts one, and writes the result back.
4. Writes the temporary result word to the output address.

The output store is the final direct write on normal completion.

There is no zero-count guard. The count update has 32-bit storage, but portable signed-overflow semantics are not established by the supplied representation.

## Side effects

Direct writes on the removal path can affect:

- Header root and cached extremes.
- Parent and child links in the target, successor, replacement, and surrounding nodes.
- One-byte color fields encountered during transplantation and repair.
- The removed target’s +12 field and container +4 during chain insertion.
- Container +20 during decrement.
- Caller-provided output storage.

The function does not directly allocate, deallocate, destroy payloads, or copy payloads. This does not constrain dependency behavior.

Chain insertion establishes a structural linkage, not a complete ownership contract. Whether the container owns, reuses, or eventually releases the node is **documented_only**.

There is no local rollback, explicit error status, exception handling, or recovery path. Invalid memory, inconsistent links, nonterminating traversal, or dependency failure can prevent completion after partial modification.

No synchronization or concurrent-mutation guarantee is established. Header and link rereads must not be replaced with an assumption of one immutable snapshot.

## Dependencies

### F-626327d2116a

Receives writable addresses of two temporary 32-bit words, initialized from the selection and container header respectively. Its return is tested only for zero versus nonzero.

An interpretation as a comparison or end-position check is inferred. Its mutations, global effects, ownership effects, and failure behavior are not established.

### F-e6902067ce91

Called only after a zero decision. Receives the writable address of the temporary result word.

Advancing a position is inferred intent. The visible contract establishes the call before the selection reread and the eventual use of the temporary result as output. Preservation of the selection, container, globals, or memory ownership is not assumed.

### F-7265777c1e19 and F-726c5ad3d4e0

These dependencies occupy mirrored structural-repair positions, consistent with opposite rotations.

The supplied calls show:

- A container and explicit parent-node argument for parent restructuring.
- Only the container as an explicit argument for intermediate sibling restructuring.

Complete signatures and effects are **documented_only**. A conventional rotation API must not be supplied by assumption.

In particular, either intermediate dependency may leave the subsequently read sibling null. The caller has an explicit terminating transition for that result in both orientations.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume valid accessed memory and normally returning dependencies.

1. **Nonzero decision with changed header.** If F-626327d2116a changes container +16 and returns nonzero, output receives the changed header value. F-e6902067ce91, direct removal, chain insertion, and count decrement do not execute.

2. **Selection changes before removal.** If the zero-decision path reaches F-e6902067ce91 and the selection word changes before the later selection read, removal uses the later value. The output remains determined separately by temporary result storage.

3. **Sole-node removal.** For a leaf whose parent is the header, the direct structural writes set cached right extreme to header, cached left extreme to header, and root to zero, in that order. The target is then prepended and the count decremented once, provided execution completes without interfering alias effects.

4. **Non-root left leaf.** The parent’s left link is cleared using four ascending-address byte writes. If the cached left extreme equals the target, it becomes the parent before repair. The cached right extreme is not directly changed by this leaf branch.

5. **Deeper successor.** The successor’s former parent receives the successor’s right child as its left child. The successor takes the target’s structural position and exchanges colors with it. The original target is prepended to the chain.

6. **Sibling absent at iteration entry.** If iterative repair finds no sibling, it colors the reference zero and continues from the parent. It does not take the post-intermediate termination transition.

7. **Left-reference intermediate call yields no sibling.** Suppose repair reaches F-726c5ad3d4e0 as the intermediate dependency for a left-child reference. If the subsequent read through the reference’s current parent yields a zero right link, the reference is colored one and repair ends. F-7265777c1e19 is not subsequently called as final restructuring for that iteration. Chain insertion, decrement, and output still follow.

8. **Opposite intermediate call yields no sibling.** Suppose repair reaches F-7265777c1e19 as the intermediate dependency in the opposite orientation. If the current parent’s subsequently read left link is zero, the reference is colored one and repair ends. No parentward propagation or final F-726c5ad3d4e0 call occurs for that iteration.

9. **Output aliases count.** If output is container +20 and execution otherwise completes validly, the decrement occurs before the output store. The final output store determines that word’s final contents.

10. **Zero count.** A starting count of zero does not select an error branch. Subtraction is attempted. A machine-level modulo decrement would produce all ones, but portable wraparound is not established here.

## Uncertainties

The following remain **documented_only**:

- Original public types, calling convention, alignment requirements, and permitted aliases.
- Exact meanings of the decision and result-preparation dependencies.
- Complete dependency signatures, global effects, memory-lifetime effects, and failure behavior.
- Whether helpers preserve any container, header, parent, sibling, or payload state.
- Payload layout and ownership lifecycle.
- Whether inconsistent trees or noncanonical colors are permitted inputs.
- Portable arithmetic behavior at signed 32-bit boundaries.

Address calculations use fixed byte offsets in a 32-bit representation without visible bounds or overflow checks. No broader portability guarantee follows.

The possibility of a null sibling after intermediate restructuring is **not** unresolved: the caller explicitly handles it by coloring the current reference one and ending repair. What remains unknown is which dependency behaviors can produce that state.

## Evidence

The supplied function body and opaque dependency mapping establish the access widths, field offsets, branch predicates, traversal directions, direct write order, dependency-call positions, chain insertion, count update, and final output store described here.

Both intermediate restructuring paths contain a sibling reread through the reference’s current parent. A null result reaches the common reference-coloring termination point, rather than the upward-propagation transition used for other missing-sibling cases.

Balanced-tree erasure, iterator advancement, rotations, and recycling are inferred interpretations, not independently established dependency contracts.

No runtime execution, dynamic inspection, external dependency inspection, or runtime tests have been performed.
