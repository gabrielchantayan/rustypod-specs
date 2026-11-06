<!-- Trusted provenance: F-e297fd02e4fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e297fd02e4fa

## Purpose

F-e297fd02e4fa selects one 72-byte record from a mutable storage state and partially resets that record.

Selection has three paths, in priority order:

1. Remove the first record from a reusable-record list, if that list is nonempty.
2. Otherwise, select the record at the sequential cursor if the cursor differs from the stored end address.
3. Otherwise, request a new header and record block, link the header into the state, and select the first record in the new block.

The function has no return value. It neither returns the selected record address nor reports an explicit success or failure status. Its observable results are memory writes and, on the third path, calls to F-871e64ddc6de.

The record-pool interpretation is inferred intent. The selection rules, accessed offsets, request sizes, arithmetic, and reset operations are established by the supplied static evidence.

## Interface

### Input state

The sole argument identifies a mutable state containing four consecutive 32-bit fields:

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Current allocation-header address, or zero |
| 4 | 32 bits | First reusable-record address, or zero |
| 8 | 32 bits | Sequential cursor |
| 12 | 32 bits | End address used for the exhaustion comparison |

Addresses are carried in 32-bit fields in the supplied representation. This contract does not establish portability to a wider-pointer environment.

### Allocation-header representation

A newly requested header occupies 12 bytes:

| Byte offset | Width | Stored value |
|---|---:|---|
| 0 | 32 bits | Previous header address |
| 4 | 32 bits | Record capacity |
| 8 | 32 bits | Record-block base address |

The capacity is read and processed as an unsigned 32-bit value.

### Record representation

Sequential records are spaced 72 bytes apart. A reusable record provides its successor address through a 32-bit field at offset 12.

The function writes only these record locations:

| Byte offset | Write width | Value |
|---|---:|---:|
| 4 | 32 bits | 0 |
| 8 | 32 bits | 0 |
| 12 | 32 bits | 0 |
| 0 | 8 bits | 0 |

The remaining record layout and the semantic meanings of these fields are not established.

### Preconditions for ordinary successful completion

The argument must identify storage supporting the state accesses required by the selected path. A selected reusable record must support the successor read at offset 12. Every selected record must support all four reset writes.

If a new block is required:

- A nonzero current-header address must support the capacity read at offset 4.
- The dependency results must support the subsequent header and record accesses.
- The computed record address and end address must be meaningful for the surrounding storage model.
- The dependency must accept both observed argument pairs.

These are external requirements, not checks performed by this function. There is no visible validation of alignment, list structure, accessible extent, allocation success, or consistency between the cursor and end.

## Behavior

### Reusable-record selection

The first selection read obtains the reusable-list address from state offset 4.

If this address is nonzero:

1. That address becomes the selected record.
2. The function reads its 32-bit successor field at record offset 12.
3. It writes the successor address to state offset 4.
4. It performs the partial reset described below.

This path does not consult the cursor, end address, or current allocation header. It makes no call to F-871e64ddc6de.

The successor read and reusable-list update occur before the selected record’s offset-12 field is cleared. Becoming empty as a result of this removal does not trigger allocation during the same invocation.

### Sequential selection

If the reusable-list address is zero, the function reads the cursor at state offset 8 and compares it with the end address at state offset 12.

When they differ, the cursor identifies the selected record. The function writes the selected address plus 72 to state offset 8, then performs the partial reset.

The exhaustion condition is equality only. There is no ordering comparison or check that at least 72 bytes remain. A cursor numerically greater than the end still takes this path if the two addresses differ. That branch behavior does not establish that the selected memory is accessible.

### Capacity selection on exhaustion

When the reusable list is empty and the cursor equals the end, the function determines a new capacity.

If the current-header address at state offset 0 is zero, the capacity is 32 records.

Otherwise, it reads the unsigned 32-bit capacity from offset 4 of the referenced header. For an old capacity \(c\), the candidates are:

\[
a = (c + 32) \bmod 2^{32}
\]

\[
g =
\left(c + \left\lfloor c/2 \right\rfloor
        + \left\lfloor c/8 \right\rfloor\right)
\bmod 2^{32}
\]

The new capacity is \(g\) if \(a \leq g\) under unsigned comparison; otherwise it is \(a\). Equivalently, the function selects the numerically greater of the two wrapped unsigned candidates.

The divisions shown above describe unsigned right shifts. They discard fractional parts independently.

There is no overflow check or post-calculation capacity bound. “Growth by at least 32” is therefore not an unconditional property: it holds only where the relevant arithmetic does not wrap.

### Storage requests and header publication

After choosing the capacity, the function calls F-871e64ddc6de twice:

1. First argument 12; second argument 0.
2. First argument equal to capacity multiplied by 72 using unsigned 32-bit arithmetic; second argument 0.

The first result is retained as the header address while the second call executes. The function performs no intervening header initialization.

After the second call returns, the accesses occur in this order:

1. Store the second result at new-header offset 8.
2. Read state offset 0 and store that value at new-header offset 0.
3. Store the chosen capacity at new-header offset 4.
4. Store the new-header address at state offset 0.
5. Read the record-block base back from new-header offset 8 and select it as the record address.
6. Store the selected base plus the wrapped capacity-times-72 byte count at state offset 12.
7. Store the selected base plus 72 at state offset 8.
8. Perform the selected-record reset.

The previous-header address used for linking is read after both dependency calls. It is not a saved copy of the earlier observation used to choose capacity. If a dependency changes state offset 0 through an alias or shared access, the capacity can be based on one header while the new link records another.

The selected record address is likewise read back from the header after the header has been published. Aliasing between state and header storage can affect that observation. The function has no visible protection against such interference.

The record-block byte count is explicitly an unsigned 32-bit product. Address calculations add that byte count or 72 to the selected address and store the result in a 32-bit address field. The evidence does not establish that crossing an address-space boundary produces a valid accessible range.

### Ordered partial reset

After the applicable list or cursor update, the selected record receives these writes in order:

1. A 32-bit zero at offset 4.
2. A 32-bit zero at offset 8.
3. A 32-bit zero at offset 12.
4. A one-byte zero at offset 0.

This is not whole-record initialization. The function does not explicitly write record bytes 1–3 or 16–71.

## Side effects

### Direct state changes

For distinct, valid storage regions and noninterfering dependency behavior:

| Path | Direct state changes |
|---|---|
| Reusable record | Replace the reusable-list address with the selected record’s successor |
| Existing sequential region | Advance the cursor by 72 |
| New block | Publish the new header, replace the end address, and set the cursor to the new base plus 72 |

The sequential paths do not directly write the reusable-list field. The reusable-record path does not directly write the cursor or end.

These statements describe writes by field role. If state, headers, or records overlap, a write through one role may also alter another. In particular, reset writes can overwrite state or header contents under an overlapping layout. The function does not detect or compensate for such aliasing.

### Memory and ownership

The new header stores a reference to the header observed after the dependency calls. The function itself makes no explicit deallocation, destructor, or reference-count call.

That absence does not establish that dependency calls preserve existing allocations. Whether F-871e64ddc6de allocates, releases, relocates, or otherwise changes memory is not established by this body.

Nor does storing a block address establish ownership, transfer rules, or a release obligation. Those requirements must come from an external contract.

Without aliasing or external interference, bytes outside the four reset destinations retain their preceding contents. For newly supplied storage, those preceding contents and their initialization status are unknown.

### Errors and partial progress

There is no explicit error result, allocation-failure branch, or rollback.

The function uses dependency results without checking them for zero or another failure representation. If a call returns normally with an unusable address, execution proceeds toward accesses through that address.

Malformed state can similarly cause invalid reads or writes. The source does not establish the resulting runtime outcome. Writes completed before a later failure are not explicitly undone.

Dependency behavior may add side effects or prevent normal return. The function’s body does not constrain those possibilities.

## Dependencies

### F-871e64ddc6de

This dependency is called only on the empty-reusable-list, equal-cursor-and-end path.

| Call order | First argument | Second argument | Subsequent use of result |
|---|---:|---:|---|
| First | 12 | 0 | Header address |
| Second | Capacity × 72, modulo \(2^{32}\) | 0 | Record-block address stored in the header |

Allocation is inferred from the request sizes and subsequent memory use, not independently established.

The dependency’s full contract is **documented_only**. In particular, the supplied evidence does not establish:

- The meaning of the second argument.
- Failure signaling or whether failure can return normally.
- Alignment or initialization guarantees.
- Whether the two returned regions are distinct.
- Whether returned regions overlap the state or existing storage.
- Whether either call modifies state, headers, records, or other shared memory.
- Whether either call releases existing storage.
- Ownership, lifetime, and release requirements.

Successful, noninterfering allocation is a condition used in some acceptance examples, not an assumed universal dependency property.

## Acceptance examples

These examples are conditional criteria derived from static evidence. No runtime tests have been performed.

### Reuse takes priority

Given a valid nonzero reusable-list address whose record has a valid successor:

- The first reusable record is selected.
- State offset 4 becomes its successor address before reset.
- The selected record receives the four reset writes.
- No dependency call occurs.
- Cursor/end equality is irrelevant because those fields are not consulted on this path.

### Removing the final reusable record

Given a selected reusable record whose successor is zero:

- State offset 4 becomes zero.
- The selected record’s offset-12 field is subsequently cleared.
- No new block is requested during this invocation.

### Existing sequential storage

Given an empty reusable list and a valid cursor different from the end:

- The old cursor identifies the selected record.
- The cursor is replaced with that address plus 72 before reset.
- No dependency call occurs.
- The current header is not consulted.

### First block

Given an empty reusable list, equal cursor and end addresses, a zero current-header address, and successful noninterfering dependency behavior with distinct storage:

- Capacity is 32.
- Requests are 12 bytes followed by 2,304 bytes, each with second argument zero.
- The new header records previous-header address zero, capacity 32, and the returned block address.
- The state publishes the new header.
- The end becomes block base plus 2,304.
- The cursor becomes block base plus 72.
- The first record receives the partial reset.

### Growth without overflow

Given exhaustion and old capacity 32:

- The additive candidate is \(32 + 32 = 64\).
- The proportional candidate is \(32 + 16 + 4 = 52\).
- Selected capacity is 64.
- The record-block request is 4,608 bytes.

Given exhaustion and old capacity 64:

- The additive candidate is \(64 + 32 = 96\).
- The proportional candidate is \(64 + 32 + 8 = 104\).
- Selected capacity is 104.
- The record-block request is \(104 \times 72 = 7,488\) bytes.

### Equality-only exhaustion

Given an empty reusable list and a cursor numerically beyond, but unequal to, the end:

- No new block is requested.
- The cursor address is selected and advanced by 72.
- Reset proceeds only as far as the accessibility of that address permits.

This specifies branch selection, not memory safety.

### Partial reset preserves other bytes

Given a valid selected record, distinct storage, and no external interference:

- Byte 0 becomes zero.
- The 32-bit fields at offsets 4, 8, and 12 become zero.
- Bytes 1–3 and 16–71 retain their previous contents.

### Wrapped capacity arithmetic

Given an old capacity of \(2^{32}-1\):

- The additive candidate wraps to 31.
- The proportional candidate wraps to 2,684,354,557.
- Unsigned comparison selects 2,684,354,557.
- Multiplication by 72 wraps to 4,294,967,080 bytes.

These are arithmetic criteria only. They do not establish that the request can succeed or that subsequent address calculations describe accessible storage.

### State mutation during dependency calls

Given exhaustion where capacity was calculated from one header, and a dependency changes state offset 0 before the post-call linking read:

- The calculated capacity is not recomputed.
- The new header’s previous-header field receives the value observed after both calls.

This example does not assume that dependency mutation is permitted or safe; it captures the function’s observable read order.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the reset fields.
- How a caller discovers or uses the selected record despite the absence of a return value.
- Complete record layout and initialization requirements for untouched bytes.
- The dependency’s actual allocation, ownership, lifetime, and failure contract.
- External invariants governing the reusable list, cursor, end, and header chain.
- Whether unusual storage overlap is prohibited externally.
- Exact runtime consequences of inaccessible memory or invalid address arithmetic.
- Target ABI details beyond the visible 32-bit representation.
- Thread-safety, reentrancy, and permitted concurrent mutation.

No synchronization is visible. The body alone does not establish safe concurrent or reentrant use.

## Evidence

This contract is based on the supplied static body of F-e297fd02e4fa and the supplied opaque dependency mapping.

Established static evidence includes selection priority, field offsets and widths, equality-only exhaustion, 72-byte record spacing, unsigned capacity calculations, dependency call order and arguments, post-call header linking, header publication and readback order, cursor updates, and the ordered partial reset.

The record-pool interpretation and allocation terminology are inferred intent. Dependency guarantees, field semantics, ownership rules, and runtime failure outcomes remain unresolved.

No runtime tests, dependency executions, or memory-layout experiments have been performed. The acceptance examples are derived conditions, not observed results.
