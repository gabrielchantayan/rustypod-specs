<!-- Trusted provenance: F-8838ed5baf7c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8838ed5baf7c

## Purpose

F-8838ed5baf7c obtains three dependency-produced values, publishes them into a caller-supplied record, invokes another dependency on the first two published values, and conditionally inserts the third published value at the head of an indexed linked structure.

Insertion depends on a four-byte selector and bit 0 of a one-byte status field. The function does not validate the selector against a table length. Table-entry address calculation uses unsigned 32-bit arithmetic and can wrap, so different accepted selectors can identify the same entry address.

The linked-structure interpretation follows from the visible field accesses and updates. Allocation, construction, destruction, and application-level registration semantics are not established by the supplied body.

## Interface

### Arguments and result

| Argument | Observed representation | Direct use |
|---|---|---|
| Record address | 32-bit integer used as an address | Base for fixed-offset record accesses. |
| Unused argument | Four-byte value | No direct use in this body. |
| First numeric input | Signed 32-bit integer | Shifted left by 16 bits for the first call to F-545766d274bf. |
| Second numeric input | Signed 32-bit integer | Shifted left by 16 bits for the second call to F-545766d274bf. |

There is no explicit return value or status result. Observable results consist of dependency invocations and memory changes.

The unused argument has no established direct effect within this function. This does not establish its significance elsewhere in the program.

### Record layout

The function accesses the following locations relative to the record address:

| Byte offset | Width | Role |
|---|---:|---|
| `0xC0` | 4 bytes | Input read for F-2b6fe108bb2f. |
| `0xD0` | 4 bytes | First published result; subsequently reread. |
| `0xD4` | 4 bytes | Second published result; subsequently reread. |
| `0xD8` | 4 bytes | Third published result; subsequently reread as the candidate address. |

Each location must support its actual reads or writes when they occur. A suitably accessible record extending through byte `0xDB` is a sufficient ordinary layout, provided dependencies preserve the memory required by later operations. No size or accessibility validation is visible.

### Candidate layout

The value eventually read from record offset `0xD8` supplies the candidate address:

| Byte offset | Width | Role |
|---|---:|---|
| `0x08` | 4 bytes | Selector, read as a signed 32-bit integer before subtraction. |
| `0x0C` | 4 bytes | Previous-link field. |
| `0x10` | 4 bytes | Next-link field. |
| `0x14` | 1 byte | Status byte; bit 0 controls insertion and is set during insertion. |

Other candidate bytes have no established meaning here.

### Global table

A global cell supplies a 32-bit integer table base. The function captures this base once, after the dependency calls.

Table entries are four bytes wide. Their addresses are derived from the captured base and a 32-bit unsigned index. Any entry actually accessed must support both a four-byte read and a four-byte write.

A nonzero next-link value used for the final update must identify storage supporting a four-byte write at offset `0x0C`. The function provides no table length, allocation extent, or alignment contract.

## Behavior

### Produce and publish the first value

The function first calls F-8157a180d901 with `0x18`, or 24 decimal. It then calls F-545766d274bf with:

- The four-byte result of that call.
- The first numeric input shifted left by 16 bits.

The four-byte result of F-545766d274bf is written to record offset `0xD0`. Neither dependency result receives an explicit validity or null check.

The shift uses a signed 32-bit operand in the supplied representation. For nonnegative inputs whose shifted result remains representable as a signed 32-bit integer, the operation is straightforward: input `1`, for example, yields `0x00010000`.

Under a wrapping 32-bit bit-vector interpretation, the shift places the input’s low 16 bits in the result’s high 16 bits and clears the low 16 result bits. That interpretation is not an unconditional guarantee for negative inputs or signed-overflow cases. Exact target behavior for those cases remains unresolved.

### Produce and publish the second value

The function next calls F-8157a180d901 with 24, then calls F-545766d274bf with that new result and the second numeric input shifted left by 16 bits.

It writes the returned four-byte value to record offset `0xD4`.

The first publication precedes both calls in this second pair. Dependencies therefore execute after the first output field has been written, although whether they can reach or modify that field is unknown.

### Produce and publish the third value

The function calls F-8157a180d901 with `0x24`, or 36 decimal. After that call returns, it calls F-2b6fe108bb2f with:

1. The four-byte result of the size-36 call.
2. A four-byte value read from record offset `0xC0`.
3. A four-byte value read from record offset `0xD0`.

The relative evaluation order of the two record reads is not established. Both occur after the size-36 call returns.

The `0xD0` value is reread from memory rather than taken from the earlier dependency result. If an intervening dependency changed the field, this call receives the changed value.

The four-byte result of F-2b6fe108bb2f is written to record offset `0xD8`.

### Process the first two published fields

The function calls F-1af11df6bb57 twice:

1. With the value read from record offset `0xD0`.
2. With the value read from record offset `0xD4`.

The first call finishes before the second argument is read. Consequently, a change to `0xD4` made by the first call can affect the second call’s argument.

Neither call’s result is used. Their effects on memory, globals, ownership, and lifetime are unknown. They are not established release operations, nor are they established to preserve the published values.

### Capture the candidate and insertion state

After both calls to F-1af11df6bb57 return, the function performs these reads in order:

1. The candidate address from record offset `0xD8`.
2. The table base from the global cell.
3. The four-byte selector at candidate offset `0x08`.

The candidate is therefore the then-current record value, not necessarily the value originally published by F-2b6fe108bb2f.

The table-base cell is read even when insertion will subsequently be rejected. Rejection does not eliminate this global read.

The selector is decremented by one, and the result is held as an unsigned 32-bit index. The supplied subtraction uses a signed 32-bit selector. For every selector except the minimum signed 32-bit value, subtracting one is representable and determines the index without signed overflow. The minimum signed selector is a separate unresolved boundary; unsigned storage of the result does not itself establish wrapping semantics for the preceding signed subtraction.

### Decide whether insertion is permitted

The function examines the signed 32-bit interpretation of the index.

- If that interpretation is nonnegative, it reads the candidate’s status byte and tests bit 0. Insertion proceeds only if the bit is clear.
- If that interpretation is negative, it skips this decision-time status read and rejects insertion. Such an index is necessarily nonzero.

For selector values whose subtraction is defined, insertion therefore requires a selector from `1` through `0x7FFFFFFF`, together with a clear low status bit.

Examples of the selector decision:

- Selector zero yields index `0xFFFFFFFF`; insertion is rejected without the decision-time status read.
- Selector one yields index zero; insertion depends on the status bit.
- Selector `0x40000001` yields index `0x40000000`; insertion also depends on the status bit.

Acceptance by this condition does not establish that the resulting table address is valid.

### Calculate the table-entry address

For an insertion, table-entry addressing has two distinct unsigned 32-bit operations:

1. Multiply the unsigned index by four, retaining the result modulo \(2^{32}\).
2. Add the captured integer table base to that displacement using unsigned 32-bit arithmetic, again retaining the result modulo \(2^{32}\).

Thus:

\[
\text{displacement}=(4\times\text{index})\bmod 2^{32}
\]

\[
\text{entry address}=(\text{unsigned table base}+\text{displacement})\bmod 2^{32}
\]

The unsigned index makes the multiplication unsigned. The addition with the 32-bit integer base is also unsigned under the supplied 32-bit model.

For example, index `0x40000000` produces displacement zero. Index zero and index `0x40000000` therefore produce the same entry address for the same captured base. Likewise, base `0xFFFFFFFC` plus displacement four produces the integer address zero.

These are established integer-arithmetic results, not guarantees that the resulting addresses are mapped, accessible, aligned, or valid for table storage. No bounds or address-validity check follows the calculation.

### Insert at the selected head

When insertion is permitted, the visible memory operations occur in this order:

1. Write four-byte zero to the candidate’s previous-link field at `0x0C`.
2. Read four bytes from the calculated table-entry address.
3. Write that value to the candidate’s next-link field at `0x10`.
4. Write the candidate address to the calculated table-entry address.
5. Read the candidate’s status byte, set bit 0, and write the updated byte back.
6. Reread the candidate’s next-link field and compare it with zero.
7. If that read is nonzero, read the next-link field again to obtain the destination object address, then write the candidate address at that object’s offset `0x0C`.

The status update preserves the other seven bits as observed at the update-time read. It does not replace the entire byte with one.

The next-link comparison is an explicit zero guard. It protects the final previous-link update when the tested value is zero. The destination is nevertheless obtained through a subsequent read; no atomic snapshot or concurrent-stability guarantee is established.

For nonoverlapping storage without intervening mutation, these operations make the candidate the new head, retain the former head as its next object, and set the former head’s previous link to the candidate.

### Complete without insertion

A rejected candidate receives none of the explicit linkage, table-entry, or status writes in the insertion sequence.

All earlier dependency calls and record publications have already occurred. Rejection is not rollback and does not produce an error result.

## Side effects

### Direct writes

On normal return through all dependencies, the function writes four-byte values to record offsets `0xD0`, `0xD4`, and `0xD8`, in that order. Existing field values are overwritten without an explicit inspection or release.

Insertion additionally writes the candidate’s previous link, next link, status byte, selected table entry, and—if the next-link guard succeeds—the addressed next object’s previous link.

The function does not explicitly clear the first two record fields after invoking F-1af11df6bb57.

### Ownership and lifetime

No ownership transfer is established. The size arguments to F-8157a180d901 are consistent with storage acquisition, but do not prove allocation, uniqueness, initialization, or failure semantics.

No dependency is independently identified as a deallocator. Conversely, the body does not establish that dependencies leave earlier results or accessible memory alive.

Successful execution requires the memory used by later accesses to remain usable. The function supplies no visible lifetime repair or recovery if a dependency invalidates it.

### Validation, failure, and concurrency

The record address, dependency results, candidate address, and captured table base receive no explicit null validation. The next-link field does receive an explicit zero check before the conditional final previous-link update.

There is no visible table-length check, error-code result, rollback, or recovery branch. Invalid addresses and dependency failures have no established safe outcome.

No locking or atomic operations are shown. Intermediate states are exposed between insertion writes.

Aliasing can change the apparent list behavior. For example, the initial previous-link write may affect a later table read if storage overlaps. The ordered accesses govern such cases; the ordinary head-insertion interpretation assumes nonoverlapping storage and stable relevant memory.

## Dependencies

| Opaque ID | Observed invocation | Established boundary |
|---|---|---|
| F-8157a180d901 | Three calls with arguments 24, 24, and 36, in that order. | Each returns a four-byte value passed to another dependency. Allocation and failure behavior are unknown. |
| F-545766d274bf | Two calls, each receiving a size-request result and a shifted numeric input. | Returns the value published at `0xD0` or `0xD4`. Other effects are unknown. |
| F-2b6fe108bb2f | One call receiving the size-36 result and reread record values from `0xC0` and `0xD0`. | Returns the value initially published at `0xD8`. Layout and lifetime guarantees are unavailable. |
| F-1af11df6bb57 | Two ordered calls using reread values from `0xD0` and `0xD4`. | Results are unused. Ownership, memory, and global effects are unknown. |

No dependency is assumed to preserve globals, preserve the record, retain memory, release memory, or return normally under invalid inputs.

## Acceptance examples

These are conditional static deductions, not executed tests. Unless otherwise stated, assume normal dependency returns, valid nonoverlapping storage, and no dependency or concurrent mutation of the relevant state.

### Empty selected chain

Given selector one, status byte `0xA4`, and a zero entry at the captured table base:

- Index and displacement are zero.
- The candidate’s previous and next links become zero.
- The selected entry becomes the candidate address.
- Status becomes `0xA5`.
- The next-link zero guard suppresses the final previous-link write.

### Nonempty selected chain

Given selector three, a clear low status bit, and a valid distinct former head at the entry address:

- Index is two; displacement is eight.
- Entry address is the captured base plus eight, modulo \(2^{32}\).
- Candidate previous link becomes zero.
- Candidate next link becomes the former head.
- The entry becomes the candidate address.
- Candidate status bit 0 becomes set.
- The former head’s previous link becomes the candidate address.

### Wrapped multiplication selects the base entry

Given selector `0x40000001` and a clear low status bit:

- Index is `0x40000000`, whose signed interpretation is nonnegative.
- Multiplication by four wraps to displacement zero.
- The selected address equals the captured table base.
- If storage there and all subsequent accesses are valid, insertion uses that entry.

The function does not reject this selector merely because it denotes a large logical index.

### Wrapped addition produces zero

Given selector two, a clear low status bit, and captured base `0xFFFFFFFC`:

- Index is one; displacement is four.
- Unsigned addition produces entry address zero.
- No explicit null check prevents the attempted entry access.

Safe completion depends on platform memory validity at the resulting address; no such validity is established.

### Already-marked candidate

Given selector one and status `0x81`, insertion is skipped. Earlier dependency calls, record publications, and the global table-base read still occur.

### Zero selector

Given selector zero, index becomes `0xFFFFFFFF`. The decision-time status read is skipped, and no insertion writes occur. The candidate selector and global table-base cell have already been read.

### Dependency changes later inputs

If the first call to F-1af11df6bb57 changes record offset `0xD4`, the second call receives the updated value.

If either call changes `0xD8`, candidate capture uses the value present after both calls return. A dependency change to the global table-base cell before capture likewise affects the captured base.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the record, numeric inputs, selector, and status bit.
- Dependency allocation, initialization, ownership, destruction, and failure semantics.
- Exact target behavior for signed left-shift boundary cases.
- Exact target behavior when decrementing the minimum signed 32-bit selector.
- Valid table extent and permitted selector range.
- Address validity, alignment, byte order, and platform memory behavior.
- Arithmetic guarantees for other signed address expressions beyond the established unsigned table-entry calculation.
- Whether overlapping layouts are supported.
- External synchronization and dependency behavior under concurrent access.
- Whether dependencies may terminate, transfer control, or fail to return.

Unsigned table-entry multiplication and addition wrapping are established, not unresolved address-overflow behavior.

An inferred intent is to obtain two related values, obtain a third associated value, process the first two, and link the third only when unmarked. This interpretation does not establish ownership or whole-function idempotence: construction and publication occur even when insertion is skipped.

## Evidence

The supplied static body establishes the dependency order, literal size arguments, field offsets, access widths, rereads, insertion conditions, unsigned 32-bit table-address calculation, ordered insertion writes, and next-link zero guard.

Dependency internals, application semantics, memory validity, and the unresolved signed-arithmetic boundaries are not established by that body.

No runtime tests have been performed. Acceptance examples are conditional deductions from static evidence, not observed executions.
