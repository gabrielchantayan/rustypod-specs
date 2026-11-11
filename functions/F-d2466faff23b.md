<!-- Trusted provenance: F-d2466faff23b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2466faff23b

## Purpose

F-d2466faff23b converts a counted sequence of 16-bit input units into a byte sequence whose first byte records its length. The output holds at most 31 payload bytes. Some input units produce a new payload byte; certain combining-mark inputs instead replace the preceding payload byte.

The established behavior includes configurable classification, table-based mappings, fixed mappings, replacement bytes, and a configurable error result. The configuration values and table contents are not supplied.

The apparent intent is conversion from Unicode-like input into a legacy single-byte encoding. That interpretation is inferred from the character values and mappings, not an established encoding declaration. The contract therefore describes numeric input units and output bytes without assigning a definitive encoding name.

## Interface

### Inputs

The function receives three arguments:

1. **Input address:** an address from which unsigned 16-bit units are read sequentially.
2. **Input count:** a signed 16-bit value. Zero requests no input processing. Positive values specify the number of units to process, subject to output capacity. Negative values are not rejected.
3. **Output address:** the start of a writable byte buffer.

The function returns a **32-bit result value**.

### Output layout

| Byte offset | Meaning |
|---|---|
| 0 | Number of payload bytes produced, from 0 through 31 |
| 1 through recorded length | Converted payload |
| Beyond recorded length | Not initialized or cleared by this function |

There is no terminating zero byte. Payload bytes may themselves be zero.

A 32-byte output buffer accommodates the maximum result. A smaller buffer is sufficient only if the caller can independently establish the maximum payload length for the particular input and configuration.

### Preconditions and ownership

The caller must provide readable input storage for every unit that processing reaches, writable output storage for every byte written, and valid readable configuration and mapping tables.

The input count is a count of 16-bit units, not bytes, scalar Unicode characters, or terminated text. Embedded zero units do not terminate processing.

The function allocates and releases no memory and calls no external functions. It does not transfer ownership of either buffer.

Input/output overlap is not checked. Processing reads input incrementally and writes output incrementally, so overlapping output can alter input not yet read. No snapshot semantics are provided. Stable input, stable configuration, and non-overlapping buffers are the safe calling conditions.

Input byte order and alignment requirements depend on the target’s 16-bit memory access representation; they are not established by the supplied material.

## Behavior

### Initialization and finalization

Before processing, the function obtains a 32-bit classification mask from configuration. It initializes:

- Payload length to zero.
- Result value to zero.
- Remembered predecessor value to zero.
- The output position to the beginning of the buffer, immediately before the first payload position.

A zero count skips input processing. Even then, the mask is read and the output length byte is written as zero.

At normal completion, the recorded payload length is written to output offset 0. This header write occurs after all payload writes.

If processing stops with the remaining input count nonzero, the final result is forced to `0x0000003F`. Otherwise, the result is zero or the configured conversion-error result, according to the paths taken.

### Classification and direct conversion

For each attempted input unit, the function first reads its unsigned 16-bit value. It extracts the low byte and tests the full value against the cached classification mask.

If their bitwise intersection is zero:

- The low byte is the candidate output.
- That low byte becomes the remembered predecessor after a successful write.
- No mapping table or replacement-error handling is used.

Consequently, direct conversion can truncate a nonzero high byte if the configured mask permits it. A directly copied `0x3F` is ordinary successful output, not an error marker.

If the intersection is nonzero, the function uses the special conversion rules below. Every successfully written unit handled through these rules clears the remembered predecessor to zero, including a combining replacement.

The mask therefore gates all special mappings. A numeric input value alone is insufficient to establish that a mapping is taken.

### Table-based mappings

Subject to the mask selecting special conversion, these ranges use byte-valued tables:

| Input range | Table index |
|---|---|
| `0x00A0`–`0x00FF` | Input minus `0x00A0` |
| `0x02C0`–`0x02DF` | Input minus `0x02C0` |
| `0x2010`–`0x20AF` | Input minus `0x2010` |
| `0x2200`–`0x2268` | Low byte of input |

Each of these mappings normally appends one payload byte.

For a table result other than `0x3F`, the byte is accepted without changing the result value. A table result equal to `0x3F` is still written but sets the result to the configured conversion-error value.

The range checks are unsigned. For example, values below `0x02C0` do not qualify by wrapping the subtraction into the accepted interval.

Inputs in the same high-byte groups but outside the listed ranges take the conversion-error path. In particular:

- Special-selected values in `0x0000`–`0x009F` are errors.
- Special-selected values in `0x0200`–`0x02BF` and `0x02E0`–`0x02FF` are errors.
- Special-selected values in `0x2000`–`0x200F` and `0x20B0`–`0x20FF` are errors.
- Special-selected values in `0x2269`–`0x22FF` are errors.

These rejected values do not proceed to the fixed-mapping fallback.

### Combining replacements

Special-selected inputs `0x0300` through `0x030A`, inclusive, attempt a table-based replacement of the preceding payload byte.

The remembered predecessor must be in the inclusive range `0x41`–`0x79`. This is a numeric range, not an alphabetic test: it includes the punctuation between uppercase and lowercase letters and excludes `0x7A`.

For an eligible predecessor, the function:

1. Reads an unsigned byte from a combining table at offset “predecessor minus `0x40`.”
2. Adds that byte to the input unit’s low byte.
3. Reads the replacement byte at that resulting offset from the same table base.

Both table reads return unsigned bytes. The second offset uses their promoted numeric values; it is not reduced modulo 256.

An eligible combining conversion replaces the current final payload byte rather than appending. The payload length is unchanged after the replacement completes.

If the replacement table returns `0x3F`, the function still replaces the preceding byte with `0x3F` and sets the configured conversion-error result. It does not preserve the previous byte or append a separate error byte.

An ineligible predecessor causes the combining input to append `0x3F` through the ordinary conversion-error path.

Special-selected input `0x0327` has two explicit replacement cases:

| Remembered predecessor | Replacement byte |
|---|---|
| `0x43` | `0x82` |
| `0x63` | `0x8D` |

For any other predecessor, `0x0327` appends `0x3F` and records a conversion error.

Remembered predecessor state comes only from the immediately preceding successfully written direct-conversion unit. A successful special conversion clears it. Thus, a second combining mark does not automatically combine again with the byte produced by the first.

### Other mappings in the `0x0300` high-byte group

Two special-selected values append fixed bytes:

| Input | Output |
|---|---|
| `0x03A9` | `0xBD` |
| `0x03C0` | `0xB9` |

Other values in this high-byte group, apart from the combining cases above, append `0x3F` and record a conversion error.

### Fixed-mapping fallback

Special-selected values in high-byte groups not handled above enter a fallback decision.

A configured 32-bit comparison value controls this decision:

- Equality with the input produces `0xAA`.
- Otherwise, a signed 32-bit comparison selects one of two fixed-mapping groups.

The unsigned 16-bit input is represented as a nonnegative 32-bit value for that signed comparison. The configured comparison value is interpreted as signed 32-bit.

If the input is greater than the configured signed value:

| Input | Output |
|---|---|
| `0x25CA` | `0xD7` |
| `0xF8FF` | `0xF0` |
| `0xFB01` | `0xDE` |
| `0xFB02` | `0xDF` |

If the input is less than the configured signed value:

| Input | Output |
|---|---|
| `0x0131` | `0xF5` |
| `0x0152` | `0xCE` |
| `0x0153` | `0xCF` |
| `0x0192` | `0xC4` |

An unmatched value appends `0x3F` and records a conversion error.

The comparison-group condition is significant. A listed numeric value is not unconditionally mapped if configuration places it in the opposite group. Equality takes priority and produces `0xAA`.

### Error accumulation and capacity

A conversion error assigns the configured 32-bit error value to the result. Processing continues. Successful conversions do not clear that result.

With stable configuration, this acts as a sticky error result. The configured value could itself be zero or `0x3F`; neither is excluded by the evidence.

Conversion selection, including applicable table reads and error-result assignment, occurs **before** the capacity check.

For an append operation:

- A current length of 0–30 permits the write.
- A current length of 31 stops processing before the new payload byte is written.

For an eligible combining replacement, the length is temporarily reduced before this check and then restored by the successful replacement write. Consequently, a replacement can succeed when the payload already contains 31 bytes.

After a successful payload write, the function advances the input address by one 16-bit unit and decrements the signed 16-bit remaining count. A capacity-rejected unit is not consumed.

If processing stops for capacity, the remaining count is nonzero and the returned result becomes `0x0000003F`, overriding any earlier configured error result.

Negative initial counts are not treated as empty input or as a separate argument error. They enter processing and ordinarily reach the capacity stop with sufficient valid storage. The signed narrowing behavior on decrementing the minimum 16-bit value is not established as a portable arithmetic contract.

## Side effects

The externally relevant access sequence is:

1. Read the classification mask.
2. For each attempted unit, read the input value.
3. Perform applicable classification, configuration, and table accesses.
4. Check capacity, accounting for a pending combining replacement.
5. If accepted, write one payload byte, either at the next position or over the current final byte.
6. After processing stops, write the length byte at output offset 0.

One low-high-byte special-conversion branch explicitly re-reads the current 16-bit input to test whether its high byte is zero. The routine therefore does not establish exactly one source read per attempted unit. Under stable input this re-read does not change the branch outcome.

Capacity rejection can occur after an input read and mapping-table reads. A caller must not assume that full output prevents further source or configuration access.

There are no explicit global writes, allocations, deallocations, callbacks, or external function calls. Buffer writes are immediate rather than transactional; an eventual error result does not roll them back.

## Dependencies

No external callable dependencies are present. There are therefore no additional opaque function IDs to enumerate.

The data dependencies are:

- A 32-bit classification mask, cached at entry.
- A table for `0x00A0`–`0x00FF`, requiring 96 byte entries.
- A table for `0x02C0`–`0x02DF`, requiring 32 byte entries.
- A table for `0x2010`–`0x20AF`, requiring 160 byte entries.
- A table for `0x2200`–`0x2268`, requiring 105 byte entries.
- A combining table supporting the two-stage byte lookup.
- A 32-bit conversion-error result value.
- A 32-bit comparison value used by the fixed-mapping fallback.

For the established combining-input range, first-stage offsets are 1–57. A first-stage byte can be as large as 255, and the mark’s low byte can be as large as 10, so second-stage offsets can reach 265. Actual reachable offsets depend on the unavailable table contents.

These dependencies must remain readable throughout processing. Only the classification mask is explicitly cached once at entry; uniform snapshot semantics for the other configuration data are not established.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Buffers are assumed valid, non-overlapping, and stable.

1. **Empty input.** With count zero, the function writes zero to output offset 0 and returns zero. It reads no input units and writes no payload bytes.

2. **Direct byte output.** If the mask does not select special conversion for input `0x0041`, a one-unit input produces length 1, payload `0x41`, and result zero.

3. **Direct truncation.** If the mask does not select special conversion for `0x1234`, the output byte is `0x34`. No error is recorded merely because the input has a nonzero high byte.

4. **Table success.** If `0x00A0` selects special conversion and its table entry is `0x80`, a one-unit input produces length 1 and payload `0x80`, returning zero.

5. **Table replacement error.** Under the same conditions, but with table entry `0x3F`, the payload is `0x3F` and the returned result is the configured error value.

6. **Combining replacement.** If `0x0043` is direct and following `0x0327` is special-selected, the two-unit input produces length 1 and payload `0x82`, returning zero.

7. **Combining without a predecessor.** A single special-selected `0x0327` produces length 1 and payload `0x3F`, returning the configured error value.

8. **Predecessor upper boundary.** A direct `0x007A` followed by special-selected `0x0300` does not use the combining table. It produces two bytes, `0x7A` and `0x3F`, and records a conversion error.

9. **Combining-table error replaces.** If direct `0x0041` is followed by special-selected `0x0300`, and the combining lookup yields `0x3F`, the final length is 1, not 2. The sole payload byte is `0x3F`, and the configured error is returned.

10. **Capacity stop.** Thirty-two ordinary append-producing units produce only the first 31 payload bytes. The function returns `0x0000003F`; the rejected unit may already have been read and classified.

11. **Replacement at capacity.** Thirty-one direct `0x0043` units followed by special-selected `0x0327` consume all 32 units. The length remains 31, the final byte becomes `0x82`, and the result remains zero.

12. **Predecessor reset.** Direct `0x0043`, special-selected `0x0327`, and another special-selected `0x0327` produce payload `0x82, 0x3F`. The second mark cannot reuse the original predecessor.

## Uncertainties

The following are **documented_only**:

- Actual mask, table contents, comparison value, and conversion-error value.
- Definitive source and destination character-encoding names.
- Target byte order and alignment requirements.
- Portable behavior of signed 16-bit count narrowing outside its representable range.
- Behavior under concurrently changing input or configuration.
- Any caller-level guarantees concerning buffer capacity, overlap, or count validity.

No validation is visible for null pointers, inaccessible memory, malformed configuration, or insufficient storage. Such cases must not be interpreted as producing a defined error result.

## Evidence

The established behavior comes solely from static inspection of the supplied representation of F-d2466faff23b: argument widths, branch conditions, numeric mappings, table-index arithmetic, payload writes, predecessor updates, count updates, and return selection.

The legacy-encoding interpretation is inferred intent. Concrete mapping contents and target execution details remain unknown.

No runtime tests, traces, or implementation-access checks have been performed.
