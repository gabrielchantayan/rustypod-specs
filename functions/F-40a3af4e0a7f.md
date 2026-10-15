<!-- Status: documented_only; source-facing review passed; no runtime validation. -->
# F-40a3af4e0a7f

## Purpose

Count records whose decoded 32-bit identifier equals a requested identifier within a container selected by an external dependency.

**Status: DOCUMENTED ONLY.** The supplied static evidence establishes the counting and decoding behavior, but not the selector contract, allocation bounds, or complete container format. Those gaps prevent an independently safe, implementation-ready specification for arbitrary inputs.

## Interface

The function receives three arguments, in this order:

| Argument | Established role |
|---|---|
| Selector value | An opaque 32-bit value forwarded unchanged to F-e81ed1edd673. Its interpretation is unknown. |
| Requested identifier | An unsigned 32-bit value compared directly against decoded record identifiers. The requested value is not itself decoded. |
| Buffer handle | A nullable pointer to a cell containing a 32-bit buffer-base value. A zero value in that cell is treated as an absent buffer. The cell's complete structure, if any, is unknown. |

The result is a 32-bit integer representing the number of matching records visited. Under defined execution, its range is 0 through 65,535 because traversal is capped by a decoded 16-bit record count.

A zero result can mean any of the following:

- The buffer handle is null.
- The handle's initial buffer-base value is zero.
- F-e81ed1edd673 returns zero.
- The selected container permits no traversal.
- Traversal visits no matching records.

There is no separate error result or diagnostic distinguishing these cases.

The function does not directly acquire ownership, release memory, or return a pointer. Ownership and lifetime requirements imposed by F-e81ed1edd673 remain unknown.

## Behavior

### Entry checks and selection

A null buffer handle causes an immediate zero result without calling F-e81ed1edd673.

A nonnull handle whose buffer-base value is initially zero also causes an immediate zero result without calling that dependency.

Otherwise, F-e81ed1edd673 is called once with:

1. The same buffer handle.
2. The opaque selector value.

A zero dependency result causes a zero result from this function. A nonzero dependency result is used as a container offset relative to the buffer base.

The buffer-base value is read again after a nonzero dependency result. Consequently, any change that F-e81ed1edd673 makes to that cell affects subsequent reads. There is no second null-base check.

### Numeric field decoding

Relevant stored numeric fields use a bytewise transformation followed by big-endian assembly:

- Multiply each stored byte by decimal 23.
- Retain only the low eight bits of each product.
- Assemble the transformed bytes in most-significant-byte-first order.

This applies to the two-byte record count and each four-byte size or identifier field described below. It is not ordinary big-endian interpretation of the stored bytes.

For example, stored bytes `01 02 03 04` decode to `0x172E455C`, not `0x01020304`.

The transformation and assembly are independent of the machine's native byte order.

### Relevant container layout

All offsets in this table are byte offsets relative to the selected container:

| Offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 4 bytes | Decoded extent used to calculate the traversal limit |
| 14 | 2 bytes | Decoded maximum number of records to visit |
| 20 | — | Starting position of the first candidate record |

The traversal limit is the container offset plus its decoded extent, expressed in the same buffer-relative coordinate system as record positions.

The function does not otherwise interpret the intervening header bytes.

### Relevant record layout

All offsets in this table are byte offsets relative to the current record:

| Offset | Width | Meaning established by use |
|---|---:|---|
| 0 | 4 bytes | Decoded advance to the next candidate record |
| 4 | 4 bytes | Decoded identifier compared with the requested identifier |

The function does not inspect record payloads beyond these fields.

The decoded advance is used relative to the current record, not relative to the container or buffer base. There is no alignment adjustment or minimum-size enforcement.

### Traversal and counting

Traversal begins 20 bytes after the selected container's start.

A record is visited only while both conditions hold:

- The decoded record-count allowance has not been exhausted.
- The current buffer-relative record position is strictly less than the calculated traversal limit, using an unsigned 32-bit comparison.

Each visit consumes one unit of the count allowance, whether or not its identifier matches.

A visited record contributes one to the result if its decoded 32-bit identifier equals the requested identifier exactly. Duplicate identifiers are counted separately. The function does not stop at the first match.

After examining the identifier, the function advances by the current record's decoded advance. The advance field is read even after the final allowed record has been examined; the next loop check then prevents further visits.

The returned value is the accumulated number of matches when either traversal condition fails.

### Arithmetic and malformed-data behavior

Record positions, traversal-limit calculation, and record advances use 32-bit arithmetic. Offset additions wrap modulo \(2^{32}\); the traversal comparison is unsigned.

No validation establishes that:

- The selected container offset is inside the allocation.
- The header is fully readable.
- A candidate record has eight readable bytes.
- The extent covers the header or a complete record.
- An advance is nonzero, forward-moving, aligned, or within the extent.
- A wrapped position still refers to the intended container.
- The post-dependency buffer base is valid.

A candidate starting below the traversal limit is eligible even if its identifier or advance field extends beyond that limit. The function checks the start position, not the full record span.

A zero advance is not rejected. It causes repeated visits to the same position until the record-count allowance is exhausted, assuming the position remains below the limit and reads remain valid. It does not by itself cause an infinite loop because every visit consumes count allowance.

A nonzero dependency result is not otherwise checked for sign or validity. This specification does not assign a meaningful container interpretation to negative or otherwise invalid selector results.

Invalid addresses or unreadable storage do not produce a defined error return in the supplied evidence. A safe interoperability precondition is that every read actually reached by traversal is valid.

## Side effects

### Direct effects

The function directly:

- Reads the buffer-base cell before selection.
- Calls F-e81ed1edd673 when the entry checks permit.
- Reads the buffer-base cell again after successful selection.
- Reads selected header and record fields.
- Returns a match count.

There are no direct writes to the buffer or handle, allocations, deallocations, logging operations, or other external calls in the supplied function.

### Dependency-mediated effects

F-e81ed1edd673 receives the buffer handle and may have side effects not established here. In particular, the caller's post-call base reload means a dependency-mediated change to that base can affect traversal.

### Stability precondition

For deterministic decoding and counting, the relevant buffer contents and post-selection base should remain stable throughout traversal. The supplied evidence establishes no synchronization or atomic snapshot mechanism.

## Dependencies

### F-e81ed1edd673 — unknown selection contract

Established interaction:

- Called only when the handle is nonnull and its initial base value is nonzero.
- Receives the handle first and the opaque selector value second.
- Called once per eligible invocation.
- A zero result prevents traversal.
- A nonzero result is used as a buffer-relative container offset.
- The handle's buffer-base value is reloaded after a nonzero result.

Unknown contract elements:

- Selector interpretation.
- Selection success and failure rules beyond this caller's treatment of zero.
- Whether zero can also describe a container at the buffer's beginning.
- Permitted result range and signedness semantics.
- Buffer validation guarantees.
- Handle mutation, allocation, ownership, and lifetime effects.
- Other side effects or failure mechanisms.

No additional function dependencies appear in the supplied evidence.

## Acceptance examples

These are specification examples derived from static evidence, **not executed or observed tests**. Unless stated otherwise, assume valid readable storage, a stable buffer, and a dependency result selecting the described container. Numeric field values below are decoded values except where stored bytes are explicitly identified.

1. **Null handle**
   - Input handle is null.
   - Expected result: 0.
   - F-e81ed1edd673 is not called.

2. **Absent initial buffer**
   - Handle is nonnull, but its initial base value is zero.
   - Expected result: 0.
   - F-e81ed1edd673 is not called.

3. **Selection returns zero**
   - Entry checks pass.
   - F-e81ed1edd673 returns zero.
   - Expected result: 0, without selected-header or record reads.

4. **Byte transformation**
   - Stored identifier bytes are `01 02 03 04`.
   - Requested identifier is `0x172E455C`.
   - That record matches.
   - Requested identifier `0x01020304` does not match those same stored bytes.

5. **Transformed count**
   - Stored count bytes at container offsets 14 and 15 are `00 01`.
   - The decoded visit allowance is 23, not 1.
   - The extent may still stop traversal before 23 visits.

6. **Duplicate matches**
   - The count allowance is 3.
   - All three record starts are below the limit.
   - Their identifiers are 7, 9, and 7.
   - Requested identifier is 7.
   - Expected result: 2.

7. **Count limits traversal**
   - The count allowance is 1.
   - Two successive records would both lie below the limit and both match.
   - Expected result: 1.
   - The first record's advance is still read; the second record's identifier is not examined.

8. **Strict extent boundary**
   - The first candidate is exactly at the traversal limit.
   - Expected result: 0, even if the count allowance is nonzero.
   - A candidate exactly at the limit is not visited.

9. **Start-only boundary check**
   - A candidate begins one byte below the traversal limit.
   - Its eight field bytes remain readable in the underlying allocation.
   - The count allowance is positive.
   - The function examines that record despite its fields extending past the logical limit.

10. **Zero advance**
    - The first candidate is below the limit.
    - Its advance is zero, its identifier matches, and the count allowance is 3.
    - Expected result: 3 from three visits to the same record position.

11. **Base changed by selection**
    - The initial handle base is nonzero.
    - F-e81ed1edd673 replaces it with another valid base and returns a nonzero container offset.
    - Header and record reads use the replacement base.

## Uncertainties

### Inferred intent

The structure is consistent with selecting a container and counting matching typed or identified records in an encoded binary format. “Container,” “record,” “identifier,” and “extent” are descriptive roles inferred from access patterns, not established format names.

The multiplication by 23 appears to be a field-byte decoding transformation. Its broader format purpose is unknown.

### Missing context

- The complete buffer and handle representations are unavailable.
- No allocation length or trusted source of bounds is supplied.
- The selector dependency's implementation and guarantees are unavailable.
- No format version, alignment requirement, payload schema, or identifier namespace is established.
- The meaning of an extent smaller than the header is not established beyond the resulting traversal comparison.
- Wrapped offsets and invalid dependency results have no established higher-level meaning.
- Fault behavior for inaccessible memory is not specified.
- Concurrent modification semantics are not established.
- No runtime behavior has been measured.

These gaps must remain explicit. Adding bounds checks, rejecting zero advances, changing wraparound arithmetic, or deduplicating repeated record positions would introduce behavior not established by this function.

## Evidence

- Snapshot: 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db
- Function: F-40a3af4e0a7f
- Referenced dependency: F-e81ed1edd673

The contract is based solely on static evidence for F-40a3af4e0a7f. F-e81ed1edd673 is identified by its call relationship; its internal behavior is unavailable.
