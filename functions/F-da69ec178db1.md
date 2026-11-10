<!-- Trusted provenance: F-da69ec178db1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da69ec178db1

## Purpose

F-da69ec178db1 writes a zero-terminated character sequence into caller-provided storage and writes three signed 32-bit fields into a separate result record.

**Established static behavior:** the function classifies an input representation, derives a sign-sensitive selector, obtains scaled state through dependencies, and generates output through one of two modes. Some attempts restart scaling and overwrite previously written bytes. A nonzero mode can fall back to mode zero with precision 17.

**Inferred intent:** the operations resemble numeric formatting with scaling, rounding, and precision control. Conventional decimal conversion is not established. In particular, the visible extraction data flow does not supply ordinary decimal digits.

**Documented-only behavior:** dependency semantics, initialization of some temporary storage, the type of an estimate-related global operand, exceptional arithmetic, and retry termination remain unresolved. No runtime tests have been performed.

## Interface

The function accepts five inputs and has no explicit return value.

| Input | Representation | Visible use |
|---|---|---|
| Result record | Pointer to writable signed 32-bit fields | Receives fields at byte offsets 0, 4, and 8. |
| Character buffer | Pointer to writable character storage | Receives character writes, reversal accesses, and a zero terminator. No capacity is supplied. |
| Input token | Opaque 32-bit value | Passed unchanged to F-2cc6ced0beea. |
| Precision | Signed 32-bit integer | Controls scaling or output length; replaced with 17 on fallback. |
| Mode | Signed 32-bit integer | Zero and nonzero select different paths. Exactly 1 has special zero-input behavior. |

The result record has the following layout:

| Byte offset | Field | Established meaning |
|---|---|---|
| 0 | Position | An estimate-derived or length-derived signed arithmetic result. A decimal-position interpretation is inferred. |
| 4 | Length | Selected output length and terminator index on ordinary completion. |
| 8 | Effective mode | Mode at completion, including fallback to zero. |

There is no separate status result, capacity result, sign field, or returned buffer pointer.

### Storage, ownership, and arithmetic requirements

The function performs no visible validation of pointers, capacity, alignment, precision, or mode.

For distinct output regions and ordinary nonnegative lengths:

- Zero-classified input with mode 1 needs one writable buffer byte.
- Zero-classified input with another mode needs precision plus one writable bytes.
- A completing mode-zero attempt with positive precision needs precision plus one writable bytes.
- A nonzero-mode attempt can write indices 0 through 16 and terminate at index 17. Eighteen writable bytes cover that span and the ordinary precision-17 fallback.
- A mode-zero attempt with precision zero can read buffer index 0 before writing its terminator.

These are conditional access requirements, not checks performed by the function. Negative precision can produce a terminator access before the supplied buffer pointer. Arithmetic boundaries can invalidate simple length calculations.

The visible representation includes signed and unsigned 32-bit quantities, 64-bit dependency results and extraction state, and single-character accesses. The estimate global’s declared width and signedness are **not supplied**. Consequently, the estimate expression’s multiplication width, conversions, shift interpretation, and narrowing behavior cannot be fixed from this excerpt.

No allocation, release, or ownership transfer is directly visible. Dependency ownership and retention behavior are unknown.

No output-aliasing prohibition is shown. The descriptions below assume distinct result and buffer regions. If they overlap, the stated access order matters because later writes can overwrite earlier output.

## Behavior

### Initial calls and classification

The function first calls F-2cc6ced0beea with the input token and a pointer into temporary storage. It then calls F-96fcc6b36b88 with two zero arguments.

Only after both calls return does it inspect the temporary words used for sign handling and zero classification. Classification therefore depends on the words present at that time, not necessarily the contents immediately after the first call. The second dependency’s preservation of temporary storage is not established.

Two 32-bit words participate:

- The first word’s high bit supplies a sign discriminator.
- Its remaining 31 bits, together with the entire second word, determine zero classification.
- The second word is read for the zero test only when the first word’s lower 31 bits are zero.

Bits 23:22 of the 32-bit result from F-96fcc6b36b88 determine an original selector, adjusted by the sign discriminator:

| Returned bits 23:22 | Sign clear | Sign set |
|---|---:|---:|
| 00 | 0 | 0 |
| 01 | 1 | −1 |
| 10 | −1 | 1 |
| 11 | −1 | −1 |

Here, −1 denotes the all-ones 32-bit pattern. Other result bits do not contribute to the selector.

Both initial dependencies run even when the function subsequently takes an immediate zero-input return.

### Zero-classified input

Zero classification requires the first word’s lower 31 bits and the entire second word to be zero. The sign discriminator may be either clear or set.

For mode exactly 1:

- No `'0'` character is generated.
- A terminator is written at buffer index 0.
- Position is the negation of the quantity precision plus one.
- Length is zero.
- Effective mode remains 1.

The addition and negation use signed 32-bit quantities; behavior at overflow boundaries is unresolved.

For every other mode:

- Starting at index 0, the function writes `'0'` while the signed index is less than precision.
- It writes the terminator at the index equal to precision.
- Position is zero.
- Length is precision.
- Effective mode remains the supplied mode.

For nonnegative precision, this produces exactly precision zero characters. For negative precision, the fill loop is skipped, but the terminator still targets the negative precision index.

On either zero-input branch, any character fill precedes the terminator. The terminator is followed by metadata stores in this order:

1. Position.
2. Length.
3. Effective mode.

### Initial estimate and global-read timing

Nonzero classification leads to an initial estimate. The expression combines:

- An exponent-like signed 32-bit temporary whose initialization is not established.
- Subtraction of 1023 from that temporary.
- A read of a global operand.
- Multiplication by that global operand.
- A right shift by 16.
- Assignment to a signed 32-bit estimate.

The global’s declaration is absent. Its width, signedness, and value are unresolved. A wider or unsigned global operand could change the usual arithmetic conversions, product width, right-shift behavior, and conversion into the signed 32-bit destination. In particular, neither a 32-bit product nor a particular point of narrowing is established.

The signed 32-bit temporary and signed 32-bit destination are established independently of those expression uncertainties. Subtraction at the temporary’s arithmetic boundary also remains unresolved.

The global read occurs after both initial dependency calls and before the scaling retry loop. Those dependencies may affect its value. The visible function does not reread this global on retries; instead, retries reuse and sometimes adjust the evolving signed 32-bit estimate.

### Scaling attempt and extraction-state construction

Each attempt calculates a signed 32-bit scaling amount:

- Mode zero: current estimate minus precision, then plus one.
- Nonzero mode: negation of precision.

These expressions have unresolved signed-overflow boundaries. A working selector begins as the original selector. A strictly positive scaling amount negates that selector in its unsigned 32-bit representation.

The visible sequence is:

1. F-99579940fdac receives a two-word temporary region and the scaling amount’s absolute magnitude.
2. F-795e866d730a receives the two input-state words and returns 64 bits.
3. The returned high word is retained, while hexadecimal 201F is subtracted from the returned low word.
4. The working selector is written to a separate unsigned 32-bit rounding-state temporary slot.
5. The first word of the scaling temporary is adjusted, and a scaling dependency is called.

For nonpositive scaling, the adjustment subtracts hexadecimal 201F and the dependency is F-bbd1fe9881a1. For positive scaling, the adjustment adds hexadecimal 201F and the dependency is F-78d399bb1bbd.

The absolute magnitude is not safely established for the minimum signed 32-bit scaling amount. The adjusted signed temporary also has unresolved overflow boundaries.

Each scaling dependency receives pointers to the adjusted temporary regions and returns 64 bits. No failure result is tested.

Extraction state is then constructed as follows:

- Normally, its high word comes from the scaling result and its low word is the original selector.
- If any of the scaling result’s lowest 16 bits are nonzero, the complete extraction state is instead hexadecimal 7FFFFFFFFFFFFFFF.

Thus, the scaling result’s low word does not ordinarily become the extraction state’s low word. Sentinel substitution does not produce a separate error status.

After state construction, the function writes the original selector into the same rounding-state slot that previously received the working selector. This second write precedes either extraction path and occurs on both scaling branches, including the sentinel case. With positive scaling, it can replace a negated selector.

The original selector also starts the mode-zero character accumulator. That separate use does not eliminate or replace the second rounding-slot write. Any implicit dependency access to this slot or surrounding storage remains unknown.

### Nonzero-mode extraction

A nonzero-mode attempt starts with generated length zero and a retry flag explicitly set to zero.

While extraction state is nonzero and fewer than 17 iterations have occurred, the function:

- Sets a character temporary to zero.
- Calls F-5ecc94b34ace with the state’s low and high 32-bit halves.
- Replaces extraction state with the returned 64 bits.
- Writes the character temporary plus `'0'` at the next increasing buffer index.

The explicit data flow therefore writes `'0'`. No digit-output pointer is passed to the dependency. Ordinary digit production through an implicit effect is not established.

If state becomes zero within the limit, the generated span is reversed in place. Each exchanged pair has this access order:

1. Read the lower-index byte.
2. Read the upper-index byte.
3. Write the lower-index byte.
4. Write the upper-index byte.

The selected result is:

- Position: generated length minus precision, then minus one.
- Length: generated length.
- Effective mode: the current nonzero mode.

Signed arithmetic boundaries remain unresolved. If state is initially zero, there are no extraction calls, character writes, or reversal accesses; generated length is zero.

If state remains nonzero after 17 calls, the ordinary visible path changes precision to 17, changes mode to zero, and restarts scaling. It does not reread the estimate global or clear the bytes already written.

A separate branch tests the retry flag and, if nonzero, reaches common completion without visible position and length assignments on that branch. No direct assignment in the shown path makes the flag nonzero. Its reachability and valid output are unresolved; it does not establish another usable result contract.

### Mode-zero extraction and correction

For positive effective precision, each attempt makes exactly that many extraction calls. Writes proceed from index precision minus one down to index zero.

At each index:

- F-5ecc94b34ace receives the current state halves.
- Its 64-bit return replaces extraction state.
- Hexadecimal 30 is added to an unsigned 32-bit accumulator initially holding the original selector.
- The accumulator is converted to a character and stored.

The unsigned accumulator addition is modulo \(2^{32}\). Character narrowing and signedness are not fully specified. The accumulator is cumulative; no explicit dependency result supplies a digit.

After extraction:

- If state remains nonzero, the estimate is incremented and scaling restarts.
- If state is zero, buffer index 0 is read.
- If that byte equals `'0'`, the estimate is decremented and scaling restarts.
- Otherwise, the attempt completes.

Estimate increments and decrements retain unresolved signed-overflow boundaries. There is no visible retry limit. Repeated attempts can overwrite the same output region.

For precision zero, extraction makes no calls and no character writes. If state is zero, the function nevertheless reads buffer index 0, whose contents were not initialized by that attempt.

Negative precision also skips extraction when the initial decrement is representable. Minimum signed precision encounters an unresolved decrement boundary. A completing negative-length attempt targets a negative terminator index.

Successful mode-zero completion selects the current estimate as position and effective precision as length.

### Common completion for nonzero-classified input

After either successful extraction path, the visible stores are:

1. Terminator at the selected length index.
2. Effective mode.
3. Length.
4. Position.

This metadata order differs from the position-first order used by zero-classified input.

## Side effects

Direct effects include caller-buffer writes, caller-record writes, temporary-state writes, buffer reads during reversal, and the leading-byte read used for mode-zero correction.

Failed attempts can leave intermediate bytes in the buffer. No rollback or transactional-output guarantee exists.

On the nonzero-input path, the function visibly reads the estimate global once. Its declared type is unknown. No direct global write is shown. Dependencies may independently read or modify globals, temporary storage, or external state.

No direct dereference of the opaque input token, allocation, release, or ownership transfer is shown. This does not establish that dependencies avoid those actions or refrain from retaining pointers.

The function has no visible exception handler, dependency-failure check, or retry-exhaustion result. A dependency that does not return prevents subsequent visible operations. Correction retries have no established termination guarantee.

## Dependencies

| Opaque ID | Visible interaction | Unresolved contract |
|---|---|---|
| F-2cc6ced0beea | Receives the input token and a temporary-region pointer. | Initialized extent, representation, ownership, retention, and other effects. |
| F-96fcc6b36b88 | Receives two zeros; result bits 23:22 contribute to the selector. | Query meaning and effects on temporaries, globals, or environment. |
| F-99579940fdac | Receives two-word storage and scaling magnitude on each attempt. | Initialization, representation, supported magnitudes, and side effects. |
| F-795e866d730a | Receives two input-state words and returns 64 bits. | Conversion semantics and other effects. |
| F-bbd1fe9881a1 | Receives adjusted temporary pointers for nonpositive scaling. | Arithmetic, pointer writes, and implicit rounding-state access. |
| F-78d399bb1bbd | Receives adjusted temporary pointers for positive scaling. | Arithmetic, pointer writes, and implicit rounding-state access. |
| F-5ecc94b34ace | Receives state halves and returns replacement 64-bit state. | Extraction semantics, implicit character effects, and other side effects. |

No dependency is assumed to preserve globals, preserve all temporary fields, or acquire or release memory.

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid, distinct output regions and representable arithmetic unless explicitly discussing an unsafe boundary.

1. **Post-call classification.** If the words present after both initial calls have zero lower 31 bits in the first word and a zero second word, the zero-input branch applies. Earlier contents alone do not determine classification.

2. **Zero input, mode 1, precision 3.** The function writes only a terminator at index 0, then position −4, length 0, and effective mode 1.

3. **Zero input, precision 3, mode 0 or 2.** Three `'0'` characters precede the terminator. Position is 0 and length is 3. Effective mode remains 0 or 2 respectively.

4. **Zero input, precision 0, mode other than 1.** A terminator is written at index 0; position and length are zero.

5. **Negative zero-input precision.** With precision −1 and mode 0, no fill occurs. The terminator targets index −1, and length is −1. No rejection is shown.

6. **Selector overwrite.** Returned selector bits 01 with a clear sign discriminator produce selector 1. Positive scaling writes the all-ones working selector to the rounding slot before F-78d399bb1bbd, then writes 1 to that slot before extraction.

7. **Estimate-global timing without a width assumption.** If an initial dependency changes the estimate global, initial estimation uses its subsequently read value, interpreted according to the unresolved global type and expression conversions. A correction retry has no additional visible read of that global.

8. **Initially zero state in nonzero mode.** Conditional on state construction producing zero, extraction and reversal are skipped. Length is zero, position is negative precision minus one, and mode is unchanged.

9. **Seventeen-call fallback.** If state remains nonzero after 17 extraction calls and the explicitly initialized retry flag remains zero, scaling restarts with precision 17 and mode zero. Later successful completion reports mode zero.

10. **Zero precision in mode zero.** If state is zero, buffer index 0 is read without an extraction write. A preexisting `'0'` causes estimate decrement and retry; another byte permits completion with length zero.

## Uncertainties

The following remain `documented_only`:

- Input-token interpretation and complete temporary layout.
- Initialization and meaning of the exponent-like signed 32-bit temporary.
- The estimate global’s declared width, signedness, value, and mutation rules.
- Resulting estimate-expression conversions, multiplication width, shift behavior, and narrowing into the signed 32-bit destination.
- Signed overflow, minimum-value negation, and boundary decrements.
- Character narrowing and any implicit digit-producing effects.
- Dependency access to rounding state, surrounding storage, or output regions.
- Complete alignment, aliasing, ownership, and retention requirements.
- Handling of non-finite values, subnormal values, or numerical range errors.
- Correction-loop termination.
- Reachability and usable metadata for the nonzero retry-flag branch.

Numeric-formatting intent does not resolve these gaps.

## Evidence

The supplied static representation establishes the interface, record layout, call order, post-call classification reads, selector mapping, global-read timing, signed 32-bit estimate temporary and destination, rounding-slot writes, extraction-state construction, output access order, retry conditions, fallback, and metadata-store order.

It does **not** declare the estimate global’s type and therefore does not establish its read width or the multiplication and shift expression’s arithmetic width. It also does not establish complete dependency contracts or conventional decimal output.

No runtime execution, output comparison, memory-safety check, or termination test has been performed.
