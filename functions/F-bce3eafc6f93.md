<!-- Trusted provenance: F-bce3eafc6f93; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bce3eafc6f93

## Purpose

F-bce3eafc6f93 conditionally sets or clears a mask of bits in one of two 32-bit state fields. A selector determines both the mask and which state field is eligible for modification. When the selected field’s stored value differs from the computed replacement, the function stores the replacement and then writes a one-byte change marker.

The function distinguishes between a dependency-provided starting value and the value actually present in the destination field. It computes the replacement from the former, but decides whether to write by comparing against the latter. Their equality is not established.

The apparent intent is to update one of two flag collections and mark related state as changed. The meanings of those collections, individual bits, and the change marker are not established.

## Interface

### Inputs

The function accepts three arguments:

1. **Context address.** An address represented by a 32-bit integer in the supplied source. For a supported selector category, a four-byte field at byte offset `0xA8` supplies an intermediate object address.
2. **Selector value.** An opaque four-byte value passed unchanged, first to F-270c5b6d8d34 and then to F-266e368c1191. Its internal encoding is unknown.
3. **Operation flag.** A signed 32-bit value. Zero requests clearing the selected bits; every nonzero value requests setting them. Negative values are not a distinct operation.

### Output

There is no explicit return value. Observable outcomes consist of dependency calls, memory reads, a possible 32-bit state write, and a possible one-byte marker write.

No success flag, error code, or indication of whether a write occurred is returned.

### Relevant memory layout

All offsets below are byte offsets. Names describe roles in this function, not established object types.

| Base | Offset | Access width | Role |
|---|---:|---:|---|
| Context address | `0xA8` | 4 bytes | Intermediate object address |
| Intermediate object address | `0xF00` | 4 bytes | State-block address |
| State-block address | `0xB30` | 4 bytes | State field for category zero |
| State-block address | `0xB8C` | 4 bytes | State field for category one |
| State-block address | `0xB90` | 1 byte | Change marker |

The supplied representation uses four-byte address-bearing fields. It does not establish a portable layout for wider-pointer environments. Byte order, alignment requirements, object extents, and address-space validity are not specified.

### Preconditions

For defined execution, every memory access actually reached must designate accessible storage of the required width. Accesses that write additionally require writable storage.

The function performs no explicit null check, bounds check, lifetime check, or address validation. The dependencies may impose additional preconditions that are not available here.

Unsupported selector categories do not cause direct context-memory accesses by this function, although both initial selector dependencies still run.

## Behavior

### 1. Resolve the mask and category

The function first calls F-270c5b6d8d34 with the selector value. Its result is retained as a 32-bit unsigned mask.

It next calls F-266e368c1191 with the same selector value. Its result is treated as a signed 32-bit category:

- Exactly zero selects the first state field.
- Exactly one selects the second state field.
- Every other value selects neither field.

These calls occur in that order. There is no context-memory read by this function before them.

The result of the first call is retained for the later bit operation. This does not imply that either dependency preserves global state or reachable objects.

### 2. Obtain the starting value

For category zero, the function:

- Reads the four-byte intermediate object address from context offset `0xA8`.
- Passes that value to F-2bd8af926943.
- Treats its result as the 32-bit unsigned starting value.

For category one, it performs the same context-field read, but passes the resulting address to F-3d1cdf97ba16 instead.

For any other category, neither starting-value dependency is called. The local starting value remains zero.

The supplied source does not establish that either starting-value dependency simply reads the later destination field. It may return a derived, cached, or otherwise different value. Its side effects are unknown.

### 3. Compute the candidate state

The candidate is computed entirely within a 32-bit unsigned bit domain:

- With an operation flag of zero, clear every bit selected by the mask from the starting value.
- With any nonzero operation flag, set every bit selected by the mask in the starting value.

For clearing, the complemented mask is a 32-bit complement. No widening beyond the state width is implied.

A zero mask leaves the starting value unchanged. An all-ones mask produces zero when clearing and all ones when setting.

No arithmetic addition, saturation, bit-count interpretation, or restriction to a single mask bit is present. The mask may select any combination of the 32 bits.

For unsupported categories, the candidate is still computed from the zero starting value, but it is not used for a direct state access or write.

### 4. Resolve and compare the destination

For category zero or one, the function reads context offset `0xA8` again. This is a fresh read after the starting-value dependency returns; it is not a reuse of the address passed to that dependency.

From this freshly obtained intermediate object address, the function reads the state-block address at offset `0xF00`.

It then reads the selected 32-bit destination:

- Category zero: state-block offset `0xB30`.
- Category one: state-block offset `0xB8C`.

If the stored destination equals the candidate, the function returns immediately. It performs neither a state write nor a marker write.

This comparison is against the actual destination value just read, not against the dependency-provided starting value.

For every unsupported category, the function returns without this destination-resolution sequence.

### 5. Store a changed value and mark it

If the destination differs from the candidate, the function:

1. Writes the candidate to the selected 32-bit destination.
2. Reads the state-block address at offset `0xF00` again, using the same intermediate object address retained for the destination-resolution sequence.
3. Writes the byte value `1` at offset `0xB90` of this newly read state-block address.

The context field at offset `0xA8` is not read a third time.

The second read of the intermediate object’s state-block pointer is significant. The function does not necessarily mark the same state block whose selected field it just changed. Equality of those addresses requires stability that the supplied source does not enforce.

The marker operation is a one-byte assignment of `1`, not a bitwise update. No marker read or equality check precedes it.

## Side effects

### Direct memory effects

For a supported category, direct reads occur in this order:

1. Context offset `0xA8`, before the selected starting-value dependency.
2. Context offset `0xA8`, after that dependency.
3. Intermediate object offset `0xF00`.
4. Selected 32-bit state field.

Only when the comparison detects a difference do the following occur:

5. Write the selected 32-bit state field.
6. Reread intermediate object offset `0xF00`.
7. Write one byte with value `1` to the resulting block’s marker.

This is the order expressed by the supplied source. It does not establish atomicity, memory barriers, hardware-visible ordering, or thread-safety guarantees.

### Dependency effects

Dependency effects are unknown. In particular, dependencies may modify shared state or context-reachable memory. A no-write return from this function therefore does not establish that the overall call had no side effects.

The context address is not explicitly passed to the two selector dependencies. That alone does not establish that they cannot access the same objects through other references.

### Ownership and lifetime

The function contains no direct allocation, deallocation, retain, release, or ownership-transfer operation.

That observation does not establish ownership behavior inside its dependencies. The source also provides no lifetime protection for addresses read from the context or intermediate object.

### Failure and partial modification

There is no explicit error-handling path. Faults, exceptions, dependency failures, or invalid-memory behavior are not defined by the supplied evidence.

The selected-field write precedes the marker write. Consequently, the source establishes no rollback or all-or-nothing update guarantee. An interruption or failure after the state write may leave the state changed without a completed marker write.

## Dependencies

| Opaque ID | Call condition and order | Input | Result used here |
|---|---|---|---|
| F-270c5b6d8d34 | First call on every invocation | Opaque four-byte selector | 32-bit unsigned mask |
| F-266e368c1191 | Second call on every invocation | Same selector | Signed 32-bit category |
| F-2bd8af926943 | Only for category zero, after the first context-field read | Four-byte value read from context offset `0xA8` | 32-bit unsigned starting value |
| F-3d1cdf97ba16 | Only for category one, after the first context-field read | Four-byte value read from context offset `0xA8` | 32-bit unsigned starting value |

No call to both starting-value dependencies occurs during one invocation.

The dependencies’ internal behavior, validation rules, exceptional behavior, memory ownership, and global-state preservation are **documented_only** unknowns.

## Acceptance examples

These are conditional consequences of the supplied source, not executed tests. Unless stated otherwise, assume all reached addresses are valid, dependencies return normally, and relevant pointers remain stable.

### Representative bit operations

| Category | Starting value | Mask | Operation flag | Stored destination before comparison | Required direct outcome |
|---:|---:|---:|---:|---:|---|
| 0 | `0x0000000A` | `0x00000005` | 1 | `0x0000000A` | Write `0x0000000F` at `0xB30`, then write marker byte `1` |
| 1 | `0x0000000F` | `0x00000005` | 0 | `0x0000000F` | Write `0x0000000A` at `0xB8C`, then write marker byte `1` |
| 1 | `0x00000000` | `0x80000000` | −1 | `0x00000000` | Write `0x80000000` at `0xB8C`, then write marker byte `1` |
| 0 | `0x12345678` | `0xFFFFFFFF` | 0 | `0x12345678` | Write zero at `0xB30`, then write marker byte `1` |

### Candidate already stored

If category zero produces candidate `0x0000000F` and the destination already contains `0x0000000F`, no direct write occurs. An existing marker byte of zero remains zero unless a dependency changed it.

The starting-value dependency still runs before this early return.

### Zero mask does not guarantee no write

Suppose category one returns starting value `0x00000020`, the mask is zero, and the destination contains `0x00000010`.

The candidate is `0x00000020` for either operation flag class. The function writes that value and marks the state changed. A zero mask does not suppress synchronization from the dependency-provided starting value to the destination.

### Unsupported category

If the category result is two, or a negative value other than the supported values, both selector dependencies run. Neither starting-value dependency runs, and this function performs no direct context read or state write.

This does not establish that an arbitrary selector is valid for the dependencies.

### Context pointer changed by a dependency

Suppose a supported-category starting-value dependency receives one intermediate object address, then changes context offset `0xA8` to another valid intermediate object address.

The later destination comparison and possible state write use the new address. The function does not require the destination to belong to the object passed to that dependency.

### Marker destination changes after the state write

If the intermediate object’s `0xF00` field yields a different valid state-block address on the post-write reread, the selected-field write affects the earlier block, while the marker write affects the later block.

The cause of that change is outside this contract. No supported concurrency protocol is established.

## Uncertainties

### Established static behavior

The supplied source establishes the dependency-call order, exact category tests, 32-bit bit operations, field offsets, fresh pointer reads, comparison-based early return, and state-write-before-marker sequence.

It also establishes the absence of direct validation, explicit allocation, and explicit error reporting within this function.

### Inferred intent

Interpreting the two selected fields as flag collections and the byte at `0xB90` as a dirty or change marker is reasonable but remains an inference. No downstream consumer is supplied to confirm those meanings.

### documented_only unknowns

The following remain unresolved:

- Selector encoding and the semantic meanings of mask bits.
- Dependency contracts beyond how their results are consumed here.
- Whether starting values normally match destination values.
- Whether either category is guaranteed for valid selectors.
- Concrete object types, address validity, alignment, and byte order.
- Aliasing constraints among context, intermediate objects, and state blocks.
- Concurrency, reentrancy, atomicity, and synchronization requirements.
- Ownership or lifetime changes performed by dependencies.
- Fault behavior and recovery from partial updates.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping. Dependency implementations, callers, surrounding type definitions, and downstream state consumers were not provided.

The behavioral claims describe the supplied source-level access and control-flow sequence; they are not observations of execution or guarantees about an unspecified compiler or hardware memory model.

No runtime tests have been performed. The acceptance examples are derived conditional expectations, not measured results.
