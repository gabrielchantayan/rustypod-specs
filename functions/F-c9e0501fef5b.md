<!-- Trusted provenance: F-c9e0501fef5b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9e0501fef5b

## Purpose

F-c9e0501fef5b accepts a 32-bit word and returns a 32-bit word. Its ordinary paths extract a sign, an exponent-like field, and a fraction-like field, then produce an integer-shaped result using shifts and unsigned negation. Other paths return a sign-dependent endpoint or transfer control to an unresolved indirect dependency.

**Inferred intent:** the input layout and endpoint values strongly suggest conversion from an IEEE 754 binary32 encoding to a signed 32-bit integer, with truncation toward zero and endpoint handling for out-of-range values. This interpretation is not a complete contract: large shift counts and the unresolved indirect transfers prevent an unconditional specification as a conventional floating-point conversion.

**Established behavior:** the visible operations are integer operations on the input bits. No floating-point instruction, allocation, memory release, or output-memory write is shown.

## Interface

### Explicit input and output

- **Input:** one 32-bit unsigned word.
- **Output:** one 32-bit unsigned word, when the function returns normally.
- The input is interpreted by value, not as a pointer.
- The return word may be interpreted by a consumer as a two’s-complement signed integer, but that interpretation does not change the unsigned interface.

The input has the following field layout:

| Bits | Field used in this contract |
|---|---|
| 31 | Sign bit |
| 30–23 | Eight-bit exponent field |
| 22–0 | Twenty-three-bit fraction field |

These field names describe the observed bit manipulations; the floating-point interpretation remains inferred.

### Implicit inputs

Some paths depend on execution context beyond the explicit argument:

- A readable 32-bit control word.
- Three additional readable 32-bit words, called **auxiliary A**, **auxiliary B**, and **auxiliary C** here.
- An incoming link-context value used to construct an indirect transfer target.

These names are descriptive labels, not public symbols or addresses. The available evidence does not establish whether the readable words are writable state, fixed image data, or another kind of storage.

### Arithmetic model

Unless stated otherwise:

- Arithmetic results are retained as 32-bit words.
- Left shifts discard bits beyond bit 31.
- Unsigned negation means subtraction from zero modulo \(2^{32}\).
- Comparisons against hexadecimal thresholds are unsigned.
- Right shifts of explicitly signed values are presented as arithmetic right shifts.
- Right shifts of unsigned values are logical right shifts.

The conversion paths can request logical right shifts by counts greater than 31. Their concrete result is **documented_only** because the supplied representation does not establish the target’s oversized-shift semantics.

### Preconditions

For paths that use only ordinary-width integer operations, no buffer or ownership precondition is visible.

For paths reaching the indirect dependency, safe completion additionally requires:

- The selected data words to be readable.
- The target derived from the incoming link context to be valid for the actual transfer mechanism.
- The target to accept the machine-level argument arrangement used by that path.
- The target to return normally if a normal return from this function is expected.

Those requirements are not verified by the visible function.

## Behavior

### 1. Construct the initial shifted magnitude

The function first interprets the input as signed and arithmetically shifts it right by 23 bits.

For a nonnegative input, that result is the exponent field, from 0 through 255. For a negative input, it is negative, with the exponent field retained in its low eight bits.

A working magnitude is formed by shifting the input left by eight bits. This places the fraction field in bits 30–8 and clears bits 7–0.

If the signed-shift result is nonzero, bit 31 of the working magnitude is set.

Consequences:

- For a nonnegative input with exponent zero, no leading bit is inserted.
- For a nonnegative input with nonzero exponent, a leading bit is inserted.
- For every negative input, a leading bit is initially inserted, including negative inputs with exponent zero.
- The negative exponent-zero case has a later correction before its conversion return.

### 2. Negative-input conversion path

For a negative input, the function compares the original word with `0xCF000001`.

If the input is strictly less than that threshold:

1. The exponent is taken from the low eight bits of the signed-shift result.
2. If that exponent is zero, bit 31 of the working magnitude is cleared.
3. A right-shift count is formed as the low eight bits of \(158-\text{exponent}\).
4. The working magnitude is logically shifted right by that count.
5. The function returns the unsigned negation of the shifted result.

This path includes `0xCF000000`, but excludes `0xCF000001`.

For admitted negative inputs, the exponent ranges from 0 through 158. The shift count therefore ranges from 158 down to 0:

- Exponents 127 through 158 use counts from 31 through 0, with ordinary 32-bit shift behavior.
- Exponents 0 through 126 use counts from 158 through 32, whose concrete shift behavior is not established here.

No control-word or auxiliary-word access is visible on this return path.

If a negative input is at least `0xCF000001`, execution proceeds to the fallback classification below.

### 3. Nonnegative-input conversion path

For a nonnegative input, the direct conversion return occurs only when:

- The exponent is less than 159; and
- \(158-\text{exponent}\) is nonzero.

Together, these conditions admit exponents 0 through 157.

The function forms the low eight bits of \(158-\text{exponent}\), logically shifts the working magnitude right by that count, and returns the shifted word without negation.

Consequences:

- Exponents 127 through 157 use counts from 31 through 1.
- Exponents 0 through 126 use counts from 158 through 32, requiring the unresolved oversized-shift rule.
- Exponent 158 does not use this direct return, even though its corresponding shift count would be zero.
- Exponents 159 through 255 also proceed to fallback classification.

No control-word or auxiliary-word access is visible on this direct return path.

### 4. Fallback classification and endpoint return

The function next doubles the original input as a 32-bit unsigned value. Overflow is discarded. This doubled word is also retained for possible use as an indirect-call argument.

If the doubled word is less than `0xFF000001`, the function returns an endpoint selected solely by the original sign:

| Original sign | Returned word | Signed interpretation |
|---|---:|---:|
| Nonnegative | `0x7FFFFFFF` | 2,147,483,647 |
| Negative | `0x80000000` | −2,147,483,648 |

A separate equality check for doubled value `0xFF000000` is visible, but both outcomes return the same endpoint. It establishes no distinct output or state change.

Under the inferred binary32 interpretation:

- Positive and negative infinity return these endpoints.
- Finite inputs that reach this stage also return these endpoints.
- Encodings with an all-ones exponent and nonzero fraction proceed to control-dependent handling.

The direct negative path already returns `0x80000000` for `0xCF000000`; it does not need this endpoint branch.

### 5. Control-dependent indirect handling

This stage is reached only when the doubled input word is greater than `0xFF000000`.

The function reads the control word and retains its low four bits as the selected mode. That mode selection is not visibly recomputed after later control-word reads.

Every branch below invokes the same unresolved indirect target, designated **F-7a1d4b2086ce** in this document. The target is obtained by clearing the low two bits of the incoming link-context value.

If the dependency returns normally, its 32-bit result becomes the function’s return value without further visible modification.

#### Mode 9

The function reads the control word to test mask `0x00100000`.

- If the bit is clear, it invokes the dependency with one explicit argument: `0`.
- If the bit is set, it reads the control word again and invokes the dependency with two explicit arguments:
  1. `8`.
  2. The 32-bit result of shifting that control-word value left by 15, complementing all bits, and shifting left once more.

For the second argument:

- Bit 0 is zero.
- Bits 15–1 are ones.
- Bits 31–16 are the complement of the low 16 bits of the control value used for this argument.

The bit test and the argument construction use separate visible control-word references. A single stable snapshot is not established.

#### Mode 8

The function reads the control word to test mask `0x40`.

If that bit is set, it invokes the dependency with two explicit arguments, both zero. No auxiliary-word read is visible on this branch.

If that bit is clear:

1. Auxiliary A is read.
2. The control word is read to test mask `0x10`.
3. If `0x10` is clear, the dependency receives:
   - Auxiliary A’s value.
   - The doubled original input.
4. If `0x10` is set:
   - Auxiliary B is read.
   - The dependency receives `0` followed by auxiliary B’s value.

The read of auxiliary A precedes the `0x10` decision even when that value is subsequently discarded.

#### Mode 10

The function:

1. Reads auxiliary C.
2. Reads the control word to test mask `0x40`.
3. Selects auxiliary C’s value if the bit is clear, or `0x80000000` if the bit is set.
4. Invokes the dependency with that one explicit argument.

Auxiliary C is read even when the control bit causes its value to be replaced.

#### All other modes

For low-nibble modes other than 8, 9, and 10, the dependency is invoked with no explicit arguments shown in the supplied representation.

This does not establish that every physical argument register is cleared or that the actual target has a zero-argument interface.

## Side effects

### Direct effects

The visible body:

- Performs no explicit memory writes.
- Performs no allocation or release.
- Does not dereference the explicit input as an address.
- Reads the control and auxiliary words only after reaching control-dependent handling.
- Returns directly without those reads on ordinary conversion and endpoint paths.

The order of the relevant reads is as specified above. The evidence does not establish atomicity, volatility, concurrency safety, or whether an implementation may combine repeated reads.

### Indirect effects

Effects of F-7a1d4b2086ce are unknown. It must not be assumed to:

- Preserve the control word or auxiliary words.
- Preserve other global state.
- Avoid allocation or release.
- Avoid I/O.
- Return normally.
- Report failures through any particular channel.

No ownership of memory is visibly transferred through pointer-typed arguments. The shown arguments are words, but their interpretation by the unresolved target is unknown.

There is no visible post-call cleanup or local error translation. A normally returned dependency result is forwarded unchanged.

## Dependencies

### F-7a1d4b2086ce — unresolved indirect dependency

This is a document-local opaque ID for an unresolved target, not a verified independent function identity.

**Target selection:** incoming link context with its low two bits cleared.

**Visible explicit argument forms:**

- No arguments shown.
- One word: zero.
- One word: auxiliary C or `0x80000000`.
- Two words: zero and zero.
- Two words: auxiliary A and doubled input.
- Two words: zero and auxiliary B.
- Two words: `8` and a control-derived bit pattern.

**Return:** a 32-bit word, forwarded unchanged if the dependency returns normally.

**Unknown:** actual calling convention, target identity, whether the transfer is properly represented as an ordinary call, state preservation, ownership behavior, and failure behavior.

No other callable dependency is established by the supplied evidence.

## Acceptance examples

These examples are static consequences or explicitly conditional expectations. None has been executed.

### Direct conversion with ordinary shift counts

| Input word | Expected return | Basis |
|---|---:|---|
| `0x3F800000` | `0x00000001` | Nonnegative, exponent 127, shift count 31 |
| `0x3FC00000` | `0x00000001` | Fractional low bits are discarded by the right shift |
| `0xBF800000` | `0xFFFFFFFF` | Shifted magnitude 1, followed by unsigned negation |
| `0xBFC00000` | `0xFFFFFFFF` | Magnitude truncates to 1 before negation |
| `0x4EFFFFFF` | `0x7FFFFF80` | Exponent 157, shift count 1 |
| `0xCF000000` | `0x80000000` | Admitted negative boundary, shift count zero |

These paths must not require the control or auxiliary words.

### Endpoint behavior

| Input word | Expected return |
|---|---:|
| `0x4F000000` | `0x7FFFFFFF` |
| `0xCF000001` | `0x80000000` |
| `0x7F800000` | `0x7FFFFFFF` |
| `0xFF800000` | `0x80000000` |

The first two examples distinguish the positive and negative direct-conversion boundaries.

### Oversized shifts

For `0x3F000000`, the direct positive path requests a logical right shift by 32.

- If the actual machine operation returns zero for a 32-bit value shifted by 32, the result is zero.
- If the actual operation masks the count to five bits, the result instead uses an effective count of zero.

The contract does not choose between those behaviors without additional evidence. Positive zero, negative zero, subnormal encodings, and many values with magnitude below one also enter paths requiring oversized-shift resolution.

### Control-dependent behavior

For input `0x7FC00000`, the doubled word is `0xFF800000`, so indirect handling is selected.

Conditional examples:

- With mode 8 and control mask `0x40` set, the dependency receives `(0, 0)`; no auxiliary read is shown.
- With mode 8, mask `0x40` clear, and mask `0x10` clear, auxiliary A is read before the second control test, and the dependency receives auxiliary A and `0xFF800000`.
- With mode 8 and mask `0x10` set after the first test, auxiliary A is still read, then auxiliary B supplies the second argument; the first argument is zero.
- With a stable control value of `0x00100009`, mode 9 takes its two-argument branch and supplies `8` and `0xFFF6FFFE`.
- With mode 10 and mask `0x40` set, auxiliary C is read before the dependency receives `0x80000000`.

In every example, if the dependency returns a particular 32-bit word normally, this function returns that same word.

## Uncertainties

The following are **documented_only**:

1. **Oversized right shifts.** Counts from 32 through 158 occur on reachable direct paths. Their machine-level semantics are not established.
2. **Indirect-transfer interpretation.** The supplied representation marks these transfers as incompletely recovered. Their ordinary-call appearance is not sufficient proof of the actual control-flow mechanism.
3. **Incoming link context.** Its value and validity as a target source are not established by the explicit interface.
4. **Storage properties.** Mutability, alignment requirements, synchronization, and read side effects of the referenced words are unknown.
5. **Read stability.** Repeated references are not evidence of a coherent snapshot.
6. **Exception policy.** The control masks resemble policy selectors, but no named floating-point exception, status flag, or rounding policy is established.
7. **Dependency effects and failures.** Memory ownership, global preservation, nonlocal exits, and error signaling remain unknown.

No explicit validation error, error code, or diagnostic is visible in the local body.

## Evidence

This contract is based only on the supplied static representation of F-c9e0501fef5b.

Established evidence includes the 32-bit comparisons and arithmetic, field extraction, direct-return thresholds, endpoint results, control-mask branches, auxiliary-read ordering, explicit indirect argument expressions, and forwarding of the indirect result.

The floating-point conversion interpretation is inferred from the input layout, exponent offset, magnitude construction, and endpoint values. It is not evidence that every input follows a standard language or floating-point conversion contract.

No runtime tests, target execution, dependency inspection, or independent instruction-level verification have been performed.
