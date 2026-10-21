<!-- Trusted provenance: F-fe0df18e153a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe0df18e153a

## Purpose

F-fe0df18e153a consumes a count-limited sequence of unsigned 16-bit units through a caller-owned cursor and returns a decimal-position-weighted numeric result. Consumption changes the caller’s cursor incrementally.

For units representing the decimal digits by numeric values 48 through 57, the result is the corresponding decimal number, subject to arithmetic limits. Decimal conversion is **inferred intent**, not an input restriction: the function also consumes other unit values without rejecting them.

The established behavior is count-driven. There is no search for a delimiter or terminator, no whitespace handling, no sign recognition, and no digit validation. A zero-valued unit is data, not an end marker.

## Interface

### Inputs and result

The function receives:

- A reference to a caller-owned cursor cell. The cell contains the address used to obtain input and is writable by this function.
- A signed 32-bit requested count.

The result is a signed 32-bit integer. It carries the numeric accumulation, not a separate success or failure status.

A positive count requests exactly that many unit consumptions, provided execution completes normally. A zero or negative count produces zero without reading or writing the cursor cell and without reading input.

There is no separate buffer length, output buffer, error destination, or ownership parameter.

### Storage layout

Each input unit occupies two bytes and is interpreted as an unsigned 16-bit value, with possible values from 0 through 65,535. Ordinary progression moves the cursor forward by two bytes per consumed unit.

The cursor cell is represented as a four-byte object whose contents are used as an address. This supports an apparent 32-bit address representation; it does not establish that every four-byte value denotes an accessible address.

The evidence does not establish byte order, alignment requirements, a text encoding, or the complete address model. In particular, consuming a 16-bit unit does not imply Unicode decoding, surrogate handling, or character normalization.

### Preconditions for ordinary completion

For a positive count, the cursor cell must permit the required reads and writes. Every address actually selected for input must permit a two-byte load under the target’s memory and alignment rules. Each cursor advance must also be valid under the target’s address rules.

Arithmetic must remain within the execution model’s defined behavior if a definite numeric result is required.

A readable region of twice the requested count in bytes is sufficient for sequential input access only under additional assumptions: the cursor cell remains stable except for this function’s updates, does not overlap affected input, and advances without an address-boundary problem. The byte-count expression here is mathematical; it is not a promise that doubling every positive count fits in a signed 32-bit quantity.

The function does not acquire ownership of either the cursor cell or the input storage. It neither allocates nor releases them.

## Behavior

### Consumption boundary

Nonpositive counts cause an immediate zero result without dereferencing the cursor reference. A negative count does not request backward traversal, become a large unsigned length, or represent an error return.

For positive counts, the requested count determines the number of units consumed. Input contents do not shorten or extend that number. There is no independent maximum-length restriction or check against accessible storage.

The count reduction needed to reach zero from a positive signed 32-bit count does not itself require signed overflow.

### Observable access ordering

Within each consumption, the input value is obtained before the caller’s cursor is changed. The cursor is then advanced before the numeric accumulation for that unit is updated. Loop-control progress follows that numeric update.

The representation contains a cursor-cell read for selecting the input address and another cursor-cell read for selecting the address to advance. The advancement therefore is not expressed as a single saved-address operation covering both purposes.

This distinction matters if something can change the cursor between those logical reads. The address supplying the unit and the address from which advancement proceeds need not then be the same. No consistency check, locking operation, or snapshot mechanism is visible.

These are source-visible logical accesses and ordering relationships. They do not guarantee separate machine instructions or bus transactions. The cursor cell is not identified as volatile or atomic, and the evidence does not define safe concurrent mutation.

Cursor changes are incremental, not deferred until successful completion. There is no saved initial cursor used for rollback.

### Numeric meaning

The accumulation starts at zero. Each consumed unit contributes its unsigned numeric value offset by 48, with earlier contributions receiving one additional decimal place for every later consumed unit.

Equivalently, in an arithmetic model where all relevant operations and conversions are valid, the result is a positional sum: each unit’s value minus 48 is weighted by a power of ten determined by how many units follow it.

For values 48 through 57, each contribution is a digit from zero through nine. Leading units with value 48 therefore do not change the eventual numeric value, although they are consumed and advance the cursor.

Other values participate in the same arithmetic. For example, a single unit with value 58 contributes ten, and a single unit with value 65 contributes seventeen. Neither is rejected as a non-digit. Contributions are not restricted to one decimal digit and can exceed nine or, mathematically, be negative.

A zero-valued unit contributes mathematically negative forty-eight. Whether a resulting negative value is returned as the corresponding signed number depends on the mixed-width arithmetic and conversion rules described below.

### Arithmetic widths and limits

The input load is unsigned 16-bit. Its contribution is explicitly widened to unsigned 32-bit arithmetic. The accumulator and return value are signed 32-bit.

The decimal multiplication is a signed operation on the previous accumulator. Its result then participates in arithmetic involving the unsigned 32-bit contribution. The negative offset also participates in that mixed expression.

Consequently, the function must not be specified as either unlimited mathematical arithmetic or unconditional 32-bit wrapping arithmetic:

- Signed multiplication can overflow before unsigned arithmetic would determine a wrapped result.
- In a conventional interpretation with 32-bit unsigned arithmetic, the unsigned portion is calculated modulo \(2^{32}\).
- Storing a value outside the signed 32-bit range back into the signed accumulator requires behavior not fully established by the supplied evidence.
- A final mathematical result fitting the signed range does not, by itself, establish that every intermediate multiplication and conversion is valid.

There is no explicit overflow check, saturation rule, range-error result, or recovery branch.

A one-unit input of value 65,535 is unproblematic numerically: its contribution is 65,487, which fits the signed result range. Longer inputs require attention to the accumulated value, not merely to each unit’s width.

### Cursor postcondition

With a stable cursor, non-overlapping storage, valid ordinary address arithmetic, and normal completion, consuming \(n\) units leaves the cursor \(2n\) bytes beyond its initial address.

That postcondition is conditional. If the cursor cell overlaps input storage, its writes can alter units that would otherwise be read later. If permitted external activity changes the cursor, subsequent accesses use the values encountered during execution rather than a guaranteed contiguous sequence.

The evidence does not establish the validity of every possible overlap between the differently interpreted objects. It establishes the access pattern, not a universal aliasing contract.

## Side effects

The explicit write target is the caller’s cursor cell. There is no separate explicit write through the input address. Input storage nevertheless can be changed indirectly if it overlaps the cursor cell.

Under ordinary stable, non-overlapping conditions, every completed consumption performs one cursor update and leaves input contents unchanged. Nonpositive counts perform no cursor or input access.

No visible allocation, release, callback, logging operation, or named global-state access occurs. There are no visible function calls that could add hidden ownership or global-state effects.

Failures are not reported through a distinct status channel. Invalid addresses, invalid advancement, and arithmetic outside defined behavior have no specified successful result in this contract.

If processing fails partway through, earlier cursor updates may already have occurred. Because advancement precedes the numeric update, the cursor associated with the current unit may also already have been changed before an arithmetic-related failure. This ordering does not establish a recoverable exception state: no local recovery or rollback mechanism is visible.

Neither an entire call nor an individual consumption is established as atomic. Shared access requires an external synchronization and execution contract.

## Dependencies

There are no visible calls to other functions, so there are no dependency function IDs to enumerate.

The following execution-environment properties affect behavior but are not opaque helper functions:

- Native interpretation of two-byte unsigned loads.
- Alignment and memory-access rules.
- The relationship between the four-byte cursor representation and usable addresses.
- Address advancement and address-boundary behavior.
- Signed arithmetic overflow rules.
- Conversion from unsigned 32-bit results to the signed accumulator.
- Invalid-access and other fault handling.

No library parser semantics may be imported to fill these gaps. In particular, digit validation, terminator handling, and error reporting are not supplied by an unseen parsing dependency in the available evidence.

## Acceptance examples

Unless a row states otherwise, these examples assume readable input, a writable cursor cell, non-overlapping storage, no external mutation, valid address advancement, and defined arithmetic. Listed inputs are numeric 16-bit unit values, not byte sequences.

| Input units at the initial cursor | Requested count | Numeric result | Cursor effect |
|---|---:|---:|---|
| No accessible input required | 0 | 0 | Cursor cell is not accessed |
| No accessible input required | −3 | 0 | Cursor cell is not accessed |
| 48 | 1 | 0 | Advances by 2 bytes |
| 55 | 1 | 7 | Advances by 2 bytes |
| 49, 50, 51 | 3 | 123 | Advances by 6 bytes |
| 48, 48, 53 | 3 | 5 | Advances by 6 bytes |
| 49, 50, 51 | 2 | 12 | Advances by 4 bytes; the third unit is not read |
| 58 | 1 | 10 | Advances by 2 bytes; no digit rejection |
| 65,535 | 1 | 65,487 | Advances by 2 bytes |
| 49, 0 | 2 | Conditionally −38 | Advances by 4 bytes if execution completes |

The last row distinguishes mathematical meaning from conversion behavior. Its mathematical result is negative thirty-eight. Under conventional unsigned 32-bit arithmetic, the final unsigned value is 4,294,967,258. Returning signed −38 additionally requires the target’s unsigned-to-signed conversion to map that value accordingly. The negative result is therefore conditional, not a portable guarantee derived solely from the representation.

Additional acceptance boundaries:

- **Terminator independence:** when a positive count reaches a zero-valued unit, that unit is consumed. Processing is not terminated by its value.
- **Exact count limit:** accessible trailing units beyond the requested count are not read by the established loop.
- **Signed-range boundary:** the ten digit values representing 2,147,483,647 yield that result under the ordinary 32-bit interpretation; the last signed multiplication remains within range.
- **Conversion boundary:** the ten digit values representing 2,147,483,648 do not establish a portable signed return. The final unsigned value exceeds the signed maximum.
- **Multiplication overflow:** the ten digit values representing 3,000,000,000 require a signed multiplication beyond the signed 32-bit range. A wrapped result must not be assigned without additional target evidence.
- **Invalid positive-count access:** inaccessible input with a positive count is not equivalent to empty input and has no specified successful return.
- **Partial progress:** no acceptance condition may require restoration of the original cursor after a later access or arithmetic failure.

These are conditional static acceptance criteria, not reports of executed tests.

## Uncertainties

The following remain **documented_only**:

- **Byte order and alignment.** Two-byte unsigned loading is established; the target’s byte ordering and permitted alignments are not.
- **Complete address model.** The four-byte cursor representation does not resolve pointer validity, address-space restrictions, provenance, or boundary crossing.
- **Overflow outcomes.** The signed multiplication prevents an unconditional wrapping interpretation.
- **Unsigned-to-signed conversion.** The exact result or failure behavior for unsigned values outside the signed range is not established.
- **Fault handling.** No local handler is visible, but the surrounding environment’s response to invalid memory or arithmetic behavior is unknown.
- **Overlapping storage.** Logical ordering is visible; the validity and exact effects of arbitrary typed overlap are not.
- **Concurrent mutation.** There is no synchronization evidence from which to derive a safe concurrent-use guarantee.
- **Machine-level accesses.** The exact instruction sequence, physical read count, and external visibility of individual accesses are not established.

Decimal parsing remains inferred intent. It does not resolve these uncertainties or authorize additional validation behavior.

## Evidence

Established static evidence for F-fe0df18e153a supports:

- A zero initial accumulation and a signed positive-count processing condition.
- Unsigned 16-bit input units obtained through a caller-owned cursor.
- A separate logical cursor read for advancement.
- Forward advancement by one two-byte unit, committed before numeric accumulation for that unit.
- Decimal positional weighting and an offset of 48.
- Signed accumulation combined with an explicitly unsigned 32-bit contribution.
- Count-limited completion and return of the accumulated signed value.
- No visible helper calls, content-validation branches, ownership operations, rollback, or distinct error-reporting path.

The multiplier and offset support decimal conversion as inferred intent. Encoding, target-specific overflow results, and runtime fault outcomes remain unestablished.

No runtime tests have been performed.
