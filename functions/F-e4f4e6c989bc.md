<!-- Trusted provenance: F-e4f4e6c989bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4f4e6c989bc

## Purpose

F-e4f4e6c989bc reads two sentinel-terminated byte sequences, combines a parsed exponent with a signed adjustment, and passes a two-word result to F-657357805295.

Its structure suggests conversion of a decimal magnitude and exponent into a signed floating-point representation. That is **inferred intent**, not an established numeric specification. The supplied body establishes parsing, arithmetic, branching, dependency calls, sign manipulation, result-word selection, and conditional status writes. It does not establish the dependencies’ numeric behavior or the destination representation.

The contract below distinguishes caller-visible operations from assumptions needed about dependencies. No runtime tests have been performed.

## Interface

### Inputs

The function accepts four inputs:

1. **Destination token:** an opaque 32-bit value forwarded to F-657357805295.
2. **Exponent-sequence pointer:** addresses bytes scanned using `0xff` as the terminator.
3. **Magnitude-sequence pointer:** addresses bytes scanned using `0xff` as the terminator.
4. **Exponent adjustment:** a signed 32-bit integer.

There are no explicit sequence lengths. Input storage must support every read described below, including rereads. Stable input contents are a precondition for the ordinary parsed-value descriptions and acceptance examples.

The body performs no null-pointer, bounds, termination, or digit-range validation. Missing terminators and invalid memory have no visible recovery path.

Sequence elements are unsigned 8-bit values. Payload values are accumulated directly; no ASCII digit-base subtraction occurs. Sign markers are `0x2d` and, where recognized, `0x2b`.

### Outputs

There is no visible return value.

On every path that completes normally, F-657357805295 is called once with the original destination token and two 32-bit words. The destination token’s interpretation and the dependency’s destination layout are **documented_only**.

Some paths also obtain an address from F-4af36158f863 and store the 32-bit value 2 there. This value is established; its meaning as a named error code is not.

### Widths and arithmetic

The magnitude is accumulated as two 32-bit words representing an unsigned 64-bit quantity. Accumulation retains the low 64 bits; overflow is not reported.

Modes, masks, result words, and the sign-bit operations are 32-bit. Bit numbering starts at zero, so the sign-related bit is bit 31. Both conversion dependencies visibly return 64-bit values.

Exponent accumulation, exponent negation, addition of the adjustment, and normalization-exponent arithmetic involve signed 32-bit quantities without overflow checks. Behavior outside their representable ranges is **documented_only**. The contract does not replace that uncertainty with an assumed wraparound rule.

## Behavior

### Exponent parsing and read order

The exponent sequence is processed before the magnitude sequence.

The function first reads its first byte:

- If that read yields `0xff`, the parsed exponent is zero. No later sign read occurs on this path.
- Otherwise, scanning starts at the second byte. The first byte contributes nothing to accumulation.

For each subsequent position, the byte is read for the terminator check. If that read is not `0xff`, the byte at that position is read again for accumulation. With stable storage, each payload byte contributes its unsigned value to a base-10 accumulation.

After the remaining sequence has been scanned through a terminator, the function **rereads the first byte**. If this later read yields `0x2d`, it negates the accumulated exponent. It does not use a cached copy of the initial byte for this decision.

Consequently, for stable input:

- An arbitrary nonterminator first byte is discarded.
- A leading plus marker is compatible with positive input but is not validated as a sign.
- A single nonterminator byte followed by `0xff` produces zero.
- A minus marker immediately followed by `0xff` also produces zero.

The initial terminator check and the later sign decision are separate reads. Their values need not agree if storage changes; no concurrent-mutation guarantees are established.

### Magnitude parsing

The function reads the first magnitude byte to determine whether it is a sign marker:

- `0x2d` records a negative sign and advances past the marker.
- `0x2b` leaves the sign nonnegative and advances past the marker.
- Any other value leaves scanning at the first byte.

The accumulation loop then reads each current byte once for that iteration. A nonterminator value contributes directly to the 64-bit base-10 accumulation. For input without a recognized sign, this means the first byte is read again by the loop after the initial sign check.

The low-word multiplication and addition explicitly contribute carries to the high word. The result is the mathematical raw-byte accumulation modulo \(2^{64}\).

A leading terminator or a sign followed immediately by a terminator produces zero. An explicit minus remains recorded even when the accumulated magnitude is zero. A nonzero mathematical magnitude may also wrap to zero and take the same zero-result path.

### Control acquisition and exponent adjustment

After both sequences have been scanned, the function calls F-96fcc6b36b88 with two zero arguments.

Its low 32-bit return word determines a mode:

- For negative input, that word is multiplied by 5 modulo \(2^{32}\), then shifted right logically by one.
- For nonnegative input, the word is used without that transformation.
- Bits 22–23 are extracted.
- Extracted values 2 and 3 are both replaced by `0xffffffff`.

The final mode is therefore 0, 1, or `0xffffffff`. Interpreting it as a rounding mode is **inferred intent**.

The signed adjustment is added to the parsed exponent after this dependency call. In the visible ordering, the negative-input transformation precedes the addition, and extraction of the mode bits follows it.

### Zero magnitude

A zero accumulated magnitude bypasses exponent-range handling and all conversion dependencies.

The delivered words are:

- Nonnegative input: first word zero, second word zero.
- Negative input: first word `0x80000000`, second word zero.

F-96fcc6b36b88 has already been called, and the exponent adjustment has already been computed. Thus this path does not avoid potentially unrepresentable exponent arithmetic.

There is no direct status write on this path.

### Nonzero magnitude with adjusted exponent below −500

For an adjusted exponent of −501 or less, the function delivers the sign-only first word and a zero second word.

After F-657357805295 returns, it calls F-4af36158f863 and writes 2 through the returned address. No normalization or factor-scratch dependency is called.

This resembles underflow to signed zero, but the established behavior is the word pair and the delivery-before-status ordering.

### Nonzero magnitude with adjusted exponent above 500

For an adjusted exponent of 501 or greater, the function:

1. Reads an external address component and adds `0x08036148`.
2. Calls F-2cc6ced0beea with that resulting address, a result-scratch address, a separate factor-scratch address, and zero.
3. Reads two adjacent 32-bit result-scratch locations.
4. ORs the first scratch word with the input sign bit.
5. Delivers that word and the second scratch word.
6. Obtains a status address and writes 2, after delivery.

The caller does not visibly initialize the result-scratch locations before this call. The body does not prove that F-2cc6ced0beea initializes them either. A defined result on this path requires a dependency contract that supplies valid values for the subsequent reads.

The OR operation does not clear an existing sign bit in the first scratch word. Positive input therefore does not guarantee bit 31 is clear.

The addressed data and dependency behavior are unknown. An infinity-like result is only inferred intent.

### Nonzero magnitude with adjusted exponent from −500 through 500

The bounds are inclusive.

The function first calls F-99579940fdac with:

- A factor-scratch address.
- The absolute adjusted exponent.
- The original mode for nonnegative exponents, or its 32-bit negation for negative exponents.

Negation exchanges 1 and `0xffffffff`; zero remains zero. “Factor preparation” describes an inferred role, not a proven effect of this dependency.

After the call, the function constructs a descriptor with three adjacent 32-bit fields:

| Offset | Field |
|---|---|
| 0 | Signed exponent-like value |
| 4 | High magnitude word |
| 8 | Low magnitude word |

It first reads the external normalization seed into the exponent-like field and copies the original magnitude words into the other fields.

If the original high magnitude word is zero, it then:

- Sets the descriptor’s low word to zero.
- **Reads the external seed again** and uses this second value minus 32.
- Places the original low magnitude word in the descriptor’s high word.

The first seed read therefore occurs on all in-range, nonzero paths. The second occurs only when the original high word is zero; its value supersedes the first.

The magnitude is normalized by left shifts selected in descending sizes of 16, 8, 4, 2, and 1 bits. Bits pass from the low word into the high word, and the exponent-like field decreases by each selected shift. The final high word has bit 31 set.

For a nonzero 64-bit magnitude with leading-zero count \(L\):

- If the original high word is nonzero, the final exponent-like field is the **first seed read minus \(L\)**.
- If the original high word is zero, it is the **second seed read minus \(L\)**.

These formulas require representable signed arithmetic. Describing both cases using one seed value additionally requires a stable-seed assumption.

### Conversion, classification reads, and sign application

For an adjusted exponent at or below zero, F-5e950de591ec receives the descriptor address, factor-scratch address, and original derived mode.

For a positive adjusted exponent, F-4a2f439d1347 is called with no explicit arguments visible in the supplied body. Its hidden inputs and calling-convention requirements are **documented_only**.

Each call supplies a visible 64-bit return value:

- Its low 32 bits become the first candidate word.
- Its upper 32 bits become the second candidate word.

Immediately after conversion, **before applying the input sign**, the function reads and captures an external classification constant.

For negative input, it then toggles bit 31 of the first candidate word using XOR. This is not a forced sign-bit set: an existing set bit is cleared. Nonnegative input leaves that word unchanged.

Let the candidate’s **magnitude portion** mean the first word with bit 31 cleared. A candidate is zero for classification when this magnitude portion and the second word are both zero.

Status handling is triggered as follows:

- A zero candidate triggers it directly. Short-circuit evaluation skips a second read of the external classification constant.
- A nonzero candidate causes a **second read** of that external constant. Status handling is triggered if this later value equals bits 20–30 of the first candidate word.

The two classification reads are distinct. Their equality is not guaranteed.

### In-range status handling and replacement

When status handling is triggered, F-4af36158f863 is called and 2 is written through its returned address.

Only after that store does the function consider replacement. It compares the **initially captured classification value**, not the later status-trigger value, with the candidate magnitude portion shifted right by 20.

If equal, the function:

1. Reads another external address component and adds `0x08036324`.
2. Calls F-2cc6ced0beea with the resulting address, result scratch, factor scratch, and zero.
3. Reads the two result-scratch words.
4. ORs the candidate’s sign bit with the complete first scratch word.
5. Uses the second scratch word as the second result word.

Again, neither caller-side initialization of the result scratch nor dependency-side initialization is established.

The final candidate or replacement is delivered through F-657357805295. In-range status writes occur **before** delivery, unlike both extreme-exponent paths.

## Side effects

### Observable ordering

The important explicit ordering is:

- Exponent scanning and its conditional first-byte reread precede magnitude parsing.
- Both parses precede F-96fcc6b36b88.
- F-99579940fdac precedes normalization-seed reads.
- Conversion precedes the first classification read.
- That first classification read precedes sign application.
- Nonzero-candidate classification performs a second external read.
- In-range status storage precedes replacement-address loading and destination delivery.
- Extreme-exponent delivery precedes status-address acquisition and storage.

These placements must not be replaced by assumptions that dependencies preserve external state.

### Memory, scratch, and ownership

The function directly reads but does not directly modify the input sequences. It directly writes local descriptor storage and, conditionally, the returned status location. Destination effects are delegated to F-657357805295.

The result-scratch pair consists of two adjacent 32-bit locations at offsets 0 and 4 from its passed address. The factor-scratch address identifies separate local storage, but its complete required extent and layout cannot be established from the displayed declaration.

The body establishes no allocation, deallocation, ownership transfer, or callee pointer-retention policy. Local addresses are valid during the invocation; permission to retain or use them after return is not established.

Dependency write footprints, initialization requirements, alignment requirements, and effects on nearby storage require separate contracts. In particular, passing an output-looking address does not prove that its recipient initializes all subsequently read bytes.

### Errors

There is no visible validation-error return, allocation-failure handling, dependency-failure check, or recovery from an invalid status address. Dependencies may have additional effects or exceptional behavior not established here.

## Dependencies

| Opaque ID | Established caller interaction |
|---|---|
| F-96fcc6b36b88 | Called with two zeros after parsing; its low return word is used to derive the mode. |
| F-99579940fdac | Receives factor scratch, absolute adjusted exponent, and direction-adjusted mode. Factor preparation is inferred. |
| F-5e950de591ec | Receives descriptor, factor scratch, and mode for nonpositive in-range exponents; returns 64 bits. |
| F-4a2f439d1347 | Called for positive in-range exponents; no explicit arguments are visible; returns 64 bits. |
| F-2cc6ced0beea | Receives an externally derived address and two scratch addresses; the caller subsequently reads two result locations. Initialization and write footprint are unknown. |
| F-657357805295 | Receives the destination token and two result words once on normal completion. |
| F-4af36158f863 | Returns the address used for a conditional 32-bit store of 2. |

No dependency implementation is supplied.

## Acceptance examples

These are conditional static criteria, not executed tests. Inputs are stable, required arithmetic is representable, and dependencies return normally unless stated otherwise.

1. **Negative zero:** Magnitude bytes `0x2d, 0xff` produce words `0x80000000` and zero. F-96fcc6b36b88 still runs; exponent-range status handling is bypassed.

2. **Discarded exponent prefix:** Exponent bytes `0x7a, 0x01, 0x02, 0xff` produce exponent 12. The first byte is read initially and reread after scanning; it is not accumulated.

3. **Sign determined by the later read:** If an externally changing exponent first byte is initially a nonterminator but yields `0x2d` on the final reread, the accumulated exponent is negated. This illustrates read timing, not a guarantee of safe concurrent mutation.

4. **Raw magnitude bytes:** Magnitude bytes `0x31, 0x32, 0xff` accumulate to 540, not decimal 12.

5. **Boundary selection:** For nonzero magnitude, −500 and 500 use in-range conversion. −501 delivers sign-only zero before status storage. At 501, F-2cc6ced0beea is called and its scratch locations are read before delivery; status storage follows delivery.

6. **Conditional seed reread:** For magnitude 1, normalized magnitude words are `0x80000000` and zero. If the seed reads yield 100 and then 200, the final exponent-like field is 137, not 37. With a stable seed \(S\), it is \(S-63\).

7. **Distinct classification values:** Suppose conversion supplies first word `0x12300000` and second word zero, with nonnegative input. If the initial classification read is zero and the later read is `0x123`, status 2 is written, but replacement is not selected.

8. **Zero short circuit and captured replacement value:** If the candidate is zero, the second classification read is skipped and status 2 is written. Replacement is selected exactly when the initially captured classification value is zero, irrespective of later external changes.

9. **Scratch-word OR:** On the above-500 path, if F-2cc6ced0beea validly leaves words `0x80000001` and 7 in result scratch, positive input delivers those same words. The caller does not clear the scratch word’s sign bit.

10. **Modulo-zero magnitude:** Accumulation wrapping to zero modulo \(2^{64}\) takes the zero path, including bypassing exponent-range status handling.

## Uncertainties

The following remain **documented_only**:

- Exact numeric format, exponent bias, rounding semantics, and special-value meanings.
- Values and stability of external seeds, classification constants, and address components.
- Contents, validity, and extent of externally addressed data.
- Dependency result-scratch initialization and all dependency read/write footprints.
- Factor-scratch layout, required extent, and initialization obligations.
- Hidden inputs to F-4a2f439d1347.
- Destination layout and status-storage meaning, validity, and lifetime.
- Dependency global mutations, ownership effects, pointer retention, and failure behavior.
- Signed-overflow behavior and handling of invalid, unterminated, or concurrently modified input.

## Evidence

Established evidence is limited to the supplied function body: byte reads and rereads, arithmetic widths, masks, branches, local field accesses, explicit dependency interactions, external-value reads, and the status store.

The distinct classification reads, conditional seed reread, final exponent-prefix reread, and unproven scratch initialization follow from those operations. They are not replaced by assumptions about stable globals or helpful callee behavior.

Decimal conversion, factor preparation, rounding control, signed zero, underflow, overflow, and special-value classification are inferred interpretations. No dependency bodies, external data contents, runtime observations, or test results establish those interpretations as a complete numeric contract.
