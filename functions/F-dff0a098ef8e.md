<!-- Trusted provenance: F-dff0a098ef8e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dff0a098ef8e

## Purpose

F-dff0a098ef8e conditionally computes an eight-byte value and stores it in a caller-supplied mutable record. A validity flag can suppress the computation entirely. Otherwise, the routine constructs a date-derived base value, optionally incorporates clock-related inputs, and optionally applies an additional adjustment.

**Established static behavior:** record offsets and access widths, branch conditions, integer expressions, dependency-call order, argument positions, retained results, and explicit record writes.

**Inferred intent:** conversion of calendar and clock components into a serial date/time representation, with a possible timezone-like adjustment.

**Not established:** the eight-byte numeric format, calendar convention, epoch, units, dependency arithmetic, global operand values, or adjustment direction. The contract therefore describes observable operations without assigning those meanings.

## Interface

### Input and result

The routine receives one pointer to a mutable record. It has no explicit return value or error status. Results are observable through record modifications and dependency effects.

The supplied representation uses 32-bit address arithmetic. The following are byte offsets from the record’s starting address:

| Offset | Width | Established use |
|---|---:|---|
| 0 | 8 bytes | Output storage |
| 8 | 4 bytes | Signed year-like input |
| 12 | 4 bytes | Signed month-like input |
| 16 | 4 bytes | Signed day-like input |
| 20 | 4 bytes | First clock-related input |
| 24 | 4 bytes | Second clock-related input |
| 28 | 4 bytes | Signed adjustment input |
| 32 | 4 bytes | First word of an additional clock-related operand |
| 36 | 4 bytes | Second word of that operand |
| 40 | 1 byte | Date-input-present flag |
| 41 | 1 byte | Clock-input-present flag |
| 42 | 1 byte | Cached-value-valid flag |
| 43 | 1 byte | Adjustment-present flag |

Each flag is tested for zero versus nonzero, not for equality to one.

The fields at offsets 8, 12, 16, and 28 participate in signed 32-bit operations. The fields at offsets 20 and 24 are loaded as four-byte quantities and passed to a dependency; their signed interpretation is not independently established.

The words at offsets 32 and 36 occupy successive argument positions, with offset 32 first. This does not establish their read order, an atomic eight-byte load, host endianness, or a floating-point representation.

### Preconditions and ownership

The caller must provide storage suitable for every access on the selected path. A record spanning at least 44 accessible bytes covers all listed fields; the immediate-return path explicitly accesses only offset 42.

The routine performs no explicit null-pointer, size, alignment, component-range, or overflow validation. Required native alignment and behavior on invalid memory are not established.

No allocation, deallocation, pointer retention, or ownership transfer is explicit in this routine. Dependency implementations are unavailable. Their ownership behavior, access to aliased storage, global effects, and failure behavior remain unknown.

## Behavior

### Cache gate and date selection

The first explicit record access reads offset 42.

- If nonzero, the routine returns without dependency calls or explicit writes.
- If zero, computation proceeds.
- A nonzero value suppresses recomputation even if component fields have changed.

Next, the routine reads offset 40.

If zero, it selects the internal date-like values year 2000, month 1, and day 1. It does not explicitly read offsets 8, 12, or 16 on this branch.

If nonzero, it explicitly reads:

1. Month-like input at offset 12.
2. Day-like input at offset 16.
3. Year-like input at offset 8.

For a selected month greater than 2, the selected year and month are unchanged. Otherwise, the year is decreased by one and the month increased by 12. The fallback consequently becomes year 1999, month 13, day 1.

No valid-calendar-range restriction is enforced. Months zero or negative take the same normalization branch as months 1 and 2. Months greater than 12 are not reduced or rejected.

### External operand pairs

Six labels distinguish pairs of global operands used by dependencies:

| Label | Use |
|---|---|
| K1 | Year-related date transformation |
| K2 | Month-related date transformation |
| K3 | Final date-base transformation |
| K4 | Transformation of the offset-24 input |
| K5 | Transformation of the offset-20 input |
| K6 | Clock transformation and, separately, adjustment transformation |

Each label denotes **two separately read four-byte globals**, supplied in successive argument positions after the two words of a dependency result. It does not denote an established single eight-byte constant.

For each invocation described below, the applicable globals are read while preparing that invocation, after the preceding conversion or combination has returned. The representation establishes their argument positions but not the evaluation order between the two global reads. Neither an atomic paired read nor stable values across calls are established.

K6 is read for clock processing and read again for adjustment processing after intervening dependency calls. Those uses need not observe identical values. Even within one invocation, no coherent snapshot of the two globals is guaranteed by the evidence.

### Date-derived base

The date calculation follows this dependency sequence:

1. F-870c4fcf7295 receives the normalized year and integer 100. Its signed 32-bit result is retained as a century-related quantity.
2. The normalized year plus 4716 is passed as a 32-bit quantity to F-b3a54d881220.
3. The returned eight-byte value is supplied to F-c4b34006a13c as its low 32-bit word followed by its high 32-bit word. The two K1 globals are read for the remaining argument positions. The explicit return value is not retained.
4. F-baa332e06e32 is called with no explicit arguments. Its signed 32-bit result is retained as the year contribution.
5. The normalized month plus one is passed to F-b3a54d881220.
6. Its returned value is supplied, low word then high word, to F-c4b34006a13c, together with the two K2 globals read for this invocation. The explicit return value is not retained.
7. F-baa332e06e32 is called again with no explicit arguments. Its signed 32-bit result is retained as the month contribution.
8. A signed 32-bit date quantity is formed by accumulating the year contribution, month contribution, selected day, the correction “two minus the century-related quantity,” and a quarter correction.
9. That quantity is passed to F-b3a54d881220.
10. The returned value is supplied, low word then high word, to F-8a693c103d1c. The two K3 globals are read for its remaining argument positions. Its eight-byte result becomes the retained base value.

For accumulation, the year and month contributions are added first, then the day, then the “two minus” correction, then the quarter correction. Intermediate overflow therefore matters; a representable final mathematical sum alone does not establish safe evaluation.

The quarter correction uses the century-related quantity’s sign, a bias of three for negative values, and arithmetic right shifting by two. Under the shown signed 32-bit shift interpretation, it equals division of that quantity by four with rounding toward zero. This does not establish that F-870c4fcf7295 performs division.

The two no-argument calls are observably significant. Their results cannot be specified as functions of explicit arguments alone. A relationship to preceding calls is plausible, but the communication mechanism is unknown.

### Base publication and clock gate

After obtaining the base value, the routine explicitly:

1. Stores the eight-byte base value at offset 0.
2. Stores byte value 1 at offset 42.
3. Reads the clock flag at offset 41.

If offset 41 is zero **at this read**, the routine returns. It does not explicitly read the adjustment flag or adjustment input and does not clear input-state flags.

An entry-time value of zero at offset 41 does not by itself guarantee this branch: dependencies execute before the read and are not known to preserve that byte.

The validity write precedes all optional processing. There is no explicit rollback if a later dependency fails to return. The routine performs no subsequent explicit write to offset 42, but dependencies may affect that byte.

### Clock processing

When offset 41 is nonzero at its read:

1. The four-byte field at offset 24 is read and passed to F-b3a54d881220.
2. Its returned value is passed, low word then high word, to F-c4b34006a13c, with the two K4 globals read for this invocation. The eight-byte return is retained.
3. The four-byte field at offset 20 is read and passed to F-b3a54d881220.
4. Its returned value is passed, low word then high word, to F-c4b34006a13c, with the two K5 globals read for this invocation. This return is also retained.
5. F-1b22d7e17791 receives the transformed offset-20 value first and transformed offset-24 value second.
6. F-1b22d7e17791 is called again with the preceding result first and the words read from offsets 32 and 36 in the second operand’s argument positions. Their relative read order is not established.
7. F-596e18a33aec receives the preceding result, followed by the two K6 globals read for this invocation.
8. F-1b22d7e17791 receives the transformed clock value first and retained base value second.
9. The returned eight-byte combined value is stored at offset 0.
10. The adjustment flag at offset 43 is read.

For dependency results supplied as eight-byte operands, the low word precedes the high word. For two-operand calls, both words of the first operand precede both words of the second in argument position.

The base used in step 8 is retained from date processing, not explicitly reread from offset 0. Consequently, a dependency’s modification of output storage is not necessarily incorporated into that call.

If offset 43 is zero at its read, the routine returns without explicit flag clearing.

### Additional adjustment

If offset 43 is nonzero at its read:

1. The signed 32-bit field at offset 28 is read and multiplied by 60 in the integer domain.
2. The resulting 32-bit quantity is passed to F-b3a54d881220.
3. F-596e18a33aec receives the returned value, low word then high word, followed by fresh reads of the two K6 globals.
4. F-387e686c467e receives the transformed adjustment first and retained combined date/clock value second.
5. Its eight-byte result is stored at offset 0.
6. Zero is written to offset 40, then offset 41, then offset 43.

The combined value in step 4 is retained from clock processing rather than explicitly reread from output storage.

The routine previously wrote 1 to offset 42 and performs no later explicit write there. A final value of 1 requires that intervening dependencies have not changed it. Normal dependency return does not establish preservation.

The component fields are not explicitly cleared or rewritten. F-387e686c467e’s argument order is established; subtraction and its direction are not.

### Integer boundaries

The shown pre-conversion integer operations are 32-bit operations. Relevant signed calculations include year decrement, month increment by 12, year increment by 4716, month increment by one, date accumulation and corrections, and adjustment multiplication by 60.

The evidence does not establish a portable overflow policy. Wrapping, trapping, saturation, and rejection must not be substituted for one another without additional evidence.

For multiplication by 60 alone, adjustment inputs from −35,791,394 through 35,791,394 avoid signed 32-bit overflow. This bound says nothing about semantic validity or the conversion dependency’s accepted range.

## Side effects

Excluding dependency effects, the explicit write patterns are:

| Path taken at the actual flag reads | Writes in order |
|---|---|
| Cache valid | None |
| Base only | Output; offset 42 = 1 |
| Base and clock | Output; offset 42 = 1; output |
| Base, clock, adjustment | Output; offset 42 = 1; output; output; offsets 40, 41, 43 = 0, in that order |

“Unchanged by explicit writes” does not mean unchanged by dependencies.

There is no established atomicity, synchronization, reentrancy, or concurrent-access guarantee. The eight-byte stores are explicit logical stores in the representation, not proven atomic machine operations. Observing offset 42 as nonzero does not establish completion of optional processing.

Inputs and globals are not all snapshotted at entry. Later flag, component, and global reads occur after dependency calls. Unknown dependencies must not be assumed to preserve record storage, operand globals, or other ambient state.

## Dependencies

| Opaque ID | Established interface use |
|---|---|
| F-870c4fcf7295 | Receives normalized year and 100; returns a signed 32-bit date-correction quantity. |
| F-b3a54d881220 | Receives one four-byte quantity; returns an eight-byte value. |
| F-c4b34006a13c | Receives two words of a returned value and two separately read global words. Return ignored in date contribution processing; retained in clock processing. |
| F-baa332e06e32 | Receives no explicit arguments; returns a signed 32-bit contribution after each date-related F-c4b34006a13c call. |
| F-8a693c103d1c | Receives converted date quantity and the K3 global words; returns the base value. |
| F-1b22d7e17791 | Receives two two-word operands; returns an eight-byte value. Called three times on the clock path. |
| F-596e18a33aec | Receives a two-word value and freshly read K6 global words; used separately for clock and adjustment processing. |
| F-387e686c467e | Receives transformed adjustment first and combined date/clock value second; returns the adjusted output. |

Conversion, multiplication, addition, division, and subtraction are possible interpretations of these dependencies, not established contracts. No dependency-preservation or memory-ownership guarantee is available.

## Acceptance examples

These are conditional static acceptance criteria, not runtime results.

1. **Cache bypass.** If offset 42 is nonzero at its first read, there are no dependency calls or explicit writes, regardless of other record contents.

2. **Fallback with base-only completion.** If offset 42 is zero at its read, offset 40 is zero at its read, dependencies return normally, and offset 41 is zero when read after base publication, the routine uses normalized fallback values 1999, 13, and 1. It stores the base, writes 1 to offset 42, and returns without explicitly reading supplied date fields or adjustment fields.

3. **Late clock enablement.** Suppose offset 41 is zero on entry, but a date-stage dependency changes it to nonzero before its read. If it remains nonzero at that read, clock processing occurs. Entry-time zero does not select the base-only path.

4. **Month boundary.** With supplied month 2 and arithmetic in range, the normalized year is one less and month is 14. With supplied month 3, neither normalization change occurs.

5. **Adjustment gated by clock.** If offset 41 is zero at its actual read, even a nonzero adjustment flag does not cause an explicit adjustment-flag read, adjustment-input read, or adjustment processing.

6. **Clock without adjustment.** If the clock flag is nonzero and the later adjustment flag is zero at their respective reads, the combined value is stored. There are no explicit zero writes to offsets 40, 41, or 43.

7. **Full adjustment.** If both optional branches are taken and all calls return normally, the final explicit output store contains F-387e686c467e’s result with transformed adjustment first and retained combined value second. The routine then explicitly clears offsets 40, 41, and 43 in order. Offset 42 is finally 1 only if dependencies have preserved its earlier write.

8. **Changed K6 globals.** If intervening dependencies change either K6 global after its clock-stage read and before its adjustment-stage read, adjustment processing uses the newly read word values. It does not reuse a saved clock-stage pair.

9. **Repeated invocation.** A later invocation bypasses computation if offset 42 is nonzero at that invocation’s first read. Prior normal completion alone does not guarantee that condition without dependency-preservation assumptions.

10. **Interrupted optional processing.** If a clock-stage dependency does not return, the routine has already explicitly stored the base and written 1 to offset 42. No explicit restoration is evidenced; actual memory contents also depend on effects of calls already entered.

## Uncertainties

The following remain **documented_only**:

- Values, numeric interpretation, and mutability of all global operand words.
- Relative evaluation order of paired global reads and the two record-word reads at offsets 32 and 36.
- Eight-byte numeric format, endianness, and native alignment requirements.
- Dependency arithmetic, rounding, exceptions, ownership, aliasing, and global effects.
- The communication mechanism behind no-argument contribution calls.
- Calendar system, epoch, supported ranges, and handling of invalid dates.
- Clock units, adjustment units, and adjustment direction.
- Overflow behavior outside safely representable intermediate calculations.
- Atomicity, concurrency, and reentrancy guarantees.
- Final validity-flag contents when dependencies may modify that byte.

The date-like normalization and correction support a calendar interpretation but do not establish a particular standard or numeric answer.

## Evidence

This contract uses only the supplied static routine representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

Static evidence establishes access widths and offsets, branch timing, integer literals and expressions, dependency argument positions, repeated global reads, retained values, and explicit write order. It does not establish dependency implementations, global values, atomic access, or preservation of record fields across calls.

No runtime tests have been performed. The acceptance examples specify conditional observations rather than reported execution results.
