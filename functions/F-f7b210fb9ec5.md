<!-- Trusted provenance: F-f7b210fb9ec5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f7b210fb9ec5

## Purpose

F-f7b210fb9ec5 measures the highest occupied bit position in a bounded collection of transformed integer words. Positions are one-based. Words at higher addresses take precedence over words at lower addresses, and the highest set bit within the selected word determines the returned position.

The collection size is determined by the magnitude of a signed count multiplied by an external **extent factor**. Each examined word is multiplied by a separate external **transformation factor** before its value is considered.

For stable inputs and defined arithmetic, the function returns:

- **0** when the selected collection is empty, exceeds the supported size, or contains only zero transformed values.
- **1–896** when an eligible collection contains a nonzero transformed value.

The supported size is at most 28 words, with 32 result positions assigned to each word. An oversized collection is rejected, not truncated.

**Inferred intent:** a bounded significant-bit measurement over transformed word storage. Neither a conventional multiprecision number format nor the application-level meaning of the two factors is established.

## Interface

### Argument and descriptor

The function receives one pointer to a descriptor and returns a signed integer. The supplied representation supports the following 32-bit interpretation:

| Descriptor offset | Width | Meaning |
|---|---:|---|
| 0 bytes | 32 bits | Signed count |
| 4 bytes | 32 bits | Signed integer encoding the base address of separate word storage |

The word storage is not inline in the descriptor. Each stored word is interpreted as a signed 32-bit integer, and successive words are four bytes apart.

The address-bearing field is an integer used in address arithmetic before conversion to a pointer. It is not established as a portable native-pointer field. The execution environment must make the encoded value, address calculation, conversion, alignment, and resulting reads valid.

The descriptor count is referenced for the extent calculation. The address-bearing field is referenced only when an eligible, nonempty collection is examined.

### External inputs

Two external scalar values participate:

- The **extent factor** scales the descriptor count.
- The **transformation factor** scales each word value considered during qualification or bit inspection.

Their declarations, initialization rules, permitted application values, memory qualifiers, and synchronization properties are unavailable. The width-specific contract below uses the 32-bit signed integer model supported by the supplied representation.

The extent calculation occurs once. There is no operation-wide snapshot of the transformation factor, encoded storage address, or stored contents.

### Return value

A successful result identifies a position rather than returning a word, address, or count of set bits. Zero does not distinguish an empty selection, an oversized selection, and an all-zero transformed selection.

There is no output buffer, diagnostic record, or separate error status.

### Preconditions

Defined execution requires the following for operations actually reached:

- The descriptor count must be readable.
- Multiplication of the count by the extent factor must be representable as a signed 32-bit integer.
- If that product is negative, its positive magnitude must also be representable.
- Every reached read of the address-bearing field must be valid.
- Address arithmetic and integer-to-pointer conversion must produce a valid, suitably aligned address for each reached word read.
- Every reached signed word transformation must be representable.
- Each reached shift must use a count valid for a 32-bit operand.

Stable relevant values are sufficient to keep the bit search within its valid range. Stability is not established as an enforced property of the function.

These are access-sensitive requirements. A successful measurement in a higher-addressed word does not require readable lower-addressed words that are never examined.

## Behavior

### Selection and eligibility

The selected extent is the absolute magnitude of the signed product of the descriptor count and the extent factor, provided both the multiplication and any necessary negation are representable.

| Extent | Required behavior |
|---:|---|
| 0 | Return zero without referencing the storage-address field or word storage. |
| 1–28 | Examine the selected word collection. |
| 29 or greater | Return zero without referencing the storage-address field or word storage. |

Neither operand must individually be positive. A negative product can select an eligible collection after magnitude conversion, and two negative operands can produce an eligible positive product.

The minimum signed 32-bit value is not a valid magnitude-conversion case: its positive magnitude is not representable in that type. No special handling for this case is established.

The selected extent is fixed by the initial calculation. Changes to the descriptor count or extent factor after that calculation do not cause the extent to be recalculated.

### Position meaning

Word indices begin at zero and increase with address. Each word contributes a block of 32 positions:

| Word index | Position interval |
|---:|---:|
| 0 | 1–32 |
| 1 | 33–64 |
| 2 | 65–96 |
| 27 | 865–896 |

For stable observations, the result has a declarative definition. Let \(j\) be the greatest selected word index whose transformed value is nonzero. Let \(b\) be the highest set-bit index, from 0 through 31, in that transformed value after conversion to unsigned 32-bit form. The result is:

\[
32j+b+1
\]

If no such word exists, the result is zero.

Thus any nonzero transformed value in a higher-indexed word outranks every value in a lower-indexed word. Within the qualifying word, the highest set bit wins.

The result is not a population count, a lowest-set-bit position, or the bit length of the absolute value of a signed word.

### Transformation and signedness

Transformation uses signed multiplication. The product determines whether a word qualifies as nonzero; bit inspection uses the product’s unsigned 32-bit interpretation.

Unsigned conversion happens after multiplication. It therefore does not establish wrapping behavior for a signed product that is out of range.

A representable negative product converts to an unsigned value with bit 31 set. Such a product yields the last position assigned to its word. In particular, a transformed value of −1 in word zero yields 32, not 1.

A zero transformation factor makes every valid transformed product zero under stable observations. Unlike a zero extent, however, it does not introduce an explicit bypass of selected storage references.

### Observable ordering and repeated evaluation

The following ordering relationships are established at the expression level:

- Extent calculation and eligibility determination precede storage evaluation.
- Word qualification proceeds from the greatest selected index toward index zero.
- A lower word is reached only after all higher selected words have qualified as zero.
- Nonzero qualification at a word precedes bit inspection at that word.
- Bit inspection gives priority to bit 31, then successively lower positions.
- Once a word qualifies, lower-indexed words are not subsequently considered.

The product observed during qualification is not retained as a value snapshot for all bit inspections. Qualification and each reached bit inspection contain separate transformation evaluations, each involving the external transformation factor, encoded storage address, and selected stored word.

The representation does not establish the relative evaluation order of the count and extent factor. It likewise does not establish whether the transformation factor or associated word value is obtained first. The encoded address necessarily supplies the address for its associated word access.

These are expression-level properties, not guarantees of distinct physical loads. The unavailable declarations and execution model determine whether repeated references can be combined, require fresh reads, or have special visibility semantics.

### Values that change during evaluation

Changing inputs can separate qualification from the later bit observations. A word may qualify using one product, while subsequent inspections observe different products or even a different storage address.

There is no consistency check, restart, or fallback to lower words after nonzero qualification.

The bit-position state is unsigned and initially identifies bit 31. If every inspection through bit zero fails, decrementing that state wraps it. The shift count uses its low eight bits, which are then 255. A shift by 255 is outside the defined range for a 32-bit operand.

Consequently, nonzero qualification followed by zero at every valid bit inspection has no established numerical result or recovery behavior. Hardware-specific shift-count masking must not be assumed. The presence of repeated references does not establish support for concurrent mutation.

## Side effects

### Memory and state

The supplied body contains no explicit writes to the descriptor, word storage, external factors, or other persistent memory. Its explicit output is the returned scalar.

There are no function calls, allocations, releases, I/O operations, or diagnostic operations. No pointer retention beyond the invocation is visible.

Reads from special-purpose memory could have external effects, but no such memory mapping or access semantics are established here. The absence of explicit writes is not evidence about unknown memory-device behavior.

### Ownership and lifetime

No ownership acquisition, transfer, or release is visible. The caller or surrounding environment must keep each reached descriptor field and stored word valid for its use.

Early completion limits which words are accessed. It does not validate the allocation, establish the total allocation size, or transfer responsibility for unread storage.

### Invalid inputs and failures

No explicit validation or recovery is provided for an unreadable descriptor, invalid encoded address, misaligned storage, insufficient readable storage, signed arithmetic overflow, unrepresentable magnitude, or invalid shift.

These conditions are not specified to return zero, issue a diagnostic, or terminate in any particular manner. Zero is an ordinary measurement result, not a general failure code.

For an eligible extent, the largest word displacement is 108 bytes. Although that displacement is small, adding it to the signed encoded address must still be representable and must identify valid storage.

The successful position calculation uses unsigned arithmetic before returning through the signed result type. Under the defined 32-bit model, every ordinary successful value is at most 896 and is representable in the signed return type.

## Dependencies

There are no called functions and therefore no dependency function IDs to enumerate.

The operation depends on the following data and environmental properties:

| Dependency | Contract relevance |
|---|---|
| Descriptor count | Determines extent together with the extent factor. |
| Extent factor | Scales the count before magnitude conversion. |
| Encoded storage address | Locates each reached word evaluation. |
| Stored words | Supply signed transformation operands. |
| Transformation factor | Affects qualification and every reached bit inspection. |
| Integer model | Determines widths, overflow boundaries, unsigned conversion, and valid shifts. |
| Address model | Determines valid encoded addresses, arithmetic, conversion, and alignment. |
| Memory model | Determines permitted visibility and realization of repeated references. |

No callee-preservation or callee-ownership assumptions are needed because no calls are present. External-state stability nevertheless remains an environmental condition rather than a guarantee made by the function.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume the described 32-bit model, stable observations, valid reached addresses, and representable arithmetic. Stored values are values before transformation.

| Case | Conditions | Required result or observation |
|---|---|---|
| Empty selection | Count 0; extent factor 1. | Return 0 without referencing the storage-address field or words. |
| Extent suppressed | Count 17; extent factor 0. | Return 0 without referencing the storage-address field or words. |
| Positive oversized selection | Count 29; extent factor 1; storage exists for 29 words. | Return 0 without word evaluation. Available storage does not change the limit. |
| Negative oversized selection | Count −29; extent factor 1. | Return 0 after magnitude conversion, without word evaluation. |
| Negative count | Count −2; extent factor 1; transformation factor 1; words zero and one contain 1 and 0. | Return 1. Word one is qualified before word zero. |
| Negative extent factor | Count 2; extent factor −1; transformation factor 1; word one contains 1. | Return 33 without reading word zero. |
| Two negative extent operands | Count −2; extent factor −1; transformation factor 1; word one contains 1. | Return 33. |
| Higher-word priority | Extent 2; transformation factor 1; words zero and one contain 1024 and 1. | Return 33; word zero is not read. |
| Highest bit | Extent 1; transformation factor 1; word zero contains 40. | Return 6. |
| Transformation changes position | Extent 1; transformation factor 2; word zero contains 4. | Return 4 because the transformed value is 8. |
| Negative product | Extent 1; transformation factor 1; word zero contains −1. | Return 32. |
| All zero | Extent 2; transformation factor 1; both selected words contain zero. | Return 0 after descending qualification. |
| Zero transformation factor | Extent 2; transformation factor 0; selected storage is valid. | Return 0. No explicit factor-only shortcut eliminates the word references. |
| Maximum result | Extent 28; transformation factor 1; word 27 contains −1. | Return 896 without reading lower words. |

Additional boundary conditions:

- A count product of −2,147,483,648 is outside the defined magnitude-conversion contract.
- A transformed product outside −2,147,483,648 through 2,147,483,647 has no established wrapped result.
- An invalid encoded storage address is irrelevant to the storage path when extent is zero or oversized, provided the descriptor count itself remains readable.
- If separately observable evaluations permit qualification to see a nonzero value and every subsequent valid bit inspection to see zero, no numerical result is specified after the resulting invalid shift.

## Uncertainties

### documented_only: declarations and target model

The supplied types, four-byte spacing, and 32 inspected bit positions support the stated 32-bit interpretation. Complete external declarations and target ABI details are absent.

Native pointer width, required alignment, byte order, integer-to-pointer conversion, and valid address ranges remain unresolved. The descriptor description is not a serialized wire-format specification.

### documented_only: arithmetic outside defined ranges

No wrapping contract is established for signed count multiplication, magnitude-producing negation, word transformation, or encoded-address addition. Later unsigned operations do not repair earlier invalid signed arithmetic.

### documented_only: visibility and mutability

External qualifiers, atomicity, synchronization, special-memory behavior, and physical load realization are unavailable. Stable-input behavior is well characterized; a general deterministic result for changing observations is not.

### documented_only: invalid-shift continuation

Restricting the shift count to its low eight bits does not restrict it to the valid range 0–31. Once an invalid shift is reached, no portable continuation, termination, or return value is established.

## Evidence

**Established static evidence:** one descriptor argument; signed extent scaling and magnitude handling; eligibility limited to 28 words; four-byte word spacing; descending word priority; signed transformation followed by unsigned bit interpretation; repeated transformation evaluations; one-based result positions; and no explicit persistent-memory writes or function calls.

**Inferred intent:** bounded highest-significant-bit measurement over transformed word storage. This does not establish an application-specific number representation or explain either external factor.

**Unresolved:** complete declarations and ABI, address validity, arithmetic outside representable ranges, memory visibility, realization of repeated reads, and behavior after an invalid shift.

No runtime tests have been performed. The acceptance examples are static derivations under their stated conditions.
