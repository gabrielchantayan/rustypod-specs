<!-- Trusted provenance: F-ffe3c25e7ae8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ffe3c25e7ae8

## Purpose

F-ffe3c25e7ae8 selects a 20-byte record from either a reusable-record chain or a sequential allocation region, then clears specified fields in that record. If neither source has an available record, it obtains a new region and links its descriptor into the supplied state.

The established behavior is selection, allocation when necessary, state advancement, and partial record initialization. The interpretation as a record pool is inferred from those operations. The record’s application-level meaning is unknown.

The function has no explicit return value. In particular, it does not return the selected record’s address or explicitly publish that address in a dedicated output field. Its observable results are memory changes and any effects of dependency calls.

## Interface

### Input

The function accepts one pointer to a mutable state structure. The supplied representation uses four consecutive 32-bit words:

| Byte offset | Contract name | Observed use |
|---|---|---|
| 0 | Region-chain head | Null, or the address of the current region descriptor |
| 4 | Reusable-record head | Null, or the address of the next reusable record |
| 8 | Sequential cursor | Address of the next record available through sequential selection |
| 12 | Sequential limit | Address compared with the cursor to decide whether another region is needed |

These names describe observed roles, not independently established type declarations.

Pointer-bearing fields occupy 32 bits in the supplied representation. Address arithmetic and allocation-size arithmetic must therefore be interpreted in that representation, not silently widened to native 64-bit arithmetic.

### Region descriptor

A region descriptor occupies 12 bytes:

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Link to the previously current descriptor |
| 4 | 32 bits | Record capacity |
| 8 | 32 bits | Address of the record storage |

The capacity is read and used as an unsigned 32-bit value.

### Record layout

Records have a 20-byte stride. This function uses the following portions:

| Byte offset | Width | Behavior |
|---|---:|---|
| 0 | 8 bits | Written as zero |
| 1–3 | 3 bytes | Not explicitly written |
| 4 | 32 bits | Written as zero |
| 8 | 32 bits | Written as zero |
| 12 | 32 bits | Read as the next reusable-record address on the reusable path; subsequently written as zero |
| 16–19 | 4 bytes | Not explicitly written |

The function does not clear the complete record.

### Preconditions

For ordinary successful execution:

- The state pointer must identify readable and writable storage for all four state words.
- A non-null reusable-record head must identify a record whose link at offset 12 is readable and whose cleared fields are writable.
- When the reusable-record head is null and the cursor differs from the limit, the cursor must identify a writable 20-byte record.
- When growth is required and the region-chain head is non-null, its capacity field must be readable.
- Both dependency calls on the growth path must provide addresses usable by the subsequent accesses.
- Capacity and address calculations must describe actual accessible storage despite their finite-width arithmetic.

No checks enforce these conditions. There is also no validation that a cursor is aligned, below its limit, within a known region, or associated with the current descriptor.

## Behavior

### 1. Selection precedence

The reusable-record head is read first.

If it is non-null, the function uses that address as the selected record. It reads the 32-bit link at record offset 12 and writes that value to the state’s reusable-record head.

On this path, the function does not explicitly read or change the sequential cursor, sequential limit, or region-chain head. No allocation dependency is called.

If the reusable-record head is null, the function reads the sequential cursor and compares it with the sequential limit.

- Unequal addresses select the current cursor directly.
- Equal addresses trigger region growth before selection.

The comparison is equality, not an ordering or bounds test. A cursor beyond the limit, but unequal to it, still takes the direct-selection path.

### 2. Growth capacity

Growth occurs only when the reusable-record head is null and the sequential cursor equals the sequential limit.

If the region-chain head is null, the new capacity is 32 records.

Otherwise, the function reads the unsigned 32-bit capacity from offset 4 of the descriptor referenced by the region-chain head. Let that capacity be \(C\). It computes two unsigned 32-bit candidates:

- Additive candidate: \(C + 32\).
- Proportional candidate: \(C + \lfloor C/2 \rfloor + \lfloor C/8 \rfloor\).

Both additions use 32-bit unsigned arithmetic, including wraparound. The selected capacity is the larger candidate under an unsigned comparison. Equal candidates produce the same capacity.

For values without overflow, the policy therefore chooses the larger of an increment of 32 and an approximately 62.5% increase. That description is inferred intent, not an alternative to the exact arithmetic. With overflow, the result need not exceed the old capacity and need not be nonzero.

The storage request is the selected capacity multiplied by 20, again using a 32-bit result. The function does not reject multiplication overflow.

### 3. Allocation and descriptor publication

The growth path makes two calls to F-871e64ddc6de, in this order:

1. Request 12 bytes, with a second argument of zero.
2. Request the computed 32-bit storage size, with a second argument of zero.

The returned addresses are subsequently used as the descriptor address and record-storage address, respectively. There is no intervening explicit initialization of the newly obtained descriptor.

After both calls return, the function performs these explicit operations in order:

1. Writes the returned record-storage address to descriptor offset 8.
2. Reads the state’s region-chain head and writes that value to descriptor offset 0.
3. Writes the selected capacity to descriptor offset 4.
4. Writes the new descriptor address to the state’s region-chain head.
5. Reads the record-storage address back from descriptor offset 8 and uses it as the selected record address.
6. Writes selected-record address plus capacity times 20 to the state’s sequential limit.

The predecessor link is therefore obtained from the state after both dependency calls, not preserved from the earlier capacity decision. No assumption that those calls preserve the state or globals is justified by this function alone.

There is no null-result check after either dependency call. There is no rollback or local recovery path if a call fails, transfers control elsewhere, or returns an unusable address.

### 4. Sequential advancement

For either sequential-selection case—an existing region or newly allocated storage—the function writes selected-record address plus 20 to the state’s sequential cursor.

This write occurs before record initialization. On the ordinary new-region path, the cursor advances to the second record position while the limit marks the computed end of the storage region.

No reusable-head update is performed on this path. Its earlier observed value was null; the function does not reread it after dependency calls or explicitly restore it to null.

### 5. Partial initialization

After selection and state updates, the function writes:

1. A 32-bit zero at record offset 4.
2. A 32-bit zero at record offset 8.
3. A 32-bit zero at record offset 12.
4. An 8-bit zero at record offset 0.

This is the explicit access order in the supplied representation. It is not a claim of atomicity, synchronization, or a portable inter-thread ordering guarantee.

Bytes 1–3 and 16–19 receive no explicit writes from this function. On reuse, the previous link is read before offset 12 is cleared.

## Side effects

### State changes

Under valid, non-overlapping storage and dependencies that do not modify the state:

| Path | Explicit state changes |
|---|---|
| Reusable record | Reusable-record head becomes the selected record’s previous link |
| Existing sequential region | Sequential cursor advances by 20 bytes |
| New region | Region-chain head is replaced; sequential limit is set; sequential cursor advances to storage base plus 20 |

Every normally completed path performs the same partial initialization of the selected record.

These are explicit effects of this function. Dependency effects are additional and unknown.

### Allocation and lifetime

The growth path obtains two addresses and retains them through the descriptor chain and descriptor storage field. Existing descriptors are linked rather than explicitly released.

The function contains no deallocation, destruction, or cleanup call. It does not establish who owns either allocation, how regions are eventually released, or whether F-871e64ddc6de provides any particular allocation or lifetime guarantee.

### Failure and concurrency

There is no status return, error flag, validation branch, or explicit error callback. Invalid addresses can cause an invalid access or other execution failure. A dependency might fail through mechanisms not visible here.

State updates precede record initialization. Consequently, an interrupted or failed execution need not leave either state or record unchanged.

No locking or atomic operations are established. Concurrent use of shared state requires external guarantees; thread safety is not demonstrated.

## Dependencies

### F-871e64ddc6de

This is the sole directly called dependency.

Its observed interface at these call sites consists of:

- A first argument representing a byte count.
- A second argument equal to zero.
- A result consumed as a 32-bit address.

It is called exactly twice on the growth path and not called on the other paths.

Its use suggests a storage-provision service. The following remain unknown:

- Meaning of the second argument.
- Alignment and initialization guarantees.
- Behavior for a zero-size request.
- Failure behavior.
- Whether it modifies shared state or invokes callbacks.
- Ownership and release requirements.

The contract does not assume that it preserves globals, preserves the supplied state, or transfers ownership under a particular allocation model.

## Acceptance examples

These are conditional static acceptance cases, not results of runtime testing. Unless stated otherwise, they assume valid non-overlapping storage, successful dependency calls, and no external mutation during execution.

### Reuse takes precedence

Given a non-null reusable-record head whose link points to another record:

- The original head record is selected.
- The state’s reusable-record head becomes the saved link.
- No dependency call occurs.
- Cursor, limit, and region-chain head receive no explicit writes.
- The selected record receives the four zero writes described above.

This applies even if the sequential cursor equals the sequential limit.

### Final reusable record

Given a non-null reusable-record head with a zero link, that record is selected and the head becomes zero. The function does not also consume a sequential record during the same invocation.

### Available sequential record

Given a null reusable head and a cursor different from the limit:

- The cursor address is selected.
- The cursor advances by 20 bytes.
- No dependency call occurs.
- The selected record is partially initialized.

If the cursor initially lies exactly 20 bytes before the limit, it becomes equal to the limit. Growth is deferred until a later invocation requiring sequential selection.

### Initial region

Given a null reusable head, equal cursor and limit, and a null region-chain head:

- Capacity is 32.
- Dependency requests are 12 bytes and then 640 bytes, each with second argument zero.
- The new descriptor records capacity 32 and the returned storage address.
- Its predecessor is the region-chain head read after both calls.
- The selected record is the storage base.
- The limit becomes storage base plus 640; the cursor becomes storage base plus 20.

If dependencies preserve the state, the predecessor is null.

### Growth candidates

For an existing capacity of 32, the candidates are 64 and 52. The selected capacity is 64, and the storage request is 1,280 bytes.

For an existing capacity of 128, the candidates are 160 and 208. The selected capacity is 208, and the storage request is 4,160 bytes.

### Untouched record bytes

Given known initial bytes in positions 1–3 and 16–19, and no overlapping state or dependency writes, those bytes remain unchanged. The record must not be treated as fully zero-initialized.

### Invalid cursor ordering

Given a null reusable head and a cursor greater than, but unequal to, the limit, the function still selects the cursor. It does not grow or report a bounds error. Safe completion depends on that address being valid independently of the comparison.

## Uncertainties

The following are **documented_only**:

- Application meaning of records and their cleared fields.
- How a caller identifies or uses the selected record after this void operation.
- Full behavior, side effects, and ownership contract of F-871e64ddc6de.
- Required allocator alignment and handling of unusable results.
- Whether surrounding logic prevents capacity and address overflow.
- Whether surrounding logic guarantees cursor ordering and region membership.
- Whether state, descriptors, and records are guaranteed not to overlap.
- Eventual region reclamation and reusable-chain construction.
- Concurrency, reentrancy, and recovery after partial progress.

Aliasing could make writes alter later reads or overlap otherwise untouched bytes. The stated access order captures the available evidence; ordinary pool semantics require additional non-overlap assumptions.

## Evidence

Established static evidence consists of the supplied function representation: its field accesses, branch conditions, unsigned capacity arithmetic, two ordered dependency calls, descriptor publication, cursor advancement, and partial record initialization.

The interpretation as a pool with a reusable chain and region-based sequential allocation is inferred intent. It is consistent with the observed structure but does not establish undocumented caller or dependency behavior.

No runtime tests have been performed. Acceptance examples are derived from the static behavior and their stated assumptions.
