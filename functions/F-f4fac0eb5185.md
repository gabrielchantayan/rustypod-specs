<!-- Trusted provenance: F-f4fac0eb5185; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f4fac0eb5185

## Purpose

F-f4fac0eb5185 creates and links a node into a tree-like container, performs color-based rebalancing, and returns the created node through an output location.

The supplied static evidence establishes node linking, endpoint maintenance, a count increment, recoloring, and calls to structural helpers. The organization strongly suggests red-black-tree insertion. That interpretation is inferred intent, not proof that arbitrary inputs produce a valid red-black tree.

This is a placement-oriented operation, not an established search-and-insert API. The caller supplies an attachment node and a placement selector. The function performs at most one visible comparison before linking; it does not search for a location or reject duplicates.

## Interface

### Inputs and output

The function has five inputs, in this order:

| Position | Contract role | Visible representation |
|---|---|---|
| 1 | Output location for the created node address | Pointer to writable 4-byte storage |
| 2 | Container address | Address-bearing 32-bit value |
| 3 | Placement selector | 32-bit integer, tested only for zero versus nonzero |
| 4 | Attachment node or container header address | Address-bearing 32-bit value |
| 5 | Payload source | Address of two consecutively accessed 4-byte words |

No meaningful direct return value is established. On normal completion, the function writes the created node address to the first input’s storage.

The visible representation uses 4-byte words for stored addresses, counters, and payload components. Color fields are accessed as single bytes. Wider native-pointer behavior is not established.

### Container layout

The following offsets are relative to the container address:

| Offset | Accessed content |
|---|---|
| +16 | 4-byte header address |
| +20 | 4-byte node count |
| +25 | Address passed as comparison context |

The comparison context’s size and internal layout are unknown. The function supplies its address to a dependency; it does not directly read its contents.

### Header layout

The object addressed by the container’s header field is used as follows:

| Offset | Inferred role |
|---|---|
| +4 | Root address |
| +8 | Leftmost-node address |
| +12 | Rightmost-node address |

These role names describe the observed linking and balancing pattern. They do not establish a separate header type or ownership contract.

### Node layout

| Offset | Accessed content |
|---|---|
| +0 | One-byte color or balancing marker |
| +4 | 4-byte parent address |
| +8 | 4-byte left-child address |
| +12 | 4-byte right-child address |
| +16 | First 4-byte payload word |
| +20 | Second 4-byte payload word |

The function visibly writes payload and parent fields. It does not visibly initialize the created node’s child fields or initial color. Any such initialization must come from a dependency or an external precondition.

### Preconditions

For ordinary successful execution:

- All addresses actually dereferenced must identify readable or writable storage of the required width.
- The output location must permit a final 4-byte write.
- The payload source must permit two ordered 4-byte reads unless the payload-copy guard suppresses both.
- The container must provide a usable header address whenever that field is read.
- The attachment node and the ancestors traversed during balancing must have usable links and color fields.
- The dependency that creates or obtains the node must return storage suitable for the subsequent operations.
- The tree state and requested attachment must be consistent with the intended insertion operation.

The function does not validate these conditions. It does not check that the selected child slot is empty or that the payload belongs at the supplied location.

No non-aliasing guarantee is established. Aliasing can make intermediate writes affect later reads.

## Behavior

### 1. Obtain a node and copy the payload

The function first calls F-2ca62cf8d9c9 with the container address. Its result becomes the created node address retained for eventual output.

Before copying the payload, the function tests whether the address 16 bytes beyond that result is nonzero. This is not an ordinary null-result check.

When the guard permits copying, the observable order is:

1. Read the first 4-byte word from the payload source.
2. Write that word at created-node offset +16.
3. Read the second 4-byte word from payload-source offset +4.
4. Write that word at created-node offset +20.

The source is not visibly snapshotted before either write. Consequently, overlapping source and destination storage can affect the second read.

If the guard suppresses the copy, later linking and node accesses still proceed. The guard therefore does not establish a safe allocation-failure path. In particular, a zero node result is not explicitly handled as failure.

### 2. Increment the count

After the payload-copy stage, the function reads the container’s 4-byte count, adds one, and stores the result back to the same field.

This happens before comparison, attachment, or rebalancing. No rollback is visible if a later operation fails.

The stored width is 32 bits. The evidence does not establish a portable signed-overflow contract, saturation, overflow rejection, or a wider intermediate count. Behavior at the representational boundary is documented_only.

### 3. Choose the attachment side

The function first reads the container’s header address and compares it with the supplied attachment address.

It chooses left attachment without calling the comparison dependency if either:

- The attachment address equals that header address; or
- The placement selector is nonzero.

Only when the attachment differs from the header and the selector is zero does it call F-a073c1bc0f3f, supplying:

1. The address at container offset +25.
2. The payload-source address.
3. The address at attachment-node offset +16.

A nonzero result selects left attachment. A zero result selects right attachment.

The comparison receives the original payload-source address, not the created node’s copied payload. Mutations or aliasing before the call can therefore affect what it observes.

The function uses only zero versus nonzero from the comparison result. Ordering, equality, and duplicate-key semantics are not established.

### 4. Link the node and maintain endpoints

For left attachment, the function first writes the created node address into attachment offset +8. It then rereads the container’s header address.

If this newly read header address differs from the attachment address:

- Read the header’s leftmost field at +8.
- Replace that field with the created node address only if its current value equals the attachment address.

If the newly read header address equals the attachment address:

- Write the created node address into that object’s root field at +4.
- Reread the container’s header address.
- Write the created node address into the resulting header’s rightmost field at +12, without an endpoint-equality check.

With a stable header and conventional empty-container state, this path sets leftmost, root, and rightmost to the created node.

For right attachment:

- Write the created node address into attachment offset +12.
- Read the container’s current header address.
- Replace its rightmost field only if that field currently equals the attachment address.

After either attachment path, the function writes the supplied attachment address into the created node’s parent field at +4.

These writes are not transactional. The child link and applicable header fields are written before the created node’s parent field.

The function rereads header and endpoint fields rather than treating earlier values as immutable. This matters for aliasing and for any dependency-induced changes made before attachment.

### 5. Rebalance using parent colors

The balancing cursor initially identifies the created node.

At each iteration, the function first reads the current root through the container’s current header. If the cursor equals that root, balancing stops. Otherwise, it reads the cursor’s parent and stops if the parent’s color byte is nonzero.

Thus:

- Zero is the color that permits repair to continue.
- Any nonzero color stops this parent-based repair check.
- Explicit recoloring writes use only zero and one.

When repair continues, the function obtains the grandparent through the parent’s parent link and determines whether the parent equals the grandparent’s left-child address.

#### Parent is the grandparent’s left child

The grandparent’s right child is treated as the opposite-side relative.

If that address is nonzero and its color byte is zero:

1. Write one to the parent’s color.
2. Write one to the opposite-side relative’s color.
3. Write zero to the grandparent reached through the cursor’s then-current parent links.
4. Move the cursor to the grandparent reached through those links.

Otherwise:

- If the cursor equals the parent’s right child, call F-b27074c83af8 with the container and parent addresses, then make the previous parent the cursor.
- Write one to the color of the cursor’s current parent.
- Write zero to the color of that parent’s current parent.
- Call F-d6635d4fad60 with the container and the grandparent address obtained through those current links.

The post-call parent links are read again. The function does not assume a structural helper leaves them unchanged.

#### Parent is not the grandparent’s left child

The grandparent’s left child is treated as the opposite-side relative.

If that address is nonzero and its color byte is zero, the same recoloring and upward-cursor movement described above occurs.

Otherwise:

- If the cursor equals the parent’s left child, call F-d6635d4fad60, then make the previous parent the cursor.
- Write one to the color of the cursor’s current parent.
- Write zero to the color of that parent’s current parent.
- Call F-b27074c83af8 with the container and the grandparent address obtained through those current links.

The preliminary F-d6635d4fad60 invocation has additional visible arguments beyond the container and parent. Their significance is unresolved and is described under Dependencies and Uncertainties.

### 6. Force the current root color and publish the result

After the loop stops, the function rereads the header and root addresses and writes one to the root’s color byte.

Only after that write does it store the originally created node address into the caller’s output location.

There is no final null-root check, explicit success status, or visible recovery path.

## Side effects

Established direct effects include:

- Up to two 4-byte payload writes.
- One 4-byte count increment.
- A child-link write at the supplied attachment.
- Conditional endpoint updates and, in the header-attachment case, a root update.
- A parent-link write in the created node.
- Color-byte writes during repair.
- A final color-byte write to the current root.
- A final 4-byte write to the output location.

Structural dependencies may additionally change links, the root, or other state. Their complete effects are unknown.

The operation visibly integrates the returned node into the container. It does not directly release memory or clean up after failure. Whether allocation, reuse, transfer, or release occurs inside a dependency is not established.

The implementation provides no visible synchronization. Intermediate state is observable to a dependency or other observer capable of accessing the same storage. Thread safety, reentrancy, and atomic insertion are not established.

No error return, exception translation, or rollback protocol is visible. A failure after the count increment or partial linking can leave prior writes in place.

## Dependencies

| Opaque ID | Established use | Limits |
|---|---|---|
| F-2ca62cf8d9c9 | Called first with the container; result used as the created node address | Allocation, initialization, ownership, failure behavior, and other mutations unknown |
| F-a073c1bc0f3f | Conditionally called with comparison context, original payload source, and attachment payload address | Nonzero selects left; zero selects right; comparison semantics and side effects unknown |
| F-b27074c83af8 | Called during two balancing configurations with container and a node address | Left-rotation-like role inferred; exact effects unknown |
| F-d6635d4fad60 | Called during complementary balancing configurations | Right-rotation-like role inferred; exact signature and effects unknown |

One F-d6635d4fad60 call is represented with six supplied arguments rather than two. The additional visible values are:

- The opposite-side relative’s address.
- Either the grandparent address, when that relative is null, or its nonzero color byte widened to a 32-bit unsigned value.
- A retained placement-selector value.
- A retained attachment-address value.

The last two are retained from the original inputs on the forced-left path. Both are set to zero on entry to the comparison path. Whether the extra arguments are meaningful to the actual callee is documented_only; they must not be assigned invented semantics.

No dependency is assumed to preserve unrelated memory or globals, or to own or release particular storage.

## Acceptance examples

These examples are conditional static expectations, not runtime test results.

### Empty-container-style attachment

Given a stable header, attachment equal to that header, valid created-node storage, and no interfering dependency effects:

- The comparison dependency is not called.
- The count increments once.
- Header leftmost, root, and rightmost become the created node.
- The created node’s parent becomes the header.
- The repair loop stops immediately because the cursor is the root.
- The root color becomes one.
- The output receives the created node address.

### Forced left attachment

Given a non-header attachment and any nonzero selector:

- No comparison call occurs.
- Attachment offset +8 receives the created node.
- The leftmost field changes only if its observed value equals the attachment.
- The created node’s parent becomes the attachment.
- Subsequent repair depends on current root and parent colors.

This does not establish that overwriting an occupied left-child slot is valid.

### Comparison-selected right attachment

Given a zero selector, a non-header attachment, and a zero comparison result:

- The comparison receives the original payload-source address.
- Attachment offset +12 receives the created node.
- Rightmost changes only if its observed value equals the attachment.
- No duplicate rejection is established by the zero comparison result.

### Nonzero parent color

Given that the created node is not root and its parent’s color is nonzero:

- Repair stops at the parent-color test.
- No balancing helper is called on that iteration.
- The current root is still recolored to one before output publication.

### Zero-colored opposite-side relative

Given valid ancestry, a zero-colored parent, and a nonnull zero-colored opposite-side relative:

- Parent and opposite-side relative become one.
- Grandparent becomes zero.
- The cursor moves upward.
- Another root and parent-color decision follows.

### Output aliases container storage

If the output location aliases a container field, the final output write occurs after root recoloring and can overwrite that field. The function does not prevent this. Such aliasing is not a safe normal-use assumption.

## Uncertainties

The following remain documented_only:

- The node-acquisition dependency’s allocation, initialization, and failure contract.
- Initial child-link and color values.
- Exact ownership and release responsibilities.
- Comparator ordering and equality semantics.
- Structural helpers’ complete signatures and side effects.
- The significance of the extra arguments on one structural-helper call.
- Arithmetic overflow and address-wrap behavior.
- Alignment requirements and byte order.
- Behavior under malformed trees, cycles, occupied child slots, concurrent mutation, or reentrancy.
- Termination when dependencies or malformed links prevent repair from converging.
- Preservation of binary-search ordering or red-black invariants under arbitrary inputs.

The intended interpretation of zero as red, one as black, and the helpers as complementary rotations is strongly suggested by the repair pattern. It is not independently verified.

## Evidence

This contract is based only on the supplied static function representation and opaque dependency mapping.

Established observations include access widths and offsets, branch conditions, visible call arguments, ordered writes, repeated state reads, absence of visible validation and rollback, and final output timing.

Tree-role names and balancing intent are structural inferences. Dependency internals, runtime memory behavior, and global invariant preservation are not established.

No runtime tests, execution traces, or implementation-access verification have been performed.
