<!-- Trusted provenance: F-d14620fdf8dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d14620fdf8dc

## Purpose

F-d14620fdf8dc conditionally transforms a 32-bit quantity using context flags, a selector, two option words, and reference values in a selected record.

Two processing modes are established:

- **Adjustment mode:** optionally bypass processing for small quantities; otherwise apply a minimum adjustment, optionally reuse the first reference, or apply a grid adjustment.
- **Reference-list mode:** search for a sufficiently close reference, conditionally adopt it through an asymmetric window, then apply selector-dependent minimums and quantization.

When processing is disabled, the original quantity word is returned unchanged. When processing is enabled, the function records the input’s signed interpretation, conditionally negates the input word, performs the selected transformation, and conditionally negates the result.

The constants and masks suggest grid fitting or size adjustment. That interpretation is inferred intent, not an established domain meaning.

## Interface

The function receives five arguments and returns one unsigned 32-bit word.

| Input | Representation | Role |
|---|---|---|
| Context | 32-bit address-bearing argument | Locates the flags and record-array address. |
| Selector | Signed 32-bit integer | Selects a record and a behavioral category. |
| Quantity | Unsigned 32-bit word | Processed using both unsigned arithmetic and signed interpretations. |
| Option A | Unsigned 32-bit word | Bit `0x01` controls the minimum adjustment in adjustment mode. |
| Option B | Unsigned 32-bit word | Bit `0x02` enables a selector-1 bypass in adjustment mode. |

There is no separate status result or explicit error indicator. Negative numerical results are returned as their unsigned 32-bit representations.

### Accessed storage

| Location | Access width | Use |
|---|---:|---|
| Context + `0x70` | 32 bits | Record-array base address |
| Context + `0x6C` | 32 bits | Flags |
| Selected record + `0x2C` | Signed 32 bits | Reference count in reference-list mode; presence indicator in adjustment mode |
| Selected record + `0x34` | 32 bits | First reference |
| Selected record + `0x34 + 12 × index` | 32 bits | Reference at a list index |

The selected record’s displacement is 392 times the selector narrowed to a signed 16-bit value.

Record addressing and behavioral classification deliberately use different selector forms:

- Addressing uses the signed low-16-bit selector.
- Selector-1 behavior requires the complete 32-bit selector to equal 1.

Thus, selector 65537 selects the same record displacement as selector 1 under the stated narrowing interpretation, but receives the non-1 behavioral rules.

No selector bounds check, count bound, or reference-storage bound is visible. The 392-byte record stride does not establish an enforced reference-list capacity.

### Flags

| Mask | Meaning |
|---|---|
| `0x04` | Enables transformation |
| `0x02` | Selects reference-list mode for selector exactly 1 |
| `0x01` | Selects reference-list mode for every other selector |
| `0x08` | Changes final reference-list adjustment for selectors other than 1 |

Other flag bits have no visible effect. Bits of Option A and Option B other than those identified above have no visible effect.

## Behavior

### Numeric conventions

The supplied representation uses 32-bit unsigned quantities and signed 32-bit comparisons. Unless explicitly identified otherwise, thresholds and order comparisons below are signed comparisons.

Unsigned additions, subtraction, and negation operate modulo \(2^{32}\). Clearing the low six bits means applying `0xFFFFFFC0`. For a nonnegative ordinary-range value, this produces the preceding or equal multiple of 64.

Enabled processing first tests whether the original quantity has a negative signed interpretation. If so, it negates the **unsigned** quantity word. At return, it negates the **unsigned** result word if the original input was negative.

These unsigned negations do not themselves introduce signed-overflow uncertainty. In particular, negating `0x80000000` leaves that word unchanged; it does not produce a positive signed magnitude. Ordinary-range descriptions using “magnitude” exclude that exceptional input.

Distance calculations subtract two unsigned words, interpret the resulting word as signed, and negate that signed value when negative. Signed conversion and the negation of the most-negative signed value require the qualifications in **Uncertainties**.

### Entry and disabled processing

The visible evaluation order is:

1. Determine whether the complete selector equals 1.
2. Read the record-array address from context offset `0x70` and form the selected-record address.
3. Read the flags from context offset `0x6C`.
4. Test enable bit `0x04`.

If the enable bit is clear, return the original quantity word immediately.

This path does not normalize the quantity, read record contents, or apply the options. It nevertheless contains the earlier array-address read and record-address calculation. Disabling processing therefore does not remove all context-validity requirements.

When enabled, record the original sign interpretation and perform the unsigned normalization described above.

### Mode selection

For selector exactly 1, flag `0x02` selects reference-list mode.

For every other selector, flag `0x01` selects reference-list mode.

If the applicable bit is clear, use adjustment mode. The flags are read once; subsequent decisions use that captured word.

### Adjustment mode

#### Small-quantity bypass

Before minimum adjustment or record-content reads, return the normalized quantity, with the original sign operation reapplied, if all of these conditions hold:

- Option B contains bit `0x02`.
- The complete selector equals 1.
- The normalized quantity’s signed interpretation is less than 192.

The comparison is strict. An ordinary magnitude of 192 does not bypass.

#### Minimum adjustment

If there is no bypass:

- If Option A bit `0x01` is clear, replace a quantity signed-less-than 56 with 56.
- If that bit is set, replace a quantity signed-less-than 80 with 64.

The second rule is not a clamp to 80: values 65 through 79 are reduced to 64, while 80 is unchanged.

Next, read the signed field at record offset `0x2C`.

- If it is zero, return the minimum-adjusted quantity after the final sign operation.
- If it is nonzero, read the first reference at offset `0x34`.

A negative field therefore enables the first-reference read in this mode. Its magnitude is not used, and no list traversal occurs.

#### First-reference reuse

Compute the signed absolute-distance operation between the adjusted quantity and the first reference.

If that distance is strictly less than 40:

- Select the first reference as the result.
- If its signed interpretation is less than 48, replace it with 48.
- Apply the final sign operation and return.

A distance of exactly 40 does not qualify.

#### Grid adjustment without reference reuse

If reference reuse fails and the adjusted quantity is signed-less-than 192, divide its word into a block base, obtained by clearing the low six bits, and a remainder from 0 through 63.

| Remainder | Result |
|---:|---|
| 0–9 | Original quantity |
| 10–31 | Block base + 10 |
| 32–53 | Block base + 54 |
| 54–63 | Original quantity |

This transformation may raise or lower the quantity. It is not ordinary rounding to a multiple of 64.

If the adjusted quantity is signed-at-least 192, add 32 using unsigned arithmetic and clear the low six bits. For ordinary nonnegative values without wraparound, this rounds to the nearest multiple of 64, with midpoint ties upward.

Apply the final sign operation before returning.

### Reference-list mode

Neither option word affects this mode.

#### Search

Initially:

- The candidate is the normalized input.
- The distance limit is 98.
- The list index is zero.

For every index that is signed-less-than the current signed count:

1. Read the reference at offset `0x34 + 12 × index`.
2. Advance the index.
3. Calculate the signed absolute-distance operation between the normalized input and that reference.
4. Replace both candidate and distance limit only if the new distance is strictly smaller than the current limit.

The count at offset `0x2C` is read at every loop condition, including the terminating condition.

For ordinary-range distance calculations:

- A first qualifying reference must be less than 98 units away.
- A reference exactly 98 units away does not qualify.
- Equal-distance ties retain the earlier reference.
- Zero or negative counts cause no reference-entry reads.
- If no reference qualifies, the candidate remains the normalized input.
- A zero-distance match does not terminate traversal early.

#### Candidate-adoption window

Construct a grid value by adding 32 to the candidate and clearing its low six bits. This grid value defines the adoption window; it is not itself adopted.

Then:

- If the normalized input is signed-less-than the candidate, adopt the candidate only if the input is strictly greater than the grid value minus 48.
- Otherwise, adopt the candidate only if the input is strictly less than the grid value plus 48.

The boundary arithmetic is unsigned, followed by signed interpretation for comparison.

Equality with the tested boundary prevents adoption. Winning the distance search is therefore insufficient by itself. If adoption succeeds, the working quantity becomes the unrounded candidate; otherwise, it remains the normalized input.

#### Final adjustment for selector exactly 1

- If the working quantity is signed-less-than 64, use 64.
- Otherwise, add 16 and clear the low six bits.

For ordinary nonnegative values, the latter rule advances to the next 64-unit boundary only when the original remainder is at least 48.

Apply the final sign operation and return.

#### Final adjustment for every other selector

If flag `0x08` is set:

- Below 64, use 64.
- Otherwise, add 32 and clear the low six bits.

If flag `0x08` is clear:

- Below 48, add 64, interpret the sum as signed, and right-shift it by one.
- From 48 through 127, add 22 and clear the low six bits.
- At 128 or above, add 32 and clear the low six bits.

For ordinary nonnegative values below 48, the first rule yields the floor of half the working quantity plus 32. Consequently, zero can become 32.

Apply the final sign operation and return.

## Side effects

No explicit memory writes, allocation, release, external calls, or callbacks appear in the supplied body. The explicit output is the returned word.

The function uses the context and selected storage for reads. No ownership transfer or address retention is visible.

The source-visible read order is:

- **Every path:** array-address field, then flags.
- **Disabled processing:** no record-content reads.
- **Adjustment bypass:** no record-content reads.
- **Other adjustment paths:** one count/presence read, followed by one first-reference read only if that field is nonzero.
- **Reference-list mode:** alternating count-condition reads and reference-entry reads in ascending index order, followed by a terminating count read.

Only the first 32-bit word of each 12-byte reference entry is read.

No synchronization or snapshot of record contents is visible. The count is repeatedly accessed, while flags are captured once. Stable-input acceptance examples require the accessed storage to remain valid and stable throughout the call. These source-visible accesses do not independently establish machine-level observation or a supported concurrent-mutation protocol.

There is no explicit recovery or error result for invalid addresses, invalid record selection, or an excessive count. Such inputs are outside the established valid-storage contract.

## Dependencies

No outgoing function dependency appears in the supplied body. The supplied identity mapping identifies only F-d14620fdf8dc itself.

External requirements are data and execution properties:

- Readable context fields.
- A valid record-array address and selected displacement whenever record contents are reached.
- Readable reference entries for every visited index.
- Compatible 32-bit numeric and address representations.
- Valid address and index arithmetic on the exercised path.

There is no unknown callee whose global preservation, memory ownership, or cleanup behavior must be assumed.

## Acceptance examples

These are conditional arithmetic examples, not executed tests. Except for the disabled case, transformation is enabled. Storage is valid and stable, and arithmetic remains in ordinary signed range.

| Conditions | Expected result |
|---|---|
| Enable bit clear; quantity represents −73 | Original word representing −73 |
| Adjustment mode; selector 1; Option B bit `0x02` set; quantity 37 | 37; no record-content read |
| Same conditions; quantity −37 | Word representing −37; no record-content read |
| Adjustment mode; Option A bit clear; quantity 20; presence field zero | 56 |
| Adjustment mode; Option A bit set; quantity 79; presence field zero | 64 |
| Same conditions; quantity 80 | 80 |
| Adjustment mode; adjusted quantity 56; first reference 20; presence field nonzero | 48: distance 36 qualifies, then the reference minimum applies |
| Adjustment mode; adjusted quantity 80; first reference 40 | 74: distance 40 fails reuse; remainder 16 maps to 10 within the block beginning at 64 |
| Adjustment mode; quantity 100; first reference 0; presence field nonzero | 118 |
| Adjustment mode; quantity 192; first reference 0; presence field nonzero | 192 |
| Adjustment mode; quantity 224; first reference 0; presence field nonzero | 256 |
| Reference-list mode; selector 1; count zero; quantity 100 | 64 |
| Same conditions; quantity 112 | 128 |
| Reference-list mode; selector other than 1; flag `0x08` clear; count zero; quantity 0 | 32 |
| Same conditions; quantity 47 | 55 |
| Same conditions; quantity 48 | 64 |
| Reference-list mode; selector other than 1; flag `0x08` set; count zero; quantity 47 | 64 |

Additional boundary and ordering cases:

- **Equal-distance selection without an output difference:** With selector 1, quantity 100, and exactly two references 90 then 110, reference 90 wins and is adopted; the result is 64. Reversing the references selects and adopts 110, but the result remains 64: adding 16 gives 126, whose low-six-bit clearing yields 64.
- **Output-visible tie order:** With selector 1, quantity 112, and exactly two references 102 then 122, both distances are 10. Reference 102 wins, passes the adoption window, and produces 64 after the final adjustment. Reversing the order selects 122, also passes the window, and produces 128.
- **Window interior:** With selector 1, quantity 90, and sole reference 130, the candidate’s grid value is 128. The input exceeds the lower boundary 80, so 130 is adopted and the result is 128.
- **Window equality:** With selector 1, quantity 80, and sole reference 130, the reference wins the distance search, but equality with boundary 80 prevents adoption. The result is 64.
- **Strict distance limit:** A sole reference exactly 98 units from the normalized input is not selected.
- **Negative count:** In adjustment mode, a negative presence field permits the first-reference read. In reference-list mode, a negative count prevents all reference-entry reads.

## Uncertainties

**documented_only — signed conversions and exceptional distance arithmetic.** Unsigned operations have 32-bit modular behavior. Interpreting an unsigned word above the signed maximum as signed depends on the representation’s conversion rules. The ordinary examples use the conventional two’s-complement interpretation and avoid problematic intermediate ranges. If a distance word becomes the most-negative signed value, its subsequent signed negation is not established as a valid mathematical absolute value.

**documented_only — the most-negative input.** Input word `0x80000000` remains unchanged by unsigned normalization. Under two’s-complement signed interpretation it therefore remains negative and can satisfy signed “less than” tests intended for ordinary magnitudes. It must not be described as successfully converted to a positive magnitude.

**documented_only — signed right shift.** The below-48 non-1 rule shifts a signed value. Its ordinary nonnegative behavior is established; behavior for a negative shifted operand depends on the applicable signed-shift rules.

**documented_only — address and traversal arithmetic.** Address formation, entry displacement, and sufficiently long traversal can exceed representable signed ranges. No bounds checks or independent overflow rules are supplied.

**documented_only — storage and concurrency.** Endianness, alignment, complete object layout, allocation sizes, compiler treatment of repeated reads, and a supported concurrent-update protocol are not established.

**Inferred intent.** Reference fitting, dimensional adjustment, and quantization are plausible interpretations. Units, valid selector ranges, and reference-validity constraints remain unknown.

## Evidence

The evidence is the supplied function representation: argument and return types, field offsets, selector narrowing, flag masks, comparison casts, unsigned arithmetic, signed distance handling, loop structure, and return paths.

The corrected tie examples follow from the selector-1 final rule: add 16, then clear the low six bits. Both 90 and 110 produce 64; 102 produces 64, while 122 produces 128.

No runtime tests, executable inspection, memory traces, caller inspection, or independent dependency inspection have been performed. Acceptance examples are static arithmetic consequences under their stated preconditions.
