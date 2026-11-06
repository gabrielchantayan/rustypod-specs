<!-- Trusted provenance: F-efe92a09e55f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efe92a09e55f

## Purpose

F-efe92a09e55f selects a record with a 20-byte stride and partially initializes it. Selection has three priorities:

1. Take the record identified by the reusable-record head, when that head is nonzero.
2. Otherwise, take the current sequential cursor, when it differs from the sequential limit.
3. Otherwise, calculate a capacity, call F-871e64ddc6de twice, publish a descriptor, and select an address read from that descriptor.

The function has no explicit return value. Its observable behavior consists of memory reads and writes, dependency calls, and any effects of those calls.

**Established static evidence:** the supplied body establishes the branch conditions, represented field widths, capacity arithmetic, dependency arguments, use of dependency results as addresses, explicit write order, and partial record initialization.

**Inferred intent:** the state resembles a record pool with a reusable-record chain, sequential storage, and linked storage descriptors. The dependency calls resemble allocation requests. The supplied body does not establish allocation semantics or the meaning of the dependency arguments.

“Record,” “descriptor,” and “capacity” describe the visible access pattern; they do not establish ownership or a complete external data model.

## Interface

### Argument and result

The sole argument identifies mutable state containing four consecutive 32-bit fields:

| State byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Descriptor-head address |
| 4 | 32 bits | Reusable-record-head address |
| 8 | 32 bits | Sequential cursor |
| 12 | 32 bits | Sequential limit |

Address-bearing fields use a represented 32-bit layout. Compatibility with another pointer width is not established.

There is no explicit return value and no dedicated selected-record output field. On either sequential path, the state cursor is advanced past the selected record before initialization.

The state argument and dependency results are not explicitly checked for null. No bounds, alignment, or structural validation is visible. There **are** explicit zero checks on the reusable-record head and, on the growth path, the descriptor head; these checks select behavior rather than validate the complete structure.

### Descriptor layout

The first dependency result is used as the address of a descriptor with the following fields:

| Descriptor byte offset | Width | Value explicitly written |
|---|---:|---|
| 0 | 32 bits | State descriptor head read during publication |
| 4 | 32 bits | Calculated capacity |
| 8 | 32 bits | Second dependency result |

These writes span 12 bytes. The first dependency call receives the numerical argument 12, but its interpretation as a requested byte count is not established.

When an existing descriptor is used to calculate growth, its offset-4 field is read as an unsigned 32-bit capacity.

### Record layout

The selected record has a 20-byte stride. Explicit accesses are:

| Record byte offset | Width | Access |
|---|---:|---|
| 0 | 1 byte | Written as zero |
| 1–3 | 3 bytes | No explicit access |
| 4 | 32 bits | Written as zero |
| 8 | 32 bits | Written as zero |
| 12 | 32 bits | Read as a link on reuse; subsequently written as zero |
| 16–19 | 4 bytes | No explicit access |

Initialization is not a full-record clear.

### Preconditions for ordinary completion

The selected path requires accessible storage for every read and write:

- The state must support the fields accessed on that path.
- A nonzero reusable-record head must identify storage readable at offset 12 and writable at the initialization offsets.
- A selected sequential address must support the initialization writes.
- An existing descriptor used for growth must support the capacity read.
- Dependency results must support their subsequent address-based uses.
- The execution environment must support the represented address conversions and address arithmetic.

The body does not establish these conditions. In particular, cursor inequality neither proves that the cursor lies below the limit nor that a complete record remains available.

## Behavior

### Reusable-record selection

The function first reads the state’s reusable-record head.

If that value is nonzero:

1. The head address becomes the selected record address.
2. A 32-bit link is read from selected-record offset 12.
3. That link is written into the state’s reusable-record-head field.
4. The common initialization writes occur.

The old link is consumed before the selected record’s link field is cleared.

This path performs no dependency calls and no explicit accesses to the descriptor head, sequential cursor, or sequential limit. With nonoverlapping storage and no external writers, those fields remain unchanged.

Only one record is removed from the reusable chain. There is no traversal, cycle detection, or validation of the next link.

### Sequential selection without growth

If the reusable-record head is zero, the function reads the sequential cursor and compares it with the sequential limit.

When the two addresses differ:

1. The cursor value already read becomes the selected address.
2. The state cursor is written with the selected address advanced by 20 bytes.
3. The common initialization writes occur.

The comparison tests equality, not ordering. A cursor beyond the limit takes this branch. A cursor fewer than 20 bytes below the limit also takes this branch. Neither case triggers an explicit error or growth.

No descriptor-head access or dependency call occurs on this path.

### Capacity calculation on exhaustion

Growth occurs only when the reusable-record head is zero and the sequential cursor equals the sequential limit.

The descriptor head is explicitly checked for zero:

- A zero head selects capacity 32.
- A nonzero head leads to an unsigned 32-bit capacity read at descriptor offset 4.

The head used to locate that capacity is obtained through a state-head read distinct from the earlier zero test. The body supplies no synchronization guaranteeing that those reads observe the same value.

For an existing capacity \(c\), calculate two unsigned 32-bit candidates:

\[
A=(c+32)\bmod 2^{32}
\]

\[
P=\left(c+\left\lfloor c/2\right\rfloor+
\left\lfloor c/8\right\rfloor\right)\bmod 2^{32}.
\]

The divisions correspond to unsigned right shifts. Additions use unsigned 32-bit arithmetic.

The selected capacity is the larger of \(A\) and \(P\), using unsigned comparison after wrapping. Equality selects the proportional candidate, without changing the numerical result.

The separately truncated terms matter: the proportional expression is not generally equivalent to rounding down \(1.625c\).

Let \(n\) be the selected capacity. The numerical product used for the second dependency call is:

\[
b=(20n)\bmod 2^{32}.
\]

No arithmetic-overflow check is visible. Neither the selected capacity nor the wrapped product is guaranteed to increase as the previous capacity increases.

This unsigned wrapping rule does not establish portable behavior for address additions that exceed valid storage or the target address space.

### Dependency calls and descriptor publication

Growth invokes F-871e64ddc6de twice, in sequence:

| Invocation | First argument | Second argument | Use of result |
|---|---:|---:|---|
| First | 12 | 0 | Address for descriptor accesses |
| Second | \(b\) | 0 | Value stored in descriptor offset 8 |

The first invocation returns before the second begins. No explicit descriptor-field write occurs between them.

After both calls return, the function performs these operations in order:

1. Write the second result into descriptor offset 8.
2. Read the current state descriptor head and write it into descriptor offset 0.
3. Write the previously calculated capacity into descriptor offset 4.
4. Write the first result into the state descriptor-head field.
5. Read the selected address from descriptor offset 8.
6. Write the selected address advanced by \(b\) bytes into the state sequential-limit field.
7. Write the selected address advanced by 20 bytes into the state sequential-cursor field.
8. Perform the common initialization at the selected address.

The selected address comes from the descriptor readback. It is not unconditionally interchangeable with the second dependency result: intervening overlapping writes or external mutation may change that field.

Likewise, the descriptor link is taken from the state head after both dependency calls and after the offset-8 descriptor write. It need not equal the head used for capacity calculation.

Under valid, nonoverlapping storage, no concurrent mutation, and dependencies that preserve the supplied state, this sequence links the new descriptor to the previous head, publishes it, selects the second result as the first record, and sets cursor and limit relative to that address. Those preservation conditions are assumptions for that interpretation, not guarantees of the function.

### Common initialization

Every path finishes with these writes, in this order:

1. A 32-bit zero at record offset 4.
2. A 32-bit zero at record offset 8.
3. A 32-bit zero at record offset 12.
4. A one-byte zero at record offset 0.

Bytes 1–3 and 16–19 receive no explicit writes.

The reusable-head update or cursor update precedes initialization. On growth, descriptor publication and limit assignment also precede initialization. Failure during initialization can therefore leave state advanced or a descriptor published without a fully initialized selected record.

## Side effects

### Direct memory changes

| Path | Explicit state-field writes |
|---|---|
| Reuse | Reusable-record head |
| Sequential selection without growth | Sequential cursor |
| Growth | Descriptor head, sequential limit, sequential cursor |

Every path also performs the four common initialization writes. Growth writes all three descriptor fields.

These categories identify access locations, not unconditional preservation guarantees. If record, descriptor, and state storage overlap, a descriptor or record write can alter state bytes. Such overlap can also change values read later in the sequence. The body does not reject overlap.

### Ownership and dependency effects

On the ordinary nonoverlapping growth path, the first returned address becomes represented in the state descriptor head, and the second becomes represented in the descriptor’s offset-8 field.

Storing these addresses does not establish ownership transfer. The body does not determine:

- Whether either dependency call allocates, retrieves, shares, or reuses storage.
- Whether its first argument denotes bytes or another quantity.
- Whether returned storage is fresh, aligned, or initialized.
- Whether the dependency changes the supplied state, other reachable storage, or globals.
- Whether either call releases existing storage.
- Who owns returned storage or must eventually release it.

There is no separate visible release or cleanup operation. That does not justify classifying the dependency itself as incapable of releasing memory.

### Errors and partial completion

No explicit error result, dependency-result null check, rollback, or recovery branch is visible.

The first result is retained across the second call and then used for descriptor writes without validation. The second result is stored, read back, used in address calculations, and ultimately used for initialization without validation.

If the second call does not return, no descriptor-field writes have yet been explicitly performed by this function. Effects and resource consequences of the first call remain unknown.

Invalid addresses, unsupported alignment, dependency failures, exceptions, interruptions, and invalid address arithmetic have no established recovery semantics. Earlier writes are not explicitly restored.

## Dependencies

### F-871e64ddc6de

**Established at this call site:**

- Called exactly twice on growth and not called on the other paths.
- First invocation receives 12 and zero.
- Second invocation receives the wrapped unsigned 32-bit capacity-times-20 product and zero.
- Calls are sequential.
- The first result is used as a descriptor address.
- The second result is written into an address-bearing descriptor field.
- A subsequent read of that field supplies the selected record address.
- Neither result is explicitly checked for null.
- No descriptor-field write occurs between the calls.

**Inferred intent:** obtain descriptor storage and a sequential backing region, with the first arguments representing byte counts.

**documented_only:** actual parameter semantics, meaning of zero, result representation beyond the visible use, allocation behavior, failure signaling, alignment, initialization, lifetime, ownership, global effects, and treatment of zero or wrapped request values.

The dependency’s implementation is not supplied. No preservation of state, globals, previously returned storage, or ownership is assumed.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, assume valid nonoverlapping storage, supported address additions, no concurrent mutation, and dependency calls that return suitable addresses without modifying the supplied state.

### Reuse takes priority

Given a reusable-record head identifying record R, with link S at R’s offset 12:

- R is selected regardless of cursor/limit equality.
- The reusable head becomes S before initialization.
- No dependency calls occur.
- The descriptor head, cursor, and limit are not explicitly accessed.
- R’s old link is read before its field is cleared.

A zero reusable head does not select address zero for reuse; it selects the sequential-selection branch.

### Sequential inequality is sufficient for selection

Given a zero reusable head and cursor R differing from the limit:

- R is selected.
- The cursor becomes R plus 20 bytes before initialization.
- No dependency calls occur.
- The descriptor head and limit receive no explicit writes.

If R is beyond the limit, the same branch is selected. If R is only four bytes below the limit, the same branch is selected. Safe completion still requires storage supporting the record writes.

### Zero descriptor head selects initial capacity

Given a zero reusable head, equal cursor and limit, and zero descriptor head:

- Capacity is 32.
- The two dependency calls receive first arguments 12 and 640, respectively, with second argument zero.
- If the returned addresses are D and B, D receives a zero link, capacity 32, and B at offset 8.
- The state descriptor head becomes D.
- The limit becomes B plus 640 bytes.
- The cursor becomes B plus 20 bytes.
- The record at B receives common initialization.

This behavior depends on the explicit descriptor-head zero check. It does not require the dependency to interpret 12 or 640 as byte counts.

### Additive growth dominates

Given exhaustion and an existing capacity of 32:

- The additive candidate is 64.
- The proportional candidate is 52.
- Selected capacity is 64.
- The second call receives 1,280 as its first argument.

### Proportional growth dominates

Given exhaustion and an existing capacity of 64:

- The additive candidate is 96.
- The proportional candidate is 104.
- Selected capacity is 104.
- The second call receives 2,080 as its first argument.

### Wrapped arithmetic is compared unsigned

Given existing capacity 4,294,967,295:

- The additive candidate wraps to 31.
- The proportional candidate wraps to 2,684,354,557.
- Unsigned comparison selects 2,684,354,557.
- Multiplication by 20 wraps to 2,147,483,588.

The last value is the second call’s first argument. This arithmetic example establishes neither safe address arithmetic nor dependency acceptance of that value.

### Dependency mutation affects the published link

Suppose capacity is calculated from descriptor H, but a dependency changes the state descriptor head to K before returning.

With otherwise nonoverlapping storage:

- Capacity remains the value already calculated from H.
- The new descriptor’s link receives K from the later state-head read.
- The new descriptor then becomes the state head.

The function does not restore H or recalculate capacity from K.

### Initialization leaves gaps

Given selected record storage initially containing known nonzero bytes:

- Bytes 0 and 4–15 become zero.
- Bytes 1–3 and 16–19 receive no explicit writes.
- The 32-bit fields are cleared in offset order 4, 8, 12.
- Byte 0 is cleared last.

Preservation of the other bytes depends on the stated nonoverlap and no-external-writer assumptions.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the initialized record fields.
- How callers identify the selected record without a returned pointer.
- State construction, reusable-chain insertion, and destruction.
- Whether the dependency allocates or interprets its arguments as sizes.
- Dependency failure behavior, memory lifetime, ownership, and release obligations.
- Thread safety, reentrancy, synchronization, and external observation of intermediate writes.
- Intended restrictions on capacities and cursor/limit relationships.
- Target-specific consequences of null or invalid addresses, misalignment, and invalid address arithmetic.
- Behavior under overlapping state, descriptor, and record storage beyond the established ordered accesses.

The explicit zero checks establish branch selection for the two heads. They do not resolve these missing contracts or validate the state argument and dependency results.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence.

The body establishes:

- Three record-selection paths.
- Explicit reusable-head and descriptor-head zero checks.
- No explicit null checks for the state argument or dependency results.
- Represented field widths and offsets.
- Unsigned capacity arithmetic and wrapped multiplication.
- Dependency argument values and subsequent address-based uses.
- Descriptor publication, cursor advancement, and initialization order.
- Partial rather than complete record clearing.

No dependency implementation, caller contract, runtime trace, or complete execution-environment contract was supplied. Allocation semantics, ownership, and dependency preservation guarantees remain unresolved.

No runtime tests have been performed. All acceptance examples are conditional static criteria, not observed executions.
