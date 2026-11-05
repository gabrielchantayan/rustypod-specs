<!-- Trusted provenance: F-6dc9d8a81ab1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6dc9d8a81ab1

## Purpose

F-6dc9d8a81ab1 searches a linked collection for the first node whose one-byte discriminator matches the requested discriminator and whose payload is accepted by a handler-selected predicate.

On a successful match, the function moves that node to the front of the collection unless its predecessor field is already zero. If traversal reaches a zero node pointer, it invokes a handler-selected fallback operation.

The established behavior is search, conditional list reordering, and fallback dispatch. A move-to-front cache or recency policy is a plausible interpretation, not an established purpose. The supplied evidence does not establish that fallback creates a node, that matching updates a value, or that any operation allocates or releases memory.

## Interface

### Arguments and result

The function accepts four arguments:

| Argument | Width and interpretation | Use |
|---|---|---|
| Collection address | A pointer represented by the supplied 32-bit address model | Locates the collection’s head and tail fields and is passed to fallback. |
| Requested discriminator | Unsigned 32-bit integer | Passed unchanged to F-ab6062dd4d3b and compared against each visited node’s unsigned one-byte discriminator. |
| Match context | Opaque 32-bit value | Passed to the predicate as its third argument and to fallback as its fourth argument. |
| Fallback context | Opaque 32-bit value | Passed to fallback as its third argument; not otherwise used directly. |

The exposed interface has no result value. A predicate result is interpreted only as zero versus nonzero. No status, selected-node pointer, or fallback result is explicitly returned.

The two opaque context arguments are not interchangeable: their order differs between the outer interface and fallback invocation.

### Collection layout

The supplied representation uses the following collection fields:

| Byte offset | Width | Meaning in this function |
|---|---|---|
| +0x10 | 32 bits | Head-node address; zero denotes the end of traversal when loaded as the current node. |
| +0x14 | 32 bits | Tail-node address; written only when relocating a matched node whose successor is zero. |

Other collection fields are not directly accessed.

### Node layout

| Byte offset | Width | Meaning in this function |
|---|---|---|
| +0x00 | 32 bits | Predecessor-node address; zero suppresses relocation after a match. |
| +0x04 | 32 bits | Successor-node address; zero terminates forward traversal. |
| +0x08 | Address only | Beginning of the region passed to the predicate. Its size and contents are unspecified. |
| +0xC8 | 8 bits | Unsigned discriminator. |

The payload argument is an interior address within the node, not a separately loaded pointer.

### Handler layout

F-ab6062dd4d3b returns a handler address. The first 32-bit field at that address is treated as a dispatch-table address.

The function uses two operation slots:

- Dispatch-table byte offset +0x04: predicate operation.
- Dispatch-table byte offset +0x08: fallback operation.

The identities and internal contracts of these indirect operations are not provided. Slot descriptions here identify dispatch locations, not additional known functions.

### Preconditions

For defined operation under the supplied memory model:

- F-ab6062dd4d3b must return an address usable for any dispatch subsequently required.
- The collection address must permit the field accesses taken by the execution path.
- Every visited nonzero node must permit access to its discriminator and successor fields.
- A matching node subject to relocation must have readable and writable link fields and valid linked neighbors for all accesses performed.
- The indirect operations must be callable with the argument arrangements described below.
- Memory needed after a predicate call must remain usable when the function resumes.

No checks enforce these requirements. Callback-driven mutation, concurrent mutation, object destruction, malformed pointers, and inconsistent list topology are not made safe by this function.

## Behavior

### 1. Resolve the handler before reading the head

The function first calls F-ab6062dd4d3b with the full unsigned 32-bit requested discriminator.

Only after that call returns does it read the collection’s head field. Consequently, any changes the dependency makes to collection memory before returning can affect the starting node. The function does not capture the head before resolving the handler.

The returned handler address is retained for subsequent dispatch. The evidence does not establish that the handler’s dispatch-table field or dispatch-table contents remain unchanged across indirect calls.

### 2. Search in successor order

Traversal starts at the loaded head address.

For each current node:

1. A zero current-node address selects fallback immediately.
2. Otherwise, the function reads the node’s unsigned discriminator byte.
3. It compares that byte, promoted to an integer value from 0 through 255, with the full requested unsigned 32-bit discriminator.
4. Only if those values are equal does it invoke the predicate.
5. A nonzero predicate result selects the current node.
6. A discriminator mismatch or zero predicate result advances traversal using the current node’s successor field.

There is no length bound, cycle detection, or secondary termination condition. A cycle that never produces a successful predicate result can prevent the function from returning.

### 3. Predicate invocation

For a discriminator-compatible node, the predicate receives, in order:

1. The retained handler address.
2. The current node’s address plus eight bytes.
3. The match context.

A zero predicate result rejects that node. Any nonzero result accepts it; no particular nonzero value has special meaning.

The function does not invoke the predicate for discriminator-incompatible nodes.

If the predicate rejects a node, the successor used to continue traversal is read after the call returns. Therefore, traversal can follow a successor changed by that predicate. The evidence does not support describing traversal as operating on a frozen list snapshot.

If the predicate accepts a node, the function then reads its predecessor field to decide whether relocation is needed. That decision likewise reflects post-callback memory, not a predecessor value captured before the call.

### 4. Successful match with a zero predecessor

If the accepted node’s predecessor field is zero, the function returns without performing direct list writes.

Under ordinary doubly linked-list invariants, this is the already-at-head case. Mechanically, however, the function tests only the predecessor field. It does not verify that the accepted node equals the collection’s head.

Fallback is not invoked on this path. The fallback context has no direct use.

### 5. Successful match with a nonzero predecessor

If the accepted node’s predecessor field is nonzero, the function detaches that node from its current position and inserts it at the front.

For a stable, well-formed list, the direct writes occur in this order:

1. Set the predecessor node’s successor field to the matched node’s successor.
2. Repair the other side of the removal:
   - If the matched node’s successor is zero, set the collection’s tail field to the predecessor.
   - Otherwise, set the successor node’s predecessor field to the predecessor.
3. Set the matched node’s successor field to the collection’s current head.
4. Set the matched node’s predecessor field to zero.
5. Set the node addressed by the collection’s head field to have the matched node as its predecessor.
6. Set the collection’s head field to the matched node.

The head used in the front-insertion portion is read during relocation. The function does not validate that it is nonzero before writing through it.

These are separate writes, not an atomic list transaction. Intermediate states temporarily violate normal bidirectional-list invariants. No locking, rollback, or publication mechanism is shown.

With a valid stable list, relocation preserves the relative order of all other nodes. Relocating an interior node leaves the tail unchanged. Relocating the tail updates the tail to the matched node’s former predecessor.

### 6. Exhausted traversal

If the current-node address is zero, the function invokes fallback using the retained handler’s dispatch-table slot at byte offset +0x08.

Fallback receives, in order:

1. The handler address.
2. The collection address.
3. The fallback context.
4. The match context.

After this dispatch, the exposed function has no further specified work.

This path covers both an initially empty collection and a traversal in which no visited node satisfies both selection conditions. It also covers traversal that reaches zero because a predicate changed a successor link.

The indirect fallback transfer has an explicit recovery limitation in the supplied static representation. Its displayed argument order and terminal position are evidence, but exact machine-level call-versus-tail-transfer behavior is not independently established.

### Integer-width consequences

Discriminator comparison is not low-byte matching. A request of 0x00000105 does not match a stored byte of 0x05.

For every requested discriminator above 255, no stored byte can compare equal. Subject to valid, terminating traversal, such a request reaches fallback without invoking the predicate. F-ab6062dd4d3b still receives the original full-width request first.

No arithmetic on discriminator values is performed. Pointer offsets and link accesses rely on valid addresses; no overflow or bounds checks are shown.

## Side effects

### Direct effects

The function can directly modify only the following identified fields:

- A matched node’s predecessor and successor.
- Its predecessor’s successor.
- Its successor’s predecessor, when a successor exists.
- The old head’s predecessor.
- The collection’s head.
- The collection’s tail, only when removing a node with a zero successor.

It does not directly write the discriminator or payload region. It does not directly allocate, free, clear, or copy a node.

On a successful match whose predecessor is zero, there are no direct list writes. On exhausted traversal, there are no direct list writes before fallback.

### Indirect effects and ownership

F-ab6062dd4d3b, the predicate, and fallback may have effects not described by this function’s body. Their global-state behavior, allocation behavior, ownership transfers, and memory-release behavior are unknown.

Passing a handler, collection address, or payload interior address does not establish ownership transfer. Conversely, the absence of an explicit release here does not prove that dependencies preserve those objects.

The function continues using relevant objects after predicate calls. Safe execution therefore requires dependency behavior compatible with those later accesses.

### Errors and exceptional conditions

No explicit error reporting is shown for:

- Invalid collection or node addresses.
- A null or invalid handler.
- Invalid dispatch entries.
- Corrupted links.
- Cyclic lists.
- Inconsistent head, tail, and predecessor fields.
- Callback-induced invalidation.
- Unsupported discriminator values.

The evidence establishes neither recovery behavior nor a particular failure mechanism for these conditions.

## Dependencies

### F-ab6062dd4d3b

Established interface:

- Input: the requested discriminator, as an unsigned 32-bit value.
- Output: an address used as the handler.
- Invocation count: once per entry to F-6dc9d8a81ab1, before the initial head read.

Unknowns include selection rules, validity guarantees, caching, global effects, and ownership.

### Handler predicate slot

Established interface:

- Dispatch location: handler dispatch-table byte offset +0x04.
- Inputs: handler address, current-node payload interior address, match context.
- Result interpretation: zero rejects; nonzero accepts.
- Invocation condition: exact full-width equality between the requested discriminator and the node’s unsigned byte value.

Preservation of links, payload, collection state, handler state, and globals is not established.

### Handler fallback slot

Established interface:

- Dispatch location: handler dispatch-table byte offset +0x08.
- Inputs: handler address, collection address, fallback context, match context.
- Invocation condition: traversal reaches a zero current-node address.

Any insertion, lookup, replacement, error reporting, or memory-management behavior inside fallback is documented_only.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless stated otherwise, assume valid addresses, a stable well-formed doubly linked list, valid dispatch operations, and dependencies that do not mutate relevant list or handler state.

### Empty collection

Given a zero head, the function resolves the handler and then invokes fallback with the collection and the two contexts in the specified order. It does not invoke the predicate or directly alter head or tail.

### Accepted head

Given a list A → B → C, with A’s predecessor zero, suppose A’s discriminator matches and the predicate accepts A.

The list remains A → B → C. No direct list writes occur, and fallback is not invoked.

### Accepted interior node

Given A → B → C, suppose A is not selected and B is selected.

The resulting order is B → A → C. B becomes head, C remains tail, and both directions of the affected links reflect that order. The function stops at B rather than checking C.

### Accepted tail

Given A → B → C, suppose only C is selected.

The resulting order is C → A → B. C becomes head, B becomes tail, and B’s successor becomes zero.

### Compatible discriminator but rejected payload

Suppose A’s discriminator matches but its predicate result is zero, and B is subsequently accepted.

A is not selected merely because its discriminator matches. Traversal continues to B, which is relocated if its predecessor is nonzero.

### Request outside the byte range

Given a finite list and requested discriminator 0x00000105, a node containing discriminator 0x05 does not qualify for predicate invocation. No node can qualify by discriminator, so traversal reaches fallback.

### Predicate changes the next link

Suppose a predicate rejects the current node after setting its successor to zero.

The subsequent successor read observes zero, and fallback is selected. The function does not continue using a successor captured before the predicate.

### Zero predecessor on a reachable non-head node

Suppose malformed links make a selected non-head node’s predecessor zero.

The function returns without relocation. It does not repair the inconsistency or verify head identity.

## Uncertainties

The following remain documented_only:

- The semantic meaning of discriminators and both contexts.
- Payload size, layout, mutability, and lifetime.
- Handler ownership and lifetime.
- Dependency effects on globals and shared memory.
- Whether callbacks are intended to mutate list structure.
- Thread-safety, synchronization, and reentrancy guarantees.
- Required list invariants beyond those necessary for safe observed accesses.
- Behavior on invalid pointers or dispatch entries.
- Exact machine-level handling of the terminal fallback dispatch.
- The broader algorithmic purpose, including any cache or recency interpretation.

The supplied representation establishes a 32-bit field and address model for these accesses. It does not independently establish a portable ABI, endianness, or implementation on other pointer widths.

## Evidence

Established static evidence consists of the supplied function representation and the mapping identifying F-ab6062dd4d3b as its direct dependency.

That evidence supports the argument flow, field offsets, unsigned byte-versus-32-bit comparison, short-circuit predicate invocation, successor-order traversal, conditional relocation, write ordering, and fallback argument arrangement described above.

Interpretations such as “cache lookup,” “recency promotion,” and “fallback insertion” are not required by the observed behavior and are not adopted as guarantees.

No implementation execution, runtime tests, binary-level verification, or dependency inspection has been performed.
