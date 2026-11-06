<!-- Trusted provenance: F-1c004ce19fed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1c004ce19fed

## Purpose

F-1c004ce19fed selects one 24-byte record from a caller-supplied storage context and clears selected fields in that record.

Selection has three priorities:

1. Reuse the record at the head of a reusable-record chain, when that head is nonzero.
2. Otherwise consume the next record from an existing sequential storage range.
3. If that range is exhausted, request a new block descriptor and backing storage, publish them into the context, and consume the first record of the new storage.

The function has no explicit return value. Its observable results are context updates, selected-record writes, and, when necessary, two calls to F-871e64ddc6de.

The interpretation as a record allocator or recycler is inferred intent. The selection rules, field accesses, write widths, and dependency-call sequence are established by the supplied static evidence.

## Interface

### Input context

The sole argument is an address interpreted as a context containing four consecutive 32-bit fields:

| Byte offset | Width | Contract role |
|---|---:|---|
| 0 | 32 bits | Head address of a block-descriptor chain |
| 4 | 32 bits | Head address of a reusable-record chain |
| 8 | 32 bits | Sequential allocation cursor |
| 12 | 32 bits | Sequential allocation end address |

These role names describe observed use, not a verified public type definition.

The supplied representation stores addresses in 32-bit integer fields. It does not establish a portable interface for environments with wider addresses. The context must support the reads and writes taken by the selected execution path.

### Block descriptor

A block descriptor is used as three consecutive 32-bit fields:

| Byte offset | Width | Contract role |
|---|---:|---|
| 0 | 32 bits | Previous block-descriptor head |
| 4 | 32 bits | Capacity used for subsequent growth calculations |
| 8 | 32 bits | Backing-storage address |

When creating a descriptor, the function requests 12 bytes from F-871e64ddc6de.

### Record layout

Each sequentially selected record occupies a 24-byte stride. This function accesses the following locations within the selected record:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 8 bits | Written to zero |
| 4 | 32 bits | Written to zero |
| 8 | 32 bits | Written to zero |
| 12 | 32 bits | Read as a reusable-chain link on the reuse path, then written to zero |

It performs no direct writes to bytes 1–3 or 16–23. Their meaning is not established.

### Output and validity requirements

There is no explicit returned record address, status value, or error code.

For ordinary successful execution:

- The context must be accessible.
- A nonzero reusable head must identify a record readable at offset 12 and writable at the cleared locations.
- A selected sequential record must be writable at those locations.
- A nonzero block-descriptor head used during growth must permit a 32-bit read at offset 4.
- Dependency results used as descriptor or backing-storage addresses must identify suitable accessible storage.

The function does not check these conditions.

The intended layouts imply distinct, suitably aligned storage regions. Behavior under overlapping context, descriptor, and record storage is not established as a supported use case.

## Behavior

### 1. Select the reusable-record path when available

The first selection read is the context’s reusable-record head at offset 4.

If that value is nonzero:

1. That address becomes the selected record.
2. A 32-bit successor address is read from record offset 12.
3. The successor is written to context offset 4.
4. The selected record is cleared as described below.

This path does not directly read or update the sequential cursor, sequential end, or block-descriptor head. It makes no calls to F-871e64ddc6de.

The successor is copied without validation. A zero successor empties the reusable chain. A nonzero successor becomes the head for a later invocation, subject to any other changes made outside this function.

### 2. Otherwise inspect the sequential range

If the reusable head is zero, the function reads the sequential cursor at context offset 8 and compares it for equality with the end address at offset 12.

If the two addresses differ:

1. The original cursor becomes the selected record.
2. Context offset 8 is written with that address plus 24 bytes.
3. The selected record is cleared.

This is an equality check, not a bounds check. The function does not establish that the cursor precedes the end or that 24 bytes remain.

In particular, a cursor numerically beyond the end still takes this path if the two values differ. A cursor with fewer than 24 bytes of valid storage remaining is not rejected.

### 3. Grow storage when cursor equals end

If the cursor equals the end, the function derives a capacity and makes two dependency calls.

#### Capacity selection

If the context’s block-descriptor head is zero, the new capacity is 32 records.

Otherwise, the function reads the existing head descriptor’s 32-bit capacity field at offset 4. Let that unsigned value be \(c\). It computes:

\[
a = \bigl(c + \lfloor c/2 \rfloor + \lfloor c/8 \rfloor\bigr) \bmod 2^{32}
\]

\[
b = (c + 32) \bmod 2^{32}
\]

The new capacity \(n\) is the unsigned maximum of \(a\) and \(b\). The comparison selects \(a\) when \(b \leq a\); otherwise it selects \(b\).

The shifts are unsigned right shifts. The additions use 32-bit unsigned arithmetic. Therefore, the mathematical interpretation “grow by at least 32, or by approximately 62.5%” applies only where the relevant additions do not wrap. It is not an unconditional growth guarantee.

#### Dependency calls

The calls occur in this order:

1. F-871e64ddc6de with arguments `(12, 0)`, obtaining the address subsequently used as the new descriptor.
2. F-871e64ddc6de with arguments `(24 × n modulo 2³², 0)`, obtaining the value subsequently used as the backing-storage address.

Both calls precede this function’s descriptor initialization.

Neither result is checked for failure. No alternate path, retry, rollback, or cleanup appears in this function.

#### Descriptor and context publication

After both dependency calls return, the function performs these operations in order:

1. Write the second call’s result to new-descriptor offset 8.
2. Read the context’s current block-descriptor head.
3. Write that head to new-descriptor offset 0.
4. Write capacity \(n\) to new-descriptor offset 4.
5. Write the new descriptor address to context offset 0.
6. Read the backing-storage address from new-descriptor offset 8.
7. Write the backing-storage address plus the computed byte extent to context offset 12.
8. Write the backing-storage address plus 24 bytes to context offset 8.
9. Clear the record beginning at the backing-storage address.

The byte extent is \(24 \times n\), calculated with 32-bit unsigned multiplication before its use in the address expression.

A significant timing distinction is that the descriptor used to calculate capacity is examined before the dependency calls, while the previous-head value stored into the new descriptor is read after both calls. The function does not preserve an earlier head value for that link. If a dependency changes the context through an external alias, those two observations need not describe the same descriptor.

### 4. Clear the selected record

All successful selection paths converge on the same ordered writes:

1. Write a 32-bit zero at record offset 4.
2. Write a 32-bit zero at record offset 8.
3. Write a 32-bit zero at record offset 12.
4. Write an 8-bit zero at record offset 0.

This is partial initialization, not a 24-byte clear.

On the reusable-record path, the offset-12 successor has already been read and installed as the new reusable head before that field is cleared.

On the sequential paths, the cursor has already advanced before the record-clearing writes begin. On the growth path, the descriptor and end address have also already been published.

## Side effects

### Direct memory changes

Depending on the path, this function directly changes:

- The reusable head, when taking a record from that chain.
- The sequential cursor, when consuming sequential storage.
- The descriptor head and sequential end, when growing storage.
- All three fields of a newly obtained descriptor.
- Four locations in the selected record.

There is no direct write to the reusable head on either sequential path.

There is no direct write to an old descriptor’s fields. Growth retains its address as the new descriptor’s previous link rather than traversing or modifying the existing chain.

### Ownership and lifetime

The growth path retains both dependency results through the new descriptor and context. This is consistent with persistent block storage, but allocation ownership is not independently established.

No release operation is visible. The function does not itself reclaim an old block, descriptor, or reusable record. The supplied evidence does not establish who must eventually release retained storage, whether release is necessary, or which dependency would perform it.

The function removes a record from the reusable chain but does not expose its address through a return value. How callers identify or use the selected record is outside the supplied evidence.

### Errors and interruption

There is no explicit error reporting or recovery.

Invalid addresses, unsuitable dependency results, insufficient writable storage, and arithmetic-induced invalid ranges are not detected before use. The exact platform response to such conditions is unknown.

Updates are not transactional. Because context changes precede record clearing, an interruption or failed memory access can leave a context pointing past a record whose initialization did not finish.

No synchronization is visible. Concurrent or reentrant use requires guarantees not supplied here.

## Dependencies

### F-871e64ddc6de

Established call contract from this function’s use:

| Invocation | First argument | Second argument | Result use |
|---|---:|---:|---|
| First growth call | 12 | 0 | Address for a three-field descriptor |
| Second growth call | \(24n \bmod 2^{32}\) | 0 | Backing-storage address |

The observed call sites support an allocation-like interpretation, but do not prove allocator semantics.

The following are unknown:

- Meaning of the second argument.
- Failure representation and failure behavior.
- Alignment, initialization, and minimum-size guarantees.
- Treatment of a zero-byte request.
- Ownership and release obligations.
- Whether calls may mutate globals, the supplied context through aliases, or other reachable memory.
- Whether the dependency can terminate execution or transfer control without returning normally.

No preservation of global or aliased state is assumed. The specified direct write order is distinct from any side effects performed inside this dependency.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Unless stated otherwise, they assume valid, nonoverlapping storage; representable address calculations; normal dependency returns; and no external mutation during execution.

### Reuse takes precedence over available sequential storage

Given a nonzero reusable head whose offset-12 field contains a successor:

- That head record is selected, even if sequential capacity is available.
- The context’s reusable head becomes the successor.
- The cursor, end, and descriptor head remain directly unchanged.
- No dependency call occurs.
- The selected record receives the four ordered zero writes.
- Its bytes 1–3 and 16–23 retain their previous values.

### Sequential selection without growth

Given a zero reusable head and distinct cursor and end addresses:

- The original cursor identifies the selected record.
- The cursor advances by 24 bytes.
- The end and descriptor head remain directly unchanged.
- No dependency call occurs.
- Only the specified fields of the selected record are cleared.

### Initial storage acquisition

Given zero reusable head, equal cursor and end, and zero descriptor head:

- Capacity is 32.
- The dependency receives `(12, 0)`, then `(768, 0)`.
- The new descriptor records the second result as its backing address and 32 as its capacity.
- If the context head remains zero through both calls, the descriptor’s previous link is zero.
- The context head becomes the new descriptor.
- The end becomes backing address plus 768.
- The cursor becomes backing address plus 24.
- The first backing record is partially cleared.

### Growth dominated by the fixed increment

Given an exhausted range and an existing capacity of 32:

- The proportional candidate is \(32 + 16 + 4 = 52\).
- The fixed-increment candidate is 64.
- New capacity is 64.
- The backing request is 1,536 bytes.

### Growth dominated by the proportional candidate

Given an exhausted range and an existing capacity of 128:

- The proportional candidate is \(128 + 64 + 16 = 208\).
- The fixed-increment candidate is 160.
- New capacity is 208.
- The backing request is 4,992 bytes.

### Wrapped growth arithmetic

Given an exhausted range and existing capacity \(2^{32}-1\):

- The proportional candidate wraps to 2,684,354,557.
- The fixed-increment candidate wraps to 31.
- The selected capacity is 2,684,354,557.
- The backing-size multiplication wraps to 4,294,967,224.

These values describe the arithmetic, not a guarantee that the resulting allocation or address range is usable.

### Invalid ordering is not rejected

Given zero reusable head and unequal cursor and end, with the cursor beyond the end:

- The function still selects the cursor and attempts to advance and clear it.
- There is no bounds-error result.
- Whether the accesses succeed depends on the actual memory mapping.

## Uncertainties

The following remain **documented_only**:

- The public type and semantic purpose of the context and record fields.
- The meaning of the record’s untouched bytes.
- How the selected record is communicated to its consumer.
- The full contract of F-871e64ddc6de.
- Supported ownership, aliasing, concurrency, and reentrancy rules.
- Platform behavior for invalid addresses, address overflow, and failed allocation-like requests.
- Whether external invariants prevent wrapped capacities or exhausted-range inconsistencies.

“Descriptor,” “capacity,” “reusable chain,” and “sequential range” are explanatory interpretations of observed structure. They do not establish an external API or hidden invariant.

## Evidence

The supplied static function body establishes:

- Four 32-bit context fields used for selection and storage management.
- Reusable-head priority over sequential allocation.
- A cursor/end equality test as the sole exhaustion check.
- A 24-byte record stride and 12-byte descriptor request.
- Unsigned 32-bit capacity arithmetic.
- Two ordered calls to F-871e64ddc6de on growth only.
- Descriptor publication and cursor updates before record clearing.
- Three 32-bit zero writes followed by one 8-bit zero write.
- No explicit return value, validation branch, release operation, or error-reporting path.

No runtime tests have been performed. Dependency internals, caller behavior, public declarations, and platform fault semantics were not supplied.
