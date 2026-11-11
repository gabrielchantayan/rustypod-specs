<!-- Trusted provenance: F-75a3efca3d72; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-75a3efca3d72

## Purpose

F-75a3efca3d72 selects a 64-bit result using a caller-supplied object’s status. Its observable behavior has three principal paths:

- Return two stored 32-bit words.
- Transform a captured 64-bit numeric payload, subject to a processor-condition gate and exceptional dispatch.
- Mark the object and attempt a sequence of three dependent operations.

If no path is selected, or either of the first two fallback operations reports a nonzero result, the function returns zero.

**Established static evidence:** access widths and offsets, branch precedence, explicit status mutation, dependency ordering, several exact results, and the bit-level transformation described below.

**Inferred intent:** the numeric path resembles conversion of a floating-point representation to a signed 64-bit integer. That interpretation does not establish a standard floating-point format, rounding contract, or portable treatment of exceptional values.

**documented_only:** processor-condition linkage, oversized word shifts, exceptional transfer destinations, and the successful fallback’s returned-data relationship remain unresolved.

## Interface

### Arguments and result

The supplied interface has four arguments:

| Argument | Established role |
|---|---|
| Object address | Identifies storage containing status, stored result, numeric payload, and a fallback argument |
| Second argument, 32 bits | No visible use |
| Third argument, 32 bits | Referenced as the low return word at the successful fallback’s shared return |
| Fourth argument, 32 bits | Passed to the first fallback dependency and referenced as the high word at that shared return |

The result is a 64-bit bit pattern. Combining two return words places the low word in bits 0–31 and the high word in bits 32–63. A signed interpretation is useful for the numeric path but is not established for every caller.

The second argument has no visible effect. The representation does not establish its validation requirements or any hidden ABI role.

### Object layout

Offsets are in bytes from the object address.

| Offset | Width | Use |
|---:|---:|---|
| 0 | 32 bits | Stored result, low word |
| 4 | 32 bits | Stored result, high word |
| 8 | 32 bits | Numeric payload, low word |
| 12 | 32 bits | Numeric payload, high word |
| 20 | 32 bits | Late-read argument to the final fallback dependency |
| 28 | 16 bits | Status selection and fallback marking |

The status access is 16-bit, including its write. No full-width access to a 32-bit field at offset 28 is established.

### Preconditions and ownership

The object must support the accesses selected by the active path. A status-only return requires the status read; the stored-result path additionally requires offsets 0 and 4; numeric processing requires offsets 8 and 12. Fallback requires writable status storage and, if both preliminary dependencies return zero, readable storage at offset 20.

There is no visible null check, bounds check, alignment correction, allocation, or release.

Dependencies receive either values, the object address, or an address of local storage. Their ownership rights, pointer retention, resource release, and additional memory accesses are not supplied. The local storage’s size, initialization, and output layout are also unresolved.

## Behavior

### Selection and precedence

The first explicit object access reads the status at offset 28. Selection uses that captured 16-bit value:

| Priority | Condition | Behavior |
|---:|---|---|
| 1 | Bit `0x0004` set | Return stored result |
| 2 | Otherwise, bit `0x0008` set | Numeric processing |
| 3 | Otherwise, any bit in `0x0012` set | Stateful fallback |
| 4 | Otherwise | Return zero |

Thus stored-result selection overrides numeric and fallback selection. Numeric selection overrides fallback selection. Unrelated status bits are not independently interpreted by the visible function.

### Stored result

The function reads offset 0, then offset 4, and returns them as the low and high words respectively.

This path has no visible dependency call or object write. It does not visibly inspect the numeric payload or offset 20.

### Numeric admission gate

Numeric processing captures the payload by reading offset 8 followed by offset 12. It then calls F-11ed8242c40c with those two words and two external 32-bit values.

The subsequent gate uses a processor carry condition:

- If carry is observed set at the first decision, F-c6bfc9d0fb36 is called with the captured payload words.
- If carry is observed clear at the final gate, the function returns `0x8000000000000000`.
- Otherwise, numeric transformation continues.

The supplied representation does not connect these observations to ordinary Boolean dependency results. In particular, it does not establish that a dependency preserves carry, nor which numeric predicate carry represents.

The payload is not visibly reread after these calls. The subsequent transformation refers to the captured words, not newly observed object fields. The order of reads of the two external comparison values is not established.

### Numeric transformation domain

The following describes a bit-pattern contract, not a standard numeric conversion.

A **signed header** is obtained by interpreting the payload high word as signed 32-bit and applying an arithmetic right shift by 20. Its range is −2048 through 2047.

A **normalized pair** has this layout:

| Normalized bits | Content |
|---|---|
| 0–10 | Zero |
| 11–62 | Original payload bits 0–51, preserving order |
| 63 | Zero when the signed header is 0 or −2048; one otherwise |

These are two 32-bit working words. Constructing them does not write to the object.

Eligibility for direct transformation is:

| Signed header | Direct transformation |
|---|---|
| −2048 through −963 | Eligible; negative result treatment |
| −962 | Eligible only when the normalized pair is exactly `0x8000000000000000` |
| −961 through −1 | Not eligible |
| 0 through 1085 | Eligible; nonnegative result treatment |
| 1086 through 2047 | Not eligible |

For eligible negative headers, the requested right-shift distance is −962 minus the signed header. For eligible nonnegative headers, it is 1086 minus the signed header. A distance greater than 79 is replaced with 80.

Consequently, ordinary eligible cases use a distance from 1 through 80. The special eligible header −962 uses distance zero.

### Direct-result semantics and width limitations

The direct transformation operates on individual 32-bit words rather than an established portable 64-bit shift.

For distances 1–31, its observable result is unambiguous:

- The shifted high word is the normalized high word shifted logically right by the distance.
- The shifted low word contains the normalized low word shifted logically right, with the vacated upper positions supplied from the normalized high word.

Together, those words are the normalized 64-bit pattern shifted logically right by the distance.

At other distances, the visible operations require additional target semantics:

| Distance | Contributions to the shifted low word | Shifted high word |
|---:|---|---|
| 0 | Normalized low word, combined by bitwise OR with the normalized high word shifted left by 32 | Normalized high word |
| 32 | Normalized low word shifted right by 32, combined by bitwise OR with the normalized high word | Normalized high word shifted right by 32 |
| 33–80 | Normalized low word shifted right by the distance, combined by bitwise OR with the normalized high word shifted right by distance minus 32 | Normalized high word shifted right by the distance |

Each shift is a word operation. The displayed shift counts are reduced to eight bits, not five. The supplied material does not establish whether shifts by 32 or more produce zero, wrap their counts, or follow another target rule. Treating this entire operation as a conventional 64-bit shift would therefore overstate the evidence.

For the nonnegative branch, the shifted pair is returned directly.

For the negative branch, the shifted pair is negated modulo \(2^{64}\). Equivalently, the low return word is its 32-bit negation; the high return word is the negation of the shifted high word **minus one when the shifted low word is nonzero**. This borrow is significant: adding one to the already-negated high word would describe different behavior.

### Endpoint selection

If direct transformation is not eligible, the function classifies the original payload.

The classification value is twice the original high word plus one if the original low word is nonzero, reduced to 32 bits. Comparison with `0xFFE00001` is unsigned.

For a classification value below that threshold:

| Original high word | Return |
|---|---|
| Sign bit clear | `0x7FFFFFFFFFFFFFFF` |
| Sign bit set | `0x8000000000000000` |

An additional comparison with `0xFFE00000` is visible, but both outcomes return the same sign-selected endpoint.

Classification values at or above `0xFFE00001` select exceptional dispatch. These classification rules apply only after direct transformation has been bypassed.

### Exceptional dispatch

Exceptional dispatch reads an external 32-bit control value. Its low four bits select cases 8, 9, 10, or a default case.

The representation shows indirect transfers to a destination derived from link-related state with its lowest two bits cleared. It does not establish a reliable ordinary-call interface or identify the destination as a supplied dependency.

The apparent argument patterns are:

| Selected case | Additional control condition | Apparent arguments |
|---|---|---|
| 10 | Bit `0x40` clear | One 32-bit zero |
| 10 | Bit `0x40` set | One word, `0x80000000` |
| 8 | Bit `0x40` set | Two zero words |
| 8 | Bit `0x40` clear, bit `0x10` clear | Two external words |
| 8 | Bit `0x40` clear, bit `0x10` set | Zero and an alternate external word |
| 9 | Bit `0x00100000` clear | Zero and the second external word used by case 8’s default pair |
| 9 | Bit `0x00100000` set | The word 8 and a transformed control word |
| Other | — | No explicit arguments shown |

The transformed case-9 word is formed by shifting a 32-bit control observation left by 15, complementing that 32-bit result, and shifting left by one, retaining 32-bit width.

There is an observable-read distinction in case 8: when bit `0x40` is clear, both default external words are read before the bit-`0x10` decision. If bit `0x10` is then set, those argument choices are replaced by zero and a subsequently read alternate word. The initial external reads are therefore not absent merely because their values are not ultimately passed.

Control references are not established as a single snapshot. The case selector, later flag decisions, and case-9 transformation may reference the control storage separately. Volatility, atomicity, and consistency across those references are unknown.

The apparent 64-bit transfer result is returned. Actual destinations, argument conventions, effects, and return semantics remain **documented_only**.

### Stateful fallback

Fallback first writes the captured initial status OR `0x0002` to offset 28 as a 16-bit value. This write occurs even when bit `0x0002` was already set.

The dependency sequence is short-circuited:

1. F-98085658bfd4 receives the object address, the 32-bit value 1, the captured initial status, and the incoming fourth argument.
2. A nonzero result ends the path with return zero. A zero result permits F-94f17d1d24a0, called with the object address.
3. A nonzero result from that dependency also produces return zero.
4. Only after both results are zero does the function read offset 20 and pass that current word, together with an address of local storage, to F-1e06ba66d3c3.

There is no visible rollback of the status write. The status passed to the first dependency is the captured pre-write value.

The late offset-20 read allows earlier dependencies’ mutations of that field to affect the final call.

After the final dependency, execution reaches a shared return expression referencing the third and fourth arguments as low and high words. No explicit assignment connecting the local output storage to those words is shown. The successful fallback’s actual returned-data relationship is therefore **documented_only**: neither “always returns the incoming pair” nor “returns a value produced in local storage” is established as a complete runtime contract.

## Side effects

Directly visible effects are:

- Branch-dependent object reads.
- One fallback status write, before any fallback dependency.
- Calls to the five named opaque dependencies.
- External-data reads during numeric admission and exceptional dispatch.
- Exceptional indirect transfers with unresolved destinations.

There is no direct visible write to the stored-result fields or numeric payload.

This does not establish purity elsewhere. Dependencies and exceptional transfers may modify object storage, globals, processor conditions, resource ownership, or other memory. Their preservation behavior is not supplied.

No explicit cleanup, error object, rollback operation, or separate error-status return is shown. Zero and `0x8000000000000000` occur as ordinary visible return patterns without a separate discriminator.

## Dependencies

| Opaque ID | Established use | Unresolved contract |
|---|---|---|
| F-11ed8242c40c | First numeric operation, receiving captured payload and two external words | Numeric meaning, side effects, condition-state effects |
| F-c6bfc9d0fb36 | Carry-conditional numeric operation on captured payload | Meaning, effects, subsequent carry behavior |
| F-98085658bfd4 | First fallback operation after marking status | Meaning beyond zero/nonzero control, mutations, ownership |
| F-94f17d1d24a0 | Second fallback operation after a zero first result | Meaning beyond zero/nonzero control, mutations, ownership |
| F-1e06ba66d3c3 | Final fallback operation using the late-read field and local storage | Output layout, retained pointers, ownership, relationship to returned words |

No supplied opaque function ID identifies the exceptional transfer destination. Its identity must remain unresolved rather than be replaced with an invented dependency.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Stored-result precedence.** Status `0x001C`, stored low word `0x89ABCDEF`, and stored high word `0x01234567` produce `0x0123456789ABCDEF`. Neither numeric nor fallback dependencies are selected.

2. **No selected path.** Status zero produces zero without visibly reading offsets 0, 4, 8, 12, or 20.

3. **Fallback marking before failure.** Initial status `0x0010` causes a 16-bit write of `0x0012` before F-98085658bfd4. If that dependency returns nonzero, the result is zero and neither later fallback dependency is called.

4. **Mark already present.** Initial status `0x0002` still causes the status write. The first dependency receives initial status `0x0002`.

5. **Second fallback failure.** A zero first result followed by a nonzero F-94f17d1d24a0 result produces zero without reading offset 20 for the final dependency.

6. **Late field observation.** If both preliminary results are zero and either dependency changes offset 20, F-1e06ba66d3c3 receives the value read after those calls.

7. **Numeric gate rejection.** If the final carry observation is clear, the numeric path returns `0x8000000000000000`. No payload-only acceptance predicate for this outcome is established.

8. **Defined-width positive transformation.** With numeric selection, payload low word zero, high word `0x43D00000`, and admission permitted, the signed header is 1085 and the shift distance is one. The result is `0x4000000000000000`.

9. **Defined-width negative transformation.** Under the same admission condition, low word zero and high word `0xC3D00000` produce signed header −963 and distance one. The returned pattern is `0xC000000000000000`.

10. **Conditional endpoints.** If admission succeeds, direct transformation is bypassed, and classification is below `0xFFE00001`, a nonnegative original high word produces `0x7FFFFFFFFFFFFFFF`; a negative one produces `0x8000000000000000`.

11. **Exceptional read ordering.** For case 8 with bit `0x40` clear and bit `0x10` set, the default external pair is read before the alternate external word. The apparent passed pair is nevertheless zero followed by that alternate word.

12. **Oversized-shift boundary.** A direct case requiring distance 32 cannot be accepted solely by assuming a conventional 64-bit shift. Its exact output requires the missing target word-shift semantics.

## Uncertainties

The following remain **documented_only**:

- Exact ABI and carry-condition linkage across numeric dependencies.
- Values and meanings of external comparison and dispatch data.
- Target behavior for 32-bit shifts by 32 or more.
- A standards-level numeric format and rounding interpretation.
- Exceptional destinations, callable interfaces, failures, and return behavior.
- Successful fallback output storage and returned-word linkage.
- Dependency mutations, global preservation, pointer retention, and ownership.
- Physical memory ordering, atomicity, and volatile-access guarantees.
- Behavior for invalid addresses, insufficient storage, unsupported alignment, or nonreturning dependencies.

The visible numeric structure supports a conversion interpretation, but these unresolved points prevent specifying a complete portable numeric conversion.

## Evidence

This contract is derived from the supplied static representation. The prior draft, embedded commentary, and feedback are not independent proof of behavior.

Static evidence supports the stated offsets, widths, selection precedence, explicit writes, short-circuit dependency order, normalized bit layout, direct-path eligibility, endpoint patterns, and apparent exceptional argument patterns.

Floating-point intent is inferred. Oversized shifts, processor-condition interpretation, exceptional transfer semantics, and successful fallback data flow remain explicitly unresolved.

No runtime tests or observed executions have been performed.
