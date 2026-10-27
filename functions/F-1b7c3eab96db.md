<!-- Trusted provenance: F-1b7c3eab96db; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1b7c3eab96db

## Purpose

F-1b7c3eab96db searches a count-prefixed sequence of 32-bit integers for a target. It examines elements in increasing position order, stops at the first match, and constructs a 64-bit result from that position. Exhausting the sequence without a match returns zero.

The apparent intent is a first-match position mask. That interpretation is only partially established. The successful-return calculation contains an out-of-range unsigned-to-signed conversion for positions below 32, potentially invalid shifts, and a potentially out-of-range conversion to the signed return type. These operations prevent an unconditional one-hot-mask contract.

This specification distinguishes:

- **Established static behavior:** layout, count checks, search order, first-match selection, arithmetic widths, and the represented return construction.
- **Inferred intent:** encoding the selected position as one set bit in a 64-bit result.
- **documented_only uncertainties:** execution semantics not established by the supplied representation.

No runtime tests have been performed.

## Interface

### Inputs and layout

The function accepts:

1. A pointer to a count-prefixed sequence of 32-bit integer words.
2. A signed 32-bit target value.

The sequence has this logical layout:

| Byte offset from base | Meaning |
|---|---|
| 0 | Signed 32-bit element count |
| 4 | Element at zero-based position 0 |
| 8 | Element at zero-based position 1 |
| \(4(i+1)\) | Element at zero-based position \(i\) |

The count describes the number of elements after the count word. It is neither a byte length nor a count including the prefix. There is no represented padding between the prefix and the first element.

Elements are compared with the target using 32-bit integer equality. Negative values require no special handling.

The four-byte word size follows the supplied width model. Byte order, required alignment, and the surrounding ABI are not independently established.

### Output

The declared result is a signed 64-bit integer.

An exhausted search returns zero. A successful search constructs a 64-bit result using 32-bit components; one return path explicitly uses an unsigned 64-bit intermediate.

The function returns no pointer, copied element, match count, or separate status. The signed declaration does not establish that every successful result is positive, or that every successful calculation has a defined result under literal C semantics.

### Preconditions

The base pointer must permit reading the count word. No null check precedes that read.

Each element actually reached must be readable as a 32-bit integer. A sufficient condition for a stable positive count is storage for the count word and all counted elements. An earlier match can eliminate the need to read later elements.

The function does not validate allocation size, alignment, address arithmetic, or agreement between the count and available storage. An invalid access has no represented recovery path.

Stable contents are required for the ordinary interpretation of “the first matching element in this sequence.” No synchronization or snapshot mechanism is supplied.

## Behavior

### Count checks and search order

The search begins at position zero.

Before each candidate-element read, the represented function reads the count and compares it with the current position interpreted as a signed 32-bit integer. If the count is less than or equal to that position, it returns zero without reading that candidate.

Consequently:

- A zero or negative initial count returns zero without reading any element.
- A stable positive count \(n\) permits examination of positions 0 through \(n-1\).
- After mismatches at all those positions, a count check at position \(n\) terminates the search.
- The element at position \(n\) is not read.

After an authorizing count check, the current element is read and compared with the target. A mismatch advances the position by one. A match ends the search immediately and starts result construction.

For stable data, the first matching position is therefore selected. Later duplicates do not contribute to the result, and no later element is explicitly read.

The repeated count reads describe the supplied representation. They are not a guarantee of separate physical loads: the pointer is not represented as volatile, and no concurrent-access semantics are established.

### Reachable positions and arithmetic widths

The position is an unsigned 32-bit quantity. Its conversion to signed 32-bit for the count comparison does not introduce an out-of-range conversion along the represented ordinary search path.

The reason is the signed count limit. No signed 32-bit count can exceed \(2^{31}-1\). At position \(2^{31}-1\), every possible count terminates the search. Thus:

- The greatest position that can be examined is \(2^{31}-2\).
- The greatest position reached for a terminating count check is \(2^{31}-1\).
- Neither unsigned position wraparound nor an out-of-range position conversion is needed to reach termination.

These bounds do not establish storage validity. Even a much smaller positive count can exceed the actual allocation.

### Successful-return branch selection

Let \(i\) denote the selected zero-based position.

The return branch is chosen by first subtracting 32 using unsigned 32-bit arithmetic, converting that result to signed 32-bit, and testing whether the converted value is nonnegative.

This is **not unconditionally equivalent** to testing whether \(i\) is at least 32.

For positions 32 and above, the subtraction result lies between zero and \(2^{31}-34\), inclusive, over the reachable matching-position range. It is representable as signed 32-bit. The nonnegative branch is therefore selected without this particular conversion uncertainty.

For positions 0 through 31, unsigned subtraction produces values from \(2^{32}-32\) through \(2^{32}-1\). None is representable as signed 32-bit.

Under C semantics, conversion of such a value to a signed integer type is implementation-defined and may raise an implementation-defined signal. The supplied material does not establish bit-pattern-preserving conversion.

The consequences are material:

- If conversion produces a negative value, the lower-position return branch is selected.
- If conversion produces a nonnegative value, the other return branch is selected.
- If conversion raises a signal, normal result construction is not assured.

The familiar conversion preserving the low 32-bit pattern would produce values from −32 through −1 and select the lower-position branch. That interpretation is an assumption, not established evidence.

Accordingly, every result description for positions below 32 is conditional on conversion completing normally and producing a negative value. Bit-pattern-preserving conversion is sufficient, but the branch only requires negativity.

### Lower-position return branch

For a match at position 0 through 31, **provided the branch-selection conversion completes with a negative result**, the function constructs a 64-bit pattern from two 32-bit halves:

- The lower half is formed by shifting a signed 32-bit one left by the low eight bits of the position.
- The upper half combines a shifted zero with an unsigned 32-bit one shifted right. The right-shift amount is the low eight bits of 32 minus the position.

For positions 1 through 30, these operations produce a zero upper half and a lower half equal to \(2^i\). The complete result is therefore \(2^i\), conditional on the branch-selection conversion.

Two boundary positions require additional qualification:

- **Position 0:** the lower half is one, but the upper-half calculation includes an unsigned 32-bit right shift by 32. Literal C semantics do not define that shift. An exact whole result of one is not established.
- **Position 31:** the lower-half calculation shifts a signed 32-bit one into its sign position. Literal C signed-shift rules do not establish the desired result. Under fixed-width bit-vector shift semantics, the constructed pattern would be `0x0000000080000000`.

If the out-of-range branch-selection conversion instead produces a nonnegative value for a position below 32, the other branch uses a retained shift amount between 224 and 255. Such amounts exceed the 32-bit operand width. That possibility does not establish an alternative defined numeric result.

### Return branch for positions 32 and above

For positions at least 32, the branch selection has no out-of-range unsigned-to-signed conversion.

The function retains the low eight bits of the unsigned position-minus-32 result. It uses that amount to shift a signed 32-bit one left, converts the resulting value to unsigned 32-bit, then places that 32-bit pattern in the upper half of an unsigned 64-bit value. The lower half is zero.

For positions 32 through 62, the retained amount is between zero and 30. The resulting signed 64-bit return value is exactly \(2^i\).

At position 63, the retained amount is 31. Two separate qualifications apply:

1. The signed 32-bit left shift reaches the sign position.
2. If the intended pattern is produced, the final unsigned 64-bit value exceeds the signed return type’s positive range.

Under fixed-width shifting and bit-pattern-preserving signed-return conversion, the pattern is `0x8000000000000000`, representing \(-2^{63}\) in two’s-complement interpretation. Neither assumption is independently established.

### Positions beyond 63

No explicit length restriction, rejection, clamp, or special return handles a match at position 64 or later.

Retaining eight bits of a shift amount does not ensure that it is below the 32-bit operand width. At position 64, for example, the amount is 32; literal C semantics do not define the resulting shift.

The retained amount repeats every 256 positions. This produces some established aliases even without assuming oversized-shift behavior. Position 288 retains an amount of zero and returns `0x0000000100000000`, the same result as position 32.

Thus, successful returns do not provide a unique position encoding for arbitrary positive counts. A restriction to at most 64 elements may be intended, but is not enforced; it would also not resolve all conversion and shift issues within that range.

## Side effects

The supplied body contains no explicit writes to the sequence, globals, or other storage. It contains no explicit allocation, deallocation, callback, I/O, or synchronization operation.

The function does not visibly acquire ownership of the input storage or release it. The caller must keep required storage accessible while it is read.

The represented access order is:

1. Read the count.
2. If authorized, read the current element.
3. After a mismatch, repeat at the next position.
4. After a match, perform return construction without further explicit sequence reads.

This is not a hardware memory trace or an atomicity guarantee.

Invalid memory accesses, invalid shifts, and implementation-defined conversions have no explicit function-level error handling. No reliable error return may be inferred for those cases.

## Dependencies

No external function calls are visible. There are no opaque callee IDs to enumerate.

The two-half result construction is represented as an operation, not evidence of a separately invoked dependency. It establishes no additional ownership or global-state behavior.

Relevant environmental dependencies are:

- 32-bit integer and 64-bit return widths;
- unsigned-to-signed conversion semantics;
- signed and oversized-shift semantics;
- memory accessibility and address calculation;
- interpretation of an unsigned 64-bit value returned through the signed interface.

These dependencies must not be replaced by assumed processor conventions.

## Acceptance examples

These are static, conditional contract cases—not executed tests. Unless stated otherwise, counts and elements remain stable and required memory accesses are valid.

| Case | Required or conditional outcome |
|---|---|
| Count is zero | Return zero after reading the count; read no element. |
| Count is negative | Return zero after reading the count; read no element. |
| Count is 3, with no matching element | Examine positions 0, 1, and 2 in order; return zero after the next count check; do not read position 3. |
| First match is at position 1 | Select position 1 and stop searching. Return 2 if branch-selection conversion completes with a negative value. Without that assumption, the numeric result is not established. |
| Matches occur at positions 2 and 5 | Select position 2; do not examine position 5. Return 4 conditional on negative, normally completing branch-selection conversion. |
| A negative target first matches at position 3 | Select position 3 without special sign handling. Return 8 conditional on negative, normally completing branch-selection conversion. |
| First match is at position 30 | Return `0x0000000040000000` conditional on negative, normally completing branch-selection conversion. |
| First match is at position 0 | Select position 0. If conversion selects the lower-position branch, its lower half is one, but the whole result remains unresolved because of the right shift by 32. |
| First match is at position 31 | Return pattern `0x0000000080000000` only if conversion selects the lower-position branch and the signed left shift has the stated fixed-width behavior. |
| First match is at position 32 | Return `0x0000000100000000`; the below-32 conversion uncertainty does not apply. |
| First match is at position 62 | Return `0x4000000000000000`. |
| First match is at position 63 | Pattern `0x8000000000000000` is conditional on fixed-width signed-shift behavior and bit-pattern-preserving signed-return conversion. |
| First match is at position 64 | Select position 64 if reached. The shift amount is 32; no defined numeric result, rejection, or zero result is established. |
| First match is at position 288 | Return `0x0000000100000000`, demonstrating aliasing with position 32. |

A null base fails the readable-count precondition, even when the caller intends an empty sequence.

## Uncertainties

The following remain **documented_only**:

- **Branch-selection conversion below 32:** unsigned subtraction produces an out-of-range signed-conversion input. Its implementation-defined result or signal determines whether normal branch selection occurs.
- **Oversized shifts:** no execution-specific behavior is established for a 32-bit shift by 32 or more. Literal C semantics provide no defined result.
- **Signed left shifts:** the desired sign-position patterns at positions 31 and 63 require semantics not independently established.
- **Signed return conversion:** the high-bit-set unsigned 64-bit pattern may not have a portable signed numeric interpretation.
- **Intended supported length:** no enforced 64-element maximum appears.
- **Concurrency and memory details:** synchronization, atomicity, byte order, alignment, and invalid-access handling are unspecified.
- **Caller interpretation:** no caller evidence establishes whether the result is treated as a signed integer, unsigned mask, or status.

These uncertainties cannot be resolved by silently substituting an idealized 64-bit one-hot operation.

## Evidence

Established evidence is limited to the supplied static function representation:

- A signed count check precedes each candidate-element read.
- Equality selects the first match; exhaustion returns zero.
- The position is unsigned 32-bit.
- Successful branch selection subtracts 32 before converting to signed 32-bit.
- Return construction uses 32-bit shifts, eight-bit-retained shift amounts, and 64-bit assembly.
- No explicit writes or external calls appear.

The position-mask purpose is inferred from the established results at positions 32–62 and the conditional results at positions 1–30. It does not resolve exceptional conversions or shifts.

No runtime execution, memory tracing, ABI validation, or tests have been performed.
