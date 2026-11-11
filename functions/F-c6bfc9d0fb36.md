<!-- Trusted provenance: F-c6bfc9d0fb36; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6bfc9d0fb36

## Purpose

F-c6bfc9d0fb36 consumes four 32-bit words and additional incoming state. Its visible behavior has three outcomes:

- Return the first input word unchanged.
- Construct a 32-bit status-like result from word comparisons and an incoming byte.
- Transfer control to an indirect dependency, with branch-dependent arguments, and return the result attributed to that transfer.

The paired-word layout resembles two binary64 encodings. This is inferred intent, not a requirement to interpret the inputs as language-level floating-point values. The established behavior is defined through bit fields, unsigned comparisons, signed interpretations, and fixed-width arithmetic.

This function is not established to implement a general floating-point comparison. In particular, when neither input pair has an all-ones exponent-like field, it returns the first input word without constructing comparison status.

## Interface

### Explicit inputs

| Position | Contract name | Width | Visible role |
|---|---|---|---|
| 1 | A-low | 32 bits | First pair’s low-order word; unchanged-return value |
| 2 | A-high | 32 bits | First pair’s high-order word |
| 3 | B-low | 32 bits | Second pair’s low-order word |
| 4 | B-high | 32 bits | Second pair’s high-order word |

All four inputs are unsigned words. “High” and “low” describe logical significance, not memory byte order. The function does not visibly dereference these inputs.

Each pair is classified using:

- **Sign:** bit 31 of the high word.
- **Exponent-like field:** bits 30 through 20 of the high word.
- **Fraction-like field:** bits 19 through 0 of the high word together with all 32 bits of the low word.

The following encoding classes are used below:

- **Ordinary:** the exponent-like field is not all ones.
- **Infinity encoding:** the exponent-like field is all ones and the fraction-like field is zero.
- **NaN-like encoding:** the exponent-like field is all ones and the fraction-like field is nonzero.

These classes describe bit patterns. They do not establish floating-point exception, rounding, signaling-NaN, or payload-propagation semantics.

### Additional incoming state

The visible representation also consumes:

- **Q-byte:** an incoming unsigned 8-bit value.
- **Return-link word:** an incoming 32-bit value from which the indirect target is obtained by clearing bits 0 and 1.

Neither value’s origin nor its permitted range beyond its visible width is established. In particular, Q-byte must not be restricted to zero or one.

### External data

Four symbolic external locations participate in exceptional dispatch:

| Contract name | Visible use |
|---|---|
| Control | Word-sized mode and selector reads |
| Data-1 | Argument source in several dispatch branches |
| Data-2 | Initial first-argument source in one branch |
| Data-3 | Replacement second-argument source in that branch |

The visible argument values are 32-bit words. Physical access widths, storage classes, address relationships, and synchronization properties are not independently established.

### Output and arithmetic

The visible result is one unsigned 32-bit word.

Word arithmetic uses 32-bit truncation. Addition, subtraction, multiplication by two, complement, and left shift retain only the low 32 bits. Signed predicates interpret those same bit patterns as two’s-complement signed values.

A conversion to byte retains the low eight bits. Widening that byte and shifting it left by 27 produces a 32-bit word; only byte bits 0 through 4 survive.

No success/error result convention is established.

## Behavior

### Initial classification and unchanged return

The first gate depends only on A-high and B-high.

Its signed tests and wrapping arithmetic collectively establish:

- If both pairs are ordinary, return A-low unchanged.
- If either pair has an all-ones exponent-like field, continue.

The low words do not affect this gate. Neither Q-byte nor external data is consulted.

Thus ordinary zeros, subnormal-like encodings, and ordinary encodings of either sign all take the unchanged-return path when the other pair is also ordinary. The returned word preserves every bit of A-low; it is not restricted to status-like bit positions.

### Classification after the initial gate

The next classification discards each high word’s sign by doubling it at 32-bit width and compares the resulting magnitude-like word against `0xFFE00000`. Low words distinguish an infinity encoding from a NaN-like encoding when the doubled high word equals that threshold.

The established outcome is:

- If either pair is NaN-like, enter exceptional dispatch.
- Otherwise, construct a comparison result.

Consequently, every comparison-result path has at least one infinity encoding. The other pair is ordinary or another infinity encoding.

A high word of `0x7FF00000` or `0xFFF00000` is not sufficient by itself to establish an infinity encoding: its corresponding low word must also be zero.

### Comparison with at least one sign bit set

This branch applies when either A-high or B-high has bit 31 set, including mixed-sign inputs.

Define two predicates:

- **Equal:** A-high equals B-high and A-low equals B-low.
- **Carry-like:** B-high is less than A-high under unsigned comparison, or the high words are equal and B-low is less than or equal to A-low under unsigned comparison.

The result has these bit contributions:

| Result bit | Set when |
|---|---|
| 31 | Carry-like is false |
| 30 | Equal is true or Q-byte bit 3 is set |
| 29 | Carry-like is true or Q-byte bit 2 is set |
| 28 | Never |
| 27 | Q-byte bit 0 is set |
| 26 through 0 | Never |

Q-byte bit 1 is discarded. Q-byte bit 4 cannot set result bit 31 on this branch. Q-byte bits above bit 4 also have no effect.

The comparison is unsigned high-word order followed by unsigned low-word order for a high-word tie. Mathematical floating-point ordering must not replace it, even for mixed-sign inputs.

### Comparison with neither sign bit set

When both sign bits are clear, choose the comparison words as follows:

- If the high words differ, use B-high and A-high.
- If the high words match, use B-low and A-low.

For the selected words, define:

- **Carry-like:** the selected B word is greater than or equal to the selected A word, unsigned.
- **Difference:** selected B word minus selected A word, reduced modulo \(2^{32}\).
- **Overflow:** that subtraction is outside the signed 32-bit range when both selected operands are interpreted as signed.
- **Equal:** both original word pairs match exactly.

If Overflow is false, construct a byte by ORing Q-byte with:

- Bit 4 if Difference has bit 31 set.
- Bit 3 if Equal is true.
- Bit 2 if Carry-like is true.

If Overflow is true, construct a byte by ORing Q-byte with:

- Bit 4 if Carry-like is false.
- Bit 2 if Carry-like is true.
- No equality contribution.

Widen the resulting byte and shift it left by 27 at 32-bit width.

Result bits 0 through 26 are therefore zero. Q-byte bits 0 through 4 map to result bits 27 through 31 and can force those bits to one regardless of the comparison contributions. Higher Q-byte bits are discarded by the final shift.

The overflow case’s absent equality contribution follows from the visible expression. It is also consistent with the fact that equal selected operands cannot produce subtraction overflow.

### Exceptional dispatch: initial mode selection

Exceptional dispatch occurs for any NaN-like pair. All branches use the indirect dependency F-000000000001.

Each Control reference described below is a separate logical read. Later reads are not assumed to equal earlier reads.

First, read Control and test mask `0x000D0000`.

If the masked value is nonzero, proceed directly to common dispatch.

If it is zero, read Control again and select its low four bits.

#### Selector 10

- The first transfer argument is initially zero.
- Read Control again and test bit `0x40`.
- If that bit is set, the first argument becomes `0x80000000`.
- Read Data-1 for the second argument.
- Transfer with those two explicit arguments and return the visible dependency result.

#### Selector 8

Read Control again and test bit `0x40`.

If the bit is set, transfer with two explicit zero arguments and return the visible dependency result. No data-location read is shown on this subpath.

If the bit is clear:

1. Read Data-2 as the initial first argument.
2. Read Data-1 as the initial second argument.
3. Read Control again and test bit `0x10`.
4. If that bit is set, replace the first argument with zero and read Data-3 as the replacement second argument.
5. Transfer with the resulting two arguments and return the visible dependency result.

The initial Data-2 and Data-1 reads occur even when their values are subsequently replaced.

#### Selector 9

Set the working third-input value to 8. Read Data-1 and use it as the working fourth-input value. Then continue to common dispatch.

These changes affect local working values used as later arguments. They are not writes to caller-owned input storage.

#### Any other selector

Transfer with no explicit arguments shown by the representation and return the visible dependency result.

This does not establish a zero-argument machine-level interface; hidden or residual incoming state is unresolved.

### Common dispatch

Common dispatch is reached either directly from the initial nonzero mask test or after selector 9.

The working third and fourth values are B-low and B-high unless selector 9 replaced them.

First, read Control and test bit `0x00100000`.

- If clear, transfer with one explicit zero argument and return the visible result.
- If set, read Control again and test mask `0x00070000`.

If this second masked value is zero, transfer with three explicit arguments, in order:

1. `0x30000000`.
2. The working fourth-input value.
3. `0x30000000`, with bit 27 additionally set if Q-byte bit 0 is set.

The third argument is therefore either `0x30000000` or `0x38000000`. No other Q-byte bit contributes.

If the second masked value is nonzero, read Control once more. Transfer with two explicit arguments:

1. The working third-input value.
2. A transformed value of this newly read Control word.

The transformation’s operation order is significant:

- Shift the newly read Control word left by 15 at 32-bit width.
- Complement that 32-bit intermediate value.
- Shift the complemented value left by one at 32-bit width.

Equivalently, the second argument has this layout:

| Argument bits | Value |
|---|---|
| 31 through 16 | Complement of the newly read Control word’s low 16 bits |
| 15 through 1 | All ones |
| 0 | Zero |

In arithmetic form, if \(L\) is the unsigned low-16-bit value of the newly read Control word, the argument is

\[
(65535-L)\times 65536 + 65534.
\]

Complementing Control before both shifts would produce a different value and is not this contract. In particular, the second argument’s low 16 bits are always `0xFFFE`, not zero.

## Side effects

The unchanged-return and comparison-result paths show no external-data reads, external-memory writes, or dependency transfers.

Exceptional paths perform the conditional reads specified above and show at most one indirect transfer on a completed path. Where the dependency returns through the represented result mechanism, that 32-bit result is returned unchanged.

There are no explicit external-memory writes in the supplied body. No allocation, release, buffer access, or ownership transfer is visible.

These local observations do not constrain dependency behavior. F-000000000001 may modify external state, interpret arguments as addresses or handles, allocate or release resources, fail, or not return; none of those behaviors is resolved here.

A status-like numeric result does not establish a write to architectural flags or a processor status register.

No local validation of the indirect target or external storage is visible. Valid accessible storage and a meaningful target are prerequisites for normal execution of paths that use them, but precise fault behavior is undocumented.

## Dependencies

### F-000000000001

This is a contract-local opaque identifier for the unresolved indirect dependency, not an assertion of a known implementation identity.

Its target is the incoming return-link word with bits 0 and 1 cleared. Every visible dispatch variant uses that target.

Depending on the branch, the representation supplies zero, one, two, or three explicit arguments. Their values and order are specified in Behavior.

The following remain undocumented:

- Complete machine-level argument interface and hidden inputs.
- Register, flag, global, and memory preservation.
- Ownership or lifetime rules for argument values.
- Whether the transfer is an ordinary returning call, a continuation, or another ABI-specific operation.
- Return behavior beyond the represented 32-bit result.

## Acceptance examples

These are conditional expectations derived from static evidence. No runtime tests have been performed.

### Ordinary pairs return A-low

With A-low `0x12345678`, A-high `0x3FF00000`, B-low zero, and B-high `0x40000000`, the result is `0x12345678`.

Q-byte and external data do not participate. Setting an ordinary pair’s sign bit does not change the unchanged-return rule while both exponent-like fields remain ordinary.

### Matching infinity encodings

With both high words `0x7FF00000`, both low words zero, and Q-byte zero, Equal and Carry-like are true. The result is `0x60000000`.

Replacing both high words with `0xFFF00000` also produces `0x60000000`.

With Q-byte `0x02`:

- Matching positive infinity encodings produce `0x70000000`.
- Matching negative infinity encodings still produce `0x60000000`, because the sign-set branch discards Q-byte bit 1.

Neither case reads external data.

### Mixed signs follow unsigned word order

With A-high `0xFFF00000`, B-high `0x7FF00000`, both low words zero, and Q-byte zero, Carry-like is true and Equal is false. The result is `0x20000000`.

Swapping the pairs makes Carry-like false and produces `0x80000000`.

### A nonzero low fraction selects dispatch

A-high `0x7FF00000` with A-low 1 is NaN-like.

If Control reads consistently yield selector 10, initial mask `0x000D0000` clear, and bit `0x40` clear, the dependency receives zero followed by Data-1. Any returned word becomes the function result.

### Selector 8 retains replaced reads

For NaN-like input, suppose Control consistently has selector 8, initial mask clear, bit `0x40` clear, and bit `0x10` set.

After the selector and `0x40` checks, the read sequence includes Data-2, Data-1, the later Control read, and Data-3. The final arguments are zero and Data-3.

### Selector 9 changes common-dispatch arguments

If selector 9 is selected and subsequent Control reads choose the three-argument common branch, the arguments are:

1. `0x30000000`.
2. The Data-1 value read during selector 9.
3. `0x30000000` or `0x38000000`, according to Q-byte bit 0.

The second argument is not the original B-high.

### Final two-argument transformation

For NaN-like input, suppose every Control read yields `0x00110000`.

The initial mask is nonzero, bit `0x00100000` is set, and mask `0x00070000` is nonzero. The final two-argument branch receives:

1. Original B-low.
2. `0xFFFFFFFE`.

The second argument is not `0xFFFF0000`.

As another conditional case, a stable Control value of `0x00120009` selects selector 9 and then the final two-argument branch. Its arguments are 8 and `0xFFF6FFFE`. The Data-1 read required by selector 9 still occurs, although its resulting working fourth-input value is unused by that transfer.

## Uncertainties

- **documented_only — Intended operation:** Binary64-style classification and status construction are inferred intent, not a complete floating-point API.
- **documented_only — Incoming state:** Q-byte and the return-link word have unresolved origins and validity constraints.
- **documented_only — External stability:** Separate reads may differ. Snapshot semantics, volatility, concurrency guarantees, and synchronization are not established.
- **documented_only — Physical accesses:** Logical read order is visible; exact machine access widths and bus-level ordering are not independently established.
- **documented_only — Indirect transfer:** Target validity, complete calling convention, hidden arguments, and normal return are unresolved.
- **documented_only — Failures:** No local recovery or explicit failure result is established for inaccessible storage, invalid targets, or dependency failures.
- **documented_only — Ownership:** No dependency ownership or resource-lifetime policy can be inferred.
- **documented_only — Architectural state:** Numeric status construction does not establish preservation or modification of processor flags or other hidden state.

## Evidence

The supplied static representation establishes four explicit word inputs, a word result, incoming byte and return-link state, conditional external-data references, bitwise status construction, and indirect transfers.

The simplified classification rules and comparison predicates are arithmetic deductions from the visible fixed-width operations. They distinguish unsigned comparisons from signed tests and retain wraparound behavior.

The final two-argument transformation follows specifically from shifting Control left by 15, complementing the resulting 32-bit intermediate, and then shifting left by one. The first shift creates 15 low zero bits; complement makes them ones; the final shift places ones in bits 15 through 1 and clears bit 0.

Encoding terminology and status terminology express inferred intent only. No implementation execution, runtime observation, or runtime test evidence is available.
