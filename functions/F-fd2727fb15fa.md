<!-- Trusted provenance: F-fd2727fb15fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd2727fb15fa

## Purpose

F-fd2727fb15fa transforms one 32-bit input word into a 32-bit encoded result. It interprets the input’s most significant bit as a sign, forms an unsigned magnitude, obtains a classification value from F-cc732b267cc0, and combines:

- The original sign in result bit 31.
- A dependency-derived field in result bits 30–23.
- A shifted-magnitude field in result bits 22–0.

The established behavior is a bit-level transformation. Its result has the field layout of a binary32 floating-point representation, but the available evidence does not establish a numerical conversion contract. In particular, the meaning of the dependency’s classification value is unavailable, and this function contains no explicit zero handling or rounding adjustment.

The contract below distinguishes operations established by the supplied static representation from interpretations and unresolved execution details. No runtime tests have been performed.

## Interface

**Input:** one pointer to a readable 32-bit word.

**Output:** one unsigned 32-bit word, returned by value.

**Direct input access:** the function reads the pointed-to word once, before invoking its dependency. The source does not show a second read through that pointer.

**Direct output access:** no caller-provided output buffer is used. The function performs no explicit write through the input pointer.

The arithmetic and bit layout described here use 32-bit words. Signed interpretation of the input is used only to determine whether bit 31 is set. Magnitude formation and subsequent arithmetic operate on unsigned 32-bit values.

The input pointer must support the initial 32-bit read under the execution environment’s memory-access rules. The function does not check for null, inaccessible storage, insufficient object size, or unsuitable alignment. The exact alignment requirements, byte order, pointer width, and calling convention are not established by the supplied representation.

The input need not remain unchanged after its initial read for this function’s local computation to remain stable: both sign and magnitude are derived from the captured word. This does not establish any broader concurrency guarantee or make an otherwise invalid concurrent access permissible.

No ownership transfer is expressed. The function itself does not allocate, retain, or release the input storage.

## Behavior

### Initial read and sign interpretation

The first observable operation represented is a read of the input word.

Let \(x\) denote that captured 32-bit word. Its sign indicator is:

- Zero when bit 31 of \(x\) is clear.
- One when bit 31 of \(x\) is set.

The source determines this indicator through a signed 32-bit interpretation. Under the represented two’s-complement interpretation, the test is equivalent to inspecting bit 31.

An unsigned magnitude \(m\) is then formed:

- For a clear sign bit, \(m=x\).
- For a set sign bit, \(m=(-x)\bmod 2^{32}\).

This is unsigned modular negation, not signed absolute-value arithmetic. Consequently, the input word `0x80000000` produces magnitude `0x80000000`; it does not require representing positive \(2^{31}\) as a signed 32-bit integer.

The possible magnitudes are from zero through \(2^{31}\), inclusive. The function preserves the original sign indicator independently of subsequent dependency behavior.

### Dependency invocation

After magnitude formation, the function calls F-cc732b267cc0 exactly once, passing \(m\) as its sole represented argument.

Let \(r\) denote the unsigned 32-bit value returned by that dependency.

There is no visible validation of \(r\), no explicit error-result test, and no alternate path for a dependency-defined failure indication. Any returned value is used as a classification value.

The dependency receives the magnitude by value, not the original input pointer. Nevertheless, its implementation is unknown. It must not be assumed to preserve globals, leave caller storage untouched through other aliases, avoid allocation, or return normally.

If the dependency does not return normally, the supplied representation establishes no returned result from this function.

### Magnitude-field construction

The function selects a shift direction using an unsigned comparison of \(r\) with 24.

For \(r<24\), it shifts \(m\) left by \(23-r\) positions. The represented operation also masks the count to eight bits, but this does not change the count in this branch: it is already between zero and 23.

For \(r\geq24\), it forms the effective shift count

\[
s=(r-23)\bmod256
\]

and shifts \(m\) right by \(s\) positions. The right shift is unsigned, so defined executions shift in zero bits.

Only the low 23 bits of the shifted value contribute to the returned word. Any higher bits of that intermediate value are discarded during result assembly.

The distinction between the eight-bit count reduction and the operand width matters. Masking the count to eight bits does not guarantee a valid shift of a 32-bit operand:

- Counts from zero through 31 have an ordinary 32-bit shift interpretation.
- Counts from 32 through 255 do not have a defined result under ordinary C-like shift semantics.
- The supplied representation alone does not establish whether the underlying machine masks such counts further, implements another behavior, or relies on a dependency return-range invariant that makes them unreachable.

Accordingly, exact results for the latter cases are documented_only rather than assigned a guessed machine-specific interpretation.

### Exponent-like field construction

Independently of the magnitude-field selection, the function derives an eight-bit field from \(r\):

\[
e=(r+111)\bmod256.
\]

This field occupies result bits 30–23.

The represented calculation uses unsigned 32-bit addition and multiplication, followed by a mask that excludes bit 31. Its observable contribution is equivalent to retaining the low eight bits of \(r+111\) and positioning them at bit 23.

There is no saturation or range check. If the sum exceeds the eight-bit field range, the field wraps. The function does not explicitly handle exponent-like underflow, overflow, reserved encodings, or special floating-point values.

### Result assembly

For a defined shift execution, let \(q\) be the selected shifted magnitude. The result consists of three nonoverlapping fields:

| Result bits | Width | Contents |
|---|---:|---|
| 31 | 1 bit | Original input sign indicator |
| 30–23 | 8 bits | \((r+111)\bmod256\) |
| 22–0 | 23 bits | \(q\bmod2^{23}\) |

The three fields are combined by bitwise OR.

The magnitude field cannot modify the sign or exponent-like field because it is masked to 23 bits. The exponent-like contribution cannot modify the sign because its high bit is excluded.

There is no explicit zero fast path, no normalization check after the dependency returns, and no adjustment based on discarded right-shift bits. No rounding increment is performed by this function.

## Side effects

The directly established memory behavior is limited to one initial 32-bit input read. The function contains no explicit memory write, allocation, deallocation, or global access of its own.

The dependency call occurs after that read and before result construction completes. Any observable effects of F-cc732b267cc0 therefore occur between those stages.

The following distinctions are important:

- A dependency-side mutation of the original input location does not trigger a reread by this function.
- Such a mutation could still be observable to the caller.
- No claim is made that the dependency preserves global state or unrelated memory.
- No ownership or memory-release behavior can be inferred for the dependency merely because its represented argument is an integer.

The function returns a word by value. It does not expose an address to its captured input, magnitude, or intermediate result.

No explicit error-reporting mechanism is present. Invalid input memory access, dependency failure, and an out-of-range shift are not converted into a status code by any visible branch. Their execution consequences remain dependent on the surrounding runtime and machine semantics.

## Dependencies

### F-cc732b267cc0

**Established interface at this call site:** accepts one unsigned 32-bit magnitude and returns a value consumed as an unsigned 32-bit classification.

**Established use:** its return value controls both shift direction/count and the eight-bit exponent-like field.

**Not established:**

- Its mathematical meaning.
- Its return value for zero.
- Its return range for any input.
- Whether it returns a highest-set-bit position, bit count, or another classification.
- Whether it changes globals or other accessible memory.
- Whether it allocates, retains, or releases resources.
- Whether it can fail, trap, or fail to return.

A highest-set-bit interpretation would explain the magnitude-alignment structure for nonzero values. That is inferred intent, not a dependency guarantee.

No other function calls are visible in the supplied representation.

## Acceptance examples

These examples are conditional bit-level expectations, not runtime observations. Each dependency return value is an explicit assumption, not a claim about F-cc732b267cc0.

| Captured input | Dependency receives | Assumed dependency return | Expected result |
|---|---|---:|---|
| `0x00000001` | `0x00000001` | 0 | `0x37800000` |
| `0xffffffff` | `0x00000001` | 0 | `0xb7800000` |
| `0x00800000` | `0x00800000` | 23 | `0x43000000` |
| `0x01000003` | `0x01000003` | 24 | `0x43800001` |
| `0x80000000` | `0x80000000` | 31 | `0xc7000000` |
| `0x00000000` | `0x00000000` | 0 | `0x37800000` |
| `0x00000001` | `0x00000001` | 279 | `0x43000001` |

The examples establish the following conditional acceptance criteria:

1. **Sign and magnitude separation.** With input `0xffffffff`, the dependency receives magnitude one, while the returned word retains a set sign bit.

2. **Zero-distance left shift.** With classification 23, the left-shift count is zero. For magnitude `0x00800000`, masking removes bit 23 from the magnitude contribution.

3. **Discard without rounding.** With magnitude `0x01000003` and classification 24, the right shift produces `0x00800001`. The discarded low bit causes no increment, yielding `0x43800001`.

4. **Minimum signed input.** Input `0x80000000` remains the same unsigned magnitude after modular negation. Given classification 31, the right shift is eight positions and the result is `0xc7000000`.

5. **No intrinsic zero special case.** If the dependency returns zero for input zero, the result is `0x37800000`, not an all-zero word. A numerical zero contract therefore requires additional dependency evidence.

6. **Eight-bit shift-count reduction.** Classification 279 produces a right-shift count of zero because \(279-23=256\). Its exponent-like field wraps to 134. This is an established consequence of the represented arithmetic, not an assertion that this classification is reachable.

7. **Unresolved oversized shift.** Classification 55 produces an effective right-shift count of 32. No exact returned word is specified for this case without additional execution semantics or proof that the case cannot occur.

If the dependency changes the pointed-to input after receiving its argument and then returns normally, the result must still use the originally captured sign and magnitude, subject to the same shift qualifications.

## Uncertainties

**documented_only — dependency contract.** The meaning and range of the classification value are unknown. Exact unconditional mappings from input words to output words cannot be established without that contract.

**documented_only — oversized shifts.** Effective right-shift counts above 31 require additional machine-level evidence or a dependency invariant. They must not silently be treated as counts reduced modulo 32.

**documented_only — memory and ABI details.** The representation establishes a 32-bit load, but not byte order, alignment behavior, pointer width, or external calling convention.

**documented_only — dependency side effects and failures.** Global preservation, resource ownership, and abnormal-return behavior are unspecified.

**Inferred intent — numeric encoding.** The field arrangement suggests floating-point-style encoding. If the dependency returns the zero-based position of the highest set bit for nonzero magnitudes, the alignment and bias would be consistent with encoding a signed value scaled by \(2^{-16}\), with discarded low bits rather than a rounding increment. Neither that helper interpretation nor that numerical interface is established.

## Evidence

The supplied static representation establishes one input-word read, sign extraction through a signed interpretation, unsigned modular magnitude formation, one dependency call, a classification-controlled shift, and assembly of three masked fields.

All dependency return values in the acceptance examples are stipulated conditions. No helper implementation, machine-instruction evidence, external interface documentation, or runtime observation was provided.

No runtime tests have been performed. The exact behavior for invalid memory accesses, dependency failures, and oversized shifts remains documented_only rather than experimentally verified.
