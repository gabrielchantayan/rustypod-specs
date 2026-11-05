<!-- Trusted provenance: F-3cd57bab8f84; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3cd57bab8f84

## Purpose

F-3cd57bab8f84 compares two memory-backed records using an ordered subset of their contents. It returns zero when the compared fields match. Otherwise, it returns the difference associated with the first unequal field in the comparison order.

The comparison priority is:

1. The unsigned 16-bit field beginning at byte offset 6.
2. The unsigned byte at offset 4.
3. The unsigned byte at offset 3.
4. The unsigned byte at offset 2.
5. The unsigned byte at offset 1.
6. The unsigned byte at offset 0.

Offset 5 does not participate. Offsets 6 and 7 participate together as one native-order 16-bit field, not as two separately ordered bytes.

**Established static behavior:** comparison is short-circuiting and field-prioritized. A mismatch prevents examination of lower-priority fields.

**Inferred intent:** this is an ordering comparator for a record with a deliberately selected significance order. The supplied material does not identify the record type or explain the omitted byte.

## Interface

### Inputs

The function accepts two pointers to byte-addressed storage:

- **First input:** the record used as the left operand.
- **Second input:** the record used as the right operand.

No length, format identifier, ownership flag, or output buffer is supplied.

The input layout relevant to this function is:

| Byte offset | Interpretation | Comparison priority |
|---|---|---|
| 0 | Unsigned byte | Sixth |
| 1 | Unsigned byte | Fifth |
| 2 | Unsigned byte | Fourth |
| 3 | Unsigned byte | Third |
| 4 | Unsigned byte | Second |
| 5 | Not examined | None |
| 6–7 | One unsigned native-order 16-bit field | First |

An eight-byte readable region for each input is a sufficient storage extent for every normal path. The function does not necessarily read every participating location: lower-priority reads depend on earlier equality.

### Widths and representation

The supplied representation uses byte, unsigned-short, unsigned-integer, and signed-integer types. The behavioral widths used here are:

- Byte: 8 bits.
- Unsigned-short field: 16 bits.
- Promoted unsigned arithmetic: 32 bits.
- Returned signed integer: 32 bits.

These are the conventional widths indicated by the supplied representation. An independent ABI or type-definition record was not supplied; confirmation of those widths is **documented_only**.

The 16-bit load uses the execution environment’s native byte order. No byte-order conversion is shown. Consequently, the numerical significance of offsets 6 and 7 depends on that environment.

### Output

The function returns one integer value:

- Zero if all selected fields compare equal.
- Otherwise, the unsigned subtraction result for the first unequal field, converted to the signed return type.
- At offset 0, the subtraction operands are read again after the equality test detects a mismatch.

Under conventional 32-bit two’s-complement unsigned-to-signed conversion, the return value is the ordinary signed difference between the decisive left and right values. Its magnitude is at most 65,535.

The result is **not normalized** to −1, 0, or +1.

### Preconditions

For defined execution:

- Each pointer must designate readable storage for the accesses actually reached.
- The 16-bit access at offset 6 must satisfy the applicable alignment and memory-access requirements.
- The memory representation must permit that typed 16-bit access under the execution model.
- Read failures must not occur.
- Stable input contents are required for the usual interpretation as a deterministic comparator.

Null pointers, invalid pointers, insufficient storage, and unsuitable alignment are not checked or reported by the function.

## Behavior

### Comparison progression

The first stage reads the 16-bit field at offset 6 from each input and compares the resulting unsigned values.

If these values differ, the function proceeds to the return calculation using those values. It does not examine offsets 4, 3, 2, 1, or 0.

If the 16-bit values match, the function reads offset 4 from each input. Equality permits progression to offset 3; inequality selects the offset-4 values for the return calculation.

The same rule applies successively to offsets 3, 2, and 1. Each stage is reached only when every higher-priority field matched.

If the comparison reaches offset 0, it tests the two bytes for equality:

- If they are equal, the function returns zero.
- If they differ, it reads the offset-0 bytes again and subtracts the newly obtained right value from the newly obtained left value.

For stable storage, this final reread produces the same values as the equality test. For changing or read-sensitive storage, that equivalence is not guaranteed.

### Read ordering

The established inter-stage order is:

1. Read and compare the two 16-bit fields.
2. If equal, read and compare the two offset-4 bytes.
3. If equal, read and compare the two offset-3 bytes.
4. If equal, read and compare the two offset-2 bytes.
5. If equal, read and compare the two offset-1 bytes.
6. If equal, test the two offset-0 bytes.
7. Only if the last test finds inequality, obtain the offset-0 subtraction operands again.

At the first five stages, the supplied representation places the left-input load before the right-input load in separate operations.

At offset 0, the two operands occur within a single comparison expression, and later within a single subtraction expression. The supplied material does not establish an evaluation order between the two operands of either expression.

This is a description of accesses represented statically. It does not establish exact machine instruction ordering, atomicity, or a guaranteed hardware read count. In particular, ordinary non-volatile reads may be optimized. No volatile or atomic access contract is shown.

### Arithmetic

Each loaded value is treated as unsigned before subtraction.

For a decisive byte:

- Each operand lies between 0 and 255.
- The mathematical difference lies between −255 and +255.

For the decisive 16-bit field:

- Each operand lies between 0 and 65,535.
- The mathematical difference lies between −65,535 and +65,535.

In the 32-bit arithmetic model, subtraction occurs modulo \(2^{32}\). A mathematically negative difference therefore first appears as a large unsigned result. That unsigned result is then converted to the signed return type.

Conventional two’s-complement conversion produces the expected negative difference. The supplied material alone does not independently establish the target’s rule for converting an unsigned value outside the signed range. Negative-result acceptance criteria therefore depend on that conversion rule.

Positive differences and zero are within the signed range and do not have that particular ambiguity.

### Equality and ignored storage

With stable inputs, a zero result means that all six compared fields match. It does not mean that the full records are byte-for-byte identical:

- Offset 5 may differ.
- Storage beyond offset 7 may differ.
- No record length or trailing content is examined.

The function does not compare addresses. Identical addresses are permitted if the underlying accesses are otherwise valid.

## Side effects

No memory write, allocation, deallocation, global-state access, or external call appears in the supplied body.

The function:

- Borrows access to the input storage.
- Does not take ownership.
- Does not release either input.
- Does not retain either pointer.
- Does not modify the records directly.
- Does not produce diagnostics or set an explicit error status.

Reads may have effects imposed by the surrounding environment, such as a fault on inaccessible memory or device-specific behavior on mapped storage. Such effects are not handled here.

Concurrent modification is not synchronized. The function supplies no lock, snapshot, retry, or consistency check. A changing record can produce a result based on values observed at different times. The source-level repeated offset-0 reads make that distinction especially visible, although ordinary unsynchronized concurrent access may itself fall outside the valid execution model.

## Dependencies

No dependency function is invoked in the supplied body. No additional opaque function IDs are available or required.

The behavior depends on execution-environment properties rather than a callee contract:

- Integer widths and signed-conversion rules.
- Native byte order for the 16-bit field.
- Alignment and validity requirements for that field access.
- The legality and stability of the referenced storage.
- Compiler treatment of ordinary memory reads.

There is no unknown callee whose ownership behavior or preservation of global state must be assumed.

## Acceptance examples

These are conditional static acceptance examples, not observed runtime results. Unless stated otherwise, they assume valid, stable storage, the widths listed above, and conventional 32-bit two’s-complement return conversion.

### All selected fields equal

Both inputs have equal 16-bit fields and equal bytes at offsets 4, 3, 2, 1, and 0.

**Expected:** return zero.

Offset 5 may contain different values without changing this result.

### Highest-priority field decides

The first input’s 16-bit field is 1,000 and the second input’s is 900.

**Expected:** return +100. Lower-priority bytes do not affect the result and are not reached by the represented control flow.

Reversing those field values gives −100 under the stated conversion assumption.

### Offset 4 overrides lower offsets

The 16-bit fields match. Offset 4 contains 2 in the first input and 9 in the second. All lower-priority bytes in the first input may be numerically greater than their counterparts.

**Expected:** return −7. Offsets 3 through 0 do not affect the decision.

### Descending byte priority

The 16-bit fields and offsets 4 and 3 match. Offset 2 contains 200 in the first input and 3 in the second.

**Expected:** return +197. Differences at offsets 1 and 0 are irrelevant.

### Last byte decides

Every higher-priority field matches. Offset 0 is stably 0 in the first input and 255 in the second.

**Expected:** return −255. The represented equality test detects inequality, and the subsequent subtraction reads the same stable values again.

### Maximum field difference

The first 16-bit field is 65,535 and the second is 0.

**Expected:** return +65,535, demonstrating that the comparator is not restricted to returning +1.

In the opposite direction, the unsigned subtraction result is 4,294,901,761; conventional signed conversion yields −65,535.

### Byte order changes interpretation

The first input contains bytes 1 and 0 at offsets 6 and 7. The second contains bytes 0 and 1 there.

**Expected under little-endian interpretation:** the fields are 1 and 256, producing −255 under the stated signed-conversion assumption.

**Expected under big-endian interpretation:** the fields are 256 and 1, producing +255.

No single endian-independent result is established.

### Aliased inputs

Both pointers designate the same stable, valid storage.

**Expected:** return zero. No special alias handling is needed because the function performs no writes.

## Uncertainties

The following remain **documented_only**:

- The record’s domain meaning and the reason for the comparison priority.
- The reason offset 5 is excluded.
- Independent confirmation of the type widths and return ABI.
- The target’s native byte order.
- Target-specific unsigned-to-signed conversion behavior.
- Alignment guarantees supplied by callers.
- Whether callers always provide eight-byte objects.
- Exact generated load instructions and physical read counts.
- Supported behavior, if any, for volatile, device-backed, or concurrently changing storage.

These unknowns do not justify adding validation, normalization, byte swapping, retries, or ownership operations to the contract.

## Evidence

The supplied static body establishes:

- Two input pointers and one integer return.
- A 16-bit comparison at offset 6, followed conditionally by byte comparisons at offsets 4, 3, 2, 1, and 0.
- Short-circuiting at the first unequal field.
- Unsigned promotion and subtraction.
- A separate equality test and subtraction expression at offset 0.
- No explicit read of offset 5.
- No writes, calls, allocation, cleanup, or explicit error handling.

The interpretation as an ordering comparator is inferred from that structure. ABI-dependent details are identified above rather than treated as independently verified facts.

No runtime tests have been performed.
