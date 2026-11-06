<!-- Trusted provenance: F-eebba63db7b5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eebba63db7b5

## Purpose

F-eebba63db7b5 obtains and initializes two subordinate values and one composite value, publishes their 32-bit representations in a caller-supplied object, calls a dependency on each stored subordinate value, and conditionally inserts the subsequently selected composite into an indexed linked structure.

The established behavior is the sequence of dependency calls, field accesses, arithmetic operations, and conditional writes. Describing the size requests as allocations, the shifted inputs as fixed-point quantities, or the final dependency calls as releases would require additional evidence.

There is no explicit return value or status. Observable outputs are memory changes and dependency calls.

## Interface

### Arguments and result

| Argument | Representation | Established role |
|---|---|---|
| First | Signed 32-bit integer used as an address representation | Identifies the mutable containing object. |
| Second | 32-bit value | Not directly read or forwarded. |
| Third | Signed 32-bit integer | Shifted left by 16 positions and supplied to the first subordinate initializer. |
| Fourth | Signed 32-bit integer | Shifted left by 16 positions and supplied to the second subordinate initializer. |

The unused second argument has no established direct effect. This does not imply that dependencies are isolated from caller state or global memory.

The function returns no explicit value. Callers cannot rely on a success indicator or constructed address.

### Containing-object layout

Offsets are byte offsets from the first argument’s address representation.

| Offset | Access width | Direct use |
|---|---:|---|
| 0xC0 | 32 bits | Read as an input to composite initialization. |
| 0xC4 | 32 bits | Receives the first subordinate result; later read for composite initialization and a dependency call. |
| 0xC8 | 32 bits | Receives the second subordinate result; later read for a dependency call. |
| 0xCC | 32 bits | Receives the composite result; later read to select the insertion candidate. |

The function accesses these fields individually. It does not initialize or copy the containing object as a whole.

### Candidate and table layout

The value eventually read from offset 0xCC is used as the candidate address.

| Candidate offset | Access width | Direct use |
|---|---:|---|
| 0x08 | Signed 32 bits | Read and decremented to derive the unsigned index. |
| 0x0C | 32 bits | Written with zero during insertion. |
| 0x10 | 32 bits | Receives the previous table entry and is later read as a link. |
| 0x14 | 8 bits | Bit 0 gates insertion and is set during insertion. |

A nonzero link read from offset 0x10 is used as an address with a writable 32-bit field at offset 0x0C.

A global location supplies a 32-bit table-base representation. Each selected table entry is a 32-bit value. The global’s declared storage type is not otherwise established.

### Preconditions

For a defined, normally completing execution:

- Every reached read and write must address suitable storage with the required alignment.
- Dependencies must return normally and provide values suitable for subsequent uses.
- Signed shifts and signed additions used in address formation must satisfy the execution model’s arithmetic requirements.
- The signed decrement of the candidate’s offset-0x08 field must not overflow unless additional target evidence establishes a meaning for that case.
- Integer-to-address conversions must have the intended meaning in the execution environment.
- Any shared-memory access must satisfy the environment’s synchronization requirements.

The global table-base location is read even if insertion will be skipped. A table entry itself is accessed only when insertion proceeds.

There are no explicit checks enforcing these preconditions.

## Behavior

### Construction and publication

The function performs the following dependency calls and publications in order:

1. Calls F-8157a180d901 with size 24.
2. Calls F-545766d274bf with that result and the third argument shifted left by 16 positions.
3. Writes the initializer’s 32-bit result to containing-object offset 0xC4.
4. Calls F-8157a180d901 with size 24 again.
5. Calls F-545766d274bf with that result and the fourth argument shifted left by 16 positions.
6. Writes the second initializer’s result to offset 0xC8.
7. Calls F-8157a180d901 with size 36.
8. Calls F-2b6fe108bb2f with that result and the current containing-object values at offsets 0xC0 and 0xC4.
9. Writes the composite initializer’s result to offset 0xCC.

The reads of offsets 0xC0 and 0xC4 occur after the third size request returns and before composite initialization begins. Their relative evaluation order is not established.

The second subordinate field, offset 0xC8, is not directly supplied to composite initialization.

Each size-request result is forwarded without a null check. Each initializer result is stored without a success check. The stored result need not equal the address or value supplied to that initializer.

Existing contents of offsets 0xC4, 0xC8, and 0xCC are overwritten without an explicit disposal operation on those previous values.

### Shifted inputs

The third and fourth arguments are signed 32-bit inputs. Each is shifted left by 16 positions before being passed to F-545766d274bf.

For nonnegative inputs from 0 through 0x7FFF, the result is representable as a signed 32-bit value:

- 0 supplies 0.
- 1 supplies 0x00010000.
- 0x7FFF supplies 0x7FFF0000.

Negative operands and nonnegative operands whose mathematical shifted result exceeds the signed 32-bit range are **documented_only**. The supplied representation does not establish a portable result for them.

A wrapping bit-pattern transformation or a fixed-point interpretation must not be assumed for those inputs.

### Calls on the published subordinate fields

After publishing the composite result, the function:

1. Reads the current value at offset 0xC4 and passes it to F-1af11df6bb57.
2. After that call returns, reads the current value at offset 0xC8 and passes it to F-1af11df6bb57.

These calls use fresh field reads, not necessarily the earlier initializer results.

The first call can therefore affect the argument observed by the second call. Either call may change offset 0xCC, the global table-base location, or other reachable state. No preservation guarantee is established.

The function does not directly clear either subordinate field afterward.

### Candidate selection and index derivation

After both calls return, the function reads, in order:

1. The containing object’s offset-0xCC field, obtaining the candidate address representation.
2. The global table-base representation.
3. The candidate’s signed 32-bit field at offset 0x08.

The candidate and table base used afterward are the captured values, not necessarily the values originally published or present before the dependency calls.

The index is derived in two distinct arithmetic stages:

- First, signed one is subtracted from the signed 32-bit field.
- Then, if that subtraction has a defined result, the result is converted to an unsigned 32-bit index.

The unsigned destination does not make the preceding subtraction unsigned.

For a field value of −2147483648, whose 32-bit representation is 0x80000000, subtracting one exceeds the signed range. Behavior at that boundary is **documented_only**. The supplied source alone does not establish an index of 0x7FFFFFFF, wrapping behavior, or any subsequent insertion outcome.

For every other signed 32-bit field value, the decrement is representable. Conversion of a negative result to unsigned is modulo \(2^{32}\).

### Eligibility gate

The function converts the unsigned index to a signed 32-bit value for a nonnegative test.

If the converted value is nonnegative, the candidate’s flag byte at offset 0x14 is read, and bit 0 determines the gate:

- Clear bit 0 permits insertion.
- Set bit 0 skips insertion.

If the signed test is negative, the gate retains the unsigned index. Such an index is nonzero, so insertion is skipped without an eligibility read of the flag byte.

Conversion of an unsigned value above the signed maximum back to signed is target-dependent in the supplied language model. The following table identifies the arithmetic result and separates the ordinary same-width signed interpretation from portable guarantees:

| Offset-0x08 bit pattern | Defined decrement and unsigned index | Eligibility consequence |
|---|---|---|
| 0x00000000 | Signed −1 becomes 0xFFFFFFFF | Skips under the same-width interpretation yielding a negative signed value; conversion details remain target-dependent. |
| 0x00000001 through 0x7FFFFFFF | Index 0 through 0x7FFFFFFE | Signed conversion is representable; reaches the flag check. |
| 0x80000000 | Signed decrement overflows | **documented_only**; no resulting index is established. |
| 0x80000001 through 0xFFFFFFFF | Negative signed result becomes an unsigned value from 0x80000000 through 0xFFFFFFFE | Skips under the same-width interpretation yielding a negative signed value; conversion details remain target-dependent. |

No table-length check accompanies the eligibility gate.

### Table-address arithmetic

For insertion, the selected entry’s address representation is:

\[
\bigl(B + (4I \bmod 2^{32})\bigr) \bmod 2^{32},
\]

where \(I\) is the unsigned 32-bit index and \(B\) is the captured table-base representation interpreted modulo \(2^{32}\).

Both multiplication by four and addition of the table base use unsigned 32-bit arithmetic. This wrapping is established independently of the unresolved signed-decrement boundary.

Consequences include:

- Index 0 selects the base representation.
- Index 0x40000000 also selects the base representation because its byte displacement reduces to zero.
- A nonzero displacement can wrap again when added to the base.
- Distinct eligible indexes can select the same address representation.

Arithmetic wrapping does not establish memory validity. The resulting address must still designate accessible storage of the required width and alignment.

### Insertion writes and reads

When the gate permits insertion, direct memory operations occur in this order:

1. Write zero to candidate offset 0x0C.
2. Read the selected 32-bit table entry.
3. Write that entry value to candidate offset 0x10.
4. Write the candidate address representation to the selected table entry.
5. Read the candidate’s flag byte and write it back with bit 0 set.
6. Read candidate offset 0x10 for a nonzero test.
7. If nonzero, read candidate offset 0x10 again and use that value as the address whose offset-0x0C field receives the candidate address.

The flag byte is read once for eligibility and again for the update. Bits 1 through 7 are preserved from the update read, not necessarily from the eligibility read.

The link used for the final backlink write comes from a further read after the nonzero test. Stability between those reads is not enforced.

With distinct, stable storage, the result has the shape of head insertion: a zero predecessor on the candidate, a successor pointing to the former head, a table entry pointing to the candidate, and a former-head backlink pointing to the candidate.

That interpretation is conditional. No alias checks prevent the table entry from overlapping candidate storage or the former head from being the candidate itself. Under aliasing, the ordered accesses govern the result rather than an assumed list invariant.

### Skipped insertion

A skipped insertion performs none of the insertion writes or table-entry accesses. Earlier construction, publication, dependency calls, and candidate-selection reads have already occurred and are not undone.

An already-set flag therefore does not make the entire function a no-op.

## Side effects

The function directly:

- Makes three size-request calls and three initializer calls.
- Overwrites three containing-object fields.
- Calls F-1af11df6bb57 twice.
- Reads the global table-base representation.
- Conditionally writes candidate links, the selected table entry, the candidate flag, and a linked object’s backlink.

Ownership is unresolved:

- Size arguments do not prove allocation or freshness.
- Initializers are not proven to consume their inputs or return those same inputs.
- F-1af11df6bb57 is not proven to release, retain, register, or inspect its argument.
- Composite initialization is not proven to retain either input.
- No direct clearing prevents the containing object from retaining representations that a dependency might invalidate.

There is no visible lock, atomic operation, rollback, or recovery path. Interruption or a nonreturning dependency may leave partially published state. No explicit error status is reported.

## Dependencies

| Opaque ID | Established use | Unresolved behavior |
|---|---|---|
| F-8157a180d901 | Called with 24, 24, and 36, in that order; results are forwarded to initializers. | Allocation, freshness, failure convention, ownership, mutations. |
| F-545766d274bf | Called twice with a preceding size-request result and a shifted signed input; results are stored at offsets 0xC4 and 0xC8. | Result identity, storage layout, input consumption, errors, mutations. |
| F-2b6fe108bb2f | Receives the third size-request result and current fields at offsets 0xC0 and 0xC4; result is stored at offset 0xCC. | Composite meaning, retention, ownership, errors, mutations. |
| F-1af11df6bb57 | Receives the current offset-0xC4 value, then the subsequently read offset-0xC8 value. | Release semantics, validity afterward, preservation of memory or globals. |

No dependency is assumed to preserve shared state merely because its apparent role suggests a narrow operation.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless stated otherwise, they assume valid arithmetic, normally returning dependencies, suitable non-overlapping storage, and no intervening mutation.

### Empty selected entry

If the candidate’s index-source field is 1, its flag byte is 0xA0, and the entry at the captured table base is zero:

- The index is zero.
- Candidate offsets 0x0C and 0x10 become zero.
- The selected entry becomes the candidate address.
- The flag becomes 0xA1.
- No former-head backlink is written.

### Nonempty selected entry

If the index-source field is 3, bit 0 is clear, and the wrapped address at base plus eight contains a distinct valid head:

- The candidate’s predecessor becomes zero.
- Its successor becomes the former head.
- The selected entry becomes the candidate.
- The former head’s predecessor becomes the candidate.

### Wrapped displacement

If the index-source field is 0x40000001, signed subtraction is defined and produces index 0x40000000. This index is nonnegative in the signed test.

If bit 0 is clear, multiplication by four reduces to zero modulo \(2^{32}\). Insertion accesses the entry at the captured table base, not a distinct entry 4 GiB beyond it.

### Wrapped base addition

If the captured base representation is 0xFFFFFFFC and the index-source field is 2, the index is 1 and the displacement is four. The selected address representation is zero.

This specifies address arithmetic only. Normal completion still requires that address zero designate suitable accessible storage.

### Flag already set

For a defined nonnegative index and an already-set bit 0, insertion writes do not occur. All preceding construction and dependency operations still occur.

### Zero index-source field

A zero field is decremented to signed −1 and then converted to unsigned 0xFFFFFFFF.

On a target where conversion back to signed yields −1, insertion is skipped without an eligibility flag read. The candidate and global table-base reads have already occurred.

### Signed-decrement boundary

A field representation of 0x80000000 denotes the minimum signed 32-bit value. Its decrement overflows.

No acceptance outcome for the gate, selected entry, or insertion is established from the supplied source alone. In particular, index 0x7FFFFFFF must not be required without additional arithmetic evidence.

### Dependency changes later inputs

If the first F-1af11df6bb57 call changes offset 0xC8, the second call receives the changed value.

If either call changes offset 0xCC or the global table-base location, subsequent reads use the resulting values.

### Zero size-request result

If F-8157a180d901 returns zero, that zero is passed to the corresponding initializer without a check. No final result is specified because dependency handling and subsequent address validity are unknown.

## Uncertainties

The following remain **documented_only**:

- Signed-shift outcomes outside the nonnegative, representable range.
- Behavior when decrementing the minimum signed 32-bit index-source value.
- Target-specific conversion of out-of-range unsigned indexes to signed values.
- Integer-to-address mapping and signed address-addition requirements.
- Dependency implementations, failure conventions, ownership, and global effects.
- Table extent, intended index range, and permitted address wrapping.
- Semantic meanings of the object fields, flag, and linked structure.
- Intended aliasing, concurrency, and repeated-invocation conditions.

Repeated invocation is not established as idempotent. It repeats construction and overwrites all three result fields before checking the later-selected candidate’s insertion flag.

## Evidence

The supplied static representation establishes argument widths, field offsets, dependency call order, publication order, the signed decrement followed by unsigned assignment, the signed-conversion gate, unsigned 32-bit table-address arithmetic, and the conditional insertion accesses.

It does not establish wrapping for the signed-decrement overflow at field representation 0x80000000. That boundary is distinct from the defined modulo-\(2^{32}\) multiplication and addition used to select a table address.

Allocation, fixed-point, release, ownership, and registry descriptions are inferred intent only, not required semantics.

No implementation execution, runtime observation, or tests have been performed.
