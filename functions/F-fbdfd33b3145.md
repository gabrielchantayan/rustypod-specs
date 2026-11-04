<!-- Trusted provenance: F-fbdfd33b3145; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbdfd33b3145

## Purpose

F-fbdfd33b3145 accumulates unsigned byte values into an unsigned 32-bit result using a positional multiplier of ten. It consumes at least the first byte, then continues toward an exclusive endpoint while applying a limited arithmetic-decrease check.

The function is not, by itself, a textual decimal parser. It does not subtract a character encoding offset, restrict input values to 0–9, skip whitespace, recognize signs, or report invalid characters. Every consumed byte contributes its full unsigned value, from 0 through 255.

When a checked accumulation step produces a value smaller than the preceding accumulator, the function stops, writes the 32-bit value `0x00000080` to the supplied output location, and returns the newly computed value. Otherwise, it returns the accumulated result without writing that location.

**Established behavior:** detection is based on a decrease in the unsigned accumulator, not on a complete test for arithmetic overflow.

**Inferred intent:** the input may be intended to contain already-converted decimal digits, with the output location serving as an error indicator. Neither interpretation is enforced by the function.

## Interface

The function has three inputs:

| Input | Contract |
|---|---|
| Status destination | Address of a location that receives a four-byte write only when the decrease condition is detected. |
| Input start | Address of the first unsigned byte to consume. This byte is read unconditionally. |
| Input end | Address used as an exclusive upper boundary for additional reads. |

The return value is an unsigned 32-bit integer in the range 0 through 4,294,967,295.

### Widths and arithmetic

- Each input element is an unsigned eight-bit byte.
- A byte is widened without sign extension before contributing to the result.
- The accumulator and return value are unsigned 32-bit quantities.
- Each accumulation step multiplies the previous accumulator by ten and adds the next byte.
- Results are reduced modulo 4,294,967,296.
- The decrease check compares the previous accumulator and the reduced new value as unsigned quantities.
- The status write replaces four bytes with the integer value 128. It is not an eight-bit write and is not a bitwise addition to existing status.

The byte order and any machine-specific alignment requirements of that four-byte store are not established by the supplied material.

### Supported input conditions

For ordinary bounded use, the caller should supply a nonempty readable byte range with the endpoint immediately after the last byte.

The implementation also forms addresses beyond the current byte while deciding whether another byte exists. Its pointer comparisons and address formation require a valid ordering and address model. Exact behavior for unrelated pointers, address-space wraparound, or invalid pointer formation is not established.

The status destination must support a four-byte write whenever the decrease path is reachable. No status read or write occurs on the normal return paths. There is no null check, capacity check, or validation of either input pointer.

An empty range does not request an empty parse: the first byte is still read.

## Behavior

### Input consumption and accumulation

The first byte is read without consulting the endpoint. Its unsigned value becomes the initial accumulator.

The function then determines whether a second byte is before the endpoint. If so, it reads that byte and appends its value with the multiplier-ten rule. If a third byte exists, the function reads and incorporates it in the same manner.

If no fourth byte is before the endpoint, the function returns the result accumulated from the first one, two, or three bytes. These paths do not write the status destination.

When a fourth byte exists, the function reads and incorporates it. If the endpoint is reached immediately afterward, the four-byte accumulated value is returned, again without a status write.

For every available byte after the fourth:

1. The next byte is read.
2. A new 32-bit accumulated value is calculated.
3. That value is compared with the accumulator from immediately before this byte.
4. If the new value is greater than or equal to the previous value, processing continues.
5. If the new value is smaller, processing stops immediately, the status destination is overwritten with `0x00000080`, and the new value is returned.

This ordering matters. The byte that causes a decrease is included in the returned value. Bytes following it are not read.

The endpoint is checked before each read after the first. On an ordinary nonempty range, it is not read as an input byte.

### Arithmetic model

For any consumed byte after the first, the accumulated value is:

\[
A_{\text{new}} = (10A_{\text{old}} + B) \bmod 2^{32},
\]

where \(B\) is the next byte's unsigned value.

If the function consumes \(n\) bytes without early termination, the result is the base-ten positional sum of those byte values, reduced modulo \(2^{32}\). The byte values need not be valid decimal digits for this arithmetic description to hold.

The decrease condition is:

\[
A_{\text{new}} < A_{\text{old}}.
\]

Equality does not trigger the status write.

### Scope and limits of the decrease check

No decrease check is performed while incorporating the first four bytes. That omission cannot hide 32-bit overflow within those four bytes: even four bytes each equal to 255 produce only 283,305.

More generally, any eight-byte input remains within the unsigned 32-bit range. Its maximum unreduced value is:

\[
255 \times 11{,}111{,}111 = 2{,}833{,}333{,}305.
\]

Consequently, under the stated byte and arithmetic widths, a decrease cannot occur within the first eight bytes. Nine bytes can produce values large enough to wrap.

Once arithmetic can wrap, a decrease is sufficient to establish that the mathematical accumulation exceeded the 32-bit range. It is not a necessary consequence of overflow: a wrapped result can still be greater than or equal to the preceding accumulator.

There is no separate overflow flag, saturation behavior, or rejection of excessive mathematical values. A wrap that does not decrease the accumulator is accepted, and processing continues.

### Completion and early termination

Normal completion occurs when the next byte is not before the endpoint. The last accumulated value is returned, and the status destination is left unchanged.

Decrease termination occurs immediately after incorporating the byte that yields the smaller value. The function performs the four-byte status write and returns that smaller value without checking or consuming the remaining suffix.

The return value does not independently identify which completion path occurred. For example, zero can be a normal result or the result of a detected wrap.

### Empty or reversed boundaries

The first byte is read even when the input start equals the endpoint. If the address immediately after the start is not before the endpoint, the function returns the first byte's value and performs no further input read.

The same control-flow observation applies to a reversed boundary when the pointer ordering is meaningful. It does not establish safety for arbitrary invalid pointer combinations.

Thus, callers must not rely on an empty or reversed range to prevent memory access. Behavior after an invalid first-byte read is outside the established contract.

## Side effects

The function's explicit memory effects are limited to input-byte reads and a conditional four-byte status write.

### Read and write order

For a valid nonempty range:

- Input bytes are read in increasing address order.
- Each consumed byte is explicitly read once in the supplied logic.
- No byte after the endpoint is read.
- No suffix byte is read after a decrease is detected.
- On the decrease path, the triggering byte is read and incorporated before the status write.
- The previous contents of the status destination are not read.

These statements describe the supplied operations, not volatile-access guarantees, atomicity, or processor-level memory ordering.

### Status preservation and replacement

Normal completion does not clear the status destination. A preexisting nonzero value remains unchanged.

Decrease termination replaces the destination's entire 32-bit value with 128. Existing bits are not preserved. If the destination already contains 128, the same store is still performed.

### Aliasing and ownership

No allocation, deallocation, ownership transfer, retained pointer, or external call is visible.

The function does not explicitly prohibit the status destination from overlapping the input. If such overlap is valid in the execution environment and a decrease occurs, the four-byte status write can modify input storage, including bytes in a suffix that the function never reads. No subsequent input read follows that write.

This describes operation order; it does not establish that every overlapping arrangement satisfies the platform's alignment, object-access, and memory-validity requirements.

## Dependencies

No dependency calls are present in the supplied function. There are therefore no dependency function IDs to list.

The established behavior depends on:

- unsigned eight-bit input reads;
- unsigned 32-bit arithmetic and comparison;
- meaningful input-address ordering;
- readable storage for each consumed byte;
- writable four-byte storage for the conditional status update.

No external error handler, allocator, parser, callback, or global-state operation is visible. No assumptions about unknown callees preserving globals or managing memory are needed.

## Acceptance examples

These examples are derived from the supplied static evidence. They have not been executed.

Unless otherwise stated, input values below are raw unsigned byte values, the endpoint follows the final listed byte, and the status destination initially contains an arbitrary valid 32-bit value.

### One byte

Input: [7].

Expected result: 7.

Only the first byte is read. The status destination is unchanged.

### Several already-converted digits

Input: [1, 2, 3, 4].

Expected result: 1,234.

All four bytes are consumed. The status destination is unchanged.

Input: [0, 0, 7].

Expected result: 7.

Leading zero-valued bytes are incorporated normally; they do not end processing.

### Encoded characters are not converted

Input: [49, 50, 51], corresponding to the usual ASCII byte values for “123”.

Expected result: 5,451.

The calculation uses 49, then 50, then 51 as numerical contributions. The result is not 123, and no invalid-character indication is written.

### Values above nine are accepted

Input: [255, 255, 255, 255].

Expected result: 283,305.

The status destination is unchanged. No digit-range validation occurs.

### Largest unsigned 32-bit value without wrapping

Input: [4, 2, 9, 4, 9, 6, 7, 2, 9, 5].

Expected result: 4,294,967,295.

The final mathematical value fits exactly. No decrease occurs, so the status destination remains unchanged.

### Detected wrap with unread suffix

Input: [4, 2, 9, 4, 9, 6, 7, 2, 9, 6, 7].

After the first nine bytes, the accumulator is 429,496,729. Incorporating the tenth byte produces the mathematical value 4,294,967,296, which reduces to zero.

Expected behavior:

- Return zero.
- Overwrite the status destination with `0x00000080`.
- Do not read the eleventh byte.

If the destination previously contained another status value, that value is replaced rather than combined with 128.

### Wrap without a status update

Input: one byte of value 1 followed by ten bytes of value 0.

After ten consumed bytes, the accumulator is 1,000,000,000. The final byte produces the mathematical value 10,000,000,000, reduced to 1,410,065,408.

Expected result: 1,410,065,408.

The reduced value is greater than the previous accumulator, so the function reaches the endpoint without writing status. This example distinguishes the implemented decrease detector from complete overflow detection.

### Empty range with readable storage at the start

Condition: the input start equals the endpoint, but the first byte remains readable and contains 42.

Expected result: 42, with no status write.

This is conditional on valid address formation and comparison. It demonstrates the unconditional first read, not a supported empty-input convention.

## Uncertainties

The following items are **documented_only**:

- The semantic meaning of status value 128 outside this function.
- Whether callers normally supply digit values, character bytes, or another representation.
- Whether callers guarantee nonempty input or initialize the status destination beforehand.
- The target byte order, concrete ABI, and alignment rules for the four-byte store.
- Platform-specific consequences of invalid addresses, unrelated-pointer comparisons, or address wraparound.
- Whether any caller permits or relies on overlapping status and input storage.
- Concurrency, volatile-memory, and memory-mapped-device semantics.
- Any broader error-reporting protocol that distinguishes normal completion from detected decrease.

**Inferred intent, not an additional requirement:** positional multiplication by ten suggests decimal-style accumulation, and the conditional value 128 suggests error reporting. These interpretations do not justify adding digit validation, complete overflow detection, status clearing, or empty-input handling.

## Evidence

The supplied function body is the sole static evidence for this contract.

It establishes:

- an unconditional initial unsigned-byte read;
- endpoint checks governing additional reads;
- multiplication by ten with unsigned 32-bit accumulation;
- no character conversion or digit validation;
- unchecked accumulation through the fourth byte;
- unsigned decrease comparisons for subsequent bytes;
- immediate termination on the first detected decrease;
- a conditional four-byte replacement store of 128;
- return of the accumulated value on both normal and decrease paths;
- no visible dependency calls, allocation, or global-state accesses.

The numeric examples and overflow bounds are arithmetic consequences of those operations and widths. **No runtime tests have been performed.**
