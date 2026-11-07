<!-- Trusted provenance: F-d2e82eeae580; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2e82eeae580

## Purpose

F-d2e82eeae580 converts a bounded sequence of bytes into a signed integer using a caller-selected radix and an externally supplied byte-to-digit mapping. It recognizes one optional leading sign and, on its ordinary completion path, writes the stopping address into a caller-owned cursor slot.

The return value is the numeric result, not a success indicator. Zero can represent accepted zero-valued digits, an invalid radix, an empty span, a sign-only span, or rejection before any digit is accepted.

Established distinctions from a general-purpose integer parser include:

- No leading whitespace is skipped.
- Digit values come from a table whose contents are not supplied.
- A fixed preliminary filter rejects some bytes without consulting that table.
- The lookup-base global is evaluated separately for every candidate requiring a lookup.
- A sign-only span leaves the caller’s cursor unchanged.
- A sign followed by a rejected candidate advances the caller’s cursor past the sign.
- No overflow detection is visible.

Optional-sign positional integer conversion is inferred intent. Conventional digit spellings, standard-library parsing semantics, and overflow behavior are not established.

## Interface

### Arguments and layout

The function receives three arguments:

1. **Cursor-slot address:** the address of a caller-owned 32-bit field containing the current input-byte address. The function reads this field on every call and writes a stopping address to it on specified paths.
2. **End address:** an input-byte address used as an exclusive scanning bound.
3. **Radix:** a signed 32-bit integer. Conversion is enabled only for values from 2 through 36, inclusive.

The cursor slot occupies four bytes in the supplied representation. No accompanying length, status, or error field is shown. Address encoding, byte order, and alignment requirements are not independently supplied.

### Return value

The return value is a signed 32-bit integer. It is formed by accumulating accepted digit values and then applying a recognized leading minus.

There is no separate failure result or digit count. The return value alone cannot distinguish an unsuccessful conversion from a successful conversion whose value is zero.

### Preconditions

For ordinary bounded use:

- The cursor slot must be readable on entry.
- It must also be writable when a writeback path is reached.
- The initial cursor and end address should delimit an ordered span within live byte-addressable storage.
- Every byte examined for sign classification or scanning must be readable.
- Whenever a lookup is reached, the lookup-base global and the selected table element must be readable.

These are caller obligations, not checks implemented by the function. There is no visible null-pointer validation, object-bound validation, or general validation of the lookup base.

The entry condition checks that the cursor differs from the end address; it does not establish that the cursor precedes the end address. Therefore, a reversed or otherwise invalid span must not be treated as safely rejected before an input read.

### Ownership and aliasing

The caller retains ownership of the cursor slot and input storage. The visible body performs no allocation, release, or ownership transfer.

There is no direct store through the scanning pointer. However, the final four-byte cursor-slot write may modify input bytes if the slot overlaps input storage. The interface establishes no non-aliasing restriction.

The end-address argument is not reassigned by the visible body. That does not imply that storage at or near that address is protected from an overlapping cursor-slot write.

## Behavior

### Entry and eligibility

The function first reads the address held in the cursor slot. Its initial accumulated value is zero.

Conversion is eligible only when:

- The initial cursor is not equal to the end address.
- The radix is greater than 1 and less than 37.

If either condition fails, the function returns zero without reading an input byte, reading the lookup-base global, accessing a table element, or writing the cursor slot. The initial cursor-slot read still occurs.

These conditions do not establish a valid memory span.

### Optional leading sign

For an eligible call, the first byte is checked for minus, `0x2D`. If it is not minus, it is checked for plus, `0x2B`.

- Minus selects negative-result processing and advances the internal cursor by one byte.
- Plus advances the internal cursor by one byte without selecting negative-result processing.
- Any other byte leaves the internal cursor at its initial address.

Only one leading sign is recognized. A later sign byte is an ordinary scanning candidate; its treatment depends on the preliminary filter and table mapping.

If consuming the sign places the internal cursor exactly at the end address, the function returns zero immediately, without cursor writeback. A one-byte plus or minus span is therefore unconsumed from the caller’s perspective.

If the post-sign cursor differs from the end address, processing proceeds to the scan. The subsequent loop separately requires the cursor to be below the end address. For valid ordered spans, this means that at least one byte remains after the sign.

A non-sign first byte is subsequently examined as the first scan candidate. The representation expresses a minus check, a conditional plus check, and a later candidate read; it does not establish a guaranteed number of physical machine loads.

### Preliminary candidate filter

The scan considers candidates while the internal cursor is strictly below the end address. Each candidate is interpreted as an unsigned 8-bit value, from 0 through 255.

The preliminary filter rejects exactly the following values:

| Byte value | Conventional interpretation |
|---|---|
| `0x00` | NUL |
| `0x09` | Horizontal tab |
| `0x0A` | Line feed |
| `0x0C` | Form feed |
| `0x0D` | Carriage return |
| `0x20` | Space |
| `0x80`–`0xFF` | High-bit bytes |

A preliminary rejection stops scanning at that candidate. The candidate is not consumed, and neither the lookup-base global nor a table element is read for it.

Every other value from `0x01` through `0x7F` proceeds to lookup. In particular:

- Vertical tab, `0x0B`, reaches the table.
- DEL, `0x7F`, reaches the table.
- Other low control bytes not listed above reach the table.

The function neither skips whitespace nor applies a general control-character rejection rule.

### Lookup-base evaluation and digit acceptance

For every candidate that passes the preliminary filter, the supplied expression evaluates the lookup-base global before reading the indexed table element.

The element address is formed using that base value and the candidate byte masked with `0x7F`. Because high-bit input bytes have already been rejected, this mask does not change any candidate that actually reaches lookup. Reachable table indices extend through 127.

The base expression is evaluated for each lookup. The supplied body does not establish a single base snapshot taken at function entry or reused throughout the scan. Its initialization, stability, and relationship to the table allocation are unknown.

The selected table element is a character-sized value converted to an integer. The candidate is rejected if the converted value is:

- Negative; or
- Greater than or equal to the radix.

Otherwise, it is accepted as a digit. Thus, every accepted digit value is at least zero and strictly less than the radix.

The table contents are unavailable. No particular mapping for decimal characters, letters, signs, or control bytes is established.

The element read uses a plain character type. The source does not independently establish whether that type is signed or unsigned on the target. Consequently, interpretation of table bytes with their high bit set remains conditional on the character-to-integer conversion. The rejection conditions apply to the resulting integer, not directly to the raw table byte.

### Accumulation and cursor advancement

For each accepted digit, the new accumulator is:

radix × previous accumulator + accepted digit value.

The accumulator starts at zero. The internal cursor advances by one byte only after the digit is accepted and the accumulator is updated.

No separate digit count or successful-conversion flag is maintained.

The accumulator and radix are represented as signed 32-bit integers. No overflow check, wider accumulator, saturation, or overflow-specific return path is visible. A portable result is therefore not established when intermediate multiplication, addition, or final negation falls outside the signed 32-bit range.

This limitation includes negative inputs whose mathematical final value would fit but whose positive accumulated magnitude would not. Accumulating 2,147,483,648 and only afterward applying minus is not established as a valid way to produce the signed 32-bit minimum.

### Termination and cursor writeback

Scanning terminates when:

- The internal cursor reaches the end address.
- The preliminary filter rejects a candidate.
- The converted table value is negative.
- The converted table value is at least the radix.

On ordinary completion, a recognized leading minus negates the accumulated value. The function then writes the internal stopping address into the cursor slot and returns the result.

The rejected candidate remains unconsumed. If every remaining candidate is accepted, the stopping address equals the end address.

Writeback does not require acceptance of a digit:

- Immediate rejection without a sign writes back the original cursor address.
- Immediate rejection after a sign writes back the address immediately following the sign.
- A sign-only span instead uses the earlier return path and performs no writeback.

These statements describe execution where the relevant accesses and arithmetic have defined behavior.

## Side effects

The source-expressed access order is:

1. Read the four-byte cursor slot.
2. If conversion is eligible, inspect the initial input byte for a sign.
3. If scanning reaches a candidate, read that candidate byte.
4. If the candidate passes the preliminary filter, read the lookup-base global.
5. Using that evaluated base, read the indexed character-sized table element.
6. If accepted, update the accumulator and advance the internal cursor.
7. Repeat candidate processing until termination.
8. Apply a recognized minus, then write the stopping address to the four-byte cursor slot on the ordinary completion path.
9. Return the result.

Rejected candidates do not advance the internal cursor. Preliminary rejection avoids both the base-global read and the table-element read for that candidate.

This ordering describes the supplied representation, not guaranteed physical load counts, synchronization, or volatile-access behavior.

The sole visible store targets the cursor slot. There is no direct store through the scanning pointer or through the lookup address. Nevertheless, cursor writeback may alter overlapping input storage. It may also affect other overlapping storage; the interface supplies no general disjointness guarantee.

The visible body makes no external function call and performs no allocation, deallocation, I/O, or explicit error reporting. Invalid memory accesses and arithmetic overflow are not translated into a defined diagnostic result by the visible logic.

## Dependencies

No called-function dependency appears in the supplied body. No additional opaque function ID is available or required for a callee.

The function depends on:

- **Lookup-base global:** evaluated on every lookup after preliminary filtering and before the indexed element read. Its storage layout, initialization, stability, and relationship to a valid table allocation are not established.
- **Digit-table storage:** supplies character-sized elements at the selected base and index. Contents, allocation extent, initialization, and mutability are unavailable. Index 127 is reachable.
- **Caller-provided storage:** the cursor slot and examined input bytes must support the accesses actually reached.
- **Target execution model:** determines address representation and ordering, plain-character conversion, and behavior outside the supported arithmetic range.

Because there are no visible callees, no assumptions about callee preservation of globals or callee ownership of memory are needed. The absence of calls does not establish that the lookup-base global or table contents are stable under external mutation.

## Acceptance examples

These are static behavioral criteria, not executed tests. Unless stated otherwise, assume valid ordered storage, valid lookup accesses where reached, and representable arithmetic.

### Cases independent of table contents

| Condition | Required result and cursor effect |
|---|---|
| Cursor equals end; radix 10 | Return zero; read the cursor slot but no input byte; no writeback. |
| Nonempty span; radix 1 | Return zero; no input, lookup-base, or table read; no writeback. |
| Nonempty span; radix 37 | Same early-return behavior as radix 1. |
| Span contains only plus | Return zero; leave the cursor slot unchanged. |
| Span contains only minus | Return zero; leave the cursor slot unchanged. |
| First byte is space; radix 10 | Return zero; write the original cursor address; no lookup-base or table read. |
| Span is plus followed by space | Return zero; write the address of the space; no lookup for the space. |
| First byte is `0x80`; radix 10 | Return zero; write the original cursor address; no lookup-base or table read. |
| First byte is NUL, with later bytes before end | Stop at NUL without examining later candidates or performing a lookup for NUL. |

### Cases conditional on lookup results

Assume each relevant lookup yields conventional values for the bytes written as `1`, `2`, and `3`:

- `123` in radix 10 returns 123 and writes the end address.
- `-12` in radix 10 returns −12 and writes the end address.
- `+12` in radix 10 returns 12 and writes the end address.
- `123` in radix 3 accepts values 1 and 2, rejects value 3, returns 5, and writes the address of the third byte.

For each of those candidate lookups, the base global is evaluated before the element is read. The examples assume the resulting accesses yield the stated values; they do not establish base stability.

Additional conditional criteria:

- A candidate whose converted table value equals the radix is rejected without consumption.
- A candidate whose converted table value is negative is rejected without consumption.
- If vertical tab maps to zero, a one-byte vertical-tab span in radix 10 is consumed, returns zero, and writes the end address.
- If the second plus in a two-plus span maps to a negative value, the first plus is consumed as a sign, the second is rejected, and the returned cursor points to the second plus.
- DEL reaches lookup. Its acceptance depends on the converted table value, not a preliminary DEL exclusion.

If the cursor slot overlaps input storage and ordinary writeback occurs, the four-byte write may change those overlapping input bytes. Their resulting byte values depend on the target address encoding; preservation of those bytes is not an acceptance requirement.

No exact numeric result is specified for an overflowing accumulation or negation.

## Uncertainties

- **documented_only — Lookup-base global:** Initialization, stability, representation, and relationship to the table allocation are unavailable. Per-lookup evaluation is established; a stable base is not.
- **documented_only — Table contents and extent:** Digit spellings, case handling, control-byte mappings, and sufficient backing allocation are not supplied.
- **documented_only — Character signedness:** High-bit table-byte conversion is not independently established.
- **documented_only — Overflow:** No detection is visible. Wrapping, trapping, or another target-specific outcome is not established.
- **documented_only — Invalid spans:** Null, dangling, reversed, unrelated-object, and wrapping-address cases lack a supported safety contract.
- **documented_only — Aliasing:** Cursor writeback can affect overlapping input storage. Broader guarantees for overlapping slot, global, and table storage are not supplied.
- **documented_only — Concurrency and special memory:** No synchronization, atomicity, volatile-access, or memory-mapped-I/O contract is established.
- **Inferred intent:** Bounded positional integer conversion with an optional sign and radix 2–36. This does not establish conventional table mappings or standard parser compatibility.

## Evidence

The supplied static representation establishes the entry-slot read, radix bounds, optional-sign handling, exact preliminary rejection set, per-lookup base-global evaluation, indexed character read, digit acceptance conditions, accumulation order, and conditional cursor writeback.

It also establishes the absence of a direct store through the scanning pointer. It does not establish disjointness between the cursor slot and input storage, so input preservation cannot be asserted unconditionally.

The supplied material does not include the lookup-base initialization, table contents, authoritative character-signedness configuration, or an overflow execution model.

No runtime tests have been performed. All acceptance examples are derived static conditions, not observed executions.
