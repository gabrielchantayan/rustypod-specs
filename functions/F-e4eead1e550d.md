<!-- Trusted provenance: F-e4eead1e550d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4eead1e550d

## Purpose

F-e4eead1e550d transforms two unsigned 32-bit source words, adjusts a signed 32-bit input, and invokes F-d350614e3326 once. It then examines the dependency’s 64-bit result and selects between:

- A 64-bit result containing a retained source-derived word and a combination of the dependency’s result words.
- An unsigned 64-bit conversion of a stored fallback object.

The established behavior is a bit transformation, dependency invocation, conditional result construction, and conditional fallback access. A higher-level interpretation involving a particular numeric format, exponent, status code, or special value is not established.

The accepted-result construction contains an unsigned-to-signed conversion. Its result is established for extracted high words through 0x7FFFFFFF. For larger high words, the available evidence does not establish the conversion result or guarantee that conversion completes normally. Consequently, the usual modulo-\(2^{32}\) combination of the original result words is conditional for those larger values.

## Interface

### Inputs

The function receives four arguments in this order:

| Argument | Width and interpretation | Visible use |
|---|---|---|
| Opaque input | 32-bit quantity | Forwarded unchanged to F-d350614e3326 |
| Upper source word | Unsigned 32-bit integer | Split between two transformed arguments |
| Lower source word | Unsigned 32-bit integer | Its upper 21 bits contribute to a transformed argument |
| Adjustment input | Signed 32-bit integer | Reduced by 15,361 before forwarding |

“Upper” and “lower” identify the source words’ roles in the transformation. They do not establish an external representation, byte order, floating-point format, or memory layout.

The wrapper does not explicitly dereference an argument. The opaque input may have pointer-like or resource-like meaning to F-d350614e3326, but neither that interpretation nor its validity requirements is established.

### Output

On normal completion, the function returns an unsigned 64-bit value.

On the accepted-result path:

- Bits 63–32 contain the retained source-derived word.
- Bits 31–0 contain the combined dependency-result word.

These are numerical bit positions, not a specification of physical registers or stored byte order.

On the rejected-result path, the return value is the stored fallback object converted to unsigned 64-bit form. The object’s declaration, width, signedness, and contents are unavailable.

### Preconditions and limits

The wrapper performs no explicit argument validation.

For the adjustment subtraction to have a representable signed 32-bit result, the adjustment input must satisfy:

\[
-2{,}147{,}468{,}287
\leq
\text{adjustment input}
\leq
2{,}147{,}483{,}647.
\]

Below the lower bound, the mathematical subtraction falls below the signed 32-bit minimum. The available evidence does not establish wrapping, saturation, or another defined outcome for those inputs.

Any additional requirements imposed by F-d350614e3326 remain unknown. The wrapper’s lack of explicit dereferences does not establish that every opaque input is valid.

## Behavior

### Source-word transformation

Before invoking the dependency, the wrapper computes a retained unsigned 32-bit word:

\[
R =
(\text{lower source word} \gg 11)
\;\vert\;
(\text{upper source word} \ll 21).
\]

The right shift is logical. The left shift has unsigned 32-bit behavior: bits shifted beyond bit 31 are discarded.

The resulting layout is:

| Retained-word bits | Source bits |
|---|---|
| 31–21 | Upper source word bits 10–0 |
| 20–0 | Lower source word bits 31–11 |

The fields do not overlap.

A second transformed word is:

\[
T = \text{upper source word} \gg 11.
\]

It lies between 0 and 2,097,151 inclusive.

The lower source word’s bottom 11 bits do not contribute to either transformed word or any other explicit wrapper computation. Changing only those bits therefore leaves the explicit dependency arguments unchanged. This does not establish identical dependency behavior across separate invocations when external state may differ.

### Dependency invocation and ordering

F-d350614e3326 is invoked once with:

1. The opaque input, unchanged.
2. \(T\), the upper source word shifted logically right by 11.
3. \(R\), the retained word.
4. The adjustment input minus 15,361.

The retained word is established before the invocation and remains the source of the upper return word on the accepted path.

The evidence does not establish a finer evaluation order among independent argument expressions. It does establish the following sequence:

1. The dependency receives the transformed arguments.
2. If it returns normally, the wrapper examines its returned value.
3. The wrapper constructs the accepted result or accesses the fallback object.

If the dependency does not return normally, no subsequent wrapper result is established.

### Branch selection

For a normally returned 64-bit dependency result, define:

- \(L\): the unsigned low 32-bit word, bits 31–0.
- \(H\): the unsigned high 32-bit word, bits 63–32.

The wrapper clears bit 31 of \(L\), adds one using unsigned 32-bit arithmetic, and tests whether the result is strictly less than 2,048:

\[
(L \mathbin{\&} \text{0x7FFFFFFF}) + 1 < 2048.
\]

The masked value is at most 2,147,483,647, so this addition cannot wrap. Acceptance is equivalent to:

\[
L \mathbin{\&} \text{0x7FFFFFFF} \leq 2046.
\]

The accepted low words are exactly:

- 0x00000000 through 0x000007FE.
- 0x80000000 through 0x800007FE.

Thus:

- A masked value of 2,046 is accepted.
- A masked value of 2,047 is rejected.
- Bit 31 of \(L\) does not affect branch selection.
- \(H\) does not affect branch selection.

Branch acceptance does not itself guarantee normal completion of the accepted-result construction: that construction also converts \(H\) to signed 32-bit form.

### Accepted-result construction

The wrapper forms an unsigned 32-bit product:

\[
P = (L \times 2^{20}) \bmod 2^{32}.
\]

It also converts the extracted high word \(H\) to a signed 32-bit integer before adding it to the unsigned product.

Let \(S\) denote the signed value produced by that conversion, **if conversion completes normally**. The subsequent unsigned addition produces:

\[
C = (S + P) \bmod 2^{32}.
\]

The conversion of \(S\) for this unsigned addition is modulo \(2^{32}\). Consequently, once \(S\) is known, the addition does not require signed-overflow semantics.

The returned value is:

\[
R \times 2^{32} + C.
\]

Only the low 32 bits of the addition contribute to \(C\). Any carry out of that word is discarded rather than added to \(R\).

#### High words through 0x7FFFFFFF

For:

\[
0 \leq H \leq \text{0x7FFFFFFF},
\]

the signed conversion is representable, so \(S=H\). The result is therefore established as:

\[
C = (H + L \times 2^{20}) \bmod 2^{32}.
\]

On accepted branches, \(P\) is at most 0x7FE00000. With \(H\) in this representable range, their sum is at most 0xFFDFFFFF and does not overflow an unsigned 32-bit word.

#### High words above 0x7FFFFFFF

For:

\[
\text{0x80000000} \leq H \leq \text{0xFFFFFFFF},
\]

the original value is not representable as a signed 32-bit integer.

The available evidence does not establish the governing conversion rule. In language models where out-of-range unsigned-to-signed conversion is implementation-defined, the conversion may yield an implementation-defined result or raise an implementation-defined signal. A signed destination alone does not establish preservation of the original 32-bit pattern.

Therefore, the unconditional formula using \(H\) directly is not established for this range.

Under the **additional, unverified assumption** that conversion completes normally and yields:

\[
S = H - 2^{32},
\]

the signed value has the same residue modulo \(2^{32}\) as \(H\), and the familiar formula follows:

\[
C = (H + L \times 2^{20}) \bmod 2^{32}.
\]

That assumption is not supplied by the wrapper or dependency mapping.

For any fixed high word whose conversion completes normally with the same result, toggling bit 31 of an accepted low word changes neither branch selection nor the unsigned product. Its contribution disappears modulo \(2^{32}\).

### Rejected-result construction

When the low-word test rejects the dependency result, the wrapper returns the fallback object converted to unsigned 64-bit form.

This path does not perform the accepted-path signed conversion of \(H\). Thus an out-of-range high word does not introduce that conversion uncertainty on the rejected path.

The fallback access follows the dependency invocation and rejection decision at the represented behavioral level. The accepted branch does not require a fallback-object read.

The fallback must not be treated as constant merely because it occupies a fixed stored location. F-d350614e3326 or another component may affect it; preservation is not established.

## Side effects

The wrapper contains no explicit memory write, allocation, release, ownership transfer, or mutation of caller-provided storage.

Its explicit externally relevant actions are:

1. Calling F-d350614e3326.
2. Conditionally reading the fallback object after the call.
3. Returning a value if execution completes normally.

These observations do not establish purity. The dependency may access or modify globals, retain references, allocate or release resources, or fail to return.

No ownership rule follows from forwarding the opaque input. The wrapper does not visibly release it, but the dependency’s borrowing, retention, consumption, or release behavior is unknown.

The wrapper exposes no explicit error-reporting channel apart from its return value. The fallback may represent an error, sentinel, or ordinary value; no interpretation is established. A failure or signal during accepted-path conversion is not shown as being caught or redirected to the fallback path.

The represented ordering does not establish atomicity, volatile access, synchronization, hardware load timing, or concurrency guarantees.

## Dependencies

### F-d350614e3326

The only identified function dependency receives four arguments and supplies a 64-bit result.

Established at this call site:

- The first argument is forwarded unchanged.
- The next two arguments are unsigned 32-bit source transformations.
- The fourth is the adjusted signed input.
- The result is divided numerically into two 32-bit words.

Unknown:

- Argument meanings and validity requirements.
- Internal algorithm and result-field semantics.
- Global-state access and preservation.
- Ownership and resource management.
- Failure, signaling, and termination behavior.

### Stored fallback object

The rejected path depends on a stored object with unavailable declaration and contents.

Its conversion to unsigned 64-bit form is established, but its exact converted value is not. Width and signedness matter; for example, a negative signed value converts differently from a narrower unsigned value with a similar stored bit pattern.

## Acceptance examples

These are conditional mathematical derivations from static evidence, not runtime observations. Unless stated otherwise, each assumes defined adjustment arithmetic and a normally returning dependency.

### Transformed arguments and representable high word

Given:

- Upper source word: 0x00000801.
- Lower source word: 0x00000800.
- Adjustment input: 15,361.

The dependency receives the unchanged opaque input followed by:

- 1.
- 0x00200001.
- 0.

If it returns 0x1234567800000000, the low word is accepted and the high word is representable as signed 32-bit. The wrapper returns:

**0x0020000112345678**

### Upper accepted boundary

With both source words zero, suppose the dependency returns:

**0x00000000000007FE**

The masked low word is 2,046; adding one gives 2,047, which is less than 2,048. The signed conversion of the zero high word is representable.

The return is:

**0x000000007FE00000**

### First rejected boundary

If the dependency’s low word is 0x000007FF, the masked value plus one is 2,048. The wrapper selects the converted fallback value.

A low word of 0x800007FF is rejected identically. The dependency’s high word does not affect this selection, and its accepted-path signed conversion is not performed.

### Ignored top bit with a representable high word

With both source words zero, dependency results:

- 0x0000000000000001.
- 0x0000000080000001.

both produce:

**0x0000000000100000**

Both low words are accepted, both unsigned products are 0x00100000, and both high words are zero.

### Conditional carry-discard example

With both source words zero, suppose the dependency returns:

**0xFFF0000000000001**

The branch accepts the low word. However, the high word 0xFFF00000 is outside the signed 32-bit positive range, so the exact return is not established unconditionally.

If conversion completes normally and yields −1,048,576, the unsigned addition combines 0xFFF00000 with 0x00100000, discards the carry, and returns zero.

Without that conversion assumption, zero is not a required result; conversion behavior remains unresolved, including the possibility of a signal.

### Source-bit boundaries

An upper source word of 0x00000800 contributes 1 to the dependency’s second argument and zero to the retained word.

An upper source word of 0x00000001 contributes zero to the second argument and 0x00200000 to the retained word.

Changing only the lower source word’s bottom 11 bits changes none of the explicit dependency arguments.

### Adjustment boundary

An adjustment input of −2,147,468,287 produces −2,147,483,648, the signed 32-bit minimum.

An input one smaller would require a result below that minimum. No wrapping or saturation result is established.

## Uncertainties

The following remain **documented_only**:

- **Out-of-range high-word conversion:** for high words above 0x7FFFFFFF, neither the signed result nor normal completion is established.
- **Adjustment underflow:** inputs below the stated threshold have no established overflow interpretation.
- **Fallback declaration and value:** exact rejected-path output cannot be fixed.
- **Dependency semantics and failures:** argument meaning, result meaning, termination, and side effects remain unknown.
- **State preservation and ownership:** the dependency is not assumed to preserve the fallback or other globals, or to refrain from retaining or releasing resources.
- **Concurrency:** no stable snapshot, atomicity, or synchronization guarantee is established.
- **Higher-level intent:** a larger numeric representation may motivate the shifts and offset, but none is required by this contract.
- **Physical interface:** register placement, stack layout, and stored byte order are unspecified.

## Evidence

Established static evidence consists of the supplied wrapper body and opaque dependency mapping: a retained-word transformation, one dependency invocation, a masked unsigned comparison, an accepted-path signed conversion and word combination, and a rejected-path fallback conversion.

The bit layout, accepted low-word ranges, adjustment boundary, and qualified example outputs follow mathematically from those operations under the stated 32-bit and 64-bit widths.

No target conversion rule was supplied for high words outside the signed 32-bit range. The unconditional preservation of those high-word bit patterns is therefore not established.

No runtime tests have been performed. No dependency implementation, fallback-object declaration, caller behavior, or external representation specification was available.
