<!-- Trusted provenance: F-09dcb55e8e0a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-09dcb55e8e0a

## Purpose

F-09dcb55e8e0a produces an unsigned result from an unsigned input and an externally stored configuration table. The table controls both the progressive reduction of the input and the bits contributed to the result.

The established behavior is a five-level, table-configured transformation. A level is selected when its mask shares at least one set bit with the input value remaining at that point. Each selected level contributes its entire adjustment word to the result through bitwise OR. The low eight bits of that same word determine how far the remaining input is shifted right.

This is not unconditionally a logarithm, bit count, or highest-set-bit operation. Highest-set-bit classification is an **inferred intent** supported by suitable table contents, not an established property of the supplied configuration. The actual table contents are unavailable.

## Interface

### Explicit input and return value

The function receives one unsigned integer by value and returns an unsigned integer of the same represented type.

The supplied representation names an unsigned word type but does not provide its definition. Four-byte spacing between table entries supports a 32-bit interpretation; it does not independently establish the type width or all target execution rules.

Consequently:

- The numerical layouts and examples in this contract assume 32-bit unsigned words.
- Those examples apply only where that width and the stated execution assumptions hold.
- The general distinction between the full adjustment word and its low-eight-bit shift count does not depend on assuming a particular table configuration.

There is no output-buffer parameter. The return value is an accumulated bit field, not the final reduced input.

### Implicit configuration input

An external storage location supplies the table base address. Under the 32-bit interpretation, the nominal table layout is:

| Level | Mask byte offset | Adjustment byte offset |
|---|---:|---:|
| 0 | 0 | 20 |
| 1 | 4 | 24 |
| 2 | 8 | 28 |
| 3 | 12 | 32 |
| 4 | 16 | 36 |

Offsets are relative to the base used for the particular lookup. With one stable base, the layout contains ten words spanning 40 bytes. Each adjustment is 20 bytes beyond its corresponding mask.

Levels have descending priority: **4, 3, 2, 1, 0**.

The representation does not establish that the base is captured once for the whole invocation. A mask lookup and its associated adjustment lookup each refer to the externally stored base. A stable, single-table interpretation therefore requires base stability as an additional assumption.

### Preconditions for defined completion

The following are required for an ordinary, defined execution:

- The external base storage is readable whenever consulted.
- Every accessed mask address permits a read of the represented unsigned-word type.
- Every selected adjustment address permits the corresponding word read.
- Address formation and alignment satisfy the execution environment.
- Every executed right shift has defined semantics.

An unselected adjustment does not need to be read. The nominal table span therefore describes the configuration layout, not an unconditional requirement that all ten words be accessed during every call.

Under ordinary 32-bit language-level shift rules, every executed count must be between 0 and 31 inclusive. The function does not validate that requirement.

No ownership transfer is part of the interface. The argument is a value; the function neither accepts nor returns an owned memory object.

## Behavior

### Selection and priority

The observable selection sequence is fixed: level 4 is considered before level 3, continuing down to level 0. Successful selection does not terminate processing.

Each mask is compared with the input remaining after any higher-priority reductions. A nonzero bitwise intersection selects the level. Sharing one set bit is sufficient; the mask need not be contained entirely within the remaining input.

An unselected level contributes nothing and does not alter the remaining input. Its adjustment word is not read.

A zero mask cannot select a level. Conversely, a nonzero mask does not guarantee selection: it must overlap the remaining input, not merely the original argument.

### Coupled reduction and result contribution

A selected level uses one loaded adjustment word for two different purposes:

| Purpose | Relevant part of the adjustment |
|---|---|
| Right-shift distance | Low eight numerical bits |
| Result contribution | Entire unsigned word |

The reduction precedes the result update in the supplied representation. Later masks therefore observe the reduced input.

These two uses are not interchangeable. In particular, a word with a zero low byte can contribute nonzero result bits without reducing the input. Under the 32-bit interpretation, `0x00000100` contributes that full value while selecting a zero-bit shift.

The low eight numerical bits are not necessarily the first byte at the lowest memory address. Their location in storage depends on byte order, which is not established.

Within a selected level, both uses refer to the same loaded adjustment value. There is no separate adjustment read for the shift and for accumulation.

### Shift boundaries

The extracted shift distance ranges from 0 through 255.

Under the conditional 32-bit unsigned interpretation:

- Distance 0 preserves the remaining input.
- Distances 1 through 31 discard low-order bits and introduce zeros at the high end.
- Distances 32 through 255 are outside the defined range of an ordinary language-level shift on a 32-bit value.

The supplied material does not establish a target-specific rule for excessive counts. In particular, neither modulo-32 masking nor saturation may be assumed.

The size of the complete adjustment word is not itself the validity criterion. For example, `0x80000000` selects distance 0, whereas `0x00000120` selects distance 32. The former has a defined shift under the stated assumptions; the latter does not if its level is selected.

If a defined reduction makes the remaining input zero, all later masks fail to select their levels. Their mask reads still occur during ordinary completion, but their adjustment reads do not.

### Result meaning and completion

The result begins at zero and accumulates selected adjustment words through bitwise OR.

Accordingly:

- A contributed bit remains set.
- Repeated contributions of the same bit do not add numeric weight.
- Overlapping contributions are not equivalent to arithmetic addition.
- Bits outside the shift-count field remain eligible for inclusion in the return value.
- The result need not be bounded by the input width minus one.

There is no visible final normalization, conversion to a bit position, or special handling of zero.

For a zero input, all five mask intersections are zero. Subject to valid mask accesses, the function returns zero without reading any adjustment.

A nonzero input can also return zero. This happens when no level is selected, or when all selected adjustments are zero. Zero is therefore an ordinary result, not an error indicator.

## Side effects

### Reads and their ordering

With stable configuration storage and defined execution, the table access sequence is:

| Priority | Required access | Conditional next access |
|---|---|---|
| First | Level 4 mask | Level 4 adjustment, if selected |
| Second | Level 3 mask | Level 3 adjustment, if selected |
| Third | Level 2 mask | Level 2 adjustment, if selected |
| Fourth | Level 1 mask | Level 1 adjustment, if selected |
| Fifth | Level 0 mask | Level 0 adjustment, if selected |

An associated adjustment read occurs after its selecting mask read and before the next level’s mask read. Ordinary completion entails five logical mask-word reads and zero through five logical adjustment-word reads.

Each lookup refers to the external base storage. The representation does not provide a snapshot or synchronization mechanism. If changes between accesses are meaningful and permitted by the execution environment, a single invocation need not use one coherent table image. Unsynchronized concurrent mutation is not established as safe.

These statements describe the supplied representation’s access semantics. They do not guarantee physical transaction counts, access atomicity, compiler reload behavior for ordinary memory, or the absence of speculative hardware reads.

### Writes and ownership

No external write is visible. In particular, the function does not visibly modify the configuration table, its base storage, or caller-provided memory.

Changes are confined to the function’s working input, accumulated result, and progression through the levels. The caller’s by-value argument is not written back.

No allocation, release, retention, or pointer return is visible. These observations do not establish the lifetime or ownership of the external table; they establish only that this function contains no visible operation changing that ownership.

### Failure behavior

There is no explicit validation path, error code, or recovery behavior for:

- Unreadable configuration storage.
- Invalid or misaligned table addresses.
- Invalid address arithmetic.
- An executed shift outside the environment’s defined range.

Such conditions are outside the ordinary defined-execution contract. The supplied material does not determine whether a particular target traps, faults, or exhibits another outcome.

## Dependencies

No function calls appear in the supplied body. No opaque dependency function IDs beyond F-09dcb55e8e0a are therefore identified.

Its external dependencies are data and execution rules:

- The stored table base.
- Five mask words.
- The adjustment words whose levels are selected.
- The unsigned-word type’s width and representation.
- Address arithmetic, alignment, byte order, and shift semantics.

The level-relative offset calculations range from 0 through 36 bytes. The small fixed offsets do not establish that adding them to the external base is valid; base representation and address-range constraints remain environmental requirements.

No unknown callee is involved, so the contract requires no assumption that a callee preserves globals or releases memory. Configuration initialization, lifetime, and stability remain unresolved independently of the absence of calls.

## Acceptance examples

All examples below are **conditional static deductions**, not observed executions. Unless explicitly discussing an invalid shift, they assume 32-bit unsigned words, stable readable configuration storage, valid address formation, and defined shifts.

### No selection

With input `0xFFFFFFFF` and all five masks zero:

- The return value is zero.
- Mask reads follow descending level order.
- No adjustment is read, regardless of adjustment contents.

With input zero and any valid mask words, the same return value and adjustment-read behavior apply. Zero input does not eliminate mask reads.

### Later selection depends on the reduced input

Configure level 4 with mask `0x00000010` and adjustment 4. Configure level 3 with the same mask and adjustment 1. Set the remaining masks to zero.

For input `0x00000010`:

- Level 4 selects and reduces the remaining input to 1.
- Level 3 does not select because its mask does not overlap 1.
- The return value is 4, not 5.
- Level 3’s adjustment is not read.

This distinguishes selection against the remaining input from selection against the original argument.

### High result bits and repeated contributions

Configure levels 4 and 3 with mask 1 and adjustment `0x00000100`. Set all other masks to zero.

For input 1, both configured levels select because neither performs a nonzero shift. The return value is `0x00000100`, not `0x00000200`.

This establishes both full-word contribution and OR accumulation.

### Conditional highest-set-bit configuration

Consider the following hypothetical table:

| Level | Mask | Adjustment |
|---|---:|---:|
| 0 | `0x00000002` | 1 |
| 1 | `0x0000000C` | 2 |
| 2 | `0x000000F0` | 4 |
| 3 | `0x0000FF00` | 8 |
| 4 | `0xFFFF0000` | 16 |

Its conditional results include:

| Input | Return |
|---|---:|
| `0x00000000` | 0 |
| `0x00000001` | 0 |
| `0x00000002` | 1 |
| `0x00000003` | 1 |
| `0x00000100` | 8 |
| `0x00010000` | 16 |
| `0x80000000` | 31 |

For the last input, the successive contributions are 16, 8, 4, 2, and 1. Their OR is 31. These expectations support the inferred highest-set-bit interpretation for this fixture only; they are not evidence of actual configuration contents.

### Shift validity is conditional on selection

If a selected level’s adjustment is 32, its extracted shift distance is 32. No defined return value is specified under ordinary 32-bit language-level rules.

If that same level is not selected, its adjustment is not read and no shift is performed for it. Merely storing 32 in an unselected adjustment does not introduce that shift.

## Uncertainties

- **documented_only — Unsigned-word width:** The type definition is absent. Four-byte table spacing supports, but does not conclusively establish, the 32-bit interpretation.
- **documented_only — Actual configuration:** No table values or initialization behavior are supplied.
- **documented_only — Address semantics:** Base representation, valid ranges, alignment, byte order, and exceptional-access behavior are unspecified.
- **documented_only — Excessive shifts:** Target-specific treatment of counts outside the ordinary width bound is unresolved.
- **documented_only — Configuration stability:** No volatility, atomicity, synchronization, or coherent-snapshot guarantee is established.
- **documented_only — Storage ownership:** The table’s owner and lifetime are unknown. No ownership change is visible in this function.
- **Inferred intent:** Highest-set-bit classification is plausible only under suitable masks and adjustments.
- **Not established:** An explicit error protocol, external writes, allocation, logging, or release behavior.

## Evidence

Established static evidence consists of the unsigned value interface, zero initial result, five descending mask-selection levels, four-byte entry spacing, and a 20-byte displacement between each mask and its adjustment.

The supplied body also establishes conditional adjustment access, selection by nonzero bit overlap, use of the low eight adjustment bits for shifting, full-word OR contribution, and return of the accumulated result. No function calls or external writes are visible.

The 32-bit numerical interpretation and highest-set-bit fixture are explicitly conditional. Configuration values and target-specific execution details are not supplied.

No runtime tests have been performed. All acceptance examples are deductions under their stated assumptions.
