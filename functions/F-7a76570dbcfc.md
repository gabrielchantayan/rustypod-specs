<!-- Trusted provenance: F-7a76570dbcfc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7a76570dbcfc

## Purpose

F-7a76570dbcfc attaches a newly obtained node to a caller-selected position in a pointer-linked tree, updates tree bookkeeping, and performs color-based insertion repair. On normal completion, it publishes the newly obtained node through a caller-supplied output location.

The visible structure is consistent with red-black tree insertion using a header node that stores the root and both endpoint nodes. That interpretation is **inferred intent**. The established contract is the specific field accesses, dependency calls, link updates, and color-byte changes described below. This function does not visibly search for an insertion position, reject duplicates, or validate the caller’s selected parent.

## Interface

### Inputs and output

The function has five arguments, in this order:

1. **Result location:** address of a writable 32-bit slot. On normal completion, this receives the new node’s address.
2. **Container address:** a 32-bit address identifying mutable container state.
3. **Placement override:** a 32-bit integer tested only for zero versus nonzero when choosing the attachment side. Its original value may also be passed as an additional argument to a repair dependency.
4. **Attachment parent:** a 32-bit node address. Depending on the container’s current header pointer, this may designate either an ordinary node or the header.
5. **Payload source:** address of at least two readable 32-bit words.

No direct return value is shown. The result is delivered by the final write to the result location.

### Visible layouts

All offsets below are byte offsets. Addresses and link fields are represented as 32-bit quantities in the supplied representation.

| Object | Offset | Visible use |
|---|---:|---|
| Container | +16 | Header-node pointer |
| Container | +20 | 32-bit node-count field |
| Container | +25 | Address supplied as the first argument to the comparison dependency |
| Node | +0 | One-byte color or repair-state field |
| Node | +4 | Parent pointer; in the header, used as the root pointer |
| Node | +8 | Left-child pointer; in the header, used as the left-endpoint pointer |
| Node | +12 | Right-child pointer; in the header, used as the right-endpoint pointer |
| Node | +16 | First 32-bit payload word |
| Node | +20 | Second 32-bit payload word |

The meanings “root,” “left endpoint,” and “right endpoint” follow the observed accesses and attachment behavior. A complete container or node type, including padding and other fields, is not established.

### Preconditions for ordinary successful use

The visible implementation relies on, rather than checks, the following conditions:

- The container fields and result location are accessible with the required read/write permissions.
- The payload source provides the two words needed for copying.
- The node-producing dependency returns storage suitable for subsequent node accesses.
- The header, attachment parent, and repair traversal links form an appropriate tree structure.
- The selected attachment slot may be overwritten without violating the caller’s intended tree structure.
- Parent and grandparent nodes needed during repair are accessible.

The function does not establish these conditions itself. In particular, it does not check that the selected child slot is empty.

## Behavior

### 1. Obtain a node and copy the payload

The first operation is a call to **F-1c32ea4cd057**, passing the container address. Its returned address becomes the new node address retained for final publication.

The function then tests whether the address at new-node offset +16 is nonzero. This is **not** an ordinary test that the returned node address itself is nonzero.

If that test succeeds, it performs these accesses in order:

1. Read the first 32-bit word from the payload source.
2. Write that word to new-node offset +16.
3. Read the second 32-bit word from the payload source at offset +4.
4. Write that word to new-node offset +20.

This is two ordered word transfers, not an established overlap-safe copy operation. If the payload source aliases storage modified by the first transfer, the second read observes memory after that first transfer.

If the address test fails, both payload transfers are skipped. Subsequent count, attachment, and node-field operations are still attempted. There is no corresponding failure return.

The function does not visibly initialize the new node’s color or child pointers before repair. Any required initial values must arise from the dependency or from another precondition; they must not be attributed to this function.

### 2. Increment the count

After the payload-copy stage, the function reads the container’s 32-bit field at +20, adds one, and stores the result back to that field.

This update occurs:

- before comparison;
- before attachment;
- before repair;
- before writing the result location.

No overflow check, capacity check, or rollback is visible. The source representation uses a signed 32-bit integer operation; exact behavior at signed overflow is not established by this evidence. No widened count calculation is shown.

### 3. Choose the attachment side

The function selects the left-attachment path if either condition holds:

- The container’s current header pointer equals the attachment parent.
- The placement override is nonzero.

The header-equality test is evaluated first. When either condition selects this path, the comparison dependency is not called.

Otherwise, the function calls **F-4b52c8e01045** with:

1. The container address plus 25.
2. The original payload-source address.
3. The attachment-parent address plus 16.

A nonzero result selects left attachment. A zero result selects right attachment. No finer interpretation of the return value is visible.

The comparison uses the original payload source, not the payload copy in the new node. Consequently, changes to the source before or during this call can matter.

For calls reaching this comparison stage, two auxiliary values later supplied on one repair-call path are set to zero. On paths that bypass comparison, those values retain the original placement override and attachment-parent address.

### 4. Attach and update header links

#### Left attachment

The function first writes the new node address to the attachment parent’s +8 field.

It then rereads the container’s header pointer:

- If this header differs from the attachment parent, the header’s +8 field is replaced with the new node address **only if** that field currently equals the attachment parent.
- If this header equals the attachment parent, the function writes the new node address to the header’s +4 field, then rereads the container’s header pointer and writes the new node address to that header’s +12 field.

Thus, in the ordinary empty-tree case where the attachment parent is the header, the header’s left endpoint, root, and right endpoint all become the new node.

These decisions use reads made at the attachment stage. They are not guaranteed to use the header value observed before a dependency call.

#### Right attachment

The function writes the new node address to the attachment parent’s +12 field.

It then rereads the container’s header pointer. The header’s +12 field is replaced with the new node address only if it currently equals the attachment parent.

#### Parent assignment

After either attachment path, the function writes the attachment-parent address to new-node offset +4.

There is no visible preservation of an overwritten child link and no detachment of any previous child.

### 5. Repair color and linkage relationships

The new node is initially the current repair node.

At each loop entry, the function rereads the container’s header pointer and that header’s +4 root pointer. Repair stops if the current repair node equals that root.

Otherwise, the function reads the current node’s parent pointer and the parent’s first byte. If that byte is nonzero, repair stops. If it is zero, repair continues through the parent’s parent pointer.

Only zero versus nonzero is tested for existing colors. The function writes exact byte values of zero and one.

#### Parent is the grandparent’s left child

The grandparent’s right child is treated as the opposite-side node.

If that node exists and its first byte is zero:

1. Write one to the parent’s first byte.
2. Write one to the opposite-side node’s first byte.
3. Resolve the current node’s parent and that parent’s parent, and write zero to the latter node’s first byte.
4. Resolve those parent links again and make that ancestor the current repair node.
5. Repeat the loop.

If the opposite-side node is absent or has a nonzero first byte:

- When the current node is the parent’s right child, call **F-afe6bf6f53c9** with the container and that parent. After the call, make the previously selected parent the current repair node.
- Through freshly read parent links, write one to the current node’s parent color and zero to that parent’s parent color.
- Call **F-051feb68d086** with the container and the ancestor obtained through the current node’s parent chain.

#### Parent is not the grandparent’s left child

The grandparent’s left child is treated as the opposite-side node. The implementation does not separately verify that the parent is the grandparent’s right child.

If the opposite-side node exists and its first byte is zero, the same recoloring and upward continuation described above occurs.

Otherwise:

- When the current node is the parent’s left child, call **F-051feb68d086** using the container and parent, plus additional visible arguments described under Dependencies. After the call, make the previously selected parent the current repair node.
- Through freshly read parent links, write one to the current node’s parent color and zero to that parent’s parent color.
- Call **F-afe6bf6f53c9** with the container and the ancestor obtained through the current node’s parent chain.

The repair dependencies’ apparent rotation roles are inferred from these call sites. Their actual mutations are not available here. Post-call parent reads must not be replaced with assumptions that links remain unchanged.

### 6. Finalize and publish

After repair stops, the function rereads the container’s header pointer and the header’s root pointer. It writes one to the first byte of that root node.

Only after this write does it store the originally obtained new node address into the result location.

If the result location aliases another modified field, this final store may overwrite that field. No non-aliasing check is shown.

## Side effects

Visible direct effects include:

- Up to two 32-bit payload writes.
- A 32-bit count increment.
- Child-link attachment.
- Conditional header endpoint updates.
- Root-link assignment in the header-parent case.
- Assignment of the new node’s parent.
- One-byte color updates during repair and at completion.
- Final publication of the new node address.

Dependency calls may introduce additional effects. Their allocation behavior, global-state access, memory ownership, callbacks, and failure behavior are unknown.

There is no visible deallocation, payload-source release, error code, exception handling, or rollback. This does not establish that dependencies cannot allocate, release memory, or fail.

The count and structural changes precede publication. Therefore, interruption or failure inside a later dependency can occur after earlier state changes have already happened.

## Dependencies

| Opaque ID | Established call-site contract | Unknown or inferred aspects |
|---|---|---|
| **F-1c32ea4cd057** | Called first with the container; returns the address used as the new node. | Allocation, initialization, ownership, failure signaling, and container mutations are unknown. |
| **F-4b52c8e01045** | Conditionally receives container +25, payload source, and attachment parent +16. Zero selects right; nonzero selects left. | Comparison intent is inferred. Ordering rules, statefulness, and side effects are unknown. |
| **F-afe6bf6f53c9** | Receives the container and a node during repair. No returned value is consumed. | Its placement is consistent with one rotation direction; exact behavior is unknown. |
| **F-051feb68d086** | Receives the container and a node during repair. One visible call supplies additional arguments. No returned value is consumed. | Its placement is consistent with the opposite rotation direction. Its actual signature and treatment of additional arguments are unknown. |

The additional visible arguments to one **F-051feb68d086** call are:

- The opposite-side node address.
- A 32-bit value that retains the grandparent address when that opposite-side node is absent, or contains the opposite-side node’s color byte when it exists and is nonzero.
- The original placement override and attachment-parent address if comparison was bypassed; otherwise, two zeros.

These arguments must not be assumed irrelevant without the dependency’s contract or calling-convention evidence.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests.

### Header-parent insertion

Given valid storage, a stable header pointer equal to the supplied attachment parent, and a node-producing dependency that supplies a suitable node:

- Comparison is skipped, regardless of the placement override.
- The payload transfers precede the count increment.
- The header’s left endpoint, root, and right endpoint become the new node.
- The new node’s parent becomes the header.
- Repair immediately recognizes the new node as root.
- Its first byte becomes one.
- The result slot receives its address last.

### Explicit left placement

Given a non-header parent and a nonzero placement override:

- Comparison is skipped.
- The parent’s left link is overwritten.
- The header’s left endpoint changes only if it equals that parent.
- The right endpoint is not directly updated by this attachment branch.

### Comparison-selected right placement

Given a non-header parent, a zero placement override, and a zero comparison result:

- Comparison receives the original payload source.
- The parent’s right link becomes the new node.
- The header’s right endpoint changes only if it equals that parent.
- The auxiliary placement values for the exceptional repair call are both zero.

### Parent already nonzero-colored

Given a newly attached non-root node whose parent’s first byte is nonzero:

- That loop iteration performs no repair dependency call.
- Finalization still writes one to the current root’s first byte.
- The result slot is then written.

### Zero-colored opposite-side node

Given a non-root current node, a zero-colored parent, and an existing zero-colored opposite-side node:

- Parent and opposite-side colors become one.
- The ancestor reached through the current node’s two parent links becomes zero.
- Repair continues upward without a rotation dependency call in that iteration.

### Null result from the node-producing dependency

A null returned node address is not converted into a clean failure result. The visible payload guard does not directly reject null, and subsequent node-field accesses remain. Behavior requires additional memory and address-semantics assumptions; no successful null-return case is established.

## Uncertainties

The following remain **documented_only**:

- Complete dependency signatures, particularly the additional repair-call arguments.
- Initial new-node color and child-link values.
- Allocation ownership and lifetime obligations.
- Comparator ordering, duplicate-key policy, and valid insertion-position rules.
- Whether dependencies preserve container fields, globals, or payload memory.
- Signed count overflow and invalid-address arithmetic behavior.
- Alignment, byte order, synchronization, and concurrency requirements.
- Behavior for malformed trees, cyclic links, missing ancestors, invalid pointers, or dependency failure.

Red-black balancing, rotation direction, and endpoint semantics are inferred structural interpretations, not independently verified dependency contracts.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established observations include field offsets, access widths, branch conditions, dependency arguments, explicit writes, and the ordering of result publication.

No implementation execution, runtime tests, dependency inspection, or independent layout validation has been performed. The acceptance examples specify conditional observable behavior; they are not reports of successful runs.
