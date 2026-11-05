<!-- Trusted provenance: F-da91d86fdee5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da91d86fdee5

## Purpose

F-da91d86fdee5 updates mutable bookkeeping using a 56-byte stride. It has two mutually exclusive paths:

- When a recycled-entry head is nonzero, replace that head with the first 32-bit word of the referenced entry, then return.
- Otherwise, advance a sequential cursor by 56. If the cursor equals its recorded limit, first obtain two dependency results, initialize a header, and update the state to describe a new region.

There is no explicit return value. Neither the recycled-entry address nor the sequential position is returned.

**Inferred intent:** the state represents a record pool with recycled entries, a sequential region, and linked backing blocks. This interpretation fits the accesses but does not establish record initialization, ownership transfer, allocation units, or a larger allocation protocol.

The behavior below is derived from the supplied static representation. Acceptance examples are conditional deductions; no runtime tests have been performed.

## Interface

### Inputs and widths

The function accepts:

1. A pointer to mutable state containing four consecutive 32-bit fields.
2. A signed 32-bit mode value, examined only for equality with zero.

The representation uses 32-bit integers as address-like values. This contract describes that model; it does not establish a portable interface for arbitrary pointer widths.

### State layout

| Byte offset | Contract name | Observed role |
|---|---|---|
| 0 | Block head | Address-like value used for capacity lookup and predecessor linking |
| 4 | Recycled head | Zero or an address-like value referencing a recycled entry |
| 8 | Cursor | Compared with the limit and advanced by 56 |
| 12 | Limit | Compared with the cursor and replaced on growth |

These names describe observed roles, not independently verified type declarations.

The cursor and limit are distinct, non-overlapping 32-bit fields within the same state object. Overlapping a separate header with that object does not make these two fields overlap.

### Header layout

A header has the following 12-byte accessed footprint:

| Byte offset | Contract name | Observed role |
|---|---|---|
| 0 | Previous block | Receives a block-head value read during linking |
| 4 | Capacity | Receives the selected count; an existing header’s corresponding field supplies growth sizing |
| 8 | Storage base | Receives the second dependency result and is subsequently read back |

The footprint does not establish the total size of a dependency-provided object.

A recycled entry needs a readable first 32-bit word. That word supplies the replacement recycled head. No other part of the entry is explicitly accessed.

### Outputs and preconditions

There is no explicit result or success indicator. Observable outputs are memory updates and, on growth, two calls to F-871e64ddc6de.

Ordinary operation requires:

- At least 16 bytes of suitably aligned, readable and writable state storage.
- A readable 32-bit word at any recycled-entry address that is used.
- A readable capacity field at offset 4 of any existing header consulted for sizing.
- A first dependency result that supports the header accesses.
- Meaningful address representations and valid arithmetic for every access performed.

The function does not validate these requirements. It contains no explicit alignment check, bounds check, chain validation, synchronization, or protection against dependency-induced mutation.

Disjoint storage and dependency preservation of state are useful conditions for simple postconditions, not guarantees supplied by this function.

## Behavior

### Recycled-entry precedence

The recycled head determines the first branch.

If it is nonzero:

1. Read the first 32-bit word at the referenced address.
2. Store that word in the recycled-head field.
3. Return immediately.

The cursor, limit, block capacity, and mode do not affect this path. There are no dependency calls.

The only explicit write is to the recycled-head field. The function does not clear the entry, return its address, or advance the cursor. The block head, cursor, and limit are not explicitly written and occupy storage distinct from the recycled-head field. If the referenced entry aliases state storage, its first-word read can obtain a state-field value; the destination remains the recycled-head field.

### Sequential path without growth

If the recycled head is zero, compare the cursor and limit for equality.

When they differ:

- Do not consult the block head or any header capacity.
- Do not call F-871e64ddc6de.
- Read the cursor, add 56 using the represented signed addition, and store the result in the cursor field.
- Do not explicitly write any other state field.

Equality is the only exhaustion test. A cursor numerically beyond the limit still takes this path if the two values differ. There is no check that 56 units remain before the limit.

### Growth and count selection

Growth occurs only when the examined recycled head is zero and the compared cursor and limit are equal.

The selected count is an unsigned 32-bit value:

| Condition | Selected count |
|---|---|
| Mode is nonzero | 1 |
| Mode is zero and block head is zero | 32 |
| Mode is zero and block head is nonzero | Larger of the two capacity-derived candidates below |

Every nonzero mode value, including a negative value, has the same direct sizing effect. With nonzero mode, no existing capacity is read for sizing.

For zero mode and a nonzero block head, read the unsigned 32-bit capacity at header offset 4. Let that value be \(C\). The candidates are:

- Additive: \(C+32\).
- Proportional: \(C+\lfloor C/2\rfloor+\lfloor C/8\rfloor\).

Each quotient is truncated separately. All candidate additions use unsigned 32-bit arithmetic, wrapping modulo \(2^{32}\). Compare the resulting candidates as unsigned values and select the larger. A tie produces their common numeric value.

For example, capacity 5 gives proportional candidate \(5+2+0=7\), while the additive candidate is 37. The selected count is 37.

The calculation is not a monotonic mathematical growth guarantee: unsigned wraparound can reduce either candidate.

### Dependency calls

After selecting count \(N\), call F-871e64ddc6de twice:

| Call order | First argument | Second argument | Subsequent use of result |
|---|---:|---:|---|
| First | 12 | 0 | Header address |
| Second | \((56N)\bmod 2^{32}\) | 0 | Storage-base value |

These are numeric request arguments. Their units are not established by this caller. The first argument 12 and the header’s 12-byte accessed footprint are separate observations.

The count is selected before either call and is not recalculated afterward. The second call is reached only if the first returns normally, but the first result is not checked before attempting the second call.

The function does not recheck the recycled head, cursor, or limit after dependency effects.

### Header initialization and state publication

After both calls return normally, the following explicit memory operations occur in order:

1. Store the second result in the new header’s storage-base field.
2. Read the state’s current block head and store it in the new header’s previous-block field.
3. Store the selected count in the new header’s capacity field.
4. Store the new header address in the state’s block-head field.
5. Read the new header’s storage-base field.
6. Store that read-back value in the state’s cursor field.
7. Store that same read-back value plus the unsigned count-times-56 product in the state’s limit field.
8. Read the state’s cursor, add 56 using the represented signed addition, and store the result in the cursor field.

The predecessor read occurs after both dependency calls and after the storage-base write. Consequently, it need not equal the entry-time block head or the header address used for capacity lookup. Dependency mutation can change it; header/state overlap can also make the preceding storage-base write affect the value read.

The storage base is read back after publication of the new block head. It must not be replaced in the contract by an unconditional assumption that it still equals the second dependency result. Earlier writes can affect that field when storage overlaps.

The final increment reads the cursor field again. However, the intervening limit write targets a different, non-overlapping state field. It cannot change the cursor through ordinary storage overlap, even when the header overlaps the state. With no external interference or invalid execution, the final increment therefore uses the value just assigned to the cursor in step 6.

Header/state overlap can still change other outcomes. For example, a header field may coincide with a state field and be overwritten by a later state update. The simple disjoint-object postconditions below do not apply to such cases.

### Ordinary disjoint-object postconditions

Assume valid disjoint state and header storage, no relevant dependency mutation, and representable address arithmetic. Let the new header address be \(H\), the second result be \(B\), and the selected count be \(N\).

Completion establishes:

- Block head = \(H\).
- Header predecessor = the block head read during linking.
- Header capacity = \(N\), subject to the represented signed-field conversion.
- Header storage base = \(B\).
- Cursor = \(B+56\).
- Limit = \(B+((56N)\bmod 2^{32})\), subject to the represented conversion into the signed state field.
- Recycled head remains zero.

When the product does not wrap and all relevant values are representable, the limit simplifies to \(B+56N\). With \(N=1\), cursor and limit are equal immediately after completion.

### Arithmetic boundaries

Unsigned capacity calculations and multiplication by 56 have established modulo-\(2^{32}\) behavior.

The final cursor increment is represented as signed 32-bit addition. The supplied representation does not establish a portable result for signed overflow.

The limit expression combines the signed storage-base value with an unsigned product. The unsigned arithmetic can be described modulo \(2^{32}\), but storing an out-of-range result in a signed field does not establish a portable signed value. Similarly, storing an unsigned count above the signed maximum into the header’s signed field leaves the portable conversion result unresolved.

These numeric rules do not establish that arbitrary wrapped values are valid addresses. Address conversion, the capacity-field address calculation, and subsequent memory access require separate environmental validity.

## Side effects

### Explicit memory effects

Depending on the branch, the function:

- Reads and replaces the recycled head.
- Reads the first word of a recycled entry.
- Reads the cursor and limit.
- Reads the block head and possibly an existing capacity.
- Writes three new-header fields.
- Publishes the new block head.
- Reads back the header’s storage base.
- Replaces the cursor and limit.
- Advances the cursor by 56.

There are no explicit writes through the storage-base result to initialize record storage. Advancing the bookkeeping does not establish that a record is cleared or initialized.

There are no explicit releases, destructor calls, or cleanup operations.

### Dependency effects and ownership

F-871e64ddc6de’s effects are unknown. It must not be assumed to preserve globals, the input state, existing headers, or reachable memory.

Retaining its results is consistent with pool-managed storage, but does not establish ownership transfer or release obligations. No older block is explicitly released when a new one is linked.

### Failure and partial progress

The function has no explicit error return, recovery branch, or rollback.

If a dependency call does not return, subsequent operations are not reached. A zero first result does not prevent the second call; after both calls return, the function attempts header writes through the first result.

A zero second result is stored without rejection. The function does not explicitly dereference it as record storage, although the resulting state may be unusable.

The block head is published before the cursor and limit updates finish. No visible synchronization or transactional mechanism protects intermediate state. Invalid accesses or dependency failures can therefore leave partial effects; the precise fault behavior is not established.

## Dependencies

### F-871e64ddc6de

**Established from this caller:**

- Not called on the recycled-entry path.
- Not called when cursor and limit differ.
- Called first with arguments 12 and 0.
- Called second with the unsigned 32-bit count-times-56 product and 0, if the first call returns normally.
- First result used as a header address.
- Second result stored as a storage-base value.
- Neither result explicitly checked for success.

**Unknown contract, retained as `documented_only`:**

- Argument meanings and units.
- Whether either request acquires memory.
- Alignment, extent, and non-overlap guarantees.
- Handling of a zero request.
- Failure signaling and whether failures return.
- Initialization of returned storage.
- Mutation of globals or caller-accessible storage.
- Ownership and eventual release obligations.

Calling this dependency an allocator is inferred intent, not established behavior.

## Acceptance examples

These examples are conditional static deductions, not execution results. Unless stated otherwise, assume valid disjoint objects, no relevant dependency mutation, and representable arithmetic.

### Recycled entry overrides exhaustion

Given a nonzero recycled head referencing a word containing a next-entry value, replace the head with that value and return.

Even if cursor equals limit and mode is nonzero, no dependency call occurs. Cursor, limit, and block head remain unchanged.

### Unequal cursor and limit

Given a zero recycled head, cursor \(B\), and a different limit, the cursor becomes \(B+56\).

No dependency call occurs. The same rule applies when \(B\) is numerically beyond the limit, provided the signed addition is representable.

### First zero-mode block

Given zero recycled head, equal cursor and limit, zero block head, and zero mode:

- Select count 32.
- Use first arguments 12 and 1,792 for the two calls.
- Write zero as the predecessor if the dependency leaves the block head unchanged.
- With storage-base result \(B\), finish with cursor \(B+56\) and limit \(B+1,792\).

The request argument 1,792 is not asserted to denote bytes.

### Nonzero-mode block

Given an exhausted sequential state and any nonzero mode:

- Select count 1 without reading an existing capacity for sizing.
- Use first arguments 12 and 56.
- With ordinary successful completion, finish with cursor and limit both equal to \(B+56\).

The limit write does not advance the cursor. The subsequent explicit cursor increment does.

### Capacity candidates

For zero mode with an existing header:

| Existing capacity | Additive candidate | Proportional candidate | Selected count | Second request argument |
|---:|---:|---:|---:|---:|
| 32 | 64 | 52 | 64 | 3,584 |
| 128 | 160 | 208 | 208 | 11,648 |
| 52 | 84 | 84 | 84 | 4,704 |

Equal candidates require no distinct externally observable outcome.

### Unsigned wraparound

For existing capacity 4,294,967,295:

- Additive candidate wraps to 31.
- Proportional candidate wraps to 2,684,354,557.
- Unsigned comparison selects 2,684,354,557.
- Multiplication by 56 wraps to 4,294,967,128.

This establishes sizing arithmetic only. It does not establish dependency acceptance, valid resulting addresses, or portable signed-field conversions.

### Dependency changes the block head

Suppose a dependency changes the block head, both calls return normally, and header writes do not overlap the state. The predecessor receives the block-head value read after both calls, not necessarily the entry-time value.

### Header overlap does not merge cursor and limit

Suppose header storage overlaps the state while the state itself remains a valid four-field object. Earlier header writes and block-head publication may affect the storage-base read-back.

Once that read-back value has been assigned to the cursor, the following limit write still targets offset 12 rather than offset 8. Without external interference, the final cursor increment uses the assigned read-back value. Header/state overlap alone does not permit the limit write to overwrite the cursor.

## Uncertainties

The following remain `documented_only`:

- The larger caller protocol and how a consumer obtains the position associated with this update.
- Authoritative ABI, pointer representation, and platform conversion rules.
- Signed-overflow behavior and out-of-range signed conversions.
- Dependency semantics, request units, failures, initialization, mutation, and ownership.
- Whether overlapping objects, callbacks, or concurrent mutation are permitted.
- Whether inconsistent cursor and limit values occur in legitimate use.
- Whether intermediate publication is observable in the intended environment.
- Release and reuse rules for headers and backing storage.

The inferred pool interpretation adds no returned pointer, validation, initialization, synchronization, or cleanup guarantee.

## Evidence

The supplied static representation establishes the branch precedence, four-field state layout, three-field header footprint, 56-byte stride, capacity formula, unsigned sizing calculations, numeric dependency arguments, call order, and explicit memory-update sequence.

The same layout establishes that cursor and limit occupy distinct 32-bit fields. The final increment rereads the cursor, but the preceding limit write cannot alter that cursor through ordinary overlap of a header with the state.

No dependency body, caller contract, ABI documentation, or runtime trace was provided. Prior prose and reviewer feedback were not treated as independent implementation evidence.

No runtime tests have been performed. All acceptance examples are conditional deductions from static evidence.
