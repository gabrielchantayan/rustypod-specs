<!-- Trusted provenance: F-59b180f4ecba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-59b180f4ecba

## Purpose

F-59b180f4ecba selects one 24-byte record from a caller-supplied pool state and clears four fields in that record. Selection prefers a linked reusable record. If none is available, the function consumes the next record from the current block, allocating a new block when the current cursor equals the block boundary.

The function returns no value. Its observable results are changes to the pool state, selected record, and—when growth occurs—new block metadata.

The interpretation as a record pool is inferred intent. The field accesses, selection precedence, allocation requests, arithmetic, and writes described below are established by the supplied static evidence. No runtime tests have been performed.

## Interface

### Input and return

The sole input is an address of mutable state. The supplied representation uses four consecutive 32-bit words:

| Byte offset | Contract name | Observed role |
|---|---|---|
| 0 | Block-chain head | Address of the most recently published block descriptor, or zero |
| 4 | Reusable-record head | Address of the next reusable record, or zero |
| 8 | Sequential cursor | Address of the next record to consume from the current block |
| 12 | Sequential boundary | Address compared with the cursor to decide whether growth is necessary |

These names describe observed use, not externally established type definitions.

The function has no explicit return value, output pointer, status code, or error indicator. In particular, it does not directly return the selected record address.

### Block descriptor

A newly allocated descriptor occupies 12 bytes:

| Byte offset | Width | Value written |
|---|---:|---|
| 0 | 32 bits | Block-chain head read from the caller’s state after both allocation calls |
| 4 | 32 bits | Chosen capacity, measured in 24-byte records |
| 8 | 32 bits | Address returned by the payload allocation |

The caller’s block-chain head is then replaced with the address of this descriptor.

An existing head descriptor is consulted at offset 4 to obtain its capacity when calculating growth. Other existing descriptor fields are not traversed by this function.

### Record layout

Each consumed record has a stride of 24 bytes. The function writes only these fields:

| Byte offset | Width | Final value on ordinary completion |
|---|---:|---|
| 0 | 8 bits | Zero |
| 4 | 32 bits | Zero |
| 8 | 32 bits | Zero |
| 12 | 32 bits | Zero |

For a reusable record, the 32-bit value at offset 12 is read as its successor address before being cleared.

Bytes 1–3 and 16–23 are not initialized by this function. The function therefore does not provide whole-record zero initialization.

### Representation and access preconditions

The supplied representation uses 32-bit state words, capacity arithmetic, and stored addresses. The byte order is not established.

Normal completion requires valid readable and writable storage for every accessed location. In particular:

- The state must support the accesses through offset 12.
- A nonzero reusable-record head must identify a record whose successor word is readable and whose cleared fields are writable.
- A nonzero block-chain head consulted during growth must identify readable capacity metadata.
- A sequentially selected record must support the initialization writes.
- Allocation results used during growth must identify suitable writable storage.

The source supplies no validation of these conditions. Alignment requirements, pointer provenance rules, and behavior for invalid addresses are not established.

## Behavior

### 1. Select the source of the record

The function first reads the reusable-record head.

If that value is nonzero, the function selects that record immediately. It does not perform the cursor/boundary exhaustion check or request new storage. It reads the selected record’s 32-bit successor at offset 12 and writes that value into the reusable-record head.

The selected record is then initialized as described below. Its successor field is cleared only after the successor has been copied into the state.

If the reusable-record head is zero, the function reads the sequential cursor and compares it for equality with the sequential boundary.

- Unequal addresses: select the current cursor directly.
- Equal addresses: grow the pool, then select the payload address of the new block.

This is an equality test, not a range or remaining-space check. A cursor beyond the boundary, or a cursor with fewer than 24 usable bytes remaining before the boundary, is not detected as exhausted unless the addresses are equal.

### 2. Calculate capacity when growth is required

Growth occurs only when the initially observed reusable-record head is zero and the cursor equals the boundary.

If the block-chain head is zero, the chosen capacity is 32 records.

Otherwise, let \(c\) be the unsigned 32-bit capacity read at offset 4 of the current head descriptor. Two candidate capacities are calculated:

\[
a = \bigl(c + \lfloor c/2 \rfloor + \lfloor c/8 \rfloor\bigr) \bmod 2^{32}
\]

\[
b = (c + 32) \bmod 2^{32}
\]

The chosen capacity is the unsigned maximum of \(a\) and \(b\). At equality, \(a\) is selected, but the numeric result is identical.

When no overflow occurs, this combines a minimum increase of 32 records with an approximately 1.625-fold growth candidate. That growth interpretation does not apply reliably after unsigned wraparound. There is no saturation, overflow check, or enforced maximum.

The payload allocation size is:

\[
s = (24 \times \text{chosen capacity}) \bmod 2^{32}
\]

The allocation request therefore need not equal the mathematical, unbounded product of capacity and record size.

### 3. Allocate and publish a new block

Growth makes two calls to F-871e64ddc6de, in this order:

1. Request 12 bytes, with second argument zero.
2. Request \(s\) bytes, with second argument zero.

The first result is used as the descriptor address. The second is used as the payload address.

After both calls return, the function performs the following observable metadata operations in the order expressed by the supplied source:

1. Write the payload result to descriptor offset 8.
2. Read the caller’s current block-chain head and write it to descriptor offset 0.
3. Write the chosen capacity to descriptor offset 4.
4. Publish the descriptor address as the caller’s block-chain head.
5. Read the payload address back from descriptor offset 8.
6. Set the sequential boundary to that address plus the 32-bit product \(s\).

The old-head read used for linking occurs after both dependency calls. It is not a saved copy of the head used earlier to calculate capacity. If either call changes the caller’s state, the link can therefore differ from the head whose capacity determined the new size.

The selected record is the payload address read back from the new descriptor.

### 4. Advance sequential state

For either sequential selection path—existing space or a newly allocated block—the function writes the sequential cursor to the selected address plus 24 bytes.

This cursor write occurs before record initialization.

The reusable-record branch does not perform this cursor write.

On an ordinary fresh growth from an empty pool, with no interfering dependency effects, the resulting state is:

- Block-chain head: new descriptor.
- Reusable-record head: unchanged from its initially zero value.
- Sequential cursor: payload address plus 24.
- Sequential boundary: payload address plus the capacity-derived byte size.

### 5. Initialize the selected record

After selection and the applicable state updates, initialization writes occur in this order:

1. Clear the 32-bit field at record offset 4.
2. Clear the 32-bit field at record offset 8.
3. Clear the 32-bit field at record offset 12.
4. Clear the single byte at record offset 0.

The function does not explicitly write the remaining record bytes.

These are the source-expressed access relationships. The evidence does not establish volatile accesses, machine-level atomicity, synchronization, or an instruction-by-instruction memory trace.

## Side effects

### State changes

Under valid, non-overlapping storage and without dependency-induced interference:

- Reusable-record selection updates only the reusable-record head within the state.
- Sequential selection without growth updates only the sequential cursor within the state.
- Growth updates the block-chain head, sequential boundary, and sequential cursor.
- Every successful selection clears the four identified record fields.

Aliasing can make a write affect more than its nominal field. For example, if a selected record overlaps the state, record initialization may overwrite state already updated earlier in the function. The supplied evidence contains no overlap checks.

### Allocation and ownership

Growth requests descriptor and payload storage separately. Both returned addresses are retained through the descriptor and state structure.

No release, destruction, rollback, or traversal of older blocks is visible. Existing block descriptors and payloads are not explicitly released.

This establishes retention, not a complete ownership policy. It does not establish:

- Who must eventually release either allocation.
- Whether the two allocations share a lifetime.
- Whether F-871e64ddc6de transfers ownership.
- Whether other functions may reclaim blocks or records.

### Errors and partial updates

There is no explicit failure branch for either dependency result.

If a dependency returns zero or another unusable address, the function still attempts subsequent operations using that result. It does not intentionally convert such a result into a reported failure.

In particular, the second allocation call is reached without first checking the descriptor result. A successfully obtained allocation is not explicitly released if a later operation fails.

No transactional guarantee is present. Depending on where execution stops, allocation side effects, descriptor writes, state publication, cursor advancement, or some record-clearing writes may already have occurred.

The evidence does not determine whether the dependency can throw, terminate execution, invoke callbacks, or use a nonlocal exit.

### Concurrency

No synchronization is visible. Concurrent modification of the state, descriptors, or selected record has no established contract here. Multi-step updates must not be treated as atomic.

## Dependencies

### F-871e64ddc6de

Observed use:

| Call | First argument | Second argument | Result use |
|---|---:|---:|---|
| First growth call | 12 | 0 | Address of a writable descriptor |
| Second growth call | Wrapped 32-bit capacity × 24 | 0 | Address of record storage |

The function is allocation-like by use. Its actual API contract is not supplied.

The following remain unknown:

- Whether the first argument is formally an unsigned byte count.
- The meaning of the second argument.
- Failure signaling and zero-size-request behavior.
- Alignment and initialization guarantees.
- Ownership and release rules.
- Effects on global or caller-reachable state.
- Reentrancy and callback behavior.

The behavioral contract must not assume this dependency preserves globals, preserves the input state, or returns zero-filled storage.

No other direct function dependency is visible.

## Acceptance examples

These examples are conditional static expectations, not observed test results. Unless stated otherwise, they assume valid, non-overlapping storage, successful dependency calls, and no intervening mutation.

### Reusable record takes precedence

Given a nonzero reusable-record head pointing to record A, with A’s successor field pointing to record B:

- A is selected.
- The reusable-record head becomes B.
- No allocation request occurs.
- Cursor and boundary are not consulted for exhaustion.
- A’s fields at offsets 4, 8, and 12 become zero, followed by its byte at offset 0.
- A’s other bytes are not explicitly modified.

This remains the selected branch even if cursor and boundary would compare equal.

### Existing sequential space

Given a zero reusable-record head and distinct cursor and boundary addresses:

- The old cursor address is selected.
- The cursor advances by 24 bytes.
- Block-chain head and boundary receive no explicit writes.
- No dependency call occurs.
- Only the identified fields of the selected record are cleared.

This behavior depends on inequality alone; it does not prove that a complete record fits before the boundary.

### Initial block

Given zero block-chain head, zero reusable-record head, and equal cursor and boundary:

- Chosen capacity is 32.
- Requests are 12 bytes, then 768 bytes, each with second argument zero.
- The new descriptor links to the head read after those calls; under the example’s noninterference assumption, that link is zero.
- The first payload record is selected.
- Cursor becomes payload plus 24.
- Boundary becomes payload plus 768.

Only the selected record’s four identified fields are cleared.

### Growth from capacity 32

For an existing head capacity of 32:

- Proportional candidate: \(32 + 16 + 4 = 52\).
- Additive candidate: \(32 + 32 = 64\).
- Chosen capacity: 64.
- Payload request: 1,536 bytes.

### Growth from capacity 64

For an existing head capacity of 64:

- Proportional candidate: \(64 + 32 + 8 = 104\).
- Additive candidate: 96.
- Chosen capacity: 104.
- Payload request: 2,496 bytes.

### Overflow is not rejected

For an existing capacity of 2,147,483,648:

- The proportional candidate is 3,489,660,928.
- The additive candidate is 2,147,483,680.
- Chosen capacity is 3,489,660,928.
- The 32-bit payload-size product wraps to 2,147,483,648 bytes.

The function does not report this mismatch between mathematical capacity and requested byte size.

### Dependency changes the head

If either growth call changes the caller’s block-chain head, and execution otherwise remains valid:

- Capacity remains based on the earlier capacity calculation.
- The new descriptor links to the head observed after both calls.

A contract requiring the original head to be preserved in that link would not match the supplied source.

## Uncertainties

The following are documented_only:

- The public meaning of clearing the record’s byte and three words.
- How callers identify or use the selected record despite the absence of a return value.
- Valid external invariants for cursor, boundary, block capacity, and reusable links.
- Whether malformed or overlapping states are permitted by the surrounding API.
- Exact pointer-overflow behavior and address-space constraints.
- Byte order, ABI alignment requirements, and machine-level access atomicity.
- Allocation failure behavior and complete memory-ownership rules.
- Any global effects or reentrancy of F-871e64ddc6de.
- Release behavior elsewhere in the system.

“Pool,” “record,” “capacity,” and “reusable” are inferred structural descriptions. They do not establish a wider object lifecycle or application purpose.

## Evidence

The supplied static function body establishes the reusable-first branch, equality-based growth condition, 24-byte stride, 12-byte descriptor, unsigned capacity calculations, ordered dependency calls, metadata publication, cursor advancement, and selective record clearing.

The dependency mapping establishes F-871e64ddc6de as the opaque identity of the directly called function. No dependency implementation or external ownership contract was supplied.

No runtime execution, allocation-failure experiment, concurrency test, or consumer-level validation has been performed. The acceptance examples are deductions from static evidence under their stated conditions.
