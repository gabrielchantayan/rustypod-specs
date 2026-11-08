<!-- Trusted provenance: F-c4e0ec1bb29e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4e0ec1bb29e

## Purpose

F-c4e0ec1bb29e scans a byte sequence using a caller-supplied radix, applies special handling to an initial zero and a possible hexadecimal prefix, and repeatedly delegates character classification to F-1a5edad330b3. It maintains a multiword accumulator, records whether an overflow condition has occurred, optionally publishes a stopping address, and conditionally writes the value 2 through an address obtained from F-4af36158f863.

The established interface exposes no numeric result. The accumulator affects overflow detection but is not written to a caller-supplied numeric destination.

The apparent intent is an unsigned integer conversion with an optional end-position output and an overflow-status mechanism. That interpretation is inferred, not a substitute for the behavior below. In particular, the supplied evidence does not establish the character repertoire accepted by the classifier, a supported radix range, or the identity of the status storage.

## Interface

The function has three inputs:

| Input | Established role |
|---|---|
| Input address | Address of the first character to examine. |
| Optional stopping-address destination | Null, or an address at which one 32-bit address value is written. |
| Requested radix | A 32-bit unsigned value. Zero invokes prefix-dependent radix selection. |

The represented address model is 32-bit: the stopping-address destination receives one 32-bit store, and character-derived values are also represented as 32-bit addresses in dependency-call arguments.

There is no explicit return value in the supplied interface. Residual machine-register contents are not an established output contract.

**Memory preconditions**

- The input address must permit the reads described below. There is no null-input check and no length argument.
- If the stopping-address destination is nonnull, it must permit a 32-bit store.
- If overflow handling is reached, the address returned by F-4af36158f863 must permit a 32-bit store.
- No alignment validation, address-range validation, or recovery from inaccessible memory is visible.
- A terminating zero byte ends scanning only if F-1a5edad330b3 rejects it. Termination on zero is not independently enforced.

The source represents input elements as characters. Their classification is byte-oriented, but whether values with the high bit set are sign-extended or zero-extended when passed to the classifier is not established.

## Behavior

### Initial read and prefix processing

The function first reads the initial input character. Prefix handling occurs before any classifier call.

If the first character is not `0`:

- A requested radix of zero becomes 10.
- Any nonzero requested radix is retained.
- No character is yet considered accepted.
- The initial character is the first character submitted to the classifier.

If the first character is `0`, the function immediately reads the second character. The initial zero is initially considered accepted, without submitting that zero to the classifier.

When that second character is neither `x` nor `X`:

- A requested radix of zero becomes 8.
- Any nonzero requested radix is retained.
- Classification begins with the second character.
- The leading zero remains sufficient to establish that some input was accepted, even if the classifier immediately rejects the second character.

When the second character is `x` or `X`, behavior depends on the requested radix:

| Requested radix | Prefix handling |
|---|---|
| Zero or 16 | Consume the two-character prefix, set the effective radix to 16, read the third character, and clear the accepted-input indication. Classification begins with the third character. |
| Any other value | Retain the requested radix. Do not consume `x` or `X` as a prefix character. The leading zero remains accepted, and classification begins with `x` or `X`. |

A recognized hexadecimal prefix alone therefore does not establish successful consumption. At least one subsequent classifier result must be nonnegative for the stopping address to advance.

There is no visible initial whitespace skipping, sign processing, or general prefix parser. Such characters reach F-1a5edad330b3 unless displaced by the exact leading-zero rules above.

### Character classification and continuation

For each current character, the function calls F-1a5edad330b3 with:

1. The current character.
2. The effective 32-bit radix.

The dependency result is represented as 64 bits. The low 32 bits, interpreted as a signed integer, determine whether scanning continues:

- A negative low word terminates the loop.
- A nonnegative low word is used as the character’s numeric contribution.

There is no separate check that the contribution is smaller than the radix. The caller relies on the dependency to impose any such restriction.

For every nonnegative result, the function:

- Marks input as accepted.
- Advances its accumulator using that result.
- Reads the next input character.
- Checks the newly computed upper accumulator field for overflow.
- Retains the low accumulator field and advances its next-read address.
- Calls the classifier again for the character just read.

Consequently, accepting a character entails reading the following character before that following character is classified. The final rejected character has already been read when scanning terminates.

Overflow does not terminate scanning. The function continues making classifier calls and reading input until the classifier returns a negative low word.

### Accumulation and arithmetic widths

The accumulator comprises three fields:

- A low field retained modulo \(2^{16}\).
- A middle field retained modulo \(2^{32}\).
- An upper field retained modulo \(2^{32}\).

All three start at zero after prefix processing. A skipped leading zero does not otherwise alter them.

The exact update matters because the input radix is not range-checked. The following equations describe the arithmetic, not an additional validation policy.

Let:

- \(B\) be the effective radix, interpreted as an unsigned 32-bit value.
- \(d\) be the nonnegative low word returned by the classifier.
- \(L\), \(M\), and \(H\) be the old low, middle, and upper fields.
- \(T=(d+B L)\bmod 2^{32}\).
- \(P=B M\), evaluated as an unsigned 64-bit product.
- \(C\) be 1 when adding \(P\bmod 2^{32}\) and \(\lfloor T/2^{16}\rfloor\) carries out of bit 31, and 0 otherwise.

The replacement fields are:

\[
L' = T\bmod 2^{16}
\]

\[
M' =
\left((P\bmod 2^{32})+\left\lfloor T/2^{16}\right\rfloor\right)
\bmod 2^{32}
\]

\[
H' =
\left(
B H+
\left\lfloor P/2^{32}\right\rfloor+
C-
\begin{cases}
M,& B\geq 2^{31}\\
0,& B<2^{31}
\end{cases}
\right)\bmod 2^{32}.
\]

The subtraction for radices with bit 31 set follows from a signed interpretation of the radix in one part of the upper-field computation. It must not be replaced by ordinary unbounded unsigned multiplication for all possible radix values.

After the next character is read, the function checks whether \(H'>65535\). If so, it sets a sticky overflow indication. Once set, this indication is never cleared, even if later modular arithmetic produces a smaller upper field.

For conventional small radices and classifier contributions within the radix, the fields implement the apparent progression of an unsigned value with a 64-bit representable limit: 16 low bits, 32 middle bits, and 16 permitted upper bits. This interpretation is conditional. It does not establish equivalent behavior for arbitrary 32-bit radices or arbitrary nonnegative classifier results.

### Stopping-address publication

After the classifier rejects a character, the function optionally publishes an address.

If the stopping-address destination is nonnull:

- If any input has been accepted, it stores the address of the rejected character.
- Otherwise, it stores the original input address.

The stored value occupies 32 bits.

The accepted-input indication includes a leading zero handled outside the classifier, except when that zero is part of a recognized hexadecimal prefix. Thus:

- A standalone accepted leading zero can advance the stopping address.
- A recognized prefix with no accepted following character does not advance it.
- The rejected character is not included in the consumed span.

The stopping-address store occurs before any overflow-status dependency call.

### Overflow-status action

If the sticky overflow indication is clear, the function makes no visible call to F-4af36158f863 and performs no visible status store.

If the indication is set, it calls F-4af36158f863 with two represented arguments:

1. If a stopping-address destination was supplied, the address selected for that destination. Otherwise, the terminating classifier result’s low 32 bits, represented as an address.
2. The terminating classifier result’s high 32 bits, interpreted as a signed 32-bit quantity.

The function then writes the 32-bit value 2 through the address returned by that dependency.

The evidence does not establish whether those represented arguments are meaningful inputs to the dependency, incidental calling-convention artifacts, or ignored values. It does establish their provenance at this call site.

There is no null check on the returned address. No numeric saturation result, exception return, or early exit accompanies the status write.

## Side effects

Directly established effects are:

- Reads from the input sequence, including prefix lookahead and one next-character read for every classifier-accepted character.
- Calls to F-1a5edad330b3, including one final call whose low result is negative on any normally terminating path.
- One optional 32-bit stopping-address store.
- On detected overflow, one call to F-4af36158f863 followed by one 32-bit store of 2 through its result.

The directly visible write order is stopping-address publication first, overflow-status publication second.

No allocation, release, or input mutation is visible in this function. No ownership transfer is established. Caller-provided input and stopping-address storage remain caller-managed, subject to unknown dependency behavior.

The dependencies may read or modify global state, access memory, allocate, release resources, or fail. None of those possibilities can be excluded from this excerpt. In particular, the status-returning dependency must not be assumed to return thread-local storage or a conventional error-number location.

If the optional destination aliases input memory, its store occurs after scanning has finished. If it aliases storage later returned by the status dependency, the later store of 2 can overwrite it. The evidence supplies no non-aliasing guarantee.

## Dependencies

### F-1a5edad330b3

**Established call-site contract**

- Receives the current character and effective radix.
- Supplies a represented 64-bit result.
- A negative signed low word stops scanning.
- A nonnegative low word supplies the accumulator contribution.
- The high word from the terminating call is forwarded to the overflow-status dependency if overflow occurred.

**Not established**

- Accepted characters, letter case rules, or supported radices.
- Whether zero bytes are always rejected.
- Whether accepted contributions always fall below the radix.
- Whether the high word has a defined semantic meaning.
- Side effects, ownership behavior, or failure modes.

### F-4af36158f863

**Established call-site contract**

- Called only after sticky overflow has been detected.
- Called after any requested stopping-address publication.
- Supplies an address used immediately for a 32-bit store of 2.

**Not established**

- The meaning of its represented arguments.
- The identity, lifetime, ownership, alignment, or sharing properties of the returned storage.
- Whether the returned address can be null.
- Global-state changes, allocation behavior, or failure behavior.

## Acceptance examples

These are conditional static expectations, not executed tests. Character classifications are stated explicitly where needed.

1. **Immediate rejection without a leading zero**

   Given input beginning with `q`, radix 10, and a classifier that rejects `q`, the stopping-address output is the original input address. No accumulator overflow occurs, so no overflow-status call is made.

2. **Automatic decimal selection**

   Given input `27!`, requested radix zero, classifier contributions 2 and 7, and rejection of `!`, both digit classifications use radix 10. The stopping address identifies `!`. The accumulator represents 27 without setting overflow.

3. **Automatic octal selection**

   Given input `09`, requested radix zero, and rejection of `9` under radix 8, the initial zero is accepted without classification. The stopping address identifies `9`, not the original zero.

4. **Recognized prefix without digits**

   Given input `0x!`, requested radix zero or 16, and rejection of `!` under radix 16, the stopping-address output is the original input address. Neither the zero nor the prefix marker is submitted to the classifier.

5. **Recognized hexadecimal digits**

   Given input `0Xf!`, requested radix zero, contribution 15 for `f`, and rejection of `!`, classification uses radix 16. The stopping address identifies `!`, and the accumulator represents 15.

6. **Prefix marker incompatible with the requested radix**

   Given input `0x7`, requested radix 10, and rejection of `x`, the zero is accepted, `x` is classified using radix 10, and the stopping address identifies `x`. The function does not switch to hexadecimal.

7. **Conventional 64-bit boundary**

   Assuming ordinary decimal digit contributions and rejection of the following terminator, `18446744073709551615` does not set overflow. `18446744073709551616` does set it. In the latter case, scanning still reaches the rejected terminator, the optional stopping address is written, and the status dependency’s returned location receives 2.

8. **Null stopping-address destination**

   Given any overflowing sequence followed by a rejected character, a null stopping-address destination suppresses only that output store. Overflow handling still occurs, using the terminating classifier result as described above.

## Uncertainties

The following remain **documented_only**:

- The true externally callable ABI, including whether the represented void return omits an undocumented machine-level result.
- Character signedness for bytes outside the basic character range.
- Classifier semantics, valid radix limits, and zero-byte handling.
- Whether dependency-call argument representations reflect their actual declared interfaces.
- The identity and ownership of overflow-status storage.
- Behavior on inaccessible memory, address wraparound, dependency failure, or nonterminating classifier acceptance.
- Any concurrency guarantees or restrictions on aliasing.

Unsigned integer-conversion intent and a conventional “out of range” interpretation of the stored value 2 are inferences. Neither is required to describe the established stores, calls, and arithmetic.

## Evidence

The supplied static function body establishes the initial-zero branches, conditional hexadecimal-prefix handling, classifier-controlled loop termination, finite-width accumulator updates, sticky upper-field overflow check, optional stopping-address write, and conditional final status write.

The dependency bodies, external declarations, platform ABI documentation, callers, and runtime observations were not supplied. No runtime tests have been performed. All acceptance examples above are conditional deductions from the static evidence, not observed executions.
