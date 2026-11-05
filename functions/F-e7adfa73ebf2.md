<!-- Trusted provenance: F-e7adfa73ebf2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7adfa73ebf2

## Purpose

F-e7adfa73ebf2 conditionally enlarges a context-associated storage area, selects a position bounded by the current element count, invokes operations consistent with making room for one 20-byte element, transfers 20 bytes to the selected position, and increments the element count.

**Inferred intent:** insert a fixed-size record into an array, growing its capacity by three elements when full and treating an out-of-range insertion position as an append.

**Established limitation:** the supplied material establishes calls, argument values, local decisions, and direct context writes. It does not establish the semantics of the called functions. In particular, successful insertion, preservation of existing records, overlap-safe movement, and release of old storage depend on those functions’ contracts.

This specification therefore distinguishes the directly established behavior from the conditional array-insertion interpretation.

## Interface

### Inputs

The function accepts three 32-bit arguments:

| Argument | Interpretation supported by the supplied material |
|---|---|
| Context address | Address used to access the fields described below. |
| Requested position | Unsigned 32-bit value used to select an element position. |
| Transfer argument | Uninterpreted 32-bit value passed as the second argument to the final 20-byte transfer dependency. Its use as a source address is inferred, not independently established. |

The context is accessed at these byte offsets:

| Offset | Width | Role |
|---|---:|---|
| `0x10` | 32 bits | Element count. Compared for equality with capacity, later interpreted as unsigned for position selection. |
| `0x14` | 32 bits | Capacity-like value. Used to calculate an allocation request and increased by three on the successful growth path. |
| `0x70` | 32 bits | Buffer-address-like value used for storage operations and address calculations. |

The fixed element size is **20 bytes**, or `0x14` bytes. The coincidence between that size and the capacity field’s offset does not imply any additional layout relationship.

No other context fields are directly accessed in the supplied body.

### Return value

The return is a 32-bit value:

- `0xFFFFFF94` when the growth dependency returns zero.
- Zero after the normal completion path.

`0xFFFFFF94` is unsigned decimal 4,294,967,188, or −108 when interpreted as a signed 32-bit integer. No symbolic error meaning is established.

There is no explicit return-value check for any dependency other than the growth dependency.

### Preconditions and numeric domain

For an ordinary successful insertion interpretation, the caller needs:

- A readable and writable context containing the specified fields.
- A valid storage layout appropriate to the count and capacity.
- Valid memory arguments for every dependency reached.
- A transfer argument acceptable to the final transfer dependency.
- Counts and addresses for which the required additions and multiplications are representable and address valid storage.
- Dependency behavior compatible with allocation, copying, movement, and transfer as described conditionally below.

The function itself does not validate these conditions.

The requested position is explicitly treated as unsigned. A bit pattern that an external caller regards as a negative signed index is therefore a large unsigned position, not a request to index before the buffer.

## Behavior

### 1. Decide whether to grow

The function first reads the count and capacity fields and compares their 32-bit values for equality.

- If they differ, it skips growth.
- If they are equal, it enters the growth path.

This is an equality check, not a check that count is greater than or equal to capacity. A context with count greater than capacity does not trigger growth solely because it is over capacity.

The requested position does not affect this decision.

### 2. Request replacement storage

On the growth path, the function obtains the capacity field and calls **F-858f5b8d5984** with a size expression corresponding to:

**20 × (capacity + 3) bytes**

The result is a 32-bit address-like value.

If that result is zero, the function immediately returns `0xFFFFFF94`. It does not directly:

- Assign a replacement buffer value.
- Increase capacity.
- Perform the shift operation.
- Perform the final 20-byte transfer.
- Increment count.

This is not a guarantee that the context or other memory remains unchanged: F-858f5b8d5984 may have side effects, and its implementation is not supplied.

### 3. Invoke old-storage handling

If the allocation-like result is nonzero, the function reads the current buffer field.

If that field is nonzero, it invokes **F-ad023a375886** with these arguments, in this argument order:

1. The current buffer field.
2. The replacement address returned by F-858f5b8d5984.
3. The current count multiplied by 20.

The supplied body does not establish whether this dependency copies from its first argument to its second, whether it tolerates overlap, or what other effects it has.

After that call returns, the function invokes **F-1137792fba7d** with the buffer field’s value as read for that call. The buffer field is not guaranteed to have the same value as before F-ad023a375886, because that dependency’s effects are unknown.

**Inferred intent:** copy existing records into replacement storage, then release the old allocation.

If the buffer field was zero at the branch decision, both of these calls are skipped, even if count is nonzero. No initialization or reconstruction of missing old records is directly performed.

### 4. Publish replacement storage and capacity

After the optional old-storage calls return, the function:

1. Writes the replacement address into the buffer field.
2. Reads the capacity field, adds three, and writes the result back.

Thus, the replacement buffer is published before the capacity update.

The capacity update uses the field value available at that point. It is not necessarily based on a value captured before the allocation or old-storage calls. If a dependency changes capacity, the allocation request and final stored capacity need not correspond.

There is no rollback path in the supplied body after a nonzero allocation result.

### 5. Select the insertion position

After growth is either skipped or completed, the function reads count into an unsigned 32-bit snapshot.

The effective position is the smaller of:

- The requested unsigned position.
- That count snapshot.

Consequences:

- Position zero remains zero.
- A position equal to count selects the end.
- Any position larger than count is clamped to count.
- Position selection uses the count observed after the growth-related calls, not necessarily the count present on entry.

### 6. Invoke suffix movement when needed

If the count snapshot differs from the effective position, the function calculates the selected address from the current buffer field:

**selected address = buffer + 20 × effective position**

It calls **F-70a021fc7c24** with:

1. The selected address.
2. The selected address plus 20 bytes.
3. Twenty times the difference between the count snapshot and effective position.

Because the effective position does not exceed the unsigned count snapshot, the element-count subtraction itself is nonnegative in that unsigned comparison domain. The multiplication by 20 can still exceed the 32-bit numeric range.

If the effective position equals the count snapshot, this dependency is not called.

**Inferred intent:** move the existing suffix one element toward higher addresses, creating a 20-byte gap. That interpretation requires the dependency to treat the first address as source, the second as destination, and to preserve the result correctly when the ranges overlap.

The body does not independently establish those properties.

### 7. Perform the final 20-byte transfer

The function then invokes an external transfer dependency with:

1. The current buffer field plus 20 times the effective position.
2. The original third function argument.
3. The constant byte count 20.

The buffer field is read again for this call. If F-70a021fc7c24 changes it, the final destination calculation can use a different buffer value from the suffix-movement call.

The third function argument is passed through unchanged as a 32-bit value. Its referenced memory, if it is an address, is not snapshotted by this function before growth or movement.

**Inferred intent:** copy one record from the supplied source into the newly available slot.

### 8. Increment count and return

After the final transfer dependency returns, the function reads the count field, adds one, writes the result, and returns zero.

This increment does not necessarily use the earlier count snapshot. A dependency that modifies count can affect the value ultimately stored.

No dependency status from the old-storage, movement, or final transfer operations is checked.

## Side effects

### Direct context writes

On a normal path without growth:

1. Dependency calls may occur.
2. Count is incremented after the final transfer returns.

On a successful growth path:

1. Old-storage dependencies may be called.
2. The replacement buffer value is stored.
3. Capacity is increased by three.
4. The suffix-movement dependency may be called.
5. The final transfer dependency is called.
6. Count is incremented.

On a zero-result growth failure, there are no direct context writes by this function.

### Memory and ownership

The function invokes dependencies using buffer addresses and byte counts, but their memory effects are not supplied.

Under the inferred insertion interpretation:

- A successful growth obtains replacement storage.
- Existing record bytes are copied into it.
- The previous allocation is released.
- A suffix is moved if insertion is not at the end.
- One 20-byte record is written into the selected slot.

These are conditional effects, not independently established dependency contracts.

The supplied body contains no record-level construction, destruction, reference adjustment, or deep-copy operation. The 20-byte width alone does not establish that embedded pointers or resources are duplicated safely.

If the third argument refers into the old buffer, growth may invalidate that source before the final transfer. Without growth, suffix movement may alter the referenced source bytes before the transfer. No protective snapshot is visible.

### Ordering limits

The call order and direct-write order described above are established. A precise ordering among separate field reads used to form a single call’s arguments is not established by the supplied representation.

No concurrency control, atomic publication, or reentrancy protection is visible. No callee is assumed to preserve context fields, globals, or memory ownership.

## Dependencies

| Dependency | Established invocation | Conditional interpretation |
|---|---|---|
| **F-858f5b8d5984** | Called when count equals capacity, with `20 × (capacity + 3)`; zero triggers the failure return. | Allocates replacement storage. |
| **F-ad023a375886** | On successful growth with a nonzero current buffer, receives old-buffer-like value, replacement address, and `20 × count`. | Copies existing bytes from old storage to new storage. |
| **F-1137792fba7d** | Called after F-ad023a375886, with the then-current buffer field. | Releases old storage. |
| **F-70a021fc7c24** | Called only for a nonempty suffix; receives selected address, selected address plus 20, and suffix byte count. | Performs overlap-safe movement toward higher addresses. |

The final external transfer dependency has no opaque ID in the supplied dependency map. Its established invocation is a destination-like address, the third input argument, and a byte count of 20. Its direction, overlap behavior, failure behavior, and memory effects remain **documented_only**.

All dependency semantics beyond these call conditions, arguments, and checked results remain **documented_only**.

## Acceptance examples

These examples are conditional specifications, not results of executed tests. Unless stated otherwise, assume valid, nonoverflowing addresses and arithmetic, dependencies that return normally without changing context fields unexpectedly, and the conventional allocation/copy/move/transfer semantics described as inferred intent.

### Insert into the middle without growth

Given count 3, capacity 6, records A, B, C, requested position 1, and an independent source record X:

- No growth call occurs.
- The movement call receives the address of B, the following record address, and 40 bytes.
- The final transfer receives the position-1 address and 20 bytes.
- Count becomes 4; capacity remains 6.
- Under the assumed dependency semantics, the sequence becomes A, X, B, C.

### Clamp an oversized position to append

Given count 2, capacity 5, and requested position 99:

- Effective position is 2.
- No suffix-movement call occurs.
- The final transfer targets the address 40 bytes beyond the buffer base.
- Count becomes 3.

An unsigned position of `0xFFFFFFFF` behaves the same way when count is 2.

### Grow a full buffer

Given count 3, capacity 3, a nonzero old buffer, and requested position 1:

- The allocation-like request is 120 bytes.
- The old-storage copy-like call receives a byte count of 60.
- The release-like call follows it.
- The replacement buffer is stored before capacity becomes 6.
- The suffix-movement call receives 40 bytes.
- The final record transfer occurs before count becomes 4.

Preservation of the old records and release of the old allocation depend on the corresponding dependency semantics.

### Insert into an empty, zero-capacity context

Given count 0, capacity 0, and buffer value zero:

- Growth requests 60 bytes.
- A nonzero allocation result causes no old-storage copy or release calls.
- Capacity becomes 3.
- Effective position is zero regardless of the requested position.
- No suffix movement occurs.
- The final transfer targets the replacement buffer.
- Count becomes 1.

### Allocation-like failure

Given count equal to capacity and a zero result from F-858f5b8d5984:

- The return value is `0xFFFFFF94`.
- No subsequent dependency is invoked.
- The function performs no direct context write.

Any effects of the allocation-like dependency itself remain outside this guarantee.

### Count exceeds capacity

Given count 5 and capacity 3:

- Growth is skipped because the fields are unequal.
- Position selection and subsequent calls proceed using the count snapshot.
- No capacity error is returned explicitly.

This is not an assurance of memory safety; storage adequacy is a caller/dependency precondition.

## Uncertainties

The following remain **documented_only**:

- Exact dependency semantics, including copy direction, overlap handling, allocation ownership, release behavior, and failure reporting.
- Whether the third argument is necessarily a source pointer.
- Validity requirements for context and buffer alignment.
- Whether dependencies modify context fields or shared state.
- Behavior when a dependency does not return normally.
- Defined behavior for arithmetic overflow or invalid address calculations.

The supplied operations are 32-bit in the visible representation. Relevant overflow points include capacity plus three, multiplication of capacity or count by 20, address additions, and the final count increment. No explicit overflow checks are present. The representation does not justify a portable wraparound guarantee for every signed-looking intermediate.

The body also does not establish null-address handling, thread safety, source-alias safety, allocation-size sufficiency under inconsistent fields, or recoverability after a later dependency fails.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established evidence includes:

- Three 32-bit inputs and a 32-bit result.
- Context accesses at offsets `0x10`, `0x14`, and `0x70`.
- A fixed 20-byte element stride and final transfer size.
- Growth triggered only by count/capacity equality.
- A growth increment of three elements.
- A zero-result allocation branch returning `0xFFFFFF94`.
- Unsigned clamping of requested position to a count snapshot.
- Conditional suffix movement.
- Replacement-buffer publication before the capacity update.
- Count increment after the final transfer returns.

Array insertion, allocation, copying, release, and overlap-safe movement are inferred intent where explicitly identified, not verified callee behavior.

**No runtime tests have been performed.**
