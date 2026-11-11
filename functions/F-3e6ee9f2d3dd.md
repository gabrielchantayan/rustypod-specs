<!-- Trusted provenance: F-3e6ee9f2d3dd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e6ee9f2d3dd

## Purpose

F-3e6ee9f2d3dd obtains a context through F-7fba0909c452 and queries a selector-dependent value. Recognized queries return an unsigned byte, one bit from a byte, or a membership result from a 32-bit flag word.

A null context returns zero. A rejected selector also returns zero and conditionally records `0x00000500` in the context’s first 32-bit word.

Dispatch depends on five external selector slots whose values are not supplied. The represented body does not capture a single snapshot of these slots: a reached slot is read for equality and, after inequality, may be read again for ordering. Consequently, literal selectors are accepted only when the preceding comparisons route execution to them.

The apparent intent is a state-query interface with first-error preservation. Field meanings and the semantic meaning of the error value are not established.

## Interface

### Input and output

The input is one signed 32-bit selector.

The output is an unsigned 32-bit value:

- Byte queries return values from zero through 255.
- Byte-bit queries return zero or one.
- Flag-word queries return zero or one when their arithmetic and signed mask shift have established semantics.
- Null-context and rejected-selector paths return zero.

Zero does not distinguish rejection, an unavailable context, a zero-valued byte, or a clear bit.

Selector ordering comparisons are signed except for the explicitly unsigned range check in the D region. Equality compares the corresponding 32-bit values.

### Context and external state

F-7fba0909c452 returns the context pointer. Context offsets below are byte offsets. The represented layout has 8-bit bytes and four-byte integer words; it does not establish the complete context structure.

Five external selector slots are designated A through E:

| Slot | Equality action | Subsequent ordering role after inequality |
|---|---|---|
| A | Select flag word `0x2F4`, base `0x3000` | Choose lower or upper dispatch |
| B | Return byte `0x8FF` | Choose between two lower-dispatch literal groups |
| C | Return bit zero of byte `0x14C` | Choose the D or E region |
| D | Select flag word `0x89C`, base `0x4000` | Choose unsigned range checking or higher literals |
| E | Return byte `0x907` | Choose between two upper-dispatch literal groups |

These labels do not imply numeric values, ordering, uniqueness, or semantic names.

### Preconditions and ownership

Established results require:

- Every reached byte or word read to address readable storage.
- A rejected-selector path to have a readable word at offset zero, and writable storage there when its value is zero.
- Signed arithmetic to remain within its represented width.
- Each evaluated pointer operation to be valid, including intermediate pointers.
- Flag-mask shift counts to fall within the established range described below.

There is no explicit allocation, release, ownership transfer, reference-count change, or pointer retention in this function. The acquisition dependency’s ownership and lifetime behavior is unknown.

## Behavior

### Acquisition and dispatch-read order

F-7fba0909c452 is called once without explicit arguments, before dispatch.

If it returns null, the result is zero. No subsequent explicit selector-slot reads or context accesses occur.

Otherwise, dispatch uses the following conditional reads:

| Slot | First read | Second read, reached only after inequality |
|---|---|---|
| A | Compare selector for equality | If A is less than or equal to the selector, enter upper dispatch; otherwise enter lower dispatch |
| B | Compare selector for equality | If B is less than the selector, take B’s higher branch; otherwise take B’s other branch |
| C | Compare selector for equality | If C is less than the selector, enter the E region; otherwise enter the D region |
| D | Compare selector for equality | If the selector is less than D, take the unsigned-range branch; otherwise take the higher-literal branch |
| E | Compare selector for equality | If E is less than the selector, take E’s higher branch; otherwise take E’s other branch |

A successful equality comparison ends dispatch through that slot without its second read.

The second read need not equal the first. In particular, equality at the second read routes:

- A to upper dispatch.
- B to its other branch.
- C to the D region.
- D to its higher-literal branch.
- E to its other branch.

Descriptions such as “selector below A” are therefore valid only under an explicit assumption that A remains stable across its two reads. The same limitation applies to the other slots.

### Initial A equality

Equality with A’s first read immediately selects the flag-word family at context offset `0x2F4`, with selector base `0x3000`.

This takes precedence over every literal-selector case. It does not require the selector to belong to the literal range `0x3000` through `0x3004`.

### Lower dispatch

Lower dispatch is reached when A’s first read is unequal to the selector and its second read is greater than the selector.

Selector `0x0BD0` returns byte `0x900`.

For other selectors below `0x0BD1`, the accepted cases are:

| Region or condition | Selector | Result source |
|---|---|---|
| Before the remaining lower subregions | `0x0B60` | Byte `0x95C` |
| Below `0x0B61`, excluding `0x0B60` | Equality with B’s first read | Byte `0x8FF` |
| B’s higher branch | `0x0B50` | Byte `0x8FD` |
| B’s higher branch | `0x0B57` | Byte `0x902` |
| B’s other branch | `0x0B10` | Byte `0x944` |
| B’s other branch | `0x0B20` | Byte `0x948` |
| At least `0x0B61` | `0x0B71` | Byte `0x9D3` |
| At least `0x0B61` | `0x0B90` | Byte `0x9D8` |
| At least `0x0B61` | `0x0BA1` | Byte `0x903` |
| At least `0x0B61` | `0x0BC0` | Byte `0x9C8` |

B is not read for `0x0B60` or in the subregion at least `0x0B61`. Its equality action precedes its literal branches.

For selectors at least `0x0BD1`, the accepted cases are:

| Selector | Result source |
|---|---|
| `0x0BE2` | Byte `0x9D0` |
| `0x0BF2` | Byte `0x9D5` |
| `0x0C11` | Byte `0x130` |
| `0x0DE1` | Record-stride dynamic byte |
| `0x3000` through `0x3004` | Flag-word family `0x2F4`, base `0x3000` |

Selector `0x3000` is recognized before the remaining comparisons in this region. The four smaller literal cases are considered in the portion below `0x3001`. Other lower-dispatch selectors are rejected.

### Upper dispatch and C equality

Upper dispatch is reached when A’s first read is unequal and its second read is less than or equal to the selector.

Equality with C’s first read returns bit zero of byte `0x14C`.

Otherwise, C’s second read chooses:

- The E region when its value is less than the selector.
- The D region when its value is greater than or equal to the selector.

### D region

Equality with D’s first read selects flag word `0x89C`, using base `0x4000`.

After inequality, D’s second read determines the remaining behavior.

**Selector less than D’s second value:** accept only when the unsigned 32-bit difference from `0x4000` is at most four. This admits exactly selectors `0x4000` through `0x4004`, which select the same flag-word family.

**Selector greater than or equal to D’s second value:**

1. Selector `0x4006` selects that flag-word family without evaluating the preliminary signed difference used for the next check.
2. Otherwise, a signed 32-bit difference from `0x4000` equal to seven selects that family.
3. If neither condition succeeds, selector `0x8037` returns byte `0x950`, and selector `0x803A` returns byte `0x904`.
4. Other selectors are rejected.

When defined, the difference-equals-seven condition accepts `0x4007`. That signed difference is evaluated for other selectors in this branch before checking `0x8037` and `0x803A`; overflow behavior is not established.

D’s value is unknown. In particular, this region cannot be replaced by unconditional acceptance of the contiguous range `0x4000` through `0x4007`.

### E region

Equality with E’s first read returns byte `0x907`.

After inequality, the accepted literals are:

| Ordering route | Selector | Result source |
|---|---|---|
| E’s second value less than selector | `0x809F` | Byte `0x908` |
| Same | `0x80A0` | Byte `0x909` |
| Same | `0x8861` | Byte `0x945` |
| Same | `0x8B9C` | Bit three of byte `0x14C` |
| E’s second value greater than or equal to selector | `0x8075` | Bit one of byte `0x14C` |
| Same | `0x8076` | Bit two of byte `0x14C` |
| Same | `0x8078` | Byte-stride dynamic byte |
| Same | `0x809D` | Byte `0x906` |

Other selectors are rejected.

### Byte and dynamic-query semantics

A byte query reads one unsigned 8-bit value and zero-extends it to the result. Nonzero bytes are not normalized to one.

Each dynamic query reads a signed 32-bit index word before reading its target byte:

| Query | Index-word offset | Mathematical target byte offset |
|---|---|---|
| `0x8078` branch | `0x250` | `0x14D` plus the index |
| `0x0DE1` branch | `0x24C` | `0x964` plus 36 times the index |

These formulas apply only when the represented arithmetic and addressing operations are valid. Neither query explicitly checks bounds or rejects negative indices.

**Byte-stride query.** Address formation uses the context converted to a signed integer representation, followed by addition of the index and then `0x14D`. The mathematical final offset does not establish that the intermediate integer addition, final integer addition, or conversion back to a pointer is valid. Pointer representation and signed overflow remain relevant.

**Record-stride query.** Address formation has three distinct stages:

1. Signed 32-bit multiplication of the index by nine produces a displacement in four-byte words.
2. That displacement is added to the context as an integer-word pointer, producing an intermediate pointer.
3. A separate pointer addition advances that intermediate pointer by `0x259` further words.

The final pointer is then used to read one byte.

The multiplication must be representable as a signed 32-bit value. The first pointer addition and the subsequent `0x259`-word pointer addition each have their own validity requirements. There is no signed integer addition of `0x259` to the product before pointer scaling.

Because `0x259` words occupy `0x964` bytes, the final mathematical byte offset is still `0x964` plus 36 times the index. A readable final address alone does not establish that both preceding pointer additions are valid.

### Byte-bit queries

Byte `0x14C` supplies four independent bit queries:

- C equality selects bit zero.
- `0x8075` selects bit one.
- `0x8076` selects bit two.
- `0x8B9C` selects bit three.

Each returns zero or one. The shift-based extractions operate on unsigned 32-bit values, so other bits in the byte do not affect the selected result.

### Flag-word queries

| Family | Word offset | Selector base |
|---|---|---|
| A equality and lower-dispatch flag literals | `0x2F4` | `0x3000` |
| D equality and accepted D-region flag cases | `0x89C` | `0x4000` |

The family computes a signed selector-minus-base expression, retains its result as an unsigned 32-bit value, and reads the selected 32-bit context word. The low eight bits of the retained difference determine the mask’s shift count.

The mask is formed by shifting the **signed integer value one**. Its later combination with an unsigned word does not make the preceding shift unsigned.

For counts zero through 30, with preceding arithmetic defined, the result is one exactly when the corresponding word bit is set.

The following are **documented_only**:

- Count 31: the mathematical mask is not representable as a positive signed 32-bit integer.
- Counts 32 through 255: the count is outside the 32-bit shift width.
- Extreme selectors whose signed selector-minus-base expression overflows.

No unsigned bit-31 rule or processor-style modulo-32 count masking is prescribed.

### Rejection

Rejection reads the 32-bit word at context offset zero.

- If zero, it writes `0x00000500` to that word.
- If nonzero, it performs no error-word write.

The result is zero.

Recognized queries do not explicitly inspect or update the error word merely because their result is zero.

## Side effects

The conditional error-word assignment is the only explicit write in the supplied body.

The represented ordering establishes:

1. Context acquisition before selector-slot reads.
2. Conditional equality reads before the corresponding ordering reads.
3. Dispatch comparisons before the selected context-data access.
4. Dynamic-index reads before address calculation and target-byte reads.
5. Flag-family selector-difference calculation and word acquisition before mask testing.
6. Rejection’s error-word read before its conditional write.

No path reads all five selector slots. B belongs only to lower dispatch. C belongs only to upper dispatch, where D and E occupy mutually exclusive routes.

Separate represented reads do not establish volatile behavior, atomicity, synchronization, or permission for concurrent mutation. No coherent snapshot is guaranteed by this body.

F-7fba0909c452 may have additional effects on context storage, external slots, other globals, or memory ownership. Its effects are not established here.

## Dependencies

| Opaque ID | Established use | Unknown contract |
|---|---|---|
| F-7fba0909c452 | Called once without explicit arguments; result interpreted as the context pointer | Context selection, initialization, lifetime, ownership, memory management, global changes, synchronization, and causes of null results |

There are no other explicit function calls.

The five selector slots and accessed context locations are additional state dependencies. Their initialization, storage ownership, and access discipline are unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Arithmetic and memory-access validity are prerequisites. Slots are assumed stable across their reads except in the changing-slot example.

1. **Null context.** If F-7fba0909c452 returns null, the result is zero without subsequent explicit context or selector-slot access.

2. **Byte preservation.** If A is greater than `0x0BD0`, selector `0x0BD0` reads byte `0x900`. A stored value of `0xFE` returns 254. A zero byte returns zero without an explicit error-word access.

3. **Error preservation.** A rejected selector writes `0x00000500` when the initial error word is zero. With initial value `0x1234`, it leaves that word unchanged. Both cases return zero.

4. **A equality overrides a literal.** If A’s first read equals selector `0x0BD0`, the flag family at `0x2F4` is selected instead of byte `0x900`. There is no second A read. The resulting low-eight-bit shift count is `0xD0`, so the execution result is documented_only.

5. **Established mask boundary.** If A’s first read equals selector `0x301E`, the shift count is 30. Word `0x40000000` at `0x2F4` yields one; word `0x80000000` yields zero.

6. **Unresolved mask counts.** A equality at selector `0x301F` produces count 31, and at `0x3020` produces count 32. Neither result is prescribed.

7. **Byte-bit extraction.** For selector `0x8076`, suppose A and C are smaller than the selector and E is greater. Byte `0x04` at `0x14C` yields one; byte `0xFB` yields zero.

8. **Byte-stride addressing.** When the `0x8078` branch is reached, index two at `0x250` targets byte offset `0x14F`, subject to the integer-address operations being valid.

9. **Record-stride addressing.** When the `0x0DE1` branch is reached, index two at `0x24C` produces an 18-word displacement. The first pointer addition advances by `0x48` bytes; the separate addition of `0x259` words advances by another `0x964` bytes. The byte read is at final offset `0x9AC`, provided both pointer additions are valid.

10. **Negative record index is not automatically accepted or rejected.** Index negative one gives a multiplication result of negative nine words. The intermediate pointer is 36 bytes before the context; the mathematical final offset is `0x940`. The function performs no explicit negative-index check, but a readable byte at `0x940` does not by itself validate the intermediate pointer operation.

11. **Unsigned range boundary.** If A is below `0x4004`, C is above `0x4005`, and D is above `0x4005`, selector `0x4004` selects flag word `0x89C`; selector `0x4005` is rejected.

12. **Changing A changes dispatch.** For selector `0x0BD0`, let A’s equality read be `0x0BD1` and its ordering read be `0x0BD0`. Upper dispatch is taken. If C’s equality read then equals `0x0BD0`, the result is bit zero of byte `0x14C`, not byte `0x900`.

## Uncertainties

- **documented_only — Slot state.** A through E have unknown values, ownership, and mutation rules. Stable-slot partitions require the stated assumption.
- **documented_only — Signed mask shifts.** Counts 31 through 255 have no established result.
- **documented_only — Arithmetic and addressing.** Signed overflow, pointer representation, integer-to-pointer conversion, and invalid accesses have no supplied recovery contract.
- **documented_only — Record-pointer validity.** The multiplication’s representability and both sequential pointer additions are separate requirements. The final offset formula does not resolve them.
- **documented_only — Memory properties.** Byte order, alignment guarantees, synchronization, atomicity, and concurrent-access validity are not supplied.
- **documented_only — Dependency effects.** Acquisition may alter globals, context state, ownership, or memory lifetime.
- **Inferred intent.** The fields appear capability-like or enable-state-related. Neither those meanings nor the meaning of `0x00000500` is established.

## Evidence

The supplied static body establishes the acquisition call, null handling, conditional selector-read sites, dispatch precedence, accessed byte and word locations, bit extractions, dynamic address operations, signed mask operand, and conditional error-word assignment.

Four-byte word scaling establishes the reported byte offsets. For the record-stride query, the evidence distinguishes signed multiplication by nine, pointer addition of that word displacement, and a separate pointer addition of `0x259` words. The final byte-offset formula is a mathematical consequence, not a replacement for those intermediate validity requirements.

No dependency implementation, selector-slot values, independent context-layout definition, execution traces, or additional execution-semantics evidence were supplied. No runtime tests have been performed.
