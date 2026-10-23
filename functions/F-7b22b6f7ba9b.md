<!-- Trusted provenance: F-7b22b6f7ba9b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7b22b6f7ba9b

## Purpose

F-7b22b6f7ba9b writes a supplied four-byte value into a container field, then passes that same value to callbacks selected by nodes reachable from twelve ordered head slots.

The established behavior is an initial store followed by mutation-sensitive traversal and indirect dispatch. Each node supplies its own dispatch information. The next link is read after that node’s callback returns, and each head is read only when traversal reaches its slot.

Propagation of a container-level setting is a plausible intent, not an established semantic requirement. The meaning of the value and the callbacks’ application-level effects remain unknown.

This contract describes static evidence. No runtime tests have been performed.

## Interface

### Inputs

The function takes two arguments:

- **Container address:** an address represented by a 32-bit integer in the supplied interface. It identifies storage containing twelve head-pointer slots and a four-byte destination field.
- **Notification value:** a four-byte quantity whose bit pattern is stored and supplied to callbacks. The function does not interpret it as a Boolean, signed number, enumeration, address, or other semantic type.

The container address is used without an explicit null, bounds, or initialization check.

### Container layout

| Byte offset from container | Access width | Role |
|---|---:|---|
| 0, 4, 8, …, 44 | 4 bytes each | Twelve head-pointer slots, numbered 0 through 11 |
| 48 | 4 bytes | Destination for the notification value |

The accessed layout extends through byte offset 51. This is an access requirement, not evidence that the complete container occupies exactly 52 bytes.

The supplied representation uses four-byte pointer slots. It does not establish portability to a different pointer width.

### Node and dispatch layout

For each non-null node reached during traversal:

| Location | Access width | Role |
|---|---:|---|
| Node offset 0 | 4 bytes | Address of a dispatch structure |
| Dispatch-structure offset 8 | 4 bytes | Indirect callable address |
| Node offset 16 | 4 bytes | Next-node pointer |

The function does not directly access other node or dispatch-structure fields.

Each callback receives two arguments, in this order:

1. The current node address.
2. The original four-byte notification value supplied to F-7b22b6f7ba9b.

A callback result, if any, is not consumed. The complete callback ABI and return type are not established.

### Output and completion

F-7b22b6f7ba9b has no explicit return value.

Normal return occurs after all twelve head slots have been processed and every traversal started from those slots has ended at a null pointer. A null head counts as a completed, empty traversal.

### Preconditions for normal execution

Every access and call actually reached must be valid:

- The destination field must be writable when the initial store occurs.
- Each head slot must be readable when its turn is reached.
- Each visited node must provide readable dispatch information.
- The selected target must be callable with the two supplied arguments.
- The current node’s next-pointer field must remain readable after its callback returns.
- Callbacks must return normally for traversal to continue.
- The dynamically followed links must eventually terminate for the function to return normally.

These are conditional execution requirements, not validations performed by the function. No ownership transfer is established for either argument.

## Behavior

### Initial store and value identity

The function first writes the notification value to the four-byte field at container offset 48. This precedes every head read and callback.

There is no comparison with the previous field value. Equal old and new values do not suppress the write or traversal. An entirely empty container still receives the write.

Callback arguments use the original function argument, not a reload of the destination field. Consequently, a callback that changes the stored field does not change the value delivered by later callbacks in this invocation.

There is no final write restoring the destination field. Its value at return therefore depends on any intervening callback effects.

### Head selection and ordering

Head slots are processed in ascending order, from slot 0 through slot 11. Each slot is four bytes after its predecessor.

The function reads a slot’s head only when it reaches that slot. It does not snapshot all twelve heads before invoking callbacks. It completes the current slot’s traversal before reading the following slot’s head.

A null head produces no callback for that slot. A non-null head becomes the first current node for that slot.

Changes made by earlier callbacks can therefore affect a later head read. The relevant value is the head present when that read occurs, not necessarily the value installed by any particular earlier callback. An intervening callback can replace an earlier replacement.

The slot counter advances through the bounded range needed to process twelve slots. Its increments and four-byte slot offsets do not approach a 32-bit arithmetic boundary. Validity of adding those offsets to the supplied container address remains an external precondition; the evidence does not establish a portable overflow rule for invalid addresses.

### Per-node dispatch order

For each non-null current node, the directly expressed order is:

1. Read its dispatch-structure pointer from node offset 0.
2. Read the callable address from dispatch-structure offset 8.
3. Invoke that callable with the current node and original notification value.
4. After normal callback return, read the next pointer from the current node at offset 16.

The resulting next pointer determines whether traversal ends or continues.

Only the current node pointer is explicitly checked for null. There is no corresponding explicit null check for the dispatch-structure pointer or callable address.

The dispatch target is resolved afresh for each visit. The body does not establish one common target for all nodes or cache a target across repeated visits.

### Effects of mutation during callbacks

Traversal follows the current node’s link as read **after** the callback. It does not preserve a pre-callback successor.

Subject to valid storage and the state present at that read:

- Setting the current link to null ends that slot’s traversal.
- Replacing the current link redirects traversal.
- Installing a new node at that link can make it the next visited node.
- Changing the current slot’s head does not itself restart traversal from the changed head.
- Changing a previously processed head does not cause that slot to be revisited.
- Changing a future head affects selection only to the extent that the changed value remains present when the future slot is read.

A callback can also modify dispatch information used on subsequent visits. No general preservation of node contents, container contents, or unrelated state is established.

### Repeated visits and cycles

There is no deduplication or cycle detection.

A node reachable through multiple slots can receive a callback during each traversal. A node reached repeatedly within one traversal can also receive repeated callbacks, with dispatch resolved on each visit.

If a reachable cycle persists, callbacks keep returning, and accesses remain valid, traversal continues within that cycle. Later slots are not reached, and the function does not return normally.

Termination depends on the links actually read during execution, not solely on whether the entry-state lists were acyclic.

### Errors and interrupted execution

The body supplies no explicit error result, diagnostic, retry, recovery, or rollback.

An invalid access or indirect target can prevent normal execution. The exact manifestation is environment-dependent and remains unspecified.

If a callback does not return normally, the body does not establish continuation to the next-pointer read or later slots. The initial store and any earlier effects have already occurred; no transactional reversal is provided.

## Side effects

### Direct memory effects

The only explicit data write is the four-byte store at container offset 48.

Direct reads comprise:

- Each head slot when its turn is reached.
- Each visited node’s dispatch-structure pointer.
- The corresponding callable entry.
- Each current node’s next pointer after callback return.

The function does not explicitly write head slots, node links, or dispatch entries. This distinction does not constrain writes performed by callbacks.

### Callback effects and final state

Callback effects are unknown. The available evidence does not establish preservation of:

- The notification field or head slots.
- Node contents or links.
- Dispatch structures.
- Global or unrelated memory.
- Allocations, resources, or object lifetimes.

The initial store is therefore not an unconditional final-state guarantee. A callback may change the destination field, and the function does not subsequently restore it.

### Ownership and lifetime

No allocation, deallocation, retain, release, or reference-count operation is explicit in this function.

Passing a node to its callback does not establish that the callback owns, borrows, retains, or may destroy that node. Those policies remain unresolved.

The post-callback next-pointer read is a concrete lifetime constraint: the current node’s relevant storage must still be readable at that point for valid continuation. No visible lifetime guard makes destruction safe.

The container must likewise remain accessible for every later head read that is reached.

### Concurrency and reentrancy

No synchronization, atomic operation, or reentrancy guard is explicit.

The stated ordering describes this function’s accesses relative to callback boundaries. It does not establish inter-thread visibility or safe concurrent mutation.

Recursive invocation through a callback is not explicitly prevented. Its interaction with shared fields and traversal state depends on behavior outside the supplied body.

## Dependencies

The visible dependency is a dynamically selected indirect callback. No concrete opaque function ID is established for its possible targets. Target identities remain **documented_only** rather than being assigned invented IDs.

The established dependency boundary consists of:

- Selection through the current node’s dispatch pointer and the entry at dispatch offset 8.
- The current node address as the first argument.
- The original four-byte notification value as the second argument.
- No use of a callback return value.
- A next-pointer read after normal callback return.

The callback’s implementation, complete calling convention, ownership policy, global effects, failure behavior, and permitted mutations remain unknown. In particular, neither global preservation nor memory release may be assumed.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Each assumes valid reached accesses and callable targets unless the example specifically concerns invalidation.

### Empty container

Given twelve readable null heads and a writable destination field, with no external mutation during execution:

- The notification value is stored at offset 48.
- All twelve heads are read in slot order.
- No callback occurs.
- The function returns normally.

### Multiple populated slots

Given slot 0 pointing to a two-node terminating chain, slot 1 null, slot 2 pointing to a one-node terminating chain, and later slots null, with callbacks preserving those heads and links:

- Both nodes in slot 0 are called in chain order.
- Slot 1 produces no callback.
- The node in slot 2 is called next.
- Every callback receives the original notification value.

### Stored field changed by a callback

Given a first callback that changes the destination field to a different value:

- Later callbacks still receive the original argument.
- If no subsequent action changes the field again, that replacement remains at normal return.
- The function performs no restoring store after traversal.

### Current successor replaced

Given a callback that replaces its current node’s next pointer with another valid node, and that replacement remains present until the post-callback read:

- Traversal selects the replacement node next.
- The former successor is not selected through that replaced link.
- The former successor may still be visited through another reachable route.

### Future head changed and preserved

Given a callback during slot 0 that replaces slot 5’s head:

- If execution reaches slot 5 and no intervening callback or other mutation changes that head again, slot 5 starts from the replacement.
- If the replacement is null and remains null until slot 5 is read, slot 5 produces no callback.

Neither outcome follows solely from the earlier write. The value present at the actual slot-5 read controls selection.

### Future head changed again

Given one callback during slot 0 that installs one node as slot 5’s head, followed by another callback that replaces it with a different node:

- If the second replacement remains present until slot 5 is read, traversal starts from the second node.
- The first replacement has no priority merely because it was installed earlier.
- If the second replacement is null and remains null until the read, slot 5 is empty for this invocation.

### Same node in two slots

Given two slots pointing to the same node, with a null next link and callbacks preserving those heads and that link:

- The node is called once for each slot.
- No identity-based suppression occurs.
- Dispatch information is read separately for both visits.

### Persistent cycle

Given a reachable cycle that remains cyclic, valid accesses, and callbacks that always return normally:

- Nodes continue to be visited within the cycle.
- Later slots are not reached.
- Normal return does not occur.

### Current node invalidated

Given a callback that makes its current node’s next-pointer storage inaccessible:

- The body still proceeds toward a read from that storage after normal callback return.
- Valid completion is not guaranteed.
- No implicit retention, pre-callback successor snapshot, or recovery may be assumed.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the notification value.
- Complete container, node, and dispatch types.
- Whether the dispatch structure is formally a virtual-method table.
- Concrete callback identities and full ABI details.
- Endianness and alignment requirements.
- Environment-specific consequences of invalid accesses or calls.
- Ownership, destruction, and resource-management rules.
- Callback effects on globals and unrelated memory.
- Concurrent-use guarantees and recursive-call behavior.
- Whether callers impose acyclic or disjoint-list invariants.
- Portable behavior for invalid address arithmetic.

The four-byte fields and pointer slots are established by the supplied representation. That evidence does not justify changing their width or extending the contract to another address model.

## Evidence

The supplied static body establishes:

- A four-byte store at container offset 48 before any head traversal.
- Twelve head slots at four-byte intervals, processed in ascending order.
- A fresh head read when each slot is reached.
- Null-terminated traversal without deduplication.
- Indirect dispatch through node offset 0 and dispatch offset 8.
- Callback arguments consisting of the current node and original notification value.
- A next-pointer read at node offset 16 after normal callback return.
- No explicit validation, allocation, release, synchronization, rollback, or recovery.

Mutation-sensitive successor selection, repeated visits, and selection of the latest future-head value follow from that access order. They are conditional consequences, not guarantees that callbacks preserve any particular mutation.

Setting propagation is inferred intent only. Dependency implementations and broader type definitions are unavailable. No runtime execution or tests have been performed.
