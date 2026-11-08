<!-- Trusted provenance: F-e35b2bccb75d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e35b2bccb75d

## Purpose

F-e35b2bccb75d performs three ordered activities:

1. Conditionally processes and releases a globally referenced pointer table.
2. Applies two independently requested resets to a caller-supplied object.
3. Attempts an operation on a globally referenced handle, returning a status determined by that handle and its dependencies.

The resets and table cleanup precede all explicit return paths. A later nonzero result does not roll back their effects.

**Established static evidence:** The supplied representation establishes the access widths, offsets, branch conditions, dependency-call order, and explicit return values described below.

**Inferred intent:** The pointer-table phase resembles disposal of registered objects. The final phase resembles an ownership-sensitive release operation with a mode-dependent nesting counter. These descriptions are explanatory only; they do not establish object ownership, synchronization guarantees, or the meanings of returned status values.

**Unknown:** Dependency implementations, storage lifetimes, calling conventions for the indirect callback, and the wider subsystem contract are unavailable. No runtime tests have been performed.

## Interface

### Inputs and result

The function accepts three 32-bit input values:

| Input | Interpretation established by use |
|---|---|
| Object address | A base address used for optional writes at fixed offsets. |
| First reset selector | Zero disables the first reset; any nonzero value enables it. |
| Second reset selector | Zero disables the second reset; any nonzero value enables it. |

It returns a 32-bit integer. Explicit local results are zero, 5, and 26. Two dependency paths can instead supply the returned value.

Neither selector is restricted to 0 or 1. Negative nonzero values enable the corresponding reset.

There is no explicit validation of the object address. If both selectors are zero, this function performs no direct access through that address. That does not establish that any particular address is permitted by the surrounding API.

### Referenced storage

The function uses a global handle reference, a global cleanup-record reference, and a distinguished 32-bit tag value.

The cleanup record has these relevant offsets:

| Offset | Width | Observed use |
|---|---:|---|
| 0 | 8 bits | Unsigned entry count and cleanup gate |
| 1–3 | Three individual bytes | Cleared only when the initial cleanup gate is nonzero |
| 8 | 32 bits | Pointer-table address |
| 8–11 | Four individual bytes | Cleared after the table-release dependency returns |

Bytes 4–7 have no direct access in the supplied body.

The table consists of 32-bit pointer slots with a four-byte stride. A nonzero slot identifies an object whose first 32-bit field supplies a dispatch-table address. A callable address is fetched at byte offset 4 within that dispatch table.

The caller-supplied object has these reset locations:

| Offset | Width | Value written |
|---|---:|---|
| 0x2D0 | 32 bits | 0x7FFFFFFF |
| 0x2D4 | 32 bits | Zero |
| 0x2E0–0x2E3 | Four individual bytes | Zero |

The global handle has these relevant locations:

| Offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Compared with the distinguished tag |
| 4 | 32 bits | Compared with the result of F-196812b4b122; sometimes cleared |
| 0x0E | 16 bits, unsigned | Mode bits, captured before the identity dependency |
| 0x12 | 16 bits | Signed counter, conditionally decremented or cleared |
| 0x14 | Address within the handle | Passed to F-259d2fe281b0 |

These accesses do not establish a complete structure layout. Byte order, allocation extent, and the meanings of untouched fields are not supplied.

### Preconditions and memory safety

For defined execution, each address actually accessed must be valid for the indicated operation and width. In particular:

- The cleanup-record reference must support its initial byte read.
- Every table slot reached by iteration must be readable.
- Every nonzero slot must support the dispatch reads and lead to a callable target.
- A requested reset requires writable object storage at its respective offsets.
- A nonzero handle must support the final-phase accesses and any requirements imposed by its dependencies.

The function does not check bounds, pointer provenance, alignment, or lifetime. Dependencies must not invalidate storage that this function subsequently accesses if execution is to remain valid.

No disjointness between the object, cleanup record, table, and handle is established. Aliasing can make earlier writes affect later decisions.

## Behavior

### 1. Entry dependency

The first action is a call to an entry dependency, passing the current global handle value.

That call is unconditional, including when the handle is zero and when both reset selectors are zero. No return value from this call is used in the supplied body.

Its implementation is unavailable. In particular, the function must not be described as retaining the entry-time values of the global cleanup reference or handle across this call: the cleanup reference is obtained afterward, and the final handle is obtained later still.

### 2. Conditional cleanup

After the entry dependency returns, the function captures the cleanup-record address and reads its count byte.

If that byte is zero, the entire cleanup block is skipped. In this case:

- The pointer-table field is not inspected by this block.
- No table callback or table-release dependency is invoked.
- Bytes 1–3 are not cleared.
- A nonzero table address may remain stored.

If the initial count is nonzero, the function examines the 32-bit table address at offset 8.

#### Nonzero table address

Iteration begins at index zero. Before each iteration, the index is compared against the cleanup record’s **current** count byte. The count is not a fixed initial loop bound.

For each admitted index:

1. The current table address is read from offset 8 of the captured cleanup record.
2. The 32-bit slot at four times the index is read.
3. A zero slot produces no callback.
4. A nonzero slot causes a dispatch-table read from the pointed-to object, followed by an indirect call through dispatch offset 4.
5. The index advances by one using an eight-bit mask.

The source representation does not expose explicit arguments for this indirect call. It does not establish a receiver-passing convention, callback result, or whether the callback destroys the pointed-to object.

Because both count and table address are reread, callback changes to either can affect subsequent iterations. For example, a callback that reduces the count below the next index can end traversal early. A callback that replaces the table can cause later slots to be read from the replacement.

The count is an unsigned byte. Under ordinary completed execution, iteration can visit only indices 0 through 254, at most 255 iterations. Index 255 cannot satisfy the comparison against any representable count byte, so the masked increment does not create a wraparound traversal.

After iteration ends, F-58ba61416cc9 is called with the table address read again from the captured cleanup record. Consequently, the released argument need not equal the table address seen before iteration if callbacks changed that field.

Only after that dependency returns does the function clear offsets 8, 9, 10, and 11, in that order, using byte stores.

#### Zero table address

If the table address was zero when initially checked, iteration and F-58ba61416cc9 are skipped. There are no table-field clearing stores on this branch.

#### Common cleanup completion

For either table-address branch, provided the initial count was nonzero, offsets 0, 1, 2, and 3 of the captured cleanup record are cleared in that order using byte stores.

The function does not directly clear individual table slots. It also does not revisit the global cleanup reference after capturing the record address for this phase.

### 3. Optional object resets

The reset phases occur after cleanup and before final handle processing.

If the first selector is nonzero, the function writes:

1. The 32-bit value 0x7FFFFFFF at object offset 0x2D0.
2. A 32-bit zero at object offset 0x2D4.

If the second selector is nonzero, it then clears object offsets 0x2E0, 0x2E1, 0x2E2, and 0x2E3, in ascending order, with individual byte stores.

The selectors are independent. No dependency is called between these direct reset stores. The function does not restore their former contents if final handle processing returns an error-like value.

### 4. Final handle processing

After the resets, the function obtains the global handle reference.

If it is zero, the function returns 26. No tag comparison, identity lookup, counter access, or F-259d2fe281b0 call follows on that path.

For a nonzero handle, processing continues as follows.

#### Distinguished-tag path

The first 32-bit field is compared with the distinguished tag.

If they match, F-621dae46a146 is called with the handle and a zero second argument.

- A nonzero result is returned immediately and unchanged.
- A zero result allows ordinary handle processing to continue.

A tag mismatch skips this dependency.

The handle used for subsequent ordinary processing is the one captured before this call. There is no explicit refresh of that captured handle after F-621dae46a146 returns. Changes the dependency makes to that handle’s contents can nevertheless affect subsequent reads.

#### Identity comparison

The unsigned 16-bit field at handle offset 0x0E is captured first. F-196812b4b122 is then called without explicit arguments.

After that call returns, the handle’s 32-bit field at offset 4 is read and compared with the dependency’s result.

This ordering matters:

- Mode decisions use the value captured **before** F-196812b4b122.
- The offset-4 comparison uses the value read **after** that call.
- Later counter accesses also occur after that call.

If the comparison fails, the function returns 5. On this branch it performs no direct counter write, no direct clearing of offset 4, and no F-259d2fe281b0 call.

Those statements describe this function’s own actions; they do not imply that earlier dependencies preserved the handle.

#### Mode-dependent counter behavior

If the identity comparison succeeds, bits 4 and 5 of the captured mode field determine whether a counter decrement is required.

The counter-decrement mode is selected exactly when those two bits have numeric value 2: bit 5 is set and bit 4 is clear. Other bits do not affect this decision.

In counter-decrement mode, the 16-bit signed counter at offset 0x12 is reduced by one and stored back at that width. The resulting signed 16-bit value is compared with zero.

- If the result is nonzero, the function returns zero without calling F-259d2fe281b0 or directly clearing offset 4.
- If the result is zero, processing proceeds to the final operation.

The decrement is not saturating. An initial zero counter becomes the 16-bit representation of −1 and does not trigger the final operation. The narrowed result of decrementing −32768 is 0x7FFF; it likewise does not trigger that operation.

For every other mode, the decrement is skipped and the final operation proceeds immediately.

#### Final operation

The function performs these actions in order:

1. Writes a 16-bit zero at handle offset 0x12.
2. Writes a 32-bit zero at handle offset 4.
3. Calls F-259d2fe281b0 with the address at handle offset 0x14.
4. Returns that dependency’s result unchanged.

The argument is the address of embedded storage, not a pointer loaded from offset 0x14. No rollback is visible if the dependency returns nonzero.

## Side effects

Direct writes are limited to the cleanup-record fields, requested object-reset fields, and conditional handle fields described above.

Additional effects may arise from:

- The unconditional entry dependency.
- Each nonzero table slot’s indirect callback.
- F-58ba61416cc9.
- F-621dae46a146.
- F-196812b4b122.
- F-259d2fe281b0.

Their memory, ownership, synchronization, and global-state effects are not established here.

The byte-by-byte clears are distinct ordered writes, not demonstrated atomic 32-bit resets. No explicit atomic operation or memory-ordering primitive is visible in this function. The entry dependency may provide synchronization, but that remains unknown.

Calling the table-release dependency does not, by itself, prove an allocator pairing or ownership transfer. Likewise, the indirect callbacks do not prove that their objects are freed.

## Dependencies

| Opaque identifier | Established interaction |
|---|---|
| F-58ba61416cc9 | Receives the current table address after traversal; result is not used. |
| F-621dae46a146 | Receives the matching-tag handle and zero; a nonzero result terminates processing. |
| F-196812b4b122 | Produces the 32-bit value compared with handle offset 4. |
| F-259d2fe281b0 | Receives the address at handle offset 0x14; its result becomes the function result. |

Two additional call targets have no assigned function ID in the supplied dependency map:

- **Unmapped entry dependency:** Called first with the global handle value.
- **Unmapped indirect dependency:** Selected separately through each nonzero table entry’s dispatch table.

These labels identify unresolved dependency roles, not claimed identities. Their contracts are **documented_only**.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. Unless a case states otherwise, dependencies return normally and do not mutate storage relevant to the example.

1. **Empty cleanup with a retained table address.**  
   With count zero, a nonzero table field, both selectors zero, and the final global handle zero, no cleanup callback or F-58ba61416cc9 call occurs. The cleanup bytes remain unchanged. The result is 26 after the entry dependency.

2. **Nonempty cleanup without a table.**  
   With count three and table address zero, no callback or table-release call occurs. Cleanup offsets 0–3 are cleared byte by byte.

3. **Sparse table.**  
   With count three and slots containing a valid object, zero, and another valid object, callbacks occur for the first and third slots, in that order. F-58ba61416cc9 follows traversal. Table-field bytes are then cleared before count-region bytes.

4. **Callback shortens traversal.**  
   With an initial count of three, if the first callback changes the count to one, no second slot is visited. The current table address is still passed to F-58ba61416cc9.

5. **Independent reset selectors.**  
   With the first selector zero and the second selector negative, only object offsets 0x2E0–0x2E3 receive reset writes. A later return of 26 does not undo them.

6. **Distinguished-tag early result.**  
   If the tag matches and F-621dae46a146 returns 9, the function returns 9 without calling F-196812b4b122 or F-259d2fe281b0. Earlier cleanup and resets have already occurred.

7. **Identity mismatch.**  
   If F-196812b4b122 returns a value unequal to the subsequently read offset-4 field, the result is 5. No direct counter modification follows.

8. **Nested counter remains active.**  
   With matching identity, counter-decrement mode, and counter two, the counter becomes one. The function returns zero without directly clearing offset 4 or calling F-259d2fe281b0.

9. **Counter reaches zero.**  
   With matching identity, counter-decrement mode, and counter one, the decrement stores zero. The function then stores zero to the counter again, clears offset 4, and returns F-259d2fe281b0’s result.

10. **Counter starts at zero.**  
    With matching identity and counter-decrement mode, a zero counter becomes −1. The function returns zero without the final operation.

11. **Other mode bypasses decrement.**  
    With matching identity and any mode other than the specified two-bit value, the counter is cleared regardless of its previous value, offset 4 is cleared, and the final dependency is called.

## Uncertainties

The following remain **documented_only**:

- Semantic names for the structure fields and status values 5 and 26.
- Whether the entry dependency acquires synchronization, validates the handle, or changes global references.
- The indirect callback’s argument convention and ownership effects.
- Whether F-58ba61416cc9 releases an allocation and which allocation discipline applies.
- Whether the identity result denotes a thread, task, owner token, or something else.
- Whether counter underflow is expected, prohibited by callers, or an exceptional state.
- Reentrancy and concurrent-mutation guarantees.
- Error, exception, and nonlocal-exit behavior of all dependencies.
- The validity of storage after dependency calls.

No dependency is assumed to preserve globals, preserve referenced objects, or acquire or release memory merely because of its position in this function.

## Evidence

This contract is derived solely from the supplied static function representation and opaque dependency mapping.

Established evidence includes the explicit field offsets and access widths, conditional branches, live count and table rereads, callback dispatch, ordered stores, mode mask, 16-bit counter narrowing, and result propagation.

Purpose and subsystem terminology are identified as inferred intent rather than established behavior. Unavailable dependency contracts are left unresolved rather than supplied by analogy.

No implementation execution, runtime tests, allocation tracing, concurrency testing, or independent binary verification has been performed.
