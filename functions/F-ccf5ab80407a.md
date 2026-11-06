<!-- Trusted provenance: F-ccf5ab80407a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ccf5ab80407a

## Purpose

F-ccf5ab80407a selects one 44-byte record from a caller-supplied pool, updates the pool’s bookkeeping, and partially initializes the selected record.

Selection has two priorities:

1. Reuse the record named by a non-null reusable-record head.
2. Otherwise, consume the next record from the current sequential storage region, allocating a new region if the sequential cursor equals its endpoint.

The selected record is not returned. Its selection is reflected in bookkeeping changes and writes to the record.

**Established static evidence:** the function manages a linked collection of storage-region descriptors, a reusable-record head, a sequential cursor, and a sequential endpoint. It clears specific fields of the selected record.

**Inferred intent:** this is a record-pool acquisition or preparation operation. The reusable-record head appears to represent a free list, and the sequential cursor appears to identify storage not yet consumed from the current region. These interpretations do not establish a release protocol, record type, or ownership policy.

## Interface

### Input and result

The function accepts one address identifying a writable control structure. It has no explicit return value.

The visible representation is 32-bit:

- Addresses stored in control fields and region descriptors occupy four bytes.
- Capacity values and growth calculations use unsigned 32-bit quantities.
- Each record occupies 44 bytes.
- Each region descriptor occupies 12 bytes.

The supplied evidence does not establish a portable implementation for different pointer widths.

### Control-structure layout

The function accesses four consecutive four-byte fields:

| Byte offset | Role |
|---|---|
| 0 | Address of the current region descriptor, or zero |
| 4 | Address of the first reusable record, or zero |
| 8 | Sequential cursor |
| 12 | Sequential endpoint |

The control structure must provide at least 16 accessible bytes. Fields that the chosen execution path updates must be writable.

The cursor and endpoint are compared for equality. No ordering or range comparison is performed.

### Region-descriptor layout

A region descriptor has three four-byte fields:

| Byte offset | Role |
|---|---|
| 0 | Link to the previously current descriptor |
| 4 | Unsigned 32-bit record capacity |
| 8 | Address of the region’s record storage |

A newly allocated descriptor is requested with a size of 12 bytes.

### Record layout touched by this function

For the selected record, the following locations are written:

| Byte offset | Width | Final value |
|---|---:|---:|
| 4 | 4 bytes | Zero |
| 8 | 4 bytes | Zero |
| 12 | 4 bytes | Zero |
| 0 | 1 byte | Zero |

On the reusable-record path, the four-byte value at record offset 12 is first read as the next reusable-record address.

Bytes 1–3 and 16–43 are not directly initialized by this function. In the ordinary non-overlapping case, they retain their prior contents, whatever those contents are.

### Preconditions

For defined, successful use, the caller must supply:

- A valid control-structure address.
- Valid readable pointers for every object traversed on the selected path.
- Writable storage for every destination used on that path.
- Sufficient alignment for the four-byte accesses required by the target environment.
- Successful, suitably aligned allocation results when new storage is needed.
- A coherent pool state if the caller expects records to remain within allocated regions.

The function does not validate these conditions. It also does not establish synchronization for concurrent access.

## Behavior

### Selection priority

The function first reads the reusable-record head at control offset 4.

If that address is nonzero, it is the selected record. The function reads the four-byte next-link value at selected-record offset 12 and stores that value into the reusable-record head.

This path does not directly read or modify the current region descriptor, sequential cursor, or sequential endpoint. It does not invoke an allocation dependency.

If the reusable-record head is zero, the function reads the sequential cursor and compares it with the sequential endpoint.

- If they differ, the cursor address is the selected record.
- If they are equal, a new region is allocated, and the base of that region becomes the selected-record address.

After either sequential-path selection, the sequential cursor is written with the selected-record address plus 44 bytes. This cursor update occurs before record initialization.

### Capacity selection for a new region

When cursor and endpoint are equal, capacity depends on the current region-descriptor address.

If that address is zero, the new capacity is 32 records.

Otherwise, the function reads the unsigned 32-bit capacity at offset 4 of the current descriptor. It computes two unsigned 32-bit candidates:

- The previous capacity plus 32.
- The previous capacity plus its value shifted right by one bit, plus its value shifted right by three bits.

The larger unsigned candidate is selected. Equality selects the shifted-sum candidate, although the numerical capacity is the same in that case.

The shifts operate on the unsigned capacity. For ordinary non-overflowing values, they contribute the integer quotients obtained by dividing the previous capacity by two and by eight.

**Width requirement:** the additions are 32-bit unsigned arithmetic. The comparison is between the resulting 32-bit candidates, not between unbounded mathematical growth expressions. There is no overflow rejection, saturation, or minimum-capacity correction after wrapping.

The requested record-storage size is the selected capacity multiplied by 44 using the visible 32-bit calculation. This multiplication can also wrap. A mathematically large capacity therefore does not imply a correspondingly large allocation request.

### Allocation and publication order

New-region creation uses two calls to F-871e64ddc6de, in this order:

1. Request a 12-byte descriptor, with a second argument of zero.
2. Request record storage whose byte count is the selected capacity multiplied by 44, with a second argument of zero.

There are no direct writes to the new descriptor between these calls.

After both calls return, the function performs these direct writes in order:

1. Store the second allocation result into the new descriptor’s storage-address field.
2. Read the control structure’s current region-descriptor field and store that value into the new descriptor’s predecessor field.
3. Store the selected capacity into the new descriptor’s capacity field.
4. Publish the new descriptor address into the control structure’s current-descriptor field.
5. Read the storage address back from the new descriptor.
6. Store that address plus the selected capacity multiplied by 44 into the sequential endpoint.
7. Store the selected-record address plus 44 into the sequential cursor.

The predecessor value is read after both allocation calls. It is not a saved copy of the descriptor address used earlier for the capacity decision.

Consequently, if either dependency call changes the current-descriptor field, the new descriptor links to the value present at the later read. The visible body does not protect against that change or require the dependency to preserve control state.

The selected capacity, however, has already been determined before the allocation calls. It is not recomputed from a descriptor installed during either call.

### Selected-record initialization

After selection and bookkeeping updates, the function performs four writes, in this order:

1. Clear the four-byte field at record offset 4.
2. Clear the four-byte field at record offset 8.
3. Clear the four-byte field at record offset 12.
4. Clear the single byte at record offset 0.

On the reusable-record path, the next link is consumed before it is cleared.

This is partial initialization, not a 44-byte zero-fill. No other record fields are directly written.

### Boundary and malformed-state behavior

The following cases are not rejected:

- A cursor greater than its endpoint is treated like any other unequal cursor: it is selected and advanced.
- A zero cursor with a nonzero endpoint reaches record initialization using a zero selected-record address.
- Equal cursor and endpoint trigger allocation regardless of whether their common value is zero.
- A nonzero reusable-record head takes precedence even when the sequential region is exhausted.
- A zero or wrapped capacity can produce a zero or otherwise undersized storage request.

The function has no explicit error return, rollback, or cleanup path. Invalid accesses or dependency failures may prevent normal completion. The precise platform-level failure behavior is not established.

## Side effects

### Direct state changes

On a successful reusable-record path:

- The reusable-record head becomes the selected record’s former next link.
- The selected record’s fields at offsets 4, 8, and 12 become zero.
- The selected record’s first byte becomes zero.

On a successful sequential path without allocation:

- The cursor advances by 44 bytes.
- The same selected-record fields are cleared.
- The current descriptor, reusable-record head, and endpoint receive no direct writes.

On a successful new-region path:

- Two allocation requests occur.
- The new descriptor is initialized and published.
- The endpoint and cursor are updated.
- The selected record is partially initialized.

These statements describe direct effects. Dependency side effects are additional and unspecified.

### Ownership and lifetime

The function performs no explicit deallocation and does not unlink or destroy earlier descriptors. It stores allocation results into persistent pool bookkeeping.

**Inferred intent:** the descriptor chain retains access to earlier storage regions for later management.

**Unknown:** which component owns those allocations, when they are released, and whether F-871e64ddc6de transfers ownership in an allocator-like manner. No release protocol can be inferred solely from these stores.

### Atomicity and aliasing

The operation is not atomic. Bookkeeping changes precede record initialization, and no lock, transaction, or rollback is visible.

For ordinary pool behavior, control storage, descriptors, and record storage need appropriate non-overlap. The function contains no alias checks. If these objects overlap, a record-clear operation can overwrite bookkeeping written earlier, or a descriptor write can alter a later read. Such cases must follow the stated access order rather than an assumption that all logical fields remain independent.

## Dependencies

### F-871e64ddc6de

**Visible invocation contract:**

- Called only when the reusable-record head is zero and cursor equals endpoint.
- Called twice, sequentially.
- First call receives 12 and zero.
- Second call receives the 32-bit capacity-times-44 result and zero.
- Return values are used as storage addresses.

**Required for ordinary successful behavior:** the returned addresses must support the subsequent descriptor and record accesses.

**Not established:**

- Whether the dependency is a heap allocator.
- Whether requests are zero-filled.
- Whether zero-size requests are permitted.
- Whether allocation failure returns zero, terminates execution, or follows another convention.
- Whether it changes globals, pool bookkeeping, or other memory.
- Whether it retains, transfers, or releases ownership.

The caller performs no explicit failure check on either result. If both calls return but the descriptor result is unusable, failure may occur during descriptor initialization. If the storage result is unusable, descriptor publication and cursor/endpoint changes may occur before record initialization fails.

## Acceptance examples

These examples are conditional on valid, non-overlapping storage and successful dependency behavior unless stated otherwise. They are static contract examples, not executed tests.

### Reusable record takes precedence

Given:

- A nonzero reusable-record head.
- A valid next-link value at that record’s offset 12.
- Any cursor and endpoint values.

Expected behavior:

- No allocation call occurs.
- The reusable-record head becomes the saved next-link value.
- The cursor and endpoint receive no direct writes.
- The selected record’s four specified fields are cleared in the documented order.
- Other record bytes remain unchanged by direct writes.

### Existing sequential storage is consumed

Given:

- A zero reusable-record head.
- Cursor and endpoint differ.
- The cursor identifies a writable record.

Expected behavior:

- No allocation call occurs.
- The cursor advances by 44 bytes before record initialization.
- The endpoint and current descriptor receive no direct writes.
- Only the specified record fields are cleared.

### First region is created

Given:

- A zero reusable-record head.
- Equal cursor and endpoint.
- A zero current-descriptor address.
- Dependency calls that do not alter the control structure.

Expected behavior:

- Capacity is 32.
- Requests are made for 12 bytes and 1,408 bytes, each with second argument zero.
- The new descriptor’s predecessor is zero.
- The endpoint becomes storage base plus 1,408 bytes.
- The cursor becomes storage base plus 44 bytes.
- The first record is partially initialized.

### Growth selects the larger candidate

With an existing capacity of 32 and no overflow:

- Additive growth gives 64.
- Shifted-sum growth gives 52.
- New capacity is 64, requiring 2,816 storage bytes.

With an existing capacity of 64:

- Additive growth gives 96.
- Shifted-sum growth gives 104.
- New capacity is 104, requiring 4,576 storage bytes.

### Unsigned wrapping affects capacity

With an existing capacity of 4,294,967,280:

- The additive candidate wraps to 16.
- The shifted-sum candidate wraps to 2,684,354,534.
- The latter is selected.
- Its multiplication by 44 wraps to a request size of 2,147,482,504 bytes.

No explicit arithmetic-error path is taken.

### Dependency changes the descriptor head

Given that an allocation dependency changes the current-descriptor field before returning:

- The previously computed capacity remains in use.
- The new descriptor’s predecessor is the current-descriptor value read after both allocation calls.
- The contract does not require that predecessor to equal the descriptor used for capacity selection.

## Uncertainties

The following are **documented_only**:

- Semantic names and complete layouts of the control object and record.
- The meaning of the byte cleared at record offset 0.
- Allocation failure conventions and dependency side effects.
- Ownership, release, and destruction rules.
- Thread-safety guarantees supplied outside this function.
- Endianness and platform fault behavior.
- Intended capacity limits and whether external checks prevent arithmetic wrapping.
- Whether malformed or overlapping pool states are permitted by a larger interface.

The visible arithmetic supports documenting 32-bit wrapping of unsigned growth and size calculations. It does not establish that wrapped pointer-address calculations produce valid storage accesses.

The access order described here is the order represented by the supplied body, not an independently verified machine-instruction trace or concurrent memory-order guarantee.

## Evidence

The supplied static body establishes:

- Reusable-record selection before sequential allocation.
- Four four-byte control fields and three four-byte descriptor fields.
- A 44-byte record stride.
- Unsigned 32-bit capacity growth and size calculations.
- Two dependency calls on the exhausted-sequential path.
- Descriptor publication before selected-record initialization.
- A next-link read before clearing record offset 12.
- Three four-byte zero stores followed by one single-byte zero store.
- No explicit return value, validation, deallocation, or rollback.

Pool terminology and lifecycle interpretation are inferred intent, not additional guarantees. Dependency internals are unavailable. No runtime tests have been performed.
