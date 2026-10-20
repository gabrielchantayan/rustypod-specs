<!-- Trusted provenance: F-e86ea13ce5bb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e86ea13ce5bb

## Purpose

F-e86ea13ce5bb searches a linked sequence for the first reachable node whose stored 32-bit integer key equals a supplied 32-bit integer key.

The search starts at the node address obtained from a caller-supplied head location. It follows a successor address stored in each nonmatching node. It returns the address of the first matching node, or zero if traversal reaches a zero address.

The established behavior is a forward, equality-based search. The interpretation as a linked-list lookup is inferred intent. The supplied evidence does not establish a container type, node ownership policy, mutation protocol, or application-specific meaning for the key.

## Interface

### Inputs

The function takes two inputs:

- **Head location:** a pointer to a readable integer-sized storage location containing the initial node address, or zero for an empty sequence.
- **Search key:** a 32-bit integer compared for exact equality against each visited node’s key.

The head location and the address stored there are distinct. The function dereferences the head location to obtain its initial traversal address. A readable head location containing zero is an empty sequence; a null head-location pointer is not an alternative empty-sequence representation.

The supplied representation uses 32-bit integers and encodes traversal addresses in integer values. No broader machine ABI or portability guarantee is established.

### Node layout relevant to this function

For a nonzero node address, the function uses these byte offsets:

| Offset from node address | Width | Meaning |
|---|---:|---|
| +4 | 32 bits | Successor node address; zero terminates traversal |
| +8 | 32 bits | Key compared with the search key |

The four bytes beginning at offset +0 are not read by this function. Their purpose is unknown. No complete node type or total allocation size is established beyond the readable regions required by the accesses above.

The successor field is interpreted as another address encoded in a 32-bit integer. It is not a relative offset or node index in the supplied behavior.

### Output

The return value is a 32-bit integer:

- Zero if traversal reaches a zero address before finding a matching key.
- The current node address if that node’s key matches.

The result is an address value, not a copy of the node, the matching key, or the successor address. The function does not return the location that stores the head.

### Preconditions

For a defined successful execution:

- The head location must support the initial integer read.
- Every nonzero address whose key is examined must support a 32-bit read at offset +8.
- Every nonmatching node from which traversal continues must support a 32-bit read at offset +4.
- Address formation at these offsets must be valid under the applicable execution model.
- Alignment, object representation, and memory-access requirements of that model must be satisfied.

Termination additionally requires that traversal encounter either a matching node or a zero address after finitely many steps. Acyclicity is sufficient for many ordinary list structures, but it is not required if a match is reached before any repetition.

## Behavior

### Initialization and empty-sequence handling

The function first reads the integer stored at the head location. That value becomes the current node address.

The head location is read once by the supplied function. It is not reread between nodes. Consequently, replacing the stored head after this initial read does not, by itself, redirect this invocation’s traversal.

If the initial address is zero, the function returns zero without reading any node fields.

### Per-node access order

For each nonzero current address, the function reads the 32-bit key at offset +8 and compares it with the search key.

If the keys are equal, it returns the current address immediately. The successor field of that matching node is not read.

If the keys differ, it reads the 32-bit successor address at offset +4. That value becomes the next current address. A zero successor ends the search with a zero result; a nonzero successor leads to another key examination.

Thus, the source-level memory-access order is:

1. One read of the head value.
2. A key read for each visited nonzero node.
3. A successor read only for each visited node whose key does not match.

No node access occurs after a match is found. In particular, the function does not scan the remainder of the sequence to identify duplicates or validate later nodes.

This describes the ordering established by the supplied source representation. It does not establish instruction-level ordering, atomicity, or externally observable access counts for volatile or device-backed memory.

### Equality and duplicate keys

Matching uses exact integer equality. The function performs no ordering comparison, masking, normalization, range check, or transformation of the search key.

Negative values and zero are ordinary search keys. Zero has a termination role when used as a traversal address, not when stored in a key field.

When multiple reachable nodes have equal keys, the first encountered node is returned. “First” means earliest in successor traversal from the initially read head, not lowest address or any order established by another field.

### Traversal boundaries and arithmetic

Traversal ends only through a matching key or a zero current address. The function has no explicit node-count limit, cycle detection, timeout, or structural validation.

Field addresses are formed by adding byte offsets +8 and +4 to the current encoded address. There is no explicit overflow check, widening step, or wraparound policy. The evidence therefore does not justify accepting addresses near the representation boundary on the assumption that offset arithmetic wraps safely.

The search key does not participate in address arithmetic. No length, index, or allocation size is computed.

### Invalid memory and failure behavior

The function exposes no separate error result. Zero reports traversal termination without a match; it is not an implemented diagnostic for an invalid head location or malformed node.

The function does not validate whether a nonzero address is mapped, readable, aligned, or a legitimate node. Invalid accesses and invalid address arithmetic fall outside the defined contract. The evidence does not establish that such inputs return zero, raise any particular exception, or fail recoverably.

A matching node does not need a readable successor field for this function’s own path, because that field is not accessed. Its key field must still be readable.

## Side effects

The supplied body contains no explicit memory writes. It does not update:

- The caller’s head value.
- Any node key or successor.
- Any global state.
- Any explicit error-status location.

There are no calls to allocation, release, logging, synchronization, or other helper functions.

The function obtains and returns existing address values. It neither creates a node nor explicitly acquires, transfers, or releases ownership. A nonzero result provides no new lifetime guarantee: its continued validity depends on the caller’s surrounding memory-management rules.

No synchronization is present. The function does not lock nodes or establish a consistent snapshot. The initial head is captured once, while node fields are accessed as traversal proceeds. No supported behavior under concurrent mutation is established; a reader must not infer thread safety from the absence of writes.

The absence of explicit writes also does not establish that reads are harmless for arbitrary special-purpose address spaces. Memory-mapped devices and other read-sensitive storage are outside the documented ordinary-memory contract.

## Dependencies

No external function dependencies are visible in F-e86ea13ce5bb. The supplied body contains no calls, so no additional opaque function IDs are required.

Execution depends on the caller supplying usable memory and valid address representations for the path actually taken. The surrounding allocator, container implementation, ownership system, and synchronization policy are not identified.

No claim about an unknown callee preserving globals or releasing memory is needed: no callee participates in the supplied behavior.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They are not runtime test results.

### Empty sequence

Given a readable head location containing zero, the function returns zero. It performs the head read and does not attempt a node-field read.

### Match at the head

Given a readable head location containing node A’s nonzero address, and a readable key field in A equal to the search key, the function returns A’s address.

A’s successor is not read. An unusable successor value does not affect this path, provided that the actual head and key accesses are valid.

### Match after a nonmatching node

Given a head naming A, A’s key differing from the search key, A’s successor naming B, and B’s key matching, the function returns B’s address.

The relevant access order is the head value, A’s key, A’s successor, and B’s key. B’s successor is not read.

### No matching node

Given a valid finite chain A → B → zero, with neither key matching, the function returns zero after reading both keys and both successors.

A zero key in either node does not terminate traversal unless it equals the requested search key.

### Duplicate matching keys

Given a valid chain A → B → C, with A nonmatching and both B and C matching, the result is B’s address. C is not examined.

### Reachable cycle

Given a valid cycle whose examined keys never match, and no external change that alters traversal, the function does not terminate through any branch shown in the evidence.

If a matching node is encountered before traversal would repeat, its address is returned normally. The mere existence of a cycle does not prevent that earlier return.

### Invalid head location

Given an unreadable head location, no return value is promised. This is distinct from a readable head location containing zero.

## Uncertainties

The following are **documented_only**, not resolved by assumptions:

- **Concrete ABI:** the representation shows 32-bit integer fields and address-bearing values, but does not fully specify calling convention, endianness, or alignment rules.
- **Full node layout:** only offsets +4 and +8 are used. Other fields, padding, allocation boundaries, and metadata are unknown.
- **Address validity:** no mechanism establishes that stored nonzero integers identify valid nodes.
- **Arithmetic boundary behavior:** no defined overflow or wraparound handling is supplied for field-address formation.
- **Ownership and lifetime:** allocation origin, retention rules, permitted freeing, and returned-address lifetime are unspecified.
- **Concurrency:** no synchronization or permissible concurrent-mutation model is supplied.
- **Fault handling:** no particular signal, exception, recovery, or diagnostic behavior is established for invalid accesses.
- **Structural invariants:** acyclicity, key uniqueness, sorting, and sentinel-node conventions are not established.

## Evidence

**Established static evidence:** the supplied function reads one head value, checks each current address against zero, reads keys at byte offset +8, follows successors at byte offset +4 only after a mismatch, and returns either zero or the matching node address. It contains no explicit writes or function calls.

**Inferred intent:** the layout and traversal are consistent with a first-match lookup in a singly linked sequence. That interpretation does not establish additional container invariants.

**Verification status:** this contract is based solely on the supplied static representation. No implementation access, tools, runtime execution, or tests were used.
