<!-- Trusted provenance: F-d03fddffbc6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d03fddffbc6d

## Purpose

F-d03fddffbc6d conditionally removes a selected node from a pointer-linked structure, updates structural links and boundary references, performs color-dependent repair, and prepends the detached node to a container-maintained chain. It publishes a four-byte output and has no explicit return value.

**Established static behavior:** the operation directly manipulates four-byte references, one-byte colors, a root reference, two boundary caches, a chain head, and a signed four-byte count.

**Inferred intent:** tree erasure with red-black-style repair and retention of detached storage for reuse. Neither balancing correctness nor the chain’s ownership policy is established. The repair positions used for a leaf and for a successor without a right child must not be replaced with conventional null-child deletion semantics.

The output is separate from the identity of the detached node. On removal, it comes from an address-exposed local selection slot whose final contents may depend on dependencies.

## Interface

The operation accepts three inputs:

| Input | Required storage | Role |
|---|---|---|
| Output destination | Four writable bytes | Receives the final output value |
| Container reference | Accessible fields through offset 0x17 | Provides the header, detached-node chain, and count |
| Selection reference | Four readable bytes | Provides an initial selection and, on removal, a separately reread target |

References occupy four bytes in the supplied representation. This is not a portable native-pointer interface.

### Container layout

| Offset | Width | Interpretation |
|---|---:|---|
| 0x04 | 4 bytes | Detached-node chain head |
| 0x10 | 4 bytes | Header reference |
| 0x14 | 4 bytes | Signed integer count |

No direct accesses to the remaining container offsets are established.

### Node and header layout

| Offset | Width | Ordinary node | Header |
|---|---:|---|---|
| 0x00 | 1 byte | Color/state | No designated color meaning established |
| 0x04 | 4 bytes | Parent | Root |
| 0x08 | 4 bytes | Left child | Cached minimum |
| 0x0c | 4 bytes | Right child | Cached maximum |

A zero child reference denotes absence. Header storage must be accessible wherever it is dereferenced.

Color 0 has the role conventionally called red; color 1 has the role conventionally called black. Other values are not rejected. Any nonzero detached-node color permits entry into repair, but a repair position must have color exactly 1 for an iteration to continue.

### Preconditions and aliasing

There is no direct validation of membership, addresses, structural consistency, count range, or exclusive access. Every reached address must support the access performed at that point. Traversals must terminate.

The operation does not establish general alias safety. Selection storage, output storage, container fields, and node fields are not checked for overlap. Earlier writes can therefore affect later reads. Structural descriptions below identify intended field roles, not a guarantee that those roles occupy distinct storage.

Dependency calls may alter referenced memory or retain exposed local addresses. No preservation, allocation, release, ownership, or normal-return guarantee is inferred for an unknown dependency.

For the signed count decrement to remain within its representable range, the count read at that operation must exceed −2,147,483,648. Positivity may be appropriate for an abstract container invariant, but it is not checked and is not required merely to make decrementing zero arithmetically valid.

## Behavior

### Initial decision and output selection

The operation first reads the selection input into a four-byte local selection slot. It then reads the container’s header reference into a separate four-byte local header slot. F-54610eed0112 receives the addresses of those slots, selection first and header second.

A nonzero result selects the non-removal path. The container header is read again, assigned directly to the local selection slot, and published to the output. No further dependency call intervenes. This path performs no direct unlinking, color repair, chain insertion, or count decrement.

A zero result selects removal. F-0ff2c000bc5c receives the local selection-slot address. After that call returns, the operation rereads the original selection input to identify the **target** node.

Consequently:

- The target is not necessarily the selection value read before either dependency.
- The target is not necessarily the value left in the local selection slot.
- The eventual output is not necessarily the detached node.

On the removal path there is no subsequent direct assignment to the local selection slot. That does not establish preservation: its address has escaped to dependencies, and later calls may affect it through retained references. Publication uses its contents at the final assignment.

### Structural selection

The target’s left child is examined first.

| Target children | Structural action | Initial repair position |
|---|---|---|
| Neither child exists | Detach a leaf | Target’s parent |
| Only right child exists | Replace target with right child | Right child |
| Only left child exists | Replace target with left child | Left child |
| Both children exist | Relocate the leftmost node of the right subtree | Successor’s right child, or successor itself if that child is absent |

The leftmost node of the target’s right subtree is called the **successor** here. This is structural terminology; no key comparison is performed.

Some child fields are read more than once during branch selection and traversal. These reads are not a single atomic snapshot. The direct structural work performs no payload copying.

### Leaf removal

For a leaf, the target’s parent becomes the repair position.

If that parent equals the current container header, the operation writes the header’s cached maximum first, cached minimum second, and root third. The two caches receive the parent/header reference; the root receives zero.

Otherwise, the parent’s left link is read and compared with the target. Equality selects the left incoming link; inequality selects the right incoming link without separately validating it.

The selected incoming link is cleared through four one-byte writes in increasing address order. This is not represented as one four-byte store.

The same side decision determines the only boundary cache considered by this branch:

- Left-link removal considers cached minimum.
- Right-link removal considers cached maximum.

If that cache currently equals the target, it is replaced with the parent. The opposite cache is not directly updated in this branch.

### Replacement by a sole child

The replacement’s parent is written from the target’s parent before the incoming link is changed.

The current header’s root is then inspected. If it equals the target, it is replaced with the child. Otherwise, the target parent’s left link is inspected: equality with the target selects that link; inequality selects the parent’s right link.

Boundary processing follows in minimum-then-maximum order. Each cache changes only if its current value equals the target.

For cached minimum:

- If the target’s right child is absent, use the target’s parent.
- Otherwise, use the leftmost node reached from the replacement.

For cached maximum:

- If the target’s left child is absent, use the target’s parent.
- Otherwise, use the rightmost node reached from the replacement.

The child remains the repair position. The original target remains the detached node later inserted into the chain.

### Relocation of a distinct successor

With two children, the successor moves into the target’s structural position. The original target is detached; the operation does not substitute payloads.

The first structural changes attach the target’s left subtree to the successor: the left child’s parent is written before the successor’s left link.

Handling of the successor’s former position depends on its relationship to the target:

- **Immediate right child:** if the successor has a right child, that child’s parent becomes the successor.
- **Deeper successor:** an existing successor right child receives the successor’s former parent. That former parent’s left link is then replaced with the child, including zero when absent. The successor receives the target’s right subtree, and that subtree’s parent becomes the successor.

Next, the target’s incoming root or parent link is replaced with the successor, using the same root-first and otherwise-left-versus-right selection described above. The successor’s parent then receives the target’s parent.

The successor’s color is retained, the target’s color is written into the successor, and the retained color is written into the target. Thus the detached target carries the successor’s former color for repair-entry purposes.

The successor’s original right child is the repair position when present. When absent, the successor itself is the repair position. No direct minimum- or maximum-cache update occurs in this branch.

### Color-dependent repair

Repair entry requires both:

- The repair position differs from the current header.
- The detached target’s color is nonzero.

The header comparison precedes the detached-color read. If entry is rejected, even the terminal repair-position color write is skipped.

Within repair, each iteration first compares the position with the current root. Equality ends iteration without requiring a position-color read. Otherwise, only position color exactly 1 permits the iteration to continue. Either exit leads to coloring the position 1.

For a continuing iteration, equality between the position and its parent’s left link selects the left-side case. Otherwise, the right-side case applies.

| Case | Sibling | Inner sibling child | Outer sibling child |
|---|---|---|---|
| Left-side position | Parent’s right child | Sibling’s left child | Sibling’s right child |
| Right-side position | Parent’s left child | Sibling’s right child | Sibling’s left child |

The repair decisions have the following precedence:

**Absent sibling.** The position is colored 0, the repair position advances to the parent, and iteration repeats. No restructuring dependency is called for this step.

**Sibling color exactly 0.** The sibling becomes 1 and the parent becomes 0 before the red-sibling dependency call. After return, the parent is reread through the repair position, and the sibling is reloaded. If now absent, the absent-sibling transition uses that reread parent.

**Both sibling children absent or exactly color 1.** The sibling becomes 0 and the position advances to its parent. The inner child is examined first. The outer-child check for this case is reached only if the inner child is absent or exactly 1.

**Outer child absent or exactly color 1, after the preceding case failed.** An existing inner child becomes 1, and the sibling becomes 0. The inner-child restructuring dependency is then called. The sibling is reloaded through the repair position’s parent. If absent after return, repair ends by coloring the position 1.

**Final redistribution.** The sibling receives the currently read parent color. The parent becomes 1, followed by the outer sibling child when present. The final-redistribution dependency is called. On return, the repair position becomes 1 and repair finishes.

The observed dependency mapping is:

| Repair side | Red sibling | Inner-child restructuring | Final redistribution |
|---|---|---|---|
| Left | F-1476b82dc9cd: container and then-read parent | F-cfe689dcfe89: container only | F-1476b82dc9cd: container and then-read parent |
| Right | F-cfe689dcfe89: container and then-read parent | F-1476b82dc9cd: container only | F-cfe689dcfe89: container and then-read parent |

These describe observed call-site arguments, not complete callable signatures. The contract does not require either dependency to be a conventional rotation.

Repeated reads matter. In particular, an outer child may be examined again after the both-children case fails, and parent references are reread for later color writes and dependency arguments. Earlier observations must not be substituted for these later accesses.

### Chain insertion and publication

After structural processing and any repair, the operation:

1. Reads the current detached-node chain head and writes it into the detached target’s right-link field.
2. Writes the detached target into the container’s chain-head field.
3. Reads the signed four-byte count, subtracts one, and stores the result in that field.
4. Publishes the final four-byte contents of the local selection slot.

The target’s parent and left link are not directly cleared during chain insertion. Its right link becomes the chain link. Its color may reflect its original color, the successor-color exchange, or a repair write.

## Side effects

Direct writes affect structural links, root and boundary caches, color bytes, the detached-node chain, the signed count, and output storage.

The operation is not transactional. Structural mutation precedes repair dependencies. Chain insertion precedes count arithmetic, which precedes output publication. No rollback or explicit error result is shown.

No direct allocation, release, payload destruction, or payload copying appears. Chain insertion suggests retained storage, but does not establish ownership transfer or reclamation policy. Unknown dependencies may have additional effects.

### Signed count arithmetic

The count expression performs a **signed four-byte integer decrement**. For a representable input greater than −2,147,483,648, the mathematical result remains representable.

In particular:

- 1 becomes 0.
- 0 becomes −1; this is not signed overflow.
- −2,147,483,647 becomes −2,147,483,648.
- Decrementing −2,147,483,648 requires −2,147,483,649, outside the signed 32-bit range.

The source does not establish defined wrapping behavior at that minimum boundary. Successful continuation, a particular stored value, and final output publication cannot be guaranteed there from the supplied signed expression.

An explicitly separate, conditional **32-bit modular model** would reduce subtraction modulo 4,294,967,296. In that model, the zero word decrements to the all-ones word, and the word corresponding to the signed minimum decrements to 0x7fffffff. This model is not evidence that the supplied signed operation wraps. Likewise, interpreting −1 as an all-ones stored representation requires the corresponding signed representation assumption.

Container-level validity and arithmetic validity are distinct: decrementing a zero count may violate an intended size invariant, but the signed subtraction itself produces −1 without overflow.

Four-byte accesses are not established as atomic. The individual byte stores used for leaf-link clearing can expose intermediate representations if another observer can access the structure.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-54610eed0112 | Receives local selection-slot and header-slot addresses; integer result selects removal or non-removal | Comparison meaning, mutations, address retention, broader effects |
| F-0ff2c000bc5c | Receives the local selection-slot address on removal | Selection transformation, retained address, effects on input or container storage |
| F-1476b82dc9cd | Called during repair as specified above | Complete signature, structural effects, other mutations, failure behavior |
| F-cfe689dcfe89 | Called during repair as specified above | Complete signature, structural effects, other mutations, failure behavior |

Different observed argument counts do not establish complete signatures.

No dependency is assumed to preserve globals, exposed local storage, node storage, or container fields. If a call does not return normally, subsequent direct behavior is not established to occur.

## Acceptance examples

These are conditional static obligations, not runtime results. Unless stated otherwise, examples assume valid, nonoverlapping relevant storage and dependencies that return without interfering with the stated scenario.

### Non-removal after a header change

If F-54610eed0112 changes the container header and returns nonzero, output receives the subsequently reread header. It does not receive the initial header merely because that value was passed by local-slot address. F-0ff2c000bc5c is not called, and no direct count decrement occurs.

### Isolated root leaf

If the target is a leaf whose parent equals the header, cached maximum and cached minimum become the header, in that order, before root becomes zero. Repair is skipped because the repair position is the header. The target is then chained and the count decremented.

### Sole right child at the minimum

If the target has only a right child and is the cached minimum, the child receives the target’s parent and replaces the target’s incoming structural link. The minimum becomes the replacement’s leftmost descendant. The original target is chained.

### Deep successor with a right child

If a two-child target has a deeper successor with a right child, that child replaces the successor in its former parent’s left link. The successor moves into the target’s position and exchanges colors with it. The child is the repair position; the original target is detached.

### Successor without a right child

If the successor’s right child is absent, the successor itself becomes the repair position. Substituting a null repair position would not satisfy the observed behavior.

### Left-side red sibling

For a continuing left-side iteration with a color-0 sibling, the sibling becomes 1 and the parent becomes 0 before F-1476b82dc9cd receives the container and then-read parent. Parent and sibling are reloaded after return.

### Later output-slot mutation

If an initial dependency retains the local selection-slot address and a later repair dependency changes its contents, final output receives the later value, provided publication is reached without another intervening modification.

### Zero count

If the count read during finalization is zero, signed decrement produces −1 without overflow. Under otherwise valid continuation, that result is stored before output publication. This does not establish that removing from a logically empty container is supported.

### Signed-minimum count

If the count is −2,147,483,648 at the decrement, the signed result is outside range. The contract does not require wrapping to 2,147,483,647 or guarantee subsequent publication. Chain insertion already precedes this arithmetic boundary.

## Uncertainties

The following remain **documented_only**:

- Complete dependency semantics, signatures, retained-address use, and failure behavior.
- Meaning and ownership of the selection-like output.
- Detached-chain purpose, ownership transfer, reclamation, and payload lifetime.
- Full balancing correctness and treatment of noncanonical colors.
- Supported aliasing, concurrency, synchronization, and access atomicity.
- Whether the abstract container requires a strictly positive count before removal.
- Execution behavior at signed-minimum count overflow; wrapping is not established.
- Endianness and the representation-level interpretation of negative counts.
- Invalid-structure handling, exceptions, signals, and recovery.

The signed nature of the supplied count decrement and the validity of zero-to-minus-one subtraction are established, not unresolved.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent behavioral evidence.

Static evidence establishes field offsets and widths, signed count arithmetic, branch order, traversal directions, repeated reads, direct write order, address-exposed local slots, observed dependency arguments, chain insertion, and final output publication.

Tree terminology, red-black color names, rotation-like intent, and reuse intent are interpretations rather than verified dependency semantics.

No dependency bodies or runtime traces were supplied. No runtime tests have been performed.
