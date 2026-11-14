<!-- Trusted provenance: F-9cd4380c8dec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9cd4380c8dec

## Purpose

F-9cd4380c8dec selects one comparison operation for two records and returns a 32-bit result. Selection uses a primary selector and, only when that selector is zero, a secondary selector.

Established behavior includes direct numeric comparisons, comparisons delegated through extracted value pairs, flag comparisons, and a special two-call path. Recognized selectors are exact values, not combinable masks. Equal values do not trigger a secondary comparison.

**Inferred intent:** this is a configurable record-ordering operation. The meanings of the records, fields, extracted values, and comparison context are not established.

This contract describes the supplied static representation. It does not assert that dependencies are pure, that extracted values are strings, or that every result is normalized to −1, 0, or 1.

## Interface

The interface has six positional inputs:

| Position | Contract name | Width and role |
|---|---|---|
| 1 | Context | Opaque 32-bit value, forwarded only to F-aa53e7865743. |
| 2 | Left record | 32-bit address representation identifying the first record. |
| 3 | Right record | 32-bit address representation identifying the second record. |
| 4 | Unused argument | Opaque 32-bit value; not referenced by the supplied body. |
| 5 | Primary selector | 32-bit value selecting the main comparison operation. |
| 6 | Secondary selector | 32-bit value consulted for dispatch only when the primary selector is zero. |

The result is a 32-bit word:

- Direct less-than results use `0xffffffff`, conventionally interpreted as signed −1.
- Direct equal results use `0`.
- Direct greater-than results use `1`.
- Delegated comparison paths return the dependency result without visible normalization.
- Unsupported selection also returns `0xffffffff`. The interface therefore does not distinguish unsupported selection from a valid “left is less” result.

Offsets below are byte offsets from the relevant record base. A field described as unsigned 16-bit or unsigned 32-bit is compared using that width and signedness. The signed 16-bit field uses signed ordering.

Required memory validity is path-dependent. The function performs no visible null checks, record-size checks, alignment checks, or nested-address validation. Delegated paths additionally require whatever memory and context validity their dependencies need.

No full record layout, byte order, alignment guarantee, or calling convention is established.

## Behavior

### Selection and precedence

The primary selector chooses at most one operation. A recognized primary selector returns from that operation, even if the compared values are equal.

When the primary selector is zero, the secondary selector chooses one operation. A nonzero unsupported primary selector does not fall back to the secondary selector.

Primary selector `0x80000000` returns zero immediately, without visible record reads or dependency calls.

All other unsupported selections return `0xffffffff`. In particular, selector values formed by combining recognized bits are not automatically meaningful.

Although intermediate dispatch comparisons use signed interpretations of the primary selector, the recognized values are exact 32-bit matches. No additional accepted range follows from those dispatch comparisons.

### Direct numeric comparisons selected by the primary selector

Unless otherwise stated, these operations return −1, 0, or 1 according to ascending field order.

| Primary selector | Record offset | Field interpretation |
|---|---:|---|
| `0x8` | `0x78` | Unsigned 16-bit |
| `0x20` | `0x7a` | Signed 16-bit |
| `0x100` | `0x6c` | Unsigned 32-bit |
| `0x200` | `0x84` | Unsigned 16-bit |
| `0x400` | `0x70` | Unsigned 32-bit |
| `0x4000` | `0x18` | Unsigned 32-bit |
| `0x2000000` | `0x58` | Unsigned 32-bit |
| `0x10000000` | `0x82` | Unsigned 16-bit |
| `0x20000000` | `0x90` | Unsigned 32-bit |
| `0x40000000` | `0x98` | Unsigned 32-bit |

These comparisons do not subtract the fields. Consequently, their established arithmetic does not introduce subtraction overflow.

The supplied representation may read the fields again after the first comparison fails:

- For `0x8`, the second test is equality.
- For `0x200`, the second test is inequality.
- For the other rows, the second test is right-less-than-left.

With stable record contents, all these forms produce ordinary ascending order. The representation does not establish an atomic snapshot or a fixed evaluation order between the two operands of an individual comparison.

### Scaled comparison selected by `0x800`

Each record contributes an unsigned 32-bit effective value:

- If the unsigned 16-bit field at `0xb8` is zero, use the unsigned 32-bit field at `0x5c`.
- Otherwise, use the unsigned 16-bit field at `0xb8`, widened to 32 bits and multiplied by 1000.

The left effective value is obtained before the right effective value. When the short field is nonzero, the supplied representation reads that short field again to obtain the multiplicand. The fallback field at `0x5c` is read only for the zero case.

The effective values are compared in ascending unsigned order.

For stable inputs, the largest scaled value is 65,535,000. This fits in both unsigned and signed 32-bit ranges; multiplication does not overflow 32 bits. The fallback value retains its full unsigned 32-bit range and is not scaled.

No units or domain interpretation of either field are established.

### Extracted-pair comparisons selected by the primary selector

An extractor receives a record address and two addresses for 32-bit output words. The left record is processed first, then the right record. The resulting words form a left pair and a right pair.

| Primary selector | Extractor | Optional alternate-comparison flag |
|---|---|---|
| `0x1` | F-9b584aabce3d | Byte `0x15`, mask `0x08` |
| `0x2` | F-7445c028e81c | Byte `0x15`, mask `0x40` |
| `0x4` | F-c99cba5deb0d | Byte `0x15`, mask `0x10` |
| `0x40` | F-05f08d4e44ad | Byte `0x15`, mask `0x80` |
| `0x80` | F-8ca4d70db38d | None |
| `0x2000` | F-2035045e72a2 | None |
| `0x80000` | F-bfa281f5f170 | None |
| `0x200000` | F-3c2e26a82b7c | None |
| `0x4000000` | F-16b769eefd01 | None |
| `0x8000000` | F-0a40b63db184 | Byte `0x15`, mask `0x20` |

For rows without an alternate flag, the function calls F-aa53e7865743 with the context, left pair, and right pair.

For rows with an alternate flag:

1. Both extraction calls finish before either flag is checked.
2. The left flag byte is examined first.
3. If the relevant left bit is clear, the right flag byte need not be read; comparison uses F-aa53e7865743.
4. If the relevant left bit is set, the corresponding right bit is examined.
5. Only when both bits are set does comparison use F-01af74a46855.

F-01af74a46855 receives the four extracted words without the context argument. F-aa53e7865743 receives the context followed by those four words. Both results are returned unchanged.

Because flag reads follow both extractor calls, mutations performed by either extractor can affect which comparator is selected. The first extractor’s outputs are not visibly copied or otherwise protected from dependency side effects.

### Special primary selector `0x10`

This path passes the 32-bit field at offset `0x74` from each record to F-13e966bdb827. It subsequently passes the fields at those same offsets to F-1ba216d0a747.

In the supplied representation, the condition governing early returns is established from whether the selector exceeds `0x0f`. Since this path requires selector `0x10`, that condition is true. The literal represented behavior is therefore:

- Call F-13e966bdb827.
- Read the arguments again for a call to F-1ba216d0a747.
- Return zero.

Neither dependency’s ordinary return value is visibly used. The second call is not established to receive the same field contents as the first: the fields are read again after the first dependency returns.

This path has a material static-evidence limitation, described under Uncertainties. It must not be silently recast as an ordinary comparison based on presumed dependency results.

### Secondary-selector operations

These operations are available only when the primary selector is zero.

| Secondary selector | Operation |
|---|---|
| `0x1` | Compare unsigned 16-bit fields at `0x88`. |
| `0x2` | Compare byte-sized character fields at `0x7e`; signedness is unresolved. |
| `0x40` | Compare presence of mask `0x04` in byte `0x14`. |
| `0x80` | Compare unsigned 32-bit nested fields at offset `0x18`. |
| `0x100` | Compare unsigned 16-bit fields at `0x8c`. |
| `0x800` | Extract using F-809b8c8462b8, then compare using F-aa53e7865743. |
| `0x2000` | Extract using F-20358a94893b, then compare using F-aa53e7865743. |
| `0x10000` | Extract using F-9ae11322d229, then compare using F-aa53e7865743. |
| `0x20000` | Compare unsigned 32-bit nested fields at offset `0x20`. |
| `0x40000` | Extract using F-9f3aa7cf475e, then compare using F-aa53e7865743. |
| `0x80000` | Compare unsigned 32-bit nested fields at offset `0x1c`. |
| `0x400000` | Compare unsigned 32-bit fields at `0xa0`. |
| `0x800000` | Compare unsigned 32-bit fields at `0xa8`. |

All direct numeric rows use ascending order.

For nested-field comparisons, each record contains a 32-bit nested address at offset `0x10`. The selected unsigned 32-bit field is read relative to that address. The left nested value is obtained before the right nested value; both obtained values are then compared without visible rereads.

For secondary selector `0x40`, a clear flag sorts before a set flag. Only the selected bit matters. The left flag is checked first, followed by the right flag on either branch.

The four extracted-pair secondary paths use the same left-then-right extraction order and five-argument comparison convention described above. They have no visible alternate-comparator flag check.

## Side effects

The body shows no direct writes to either record, nested objects, or global state. Its explicit writes are to temporary 32-bit storage used for extracted results and intermediate values.

That does **not** establish a read-only operation overall:

- Extractors receive record addresses and addresses of temporary output storage.
- Comparator dependencies receive opaque values that may themselves represent addresses.
- Dependencies may modify records, shared state, or memory reachable through their arguments.
- No preservation of global state is established.
- No dependency ownership or release behavior is established.

The body itself visibly performs no allocation, deallocation, reference-count operation, or explicit ownership transfer. Extracted words have unknown meanings and unknown ownership. There is no visible cleanup after comparison, including when a dependency returns an unusual result.

No explicit error reporting, rollback, or recovery path is present. Faulting memory accesses, dependency failures, and dependencies that do not return have no specified translation into a normal result.

## Dependencies

| Opaque function ID | Established role at this call site |
|---|---|
| F-9b584aabce3d | Extract two words for primary `0x1`. |
| F-7445c028e81c | Extract two words for primary `0x2`. |
| F-c99cba5deb0d | Extract two words for primary `0x4`. |
| F-05f08d4e44ad | Extract two words for primary `0x40`. |
| F-8ca4d70db38d | Extract two words for primary `0x80`. |
| F-2035045e72a2 | Extract two words for primary `0x2000`. |
| F-bfa281f5f170 | Extract two words for primary `0x80000`. |
| F-3c2e26a82b7c | Extract two words for primary `0x200000`. |
| F-16b769eefd01 | Extract two words for primary `0x4000000`. |
| F-0a40b63db184 | Extract two words for primary `0x8000000`. |
| F-809b8c8462b8 | Extract two words for secondary `0x800`. |
| F-20358a94893b | Extract two words for secondary `0x2000`. |
| F-9ae11322d229 | Extract two words for secondary `0x10000`. |
| F-9f3aa7cf475e | Extract two words for secondary `0x40000`. |
| F-aa53e7865743 | Compare extracted pairs with context; result forwarded. |
| F-01af74a46855 | Compare extracted pairs without context; result forwarded. |
| F-13e966bdb827 | First dependency called for primary `0x10`. |
| F-1ba216d0a747 | Second dependency called for primary `0x10` in the literal representation. |

Extractor return values are not visibly consumed. Successful use of extracted-pair paths requires the extractors to initialize both output words before returning; the caller does not visibly initialize them first.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless noted otherwise, assume valid memory, stable field contents, returning dependencies, and correctly initialized extractor outputs.

1. **Unsigned ordering:** with primary `0x100`, left field `0xffffffff` and right field `1` at offset `0x6c` produce `1`, not −1.

2. **Signed ordering:** with primary `0x20`, signed 16-bit values −1 and 1 at offset `0x7a` produce `0xffffffff`.

3. **Width boundary:** with primary `0x8`, unsigned 16-bit values `0xffff` and `0x0000` produce `1`.

4. **No tie-break:** with primary `0x400` and equal fields at `0x70`, the result is zero even if the secondary selector names a field whose values differ.

5. **Scaled equality:** with primary `0x800`, left short field at `0xb8` equal to 2 and right short field zero with fallback value 2000 produce zero. The left fallback field is irrelevant.

6. **Unsigned fallback range:** a left fallback value of `0xffffffff` compares greater than a right scaled value of 65,535,000.

7. **Alternate comparator:** with primary `0x2`, if both relevant flag bits are set after extraction, F-01af74a46855 is called. Its result is returned unchanged.

8. **Short-circuited flag read:** with primary `0x2` and the left relevant flag clear after extraction, comparison uses F-aa53e7865743 without a required right flag read. Right extraction still occurs.

9. **Forwarded result:** if F-aa53e7865743 returns the 32-bit representation of −7, the caller returns that same word, not normalized −1.

10. **Secondary flag order:** with primary zero and secondary `0x40`, a clear left bit and set right bit produce −1; equal bit presence produces zero regardless of other byte bits.

11. **Exact dispatch:** primary `0x3` returns `0xffffffff`, even with a recognized secondary selector. It does not combine primary operations `0x1` and `0x2`.

12. **Immediate equality:** primary `0x80000000` returns zero without visible record access.

13. **Special-path qualification:** under the literal supplied representation, primary `0x10` calls both dependencies in order and returns zero. Acceptance against an executable requires resolving the condition-state uncertainty below.

## Uncertainties

The following remain **documented_only**:

- **Special-path condition state:** primary `0x10` contains dependency calls followed by tests of an apparently pre-established condition. The literal representation makes those tests constant, but it does not independently establish whether underlying condition-state behavior is represented faithfully. There is no basis here to invent comparison semantics for the two dependencies.
- **Character signedness:** secondary `0x2` uses a plain character type. Ordering for bytes `0x80` through `0xff` depends on an unstated signedness convention. Values below `0x80` do not encounter this ambiguity.
- **Dependency contracts:** extracted-word meanings, comparison rules, valid contexts, mutation, allocation, lifetime, ownership, and failure behavior are unknown.
- **Concurrent or volatile memory:** no synchronization, atomicity, or stable-snapshot guarantee is shown. Repeated reads may matter if data changes; exact machine-load behavior is not established.
- **Record schema:** semantic field names, complete object sizes, alignment, byte order, and valid nested-object relationships are unknown.
- **Invalid memory and incomplete outputs:** no defined normal result is established for invalid addresses or extractors that fail to populate both words.
- **Ordering laws:** antisymmetry, transitivity, and equality consistency cannot be guaranteed across all selectors and dependencies. Unsupported selections already return the same negative word used for valid less-than results.

## Evidence

The established claims derive solely from the supplied static function representation and the supplied opaque dependency mapping: selector tests, access widths and offsets, explicit call order, argument forwarding, visible temporary writes, and return expressions.

Domain interpretations are identified as inferred intent. Missing dependency, platform, and condition-state information remains documented rather than filled in by assumption.

No implementation was executed. No runtime tests, dependency inspections, or executable-level validation have been performed.
