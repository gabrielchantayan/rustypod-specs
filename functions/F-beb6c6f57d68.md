<!-- Trusted provenance: F-beb6c6f57d68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-beb6c6f57d68

## Purpose

F-beb6c6f57d68 selects a 32-bit result from a small integer selector. Depending on the selector, it returns either a value obtained from one of eleven distinct global storage locations, the fixed value 1024, or the all-ones 32-bit value.

This is a selection operation, not a calculation over the selected values. The supplied evidence establishes no units, domain meaning, validation purpose, or relationship among the stored values.

Two pairs of selectors share storage locations. Two other selectors return the same fixed constant. Several selectors within the otherwise recognized range return the same all-ones result as out-of-range selectors.

## Interface

**Input:** one integer selector.

The supplied representation identifies the input as an integer but does not independently establish its machine width or calling-convention representation. The behavior below applies to the selector’s integer value. If the interface uses a signed 32-bit integer, its domain is −2,147,483,648 through 2,147,483,647; that specific ABI remains an assumption rather than independently supplied evidence.

**Output:** one 32-bit result.

The result is best specified as a bit pattern because its signedness and semantic type are not established:

- The fixed successful-looking constant is `0x00000400`, or 1024.
- The default result is `0xFFFFFFFF`.
- `0xFFFFFFFF` is 4,294,967,295 under unsigned interpretation and −1 under signed two’s-complement interpretation. Neither interpretation establishes that it is an error code.
- Storage-backed results come from the selected global location. The declarations and widths of those locations are absent, so any underlying load-width or extension details are not independently established.

No output buffer, length, context argument, or ownership parameter is visible.

For this contract, the eleven reachable global locations are assigned neutral labels **Slot A** through **Slot K**. These labels identify distinct storage locations; they do not imply an array, a record layout, contiguous storage, or an application-level relationship.

## Behavior

### Selection rules

The complete externally relevant selection mapping is:

| Selector | Result source | Selected global data read |
|---:|---|---|
| 0 | Value from Slot A | Slot A |
| 1 | Value from Slot A | Slot A |
| 2 | Value from Slot B | Slot B |
| 3 | Fixed `0x00000400` | None |
| 4 | Fixed `0x00000400` | None |
| 5 | Value from Slot C | Slot C |
| 6 | Value from Slot C | Slot C |
| 7 | Value from Slot D | Slot D |
| 8 | Value from Slot E | Slot E |
| 9 | Fixed `0xFFFFFFFF` | None |
| 10 | Value from Slot F | Slot F |
| 11 | Fixed `0xFFFFFFFF` | None |
| 12 | Value from Slot G | Slot G |
| 13 | Fixed `0xFFFFFFFF` | None |
| 14 | Value from Slot H | Slot H |
| 15 | Value from Slot I | Slot I |
| 16 | Value from Slot J | Slot J |
| 17 | Value from Slot K | Slot K |
| Any other integer | Fixed `0xFFFFFFFF` | None |

“None” refers to explicit selected global data reads in the supplied function. It does not describe instruction fetches, calling-convention accesses, or other machine-level activity absent from the evidence.

### Selector 4 takes precedence

Selector 4 is resolved by an earlier decision and returns 1024 directly.

The supplied representation also contains a later alternative that appears to associate selector 4 with another global location. That alternative is unreachable under the earlier decision. Consequently:

- Selector 4 does not obtain its result from that additional location.
- Changes to that location do not affect the established result for selector 4.
- The additional location is not a reachable dependency of this function.

A behavioral implementation must preserve the actual result of 1024 rather than treat the unreachable alternative as a second valid outcome.

### Shared sources and distinct sources

Selectors 0 and 1 select the same global location. Selectors 5 and 6 also select the same global location.

This establishes source identity, not unconditional equality across separate invocations. If a shared location changes between calls, the returned values may differ.

All other storage-backed rows select distinct locations. Those locations may contain equal values, but equality is not guaranteed by the selection mapping.

There is no visible caching or initialization. A storage-backed invocation obtains its value through the selected global reference rather than through a value retained by this function from an earlier invocation.

### Default result and unsupported values

Selectors 9, 11, and 13 return `0xFFFFFFFF`, even though neighboring selectors have storage-backed behavior. Every selector outside 0 through 17 also returns that same bit pattern.

The function does not distinguish these categories in its output. It provides no visible reason code, diagnostic, or separate status channel.

A selected global value is not checked against `0xFFFFFFFF`. A storage-backed branch can therefore produce the same output as a default branch. The result alone cannot reliably distinguish unsupported input from a selected location containing all ones.

### Arithmetic and conversion boundaries

No addition, subtraction, multiplication, division, shift, or index arithmetic is visible. The selectors are matched against explicit integer values; the behavior is not a range-based table lookup with an inferred bound.

There is consequently no arithmetic overflow behavior to specify within the visible selection logic.

The fixed values fit exactly in 32 bits. Storage-backed values require an additional qualification: because the global declarations are omitted, the evidence does not resolve whether a value is loaded directly at result width or converted from another representation.

## Side effects

The supplied function contains no explicit writes to global storage, caller-provided memory, or other state.

For a storage-backed selector, one selected global reference supplies the result. No other selected global location is explicitly read on that path. Constant-result paths do not explicitly read any of the eleven reachable slots.

There are no visible function calls, allocations, releases, callbacks, logging operations, or synchronization operations.

No ownership transfer is expressed. The function returns a 32-bit value; it does not visibly retain, release, or dereference that returned value as an address. Whether callers interpret the bits as an address or handle is unknown and cannot be inferred from this function.

The evidence does not identify the storage as ordinary memory, volatile storage, or memory-mapped hardware. Accordingly:

- The contract establishes the selected source, but not its hardware access properties.
- Atomicity and concurrent-update behavior are not established.
- No snapshot consistency across separate calls is promised.
- A read’s possible external effects depend on storage properties not supplied here.

Normal return on a storage-backed path requires that the selected read complete successfully. No local recovery or error translation for a failed read is visible. Exception, fault, and process-level behavior remain unspecified.

## Dependencies

There are no visible callee dependencies requiring additional opaque function IDs.

The function’s reachable data dependencies are:

| Dependency | Selectors |
|---|---|
| Slot A | 0, 1 |
| Slot B | 2 |
| Slot C | 5, 6 |
| Slot D | 7 |
| Slot E | 8 |
| Slot F | 10 |
| Slot G | 12 |
| Slot H | 14 |
| Slot I | 15 |
| Slot J | 16 |
| Slot K | 17 |

The initial contents, initialization order, lifetime, mutability, and writers of these locations are unknown. The function does not visibly establish any of them.

No assumption about callee preservation of globals or callee memory ownership is needed: no calls appear in the supplied evidence.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results.

1. **Fixed-value selectors.** With any global contents, selectors 3 and 4 return exactly `0x00000400`. Neither result depends on a selected global read.

2. **Earlier decision wins.** If the global location mentioned only in the unreachable selector-4 alternative contains a value other than 1024, selector 4 still returns 1024.

3. **Shared source remains stable.** If Slot A yields `0x12345678` and remains unchanged across both calls, selectors 0 and 1 each return `0x12345678`.

4. **Shared source changes between calls.** If selector 5 reads one value from Slot C, Slot C subsequently changes, and selector 6 then reads it, the second result reflects the later selected value. Equality with the earlier result is not required.

5. **Each distinct source is respected.** For every storage-backed row in the mapping, if its selected slot yields a distinguishable 32-bit value, the result is that selected value rather than a value from another slot. This criterion is conditional on the omitted storage declarations permitting that value without a different conversion.

6. **Interior default cases.** Selectors 9, 11, and 13 each return `0xFFFFFFFF`, regardless of the contents of the eleven reachable slots.

7. **Outside-range values.** Selectors −1 and 18 return `0xFFFFFFFF`. The same applies to every other representable integer outside the listed selectors.

8. **No sentinel filtering.** If Slot K yields the result bit pattern `0xFFFFFFFF`, selector 17 returns that pattern without converting it to a different result or reporting a separate error.

9. **No selected-value normalization.** If a selected slot yields zero, the storage-backed result is zero; the visible logic does not substitute 1024 or the default value.

10. **No explicit state mutation.** Under ordinary readable-storage conditions, invocation introduces no explicit write to any slot. Read-triggered hardware effects, if applicable, are outside the supplied evidence.

## Uncertainties

The following remain **documented_only**:

- The selector’s exact ABI width and signedness.
- The declared widths, signedness, and load conversions of the global locations.
- The semantic type and units of returned values.
- Whether `0xFFFFFFFF` is intended as an error, sentinel, valid data, or another category.
- Whether storage is immutable, mutable, volatile, atomic, or hardware-backed.
- Storage initialization, accessibility, and concurrent modification rules.
- Fault behavior when a selected read cannot complete.
- Whether the unreachable alternative reflects an artifact or historical intent.

A plausible inferred intent is to expose values by category while reserving some selectors. That interpretation is not needed for the contract and is not established by the evidence.

## Evidence

This contract is based solely on the supplied static representation of F-beb6c6f57d68.

Established evidence includes the earlier selector-4 decision, the reachable selector mapping, the two fixed result patterns, eleven reachable global value sources, and the absence of explicit writes or calls.

No global declarations, caller contracts, ABI documentation, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples specify expected behavior under stated conditions; they are not verification claims.
