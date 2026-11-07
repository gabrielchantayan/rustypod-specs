<!-- Trusted provenance: F-03a561fa0ffb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-03a561fa0ffb

## Purpose

F-03a561fa0ffb conditionally transfers selected fields between indexed positions associated with an object. F-13e8c44c016a supplies a source index, a destination index, and a signed transfer count. The wrapper selects descending traversal when the source index is lower than the destination index, and ascending traversal when the destination index is lower.

Each executed transfer copies two 32-bit fields and the low bit of a byte, then clears the source’s first 32-bit field and entire byte. It does not explicitly clear the source’s second 32-bit field.

**Inferred intent:** traversal direction is consistent with moving overlapping ranges without overwriting fields needed by later transfers. Source clearing suggests transfer-like rather than ordinary copy semantics. Neither ownership transfer nor a particular container abstraction is established.

The transfer count is not necessarily fixed throughout execution. Each loop condition reads the count again. Its address has been exposed to F-13e8c44c016a, and the source does not establish that subsequent opaque calls cannot modify it.

## Interface

### Inputs

The function receives:

1. **Object reference.** Passed unchanged to F-13e8c44c016a and every indexed accessor invocation. When resolving an indexed record, the wrapper reads the object’s first word as a dispatch-table address representation.
2. **First opaque 32-bit argument.** Forwarded unchanged to F-13e8c44c016a.
3. **Second opaque 32-bit argument.** Forwarded unchanged to F-13e8c44c016a.

The wrapper does not otherwise interpret the two opaque arguments. Their semantic types and valid domains are unknown.

### Output locations supplied to F-13e8c44c016a

The preliminary dependency receives the original three arguments followed by addresses of three wrapper-local, signed 32-bit output locations, in this order:

1. Source index.
2. Destination index.
3. Transfer count.

The wrapper does not initialize these locations before the call and does not consume the dependency’s return value.

Initialization requirements are path-dependent:

- The count must be initialized before the wrapper’s first condition reads it.
- If that count is zero, neither index is read by the wrapper. Initialized indices are therefore unnecessary for this wrapper’s defined execution on that path.
- If the count is nonzero, both indices must be initialized because the wrapper compares them.

These are requirements on values actually read, not an unconditional requirement that the dependency initialize all three outputs.

The source does not establish that the dependency discards these addresses. In particular, it could retain the count’s address in state accessible to later calls during this invocation.

### Return value

The function returns no value and provides no explicit success, failure, or completed-transfer count.

### Accessed record layout

Each indirect accessor returns an address used with this layout:

| Byte offset | Width | Explicit treatment |
|---|---:|---|
| 0 | 32 bits | Read from source, written to destination, then zeroed at source |
| 4 | 32 bits | Read from source and written to destination |
| 8 | 8 bits | Source bit 0 written as the entire destination byte, then entire source byte zeroed |

The two 32-bit fields are opaque bit patterns. No signed interpretation of their contents is visible.

Only nine bytes of each returned region are directly accessed. This does not establish the complete record size, indexing stride, padding, or layout beyond offset 8.

### Preconditions for defined normal execution

The wrapper performs no visible validation. The selected path requires:

- Initialization of every output value actually read.
- Normal return from dependencies that are reached.
- Valid dispatch-table reads and callable entries.
- Valid readable and writable record storage for every field operation reached.
- Any alignment required by the execution environment.
- Defined results for every evaluated signed arithmetic operation.
- Externally satisfied lifetime, bounds, aliasing, and synchronization requirements.

The displayed representation uses signed 32-bit indices, count, and iteration counters. It does not establish a portable pointer ABI or a general rule for signed overflow.

## Behavior

### Preliminary call and initial branching

F-13e8c44c016a is called exactly once before wrapper-controlled indexed access.

After that call returns, the wrapper reads the count:

- **Zero:** return without reading either index or directly accessing any record.
- **Nonzero:** compare the signed source and destination indices.
- **Equal indices:** perform no indexed accesses or record transfers, regardless of the nonzero count.
- **Source below destination:** select descending traversal.
- **Destination below source:** select ascending traversal.

There is no error report for a negative count. With a negative count and unequal indices, no transfer executes, although descending traversal performs arithmetic before its first terminating comparison.

Nothing establishes that the preliminary call preserves object metadata, records, globals, or resources.

### Descending traversal

The wrapper initializes two private index cursors by adding the then-current count to the source index and destination index respectively. No opaque call occurs between the initial count check and this setup.

Before every loop condition, including the first and the terminating condition, it decrements both cursors by one. It then compares the current count against the number of transfers already counted. A transfer executes only when that counter is less than the current count.

For a stable positive count, the first pair is the source and destination indices each increased by count minus one. Subsequent pairs descend by one.

The cursor setup is not repeated when the count changes. In particular:

- Reducing the count can terminate traversal earlier.
- Increasing it can extend traversal below the original source and destination indices.
- The next condition still decrements both cursors before deciding whether to stop.

The completed-transfer counter is incremented after the two 32-bit destination writes and before the byte transfer and source clearing. This internal timing does not add an accessor invocation or field access.

### Ascending traversal

The wrapper initializes its cursors directly from the source and destination indices. Before each iteration, it reads the current count and compares it with the number of completed iterations.

When the comparison permits entry, it resolves both records and performs the transfer. Both cursors advance by one during the iteration, including the final executed iteration. The iteration counter advances after the field operations.

Changing the count does not reset the cursors. Increasing it can extend traversal above the original count’s implied range; decreasing it can stop traversal at the next condition.

### Count rereads and call-count guarantees

Every loop condition rereads the transfer-count location supplied to F-13e8c44c016a. It is not a private immutable snapshot.

For example, the preliminary dependency could retain the count’s address, and an indirect accessor could subsequently modify it. The source provides no preservation guarantee that excludes this behavior.

Once an iteration has begun, no further count check occurs between its two accessor calls or before its field operations finish. A count change during either accessor therefore affects a later condition, not admission to the current iteration.

Only under an explicit **count-stability assumption** does an initial positive count of N, together with unequal indices and normal defined completion, imply exactly N transfers and 2N accessor calls.

Under the same assumption, the fixed visit sequences are:

| Relationship | Source visits | Destination visits |
|---|---|---|
| Source below destination | Source plus N minus one, descending through source | Destination plus N minus one, descending through destination |
| Destination below source | Source, ascending through source plus N minus one | Destination, ascending through destination plus N minus one |

Without count stability, neither those complete ranges nor the 2N call count is guaranteed. No independent fixed upper bound is imposed; arithmetic and storage validity remain necessary throughout any extended traversal.

### Resolving the two records

Every admitted iteration invokes the indirect accessor twice:

1. With the original object reference and current source cursor.
2. With the original object reference and current destination cursor.

Each invocation resolves its target through the object’s current dispatch-table address and the callable entry at byte offset 0x10. The wrapper performs a fresh dispatch lookup for the destination call after the source call returns. It does not retain one target for the whole operation.

The source address is retained across the destination call. No source record field is read until both calls return.

Consequences:

- The destination call may use a different dispatch target if preceding activity changed dispatch state.
- The copied values are those read after both calls, not necessarily values present at entry or at source-address resolution.
- If the destination call invalidates the retained source address, the wrapper has no visible protective mechanism.
- Count changes during these calls do not cause immediate cancellation of the admitted iteration.

### Ordered field operations

After both addresses have been obtained, the externally relevant record operations occur in this order:

1. Read 32 bits from source offset 0; write those bits to destination offset 0.
2. Read 32 bits from source offset 4; write those bits to destination offset 4.
3. Read the byte at source offset 8; write its low bit, as a byte value of 0 or 1, to destination offset 8.
4. Write a zero byte to source offset 8.
5. Write a 32-bit zero to source offset 0.

The wrapper does not snapshot all source fields before writing the destination. Earlier destination writes can therefore affect later source reads when the returned regions overlap.

It performs no explicit source-offset-4 clearing operation.

### Arithmetic boundaries

All displayed index and counter arithmetic is signed 32-bit arithmetic, without visible widening or overflow checks.

Descending traversal evaluates both initial additions and decrements both cursors before every condition, including a condition that terminates without a transfer. This applies even when an initially negative count prevents any transfer.

Ascending traversal increments both cursors during every executed iteration, including the last. Both branches also increment an iteration counter as execution proceeds.

Valid accessed indices alone do not establish valid arithmetic. Setup, final cursor updates, and terminating-condition decrements must also be defined. This contract does not assign wrapping behavior to signed overflow.

## Side effects

### Record mutation

For a completed iteration with stable, distinct, non-overlapping record storage:

- The destination receives the source’s two 32-bit field values as read during that iteration.
- The destination byte becomes 0 or 1 according to source bit 0.
- The source’s first 32-bit field and byte become zero.
- The source’s second 32-bit field is unchanged by explicit source operations.

These are per-iteration effects. Later iterations may overwrite positions cleared earlier.

### Aliasing

Different indices are not checked for distinct returned addresses.

If both accessors return exactly the same record address, a completed iteration leaves the first 32-bit field and byte zero. The second 32-bit field remains unchanged by the wrapper’s field operations. Identical addresses do not make the transfer an identity operation.

For partial overlap, the ordered reads and writes govern the result. Whole-record copy semantics are not guaranteed.

### Ownership and failure

No explicit allocation, deallocation, destructor invocation, reference-count operation, or resource release appears in the wrapper. Clearing a copied field does not establish its type or ownership significance.

Opaque dependencies may independently perform resource operations or mutate shared state. No preservation guarantee is assumed.

There is no visible rollback, error translation, or cleanup path. Failure after some writes can leave partial mutation; the wrapper supplies no restoration mechanism.

## Dependencies

### F-13e8c44c016a

**Established role:** called once with the original arguments and addresses for source index, destination index, and count.

**Unknown:** argument interpretation, validation, failure behavior, which outputs it initializes on each path, retained output addresses, object or global mutation, and resource handling.

Its return value does not control wrapper execution.

### Indirect accessor — identity documented_only

No opaque function ID is supplied for the dynamically selected target; none is invented here.

**Established role:** receives the object reference and a signed index; returns an address used for record accesses.

**Dispatch:** entry at byte offset 0x10 of the object’s freshly read dispatch table.

**Ordering:** source call before destination call for each admitted iteration.

**Unknown:** valid index domain, returned-address lifetime, aliasing guarantees, allocation, error behavior, synchronization, and preservation of the count or other state.

## Acceptance examples

These are conditional static consequences, not observed executions. Unless stated otherwise, dependencies return normally, arithmetic and storage accesses are valid, records have stable non-overlapping storage, and dependencies add no relevant mutations. Fixed-count examples additionally assume the count remains unchanged.

### Zero count with uninitialized indices

F-13e8c44c016a initializes only the count to zero and leaves both index outputs uninitialized.

The wrapper reads the initialized count and returns. It does not read either index, invoke an accessor, or directly access a record. Index initialization is not required on this path.

### Equal indices

The dependency supplies source 7, destination 7, and count 3.

The wrapper reads the count and indices but invokes no accessor. No flag normalization or source clearing occurs.

### Stable descending overlap

The dependency supplies source 2, destination 3, and a stable count of 3.

Accessor pairs are source 4/destination 5, then source 3/destination 4, then source 2/destination 3.

Positions 3 through 5 receive the original two-word contents of positions 2 through 4, with the source flags reduced to bit 0. Position 2 ends with its first word and byte cleared and its second word unchanged. Positions cleared as intermediate sources are subsequently overwritten as destinations.

### Stable ascending overlap

The dependency supplies source 3, destination 2, and a stable count of 3.

Accessor pairs are source 3/destination 2, then source 4/destination 3, then source 5/destination 4.

Positions 2 through 4 receive the original corresponding source fields, subject to flag masking. Position 5 ends with its first word and byte cleared and its second word unchanged.

### Count reduction during an accessor

The dependency supplies source 2, destination 5, and initial count 3. During the first iteration’s source accessor, the count becomes zero.

The first pair remains source 4/destination 7, and the destination accessor and field operations still execute. At the next descending condition, both cursors decrement before the zero count terminates traversal. Exactly one transfer completes under these conditions.

### Count increase without cursor reinitialization

The dependency supplies source 2, destination 5, and initial count 1. During the first iteration, an accessor changes the count to 3, after which it stays stable.

Descending cursor setup used 1, so the pairs are source 2/destination 5, source 1/destination 4, and source 0/destination 3. They are not recomputed as the range that an initial count of 3 would have selected.

### Flag masking and exact aliasing

For distinct records, source byte 0xFE produces destination byte 0; source byte 0xA5 produces destination byte 1. The source byte and first word are then cleared.

If both returned addresses instead identify the same record, that record’s first word and byte finish as zero, while its second word is unchanged by the wrapper.

### Negative count

An initialized negative count and initialized unequal indices produce no accessor calls. Descending selection nevertheless requires defined setup additions and initial decrements. No error status is returned.

## Uncertainties

The following remain **documented_only**:

- Semantic types of the copied words and low flag bit.
- Meaning and accepted domains of the opaque arguments.
- Dependency validation, exceptions, faults, and resource behavior.
- Whether output addresses are retained or used to modify the count.
- Indirect target identity and returned-address guarantees.
- Complete record layout, stride, and pointer ABI.
- Actual execution-environment treatment of signed overflow.
- Bounds, alignment, lifetime, concurrency, reentrancy, and atomicity guarantees.
- Ownership and release obligations.
- Whether overlap-aware movement is an intended public guarantee.

Count stability is an additional assumption for fixed-length behavior, not an established property.

## Evidence

**Established static evidence:** the supplied representation shows the preliminary call, path-dependent output reads, signed direction comparisons, count rereads at every loop condition, one-time descending cursor initialization, fresh indirect dispatch for each call, source-before-destination resolution, ordered field accesses, and absence of an explicit result or rollback path.

**Inferred intent:** overlap-aware movement and transfer-like source clearing. These interpretations do not establish ownership or dependency guarantees.

**Unverified:** no runtime tests have been performed. Acceptance examples describe conditional consequences of the supplied static representation, not measured behavior.
