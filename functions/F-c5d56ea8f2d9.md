<!-- Trusted provenance: F-c5d56ea8f2d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c5d56ea8f2d9

## Purpose

F-c5d56ea8f2d9 conditionally copies one fixed-size record from an indexed backing store into caller-supplied storage.

The function obtains an unsigned 32-bit bound from F-7e6b192346c3. It proceeds only if the requested index is strictly below that bound and a subsequently checked backing-store reference is nonzero. On the successful path, it reads four 32-bit source words and writes them to four corresponding destination positions, transferring 16 bytes.

Both the index scaling and the addition of the backing-store reference use unsigned 32-bit arithmetic. Each retains only its low 32 bits. The resulting 32-bit value is then converted to a source pointer. These established arithmetic rules do not establish the platform’s integer-to-pointer conversion or memory-access behavior.

The normal return is a 32-bit status: zero after the four writes, or all bits set after either explicit eligibility check fails. Invalid memory accesses and abnormal dependency behavior have no established normal-return contract.

This specification describes static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The function receives three inputs:

1. **Descriptor reference.** An address-like input identifying an object with these directly accessed fields:

   | Descriptor byte offset | Width | Use |
   |---|---:|---|
   | 4 | 32 bits | Argument supplied to F-7e6b192346c3 |
   | 8 | 32 bits | Backing-store reference, checked for zero and read again for address formation |

   The function does not directly access the descriptor’s first four bytes. Its full layout, total size, and other field meanings are unknown.

2. **Index.** An unsigned 32-bit record selector. The comparison against the dependency’s result is unsigned and strict; equality is rejected.

3. **Destination reference.** An address-like input identifying storage for four consecutive 32-bit writes at byte offsets 0, 4, 8, and 12.

There is no destination-capacity argument and no independent source-capacity argument.

### Record layout

The selected record consists of four consecutive 32-bit words:

| Record byte offset | Width | Treatment |
|---|---:|---|
| 0 | 32 bits | Copied without interpretation |
| 4 | 32 bits | Copied without interpretation |
| 8 | 32 bits | Copied without interpretation |
| 12 | 32 bits | Copied without interpretation |

The transfer is shallow and bit-preserving at the word level. The function does not interpret any word as a pointer, flag, number, or owned object. It establishes no external serialization format or byte-order conversion.

### Output

| Returned 32-bit pattern | Established meaning |
|---|---|
| `0x00000000` | Execution completed the four destination writes and reached the successful return. |
| `0xFFFFFFFF` | The index failed the unsigned bound check, or the backing-store value read for the nonzero check was zero. |

The failure status is specified as a bit pattern. Interpreting it as a signed 32-bit value would produce negative one.

These statuses describe normal returns only. They do not specify outcomes for inaccessible memory, invalid pointer conversions, or a dependency that fails to return normally.

### Preconditions for normal completion

The descriptor must support each field access that execution reaches. F-7e6b192346c3 must accept the supplied argument and return normally.

For successful completion, the source pointer obtained after the complete wrapping address calculation must permit all four source-word reads. The destination must permit all four writes. Any applicable alignment, lifetime, address-representation, and memory-model requirements must hold.

A nonzero backing-store value alone does not establish these preconditions. Neither does an index below the returned bound.

There is no explicit null check on the descriptor or destination, no verification of destination capacity, and no check that the backing allocation matches the dependency’s bound.

## Behavior

### Dependency call precedes eligibility checks

The function first reads the 32-bit descriptor field at byte offset 4 and passes its value to F-7e6b192346c3. The result is consumed as an unsigned 32-bit bound.

This call occurs before either eligibility check. It is reached even when the index will be rejected or the backing-store field would be zero.

No backing-store field value is captured before the call. Any dependency-induced change to that field can therefore affect subsequent reads. The dependency is not assumed to preserve the descriptor, source storage, destination storage, globals, or ownership state.

### Eligibility checks short-circuit

After the dependency returns:

- If the index is greater than or equal to the unsigned bound, the function returns `0xFFFFFFFF`.
- Otherwise, it reads the 32-bit descriptor field at byte offset 8 and checks whether its bit pattern is zero.
- If that checked value is zero, the function returns `0xFFFFFFFF`.
- Otherwise, it proceeds to source-address formation.

The index-rejection path does not directly read descriptor offset 8. Neither explicit rejection path directly reads a source-record word or writes a destination word.

These restrictions concern this function’s direct operations, not the earlier dependency’s possible effects.

### Source-address formation and wrapping

After the nonzero check succeeds, the function reads the backing-store field at descriptor offset 8 again. This second value, rather than the value previously checked, participates in address formation.

The address calculation has two established unsigned 32-bit stages:

1. The index is multiplied by 16. Only the low 32 bits of the product are retained.
2. The 32-bit backing-store value from the second read is added to that product. Only the low 32 bits of the sum are retained.

Equivalently, the final address-forming integer is the backing-store value plus 16 times the index, reduced modulo \(2^{32}\). The representation establishes 32-bit wrapping for both operations, not merely for the multiplication.

The backing-store field’s use in the zero check does not make its addition signed. When combined with the unsigned 32-bit product, the addition operates in the unsigned 32-bit domain. A high-bit-set backing-store value therefore contributes its corresponding 32-bit unsigned value.

There is no explicit overflow check for either stage:

- A sufficiently large index can wrap the multiplication.
- A sufficiently large backing-store value can make the addition wrap even when the multiplication does not.
- A nonzero checked backing-store value does not prevent the final address-forming integer from becoming zero.

Only after this arithmetic is the resulting 32-bit integer converted to a source pointer. The integer-to-pointer mapping, pointer validity, and final machine-address behavior remain documented_only. They must not be used to weaken or replace the established modulo-\(2^{32}\) arithmetic.

The function does not separately check the resulting pointer or validate the four-word readable extent. The address calculation described here selects the record’s starting pointer; it does not establish how an invalid or boundary-crossing subsequent pointer access behaves.

### Source reads and destination writes

Once the source pointer has been formed, the represented direct access order is:

1. Read the source word at record byte offset 4.
2. Read the source word at record byte offset 8.
3. Read the source word at record byte offset 12.
4. Read the source word at record byte offset 0.
5. Write the captured offset-0 word to destination byte offset 0.
6. Write the captured offset-4 word to destination byte offset 4.
7. Write the captured offset-8 word to destination byte offset 8.
8. Write the captured offset-12 word to destination byte offset 12.

The function then returns zero.

All four source values are obtained before the first direct destination write. The writes occur in ascending destination-offset order. No source word is reread after destination writing begins.

This order is established at the supplied representation level. It is not a guarantee of individual machine-instruction order, volatile-access semantics, or inter-thread visibility.

### Aliasing and overlap

The function has no explicit overlap rejection.

For valid ordinary memory under the applicable aliasing rules, capturing all four source values before writing avoids progressive corruption from earlier destination writes overwriting later source values.

An exactly matching source and destination does not trigger a special case: the represented reads and writes still occur.

Destination storage may overlap descriptor storage or other caller state if the execution environment permits those accesses. Such writes can change that storage. After source-address formation, the function performs no further direct descriptor reads.

Overlap validity under the original language, compiler assumptions, concurrency model, or special memory mappings remains documented_only. The represented access order does not independently establish that every overlap arrangement is valid.

## Side effects

### Direct effects

The visible direct operations are:

- One 32-bit read from descriptor offset 4.
- One invocation of F-7e6b192346c3, provided execution reaches and completes the preceding argument read.
- Conditional reads from descriptor offset 8: none after index rejection, one after a zero-field rejection, and two on the path toward copying.
- Four source-word reads and four destination-word writes on successful completion.

There are no visible direct allocations, releases, reference-count updates, or metadata updates. Descriptor storage is not directly written unless the destination aliases it.

Neither explicit rejection branch clears or otherwise directly writes the destination. This is not an invocation-wide preservation guarantee, because the dependency’s effects are unknown.

### Ownership

The function visibly borrows the supplied references to access storage and copies four word values. No direct ownership transfer, reference retention, or release operation is established.

If the words encode references, only their raw values are copied. There is no visible deep-copy or lifetime-management behavior.

No ownership or memory-preservation guarantee is assigned to F-7e6b192346c3.

### Errors and partial completion

The two explicit rejection conditions return the same status. The return value alone does not identify which check failed.

No recovery or rollback is visible for invalid reads or writes. If execution stops during destination writing, a previously written prefix may remain changed. No normal return is established in that situation.

The transfer is not specified as atomic. No locking, retry, snapshot validation, or consistency mechanism is visible. Four captured source words need not represent one coherent concurrent state if another actor modifies the source during the reads.

## Dependencies

### F-7e6b192346c3

**Invocation:** Once when the initial descriptor-field read succeeds and execution reaches the call; before all eligibility checks.

**Argument:** The 32-bit value read from descriptor byte offset 4.

**Consumed result:** An unsigned 32-bit bound for the strict index comparison.

**Established role:** Supplies the comparison limit controlling whether the backing-store field is examined.

**Inferred intent:** The bound likely represents a record count or equivalent selection limit. That interpretation is not required by the observed behavior and does not establish actual storage capacity.

**Unknowns:** Accepted arguments, side effects, global-state changes, descriptor or destination mutation, ownership actions, storage invalidation, exceptional behavior, and non-returning behavior.

No other explicit function dependency appears in the supplied body.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, assume normal dependency return, stable ordinary memory, valid alignment and lifetimes, valid pointer conversion and accesses, and no interfering mutation.

### First and last in-range records

If the bound is 3, the index is 0, and both backing-store reads yield the same valid nonzero reference, the function transfers that reference’s first 16-byte record and returns zero.

If the index is instead 2 and the addition does not wrap, the source starts 32 bytes after the backing-store value. The destination receives the four corresponding source words, and the function returns zero.

### Equality and zero-bound rejection

If the bound is 3 and the index is 3, the function returns `0xFFFFFFFF`. After the dependency returns, it does not directly read the backing-store field, read a source word, or write the destination.

If the bound is zero, every unsigned 32-bit index is rejected in this manner, including zero.

### Zero backing-store check

If the bound is 1, the index is 0, and the backing-store value read for the check is zero, the function returns `0xFFFFFFFF`.

It performs no second backing-store read, source-word read, or destination write.

### Dependency mutation and separate backing-store reads

If the dependency changes descriptor offset 8 before returning, the function’s later reads use the post-call state, subject to the applicable memory model.

If the checked backing-store value is nonzero but the subsequent address-formation read produces a different value, the latter value is used in the wrapping calculation. The first value’s nonzero status does not validate the second value or the resulting pointer.

### Multiplication wrap

If the bound is `0x10000001` and the index is `0x10000000`, the unsigned comparison accepts the index. Multiplication by 16 yields a retained 32-bit offset of zero.

If the backing-store check succeeds, the address-forming integer therefore equals the second backing-store value. Successful copying remains conditional on valid pointer conversion and accesses; acceptance does not establish that this wrapped selection is intended.

### Addition wrap without multiplication wrap

If the bound is 3, the index is 2, and both backing-store reads yield `0xFFFFFFF0`, the nonzero check succeeds.

The product is `0x00000020` and does not overflow 32 bits. Adding it to the backing-store value nevertheless wraps, producing the final 32-bit address-forming integer `0x00000010`.

The function converts that value to a pointer. It does not use an unbounded sum or retain a carry into a wider address. If the resulting source accesses and destination writes are valid, it copies from that source and returns zero; otherwise, no normal outcome is established.

### Addition produces zero

If the bound is 2, the index is 1, and both backing-store reads yield `0xFFFFFFF0`, the checked value is nonzero and the product is `0x00000010`.

The wrapping sum is zero. There is no further zero check before pointer conversion and source access. Whether this results in an invalid access or any other outcome depends on the unresolved address-conversion and memory semantics.

### Overlapping destination

If a valid destination overlaps the selected record by one word, all four source values are captured before destination writing begins.

Under the stated ordinary-memory and aliasing assumptions, the destination receives those captured values rather than values progressively overwritten by earlier destination stores.

## Uncertainties

The following remain documented_only:

- The full descriptor layout and meaning of its offset-4 field.
- Whether the returned bound matches backing-store capacity.
- Integer-to-pointer conversion, actual pointer width, and machine-address mapping.
- Alignment requirements and validity of individual source and destination accesses.
- Behavior when the converted pointer is invalid or the four-word extent crosses an inaccessible boundary.
- Machine-level ordering, individual-access atomicity, and device-memory semantics.
- Which overlaps are valid under the original language and compiler rules.
- Dependency side effects, ownership actions, storage invalidation, and abnormal completion.
- Thread safety and visibility under concurrent descriptor or record mutation.
- Whether wrapped selection is intentional or reachable in ordinary use.

The arithmetic widths are not unresolved: both index multiplication and backing-reference addition retain only their low 32 bits before pointer conversion.

## Evidence

**Established static evidence:** A descriptor-field read at offset 4 supplies the sole explicit dependency call. Its result controls an unsigned strict index comparison. A conditional zero check reads descriptor offset 8, followed by a separate read for address formation. The record stride is 16 bytes. Both the multiplication and backing-reference addition use unsigned 32-bit wrapping arithmetic. The resulting value is converted to a source pointer. Four 32-bit source reads precede four ordered destination writes. Normal return patterns are zero and `0xFFFFFFFF`.

**Inferred intent:** Retrieval of one fixed-size record from a collection-like object. This does not establish capacity, ownership, record semantics, or the intended meaning of wrapping addresses.

**Verification limit:** The contract derives solely from the supplied static representation. No runtime tests, execution traces, dependency implementation, or independent platform evidence were available.
