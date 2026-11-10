<!-- Trusted provenance: F-8abffcbfb4aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8abffcbfb4aa

## Purpose

F-8abffcbfb4aa performs an ordered sequence of dependency calls using two caller-supplied values and two words from a caller-supplied object. It then derives a 32-bit unsigned result from a 64-bit dependency result.

The ordinary result-processing path resembles conversion of an IEEE 754 binary64 value to an unsigned 32-bit integer, with truncation and saturation. That interpretation is **inferred intent**, not an established dependency contract. The supplied representation does establish the bit-level decisions described below, subject to unresolved oversized-shift semantics.

Certain exceptional bit patterns enter a control-word-dependent path containing unresolved indirect transfers. Those transfers cannot be specified as ordinary callbacks with established signatures.

No runtime tests have been performed.

## Interface

### Inputs

The recovered interface has two inputs:

| Input | Established representation | Required use |
|---|---|---|
| Object address | 32-bit address-like value | Supplies readable 32-bit words at byte offsets `0x418` and `0x434`. |
| Context value | Opaque 32-bit value | Passed as the first explicit argument to F-d7be9cfb4000. |

The object’s broader layout, allocation size, ownership, and mutability are not established. The two referenced fields are four bytes wide. Their meanings are unknown.

There is no visible null check, bounds check, alignment check, or access-failure recovery. Successful execution requires both field reads to be valid under the target’s memory-access rules. Address arithmetic and alignment behavior outside that valid domain are not established.

The context value is not visibly dereferenced by this function. Whether its recipient interprets it as an address, handle, integer, or another representation is unknown.

### Output

The recovered return value is an unsigned 32-bit word.

Ordinary paths produce a shifted significand-like value, zero, or `0xffffffff`. Exceptional paths return the recovered result of an unresolved indirect transfer, if that transfer returns.

There is no established error-code namespace. Zero and `0xffffffff` are ordinary results on some paths and must not automatically be interpreted as failures.

### External data

The function also uses:

- Two fixed 32-bit words passed to F-c4b34006a13c.
- A 32-bit control word consulted only on the exceptional path.
- A shared 32-bit word used by selected exceptional branches.
- Two additional fixed words used by one exceptional branch.
- An incoming return-address-like execution value used in the recovered indirect-transfer expressions.

The supplied material does not provide the fixed words’ contents, the external data’s volatility or synchronization rules, or the execution-state contract for the indirect transfers.

## Behavior

### Ordered dependency phase

Before inspecting the final result’s bits, the function performs the following observable sequence:

1. Reads the object word at byte offset `0x418`.
2. Calls F-9e5e664c8ae7 with that word as its explicit argument. Retains the recovered 64-bit result.
3. Reads the object word at byte offset `0x434`.
4. Calls F-d7be9cfb4000 with the context value first and the newly read object word second.
5. Calls F-9e5e664c8ae7 again. This invocation has **no explicit argument in the supplied representation**.
6. Calls F-c4b34006a13c with four explicit 32-bit arguments: the low and high words of the second 64-bit result, followed by the two fixed words.
7. Calls F-596e18a33aec with four explicit 32-bit arguments: the low and high words returned by F-c4b34006a13c, followed by the low and high words retained from the first F-9e5e664c8ae7 call.
8. Processes the resulting 64-bit value.

Low word precedes high word in these argument lists. This is an argument-order fact, not evidence of the byte order used to store a 64-bit value in memory.

The intervening calls matter. In particular, the `0x434` field is read after the first dependency call, not captured alongside the `0x418` field. An implementation must not assume the object or other external state remains unchanged across that call.

The second F-9e5e664c8ae7 invocation must not be silently assigned the first invocation’s argument. Its actual calling convention and implicit inputs remain unresolved.

### Result interpretation and widths

For this section:

- **Low word** means the low 32 bits of the final 64-bit dependency result.
- **High word** means the high 32 bits.
- **Signed high word** means the same high-word bits interpreted as a signed 32-bit integer.
- **Exponent-like field** means the signed high word shifted arithmetically right by 20.
- **Candidate word** means the low 32 bits of the high word shifted left by 11, combined by bitwise OR with the low word shifted logically right by 21.

If the exponent-like field is nonzero, bit 31 of the candidate word is set. These operations construct a 32-bit word; they do not preserve bits shifted beyond that width.

The placement of these fields matches binary64 sign, exponent, and significand extraction. Nevertheless, the dependencies’ production of a binary64 number is not independently established.

### Negative-sign early return

When the exponent-like field is negative, the function first compares the signed high word plus `0x40000000` against `−0x100000`.

If that sum is strictly smaller, the function returns zero immediately.

For this negative-sign branch, the addition stays within signed 32-bit range. The comparison is equivalent to requiring the signed high word to be strictly less than `−0x40100000`.

Under the binary64 interpretation, this early return covers negative zero and negative values with magnitude below one. Equality at the boundary does not take this early return.

Negative-sign values that do not satisfy the comparison proceed to the classification described below.

### Nonnegative-sign conversion range

When the exponent-like field is nonnegative, the conversion-range path is selected if that field is less than 1055.

Its shift distance is 1054 minus the exponent-like field:

- For distances at least 256, the returned result is zero.
- For distances from 0 through 255, the recovered operation logically shifts the candidate word right by the distance masked to eight bits, then returns that word.

The mask does not reduce these distances to five bits. Counts from 32 through 255 therefore require an explicit target shift-semantics decision; the supplied expression alone does not establish a portable result for them.

For distances from 0 through 31, the operation is unambiguous: bits shifted out are discarded and zeros enter at the high end.

Under the binary64 interpretation, the unambiguous nonnegative range includes values from one through values below \(2^{32}\). Fractional information below the retained integer bits is discarded, not rounded upward.

### Saturation and exceptional classification

Values not returned by the preceding paths are classified using a 32-bit discriminator:

- Double the high word.
- Add one if the low word is nonzero; otherwise add zero.
- Retain the result modulo \(2^{32}\).

If the discriminator is less than `0xffe00001`, the function returns:

- `0xffffffff` when the high word’s sign bit is clear.
- Zero when the high word’s sign bit is set.

The supplied representation subdivides this condition internally, but both subcases return the same result. No differing observable behavior is established for that subdivision.

A discriminator equal to `0xffe00000` therefore saturates rather than taking the exceptional path. Under the binary64 interpretation, this includes positive and negative infinity, producing `0xffffffff` and zero respectively.

A discriminator at least `0xffe00001` enters the exceptional path. Under the binary64 interpretation, these are NaN encodings, including encodings whose only nonzero fraction bits are in the low word.

### Exceptional control-dependent path

The exceptional path first reads the control word and selects a mode using its low four bits. Later control-word uses are separate reads in the supplied representation. They must not be assumed to observe the same value if external state can change.

All terminal transfers in this path are unresolved. Their recovered destination is formed by clearing the low two bits of an incoming return-address-like value. This is not evidence of a caller-supplied callback or a conventional function-pointer interface.

| Initially selected mode | Subsequent condition | Recovered explicit transfer arguments |
|---|---|---|
| 9 | A later control read has bit `0x100000` clear | Zero, then the shared word. |
| 9 | That bit is set | Eight, then a derived control value obtained from another control read. |
| 8 | A later control read has bit `0x40` set | Zero, zero. |
| 8 | That bit is clear, and a subsequent read has bit `0x10` clear | First exceptional fixed word, then the shared word. |
| 8 | Bit `0x40` is clear, and the subsequent bit `0x10` is set | Zero, then the alternate exceptional fixed word. |
| 10 | A later control read has bit `0x40` clear | One explicit argument: zero. |
| 10 | That bit is set | One explicit argument: `0x80000000`. |
| Any other mode | No further visible condition | No explicit arguments recovered. |

For mode 9 with bit `0x100000` set, the second argument is formed by shifting the newly read 32-bit control word left by 15, complementing the resulting 32-bit word, then shifting left by one. Intermediate and final words are truncated to 32 bits.

For mode 8 with bit `0x40` clear, the first exceptional fixed word and the shared word are read before the subsequent bit-`0x10` test. If that test selects the alternate pair, those earlier reads have still occurred.

Where an indirect transfer returns, the recovered function returns its 32-bit result unchanged. Whether these operations actually return, resume elsewhere, or rely on additional register state is **documented_only**.

## Side effects

### Directly established effects

The supplied representation shows:

- Two object-field reads, ordered around the first dependency call.
- Four dependency identities invoked in the sequence above, including two invocations of F-9e5e664c8ae7.
- Reads of fixed data used as dependency arguments.
- Conditional control-word and exceptional-data reads.
- Conditional unresolved indirect transfers.

It shows no explicit writes to the object, the control word, or other memory. It shows no direct allocation or release operation.

### Effects not bounded by this function

The absence of explicit stores does not establish purity. Any dependency or unresolved target may mutate memory or global state, allocate or release resources, raise an exception, or fail to return.

No ownership transfer is established for either input or either object field. The function does not visibly retain an input address beyond its return, but dependency retention behavior is unknown.

There is no visible rollback if a dependency changes state and a later call fails. No synchronization or atomic snapshot of the object fields or control word is established.

## Dependencies

| Opaque function ID | Established use | Unresolved contract |
|---|---|---|
| F-9e5e664c8ae7 | Called twice; each call has a recovered 64-bit result. The first receives the word from offset `0x418`; the second has no explicit argument. | Actual signature, implicit inputs, meaning of results, state changes, and failure behavior. |
| F-d7be9cfb4000 | Called between the two preceding invocations, with context first and the word from offset `0x434` second. Its result is not visibly consumed. | Effects, ownership, return convention, and failure behavior. |
| F-c4b34006a13c | Receives a low/high pair from the second invocation plus two fixed words; supplies a recovered 64-bit result. | Whether it performs arithmetic, the constants’ meaning, and all side effects. |
| F-596e18a33aec | Receives the transformed low/high pair followed by the retained first low/high pair; supplies the final recovered 64-bit result. | Arithmetic meaning, side effects, and error behavior. |

The unresolved indirect target has no supplied opaque identity. Assigning it one of the listed dependency IDs would be unsupported.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Numerical examples assume the final dependency result has the stated binary64 encoding; they do not assert that any particular input pair produces it.

1. **Object mutation between reads.** If the first dependency changes the object’s `0x434` word, F-d7be9cfb4000 receives the value read after that call, not an entry-time snapshot.

2. **Positive one.** A final encoding of positive one returns 1. Its shift distance is 31.

3. **Fraction discarded.** A final encoding of 1.75 returns 1. A final encoding of 3.75 returns 3.

4. **Largest representable unsigned integer.** A final encoding of \(2^{32}-1\) returns `0xffffffff` through the conversion-range path.

5. **Positive overflow.** A final encoding of \(2^{32}\) returns `0xffffffff` through saturation.

6. **Negative values.** Negative zero and negative one-half return zero through the negative-sign early return. Negative one does not satisfy that strict early comparison but still returns zero through later classification.

7. **Infinities.** Positive infinity returns `0xffffffff`; negative infinity returns zero. Neither enters the exceptional control dispatch.

8. **Low-word-only NaN payload.** A NaN encoding with a zero high-word fraction and a nonzero low word enters the exceptional path. Ignoring the low word would incorrectly classify this case as infinity.

9. **Large shift uncertainty.** A positive one-half encoding selects a shift distance of 32. Its exact result remains documented_only until target shift semantics are established; a five-bit shift-count mask must not be assumed.

10. **Repeated control reads.** If the initial control read selects mode 9 but a later read changes other control bits, the later reads determine the selected arguments. An entry-time control snapshot is not an established substitute.

11. **Exceptional transfer result.** If a recovered indirect transfer returns `0x12345678`, the function returns that same word, conditional on the recovered call-and-return interpretation being valid.

## Uncertainties

The following remain **documented_only**:

- Dependency signatures, especially the second F-9e5e664c8ae7 invocation.
- The values and semantic roles of the fixed data words.
- Whether the final 64-bit result is actually a binary64 value.
- Logical-right-shift behavior for counts from 32 through 255.
- The true control-flow meaning, arguments, clobbers, and return behavior of the unresolved indirect transfers.
- Whether control and shared-data reads are volatile, synchronized, or externally mutable.
- Dependency ownership, retention, allocation, release, and exception behavior.
- Invalid-address, misalignment, concurrency, and reentrancy behavior.

A full-domain executable replacement requires resolving the oversized shifts and exceptional transfers. The available evidence supports neither inventing a callback interface nor replacing the exceptional path with an unconditional numeric result.

## Evidence

**Established static evidence:** the supplied representation supports the explicit call order, object offsets, argument-word order, 32-bit bit operations and comparisons, conditional external-data reads, ordinary return values, and recovered exceptional transfer forms.

**Inferred intent:** the field extraction and thresholds are consistent with binary64-to-unsigned-32 conversion using truncation, saturation, and special handling for NaNs.

**Evidence limitations:** the supplied material reports an omitted unreachable block and unresolved indirect control flow. It provides no dependency bodies, constant contents, authoritative calling-convention description, or runtime observations. This contract therefore distinguishes visible operations from numerical interpretation and unresolved execution semantics.
