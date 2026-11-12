<!-- Trusted provenance: F-5daf1349eecb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5daf1349eecb

## Purpose

F-5daf1349eecb coordinates an operation involving a context object, a service object, a four-word working region, operation flags, and optional indexed data and result state.

Two principal paths are selected by bit 27 of a context flag word:

- A **region-oriented path** adjusts or selects a working region and dispatches a region-based operation. It can use an indexed collection to select an alternative payload.
- A **position-oriented path** calculates a position, dispatches an extent-based operation, and optionally performs an additional region operation.

These names describe the visible argument flow, not established application semantics. Rendering, text layout, clipping, and decoration are plausible interpretations, but the supplied evidence does not establish them.

The function has no explicit return value. Its observable results are dependency calls, writes to optional result state, and any effects produced by its dependencies.

## Interface

The interface contains eleven arguments. The visible model uses 32-bit integers, 32-bit opaque values, and 32-bit addresses.

| Argument | Contract |
|---|---|
| 1. Context | Address of an object. A 32-bit flag word is read at byte offset `0x48`. The region-oriented path can also read four words at offsets `0x80` through `0x8c`. |
| 2. Service | Opaque 32-bit value supplied to most dependencies, commonly in the first argument position. |
| 3. Input bounds | Address of a word sequence. The region-oriented path conditionally reads signed 32-bit words at offsets `0` and `8`. No direct writes through this argument are visible. |
| 4. Primary payload | A 32-bit value used as the default operation payload. Some dependencies may interpret it as an address; its representation is not established. |
| 5. Operation value A | Opaque 32-bit value forwarded to operation dependencies. |
| 6. Operation value B | Opaque 32-bit value forwarded on region-oriented operations. |
| 7. Position adjustment | Signed 32-bit value used only in the position-oriented path’s arithmetic. |
| 8. Operation flags | Unsigned 32-bit word. Bits 0–2 select a dispatch mode. Bit 6 enables the optional position-oriented post-operation sequence. |
| 9. Indexed-selection request | Zero disables indexed selection. Any nonzero value requests it, subject to argument 10 being non-null. |
| 10. Indexed collection | Optional object address. Indexed selection reads its signed count at byte offset `4` and invokes an indirect accessor at dispatch-table byte offset `0x40`. It can also be forwarded without inspection. |
| 11. Result state | Optional object address. The region-oriented, mode-zero path writes fields and invokes helpers on it. |

### Working-region representation

F-412e4635fdb0 receives the address of four contiguous 32-bit working words. This contract names them **R0**, **R1**, **R2**, and **R3**, in increasing address order.

The caller uses:

- `R2 − R0` as an alignment span;
- `R3 − R1` as an operation extent;
- R0 and R2 as adjustable endpoints;
- R1 as a position coordinate.

The evidence does not establish coordinate orientation, inclusive versus exclusive endpoints, or whether these words are a conventional rectangle.

A second four-word region is copied from context offsets `0x80`, `0x84`, `0x88`, and `0x8c`. Its first and third words are named **C0** and **C2** below; only those words participate directly in the visible comparisons.

### Preconditions

For a defined execution, every address actually dereferenced must be valid for the corresponding access. Dependencies must also satisfy their own, otherwise unspecified, contracts.

In particular:

- The context must support the initial flag read.
- F-412e4635fdb0 must supply usable working-region contents before they are consumed.
- Input bounds must support the reads reached by the chosen path.
- Indexed selection requires a usable collection, dispatch table, accessor result, and referenced entries.
- A non-null result object must support the accessed fields and dependency calls.

There are no visible null checks for the context, service, or input bounds. There is no explicit validation of extents, collection ordering, integer ranges, or payload representation.

## Behavior

### Initial sequence and flag selection

The function first reads the context flag word at offset `0x48`. This snapshot determines all subsequent context-flag decisions, even if a dependency later changes the context.

It then calls, in order:

1. F-88dd67d704e5 with the context and service.
2. F-412e4635fdb0 with the context and the working-region address.

After these calls, it captures the address used to obtain a global operation word. The word itself is read only when a dispatch requiring it is reached.

If indexed selection was requested but the collection argument is null, the request is locally disabled. The collection argument remains null; no result error is reported.

The relevant context bits are:

- Bit 24: first alignment option.
- Bit 25: second alignment option.
- Bit 26: third alignment option.
- Bit 27: region-oriented path selector.

### Region-oriented path: bit 27 set

#### Alignment stage

If any of bits 24–26 is set, F-7915d17c3515 is called with:

- the service;
- the primary payload;
- the current extent `R3 − R1`.

Its signed 32-bit result is used as a measured quantity.

If bit 24 is clear, R0 is replaced by `R2 − measured quantity`. This applies whether bit 25, bit 26, or both caused the stage to run.

If bit 24 is set, F-59908d670f34 receives the working-region address and half of `R2 − R0 − measured quantity`. Division is signed division by two, truncating toward zero. The dependency’s exact mutations are not established; the caller subsequently uses the working region as left by that call.

If all three alignment bits are clear, no measurement or alignment-helper call occurs.

#### Nonzero low-three-bit modes

The mode is the operation flags masked with `7`.

For mode 1 or 4, the function calls F-0194b051f9cb with the service, primary payload, current global operation word, working-region address, operation values A and B, and respectively selector 1 or 4. It then returns.

For modes 2, 3, 5, 6, or 7, it returns without calling a region operation dependency.

These early-return paths do not copy the context’s four-word secondary region, inspect indexed entries, initialize result fields, or invoke the result finalizer. Alignment processing, including dependency effects, has already occurred.

Bit 6 has no effect on this path.

#### Mode zero: context-region copy and result initialization

The four words at context offsets `0x80` through `0x8c` are copied after the alignment stage.

If result state is non-null, initialization occurs in this order:

1. Compute an endpoint initially equal to R2.
2. If input bounds word 2 is less than C2, reduce that endpoint by `C2 − input bounds word 2`.
3. Store the endpoint at result offset `0x2c`.
4. Store zero at result offset `0x34`.
5. Call F-38583c8dfd31 with the address at result offset `0x38`.

The last call’s effects on the result object or other state are unknown. Later operations must not assume it preserves the just-written fields.

This initialization does not itself modify R2.

#### Optional indexed selection

If indexed selection remains enabled, the function searches using:

- A signed count read once from collection offset `4`.
- Inclusive search indices initially zero through `count − 1`.
- A midpoint formed from the signed sum of the current bounds, divided by two with truncation toward zero.

For each midpoint, the indirect accessor at dispatch-table offset `0x40` receives the collection and index. Its returned address is dereferenced once to obtain an entry address.

Each entry exposes these signed 32-bit words:

| Entry offset | Visible role |
|---|---|
| `0` | Candidate payload |
| `8` | Search threshold |
| `0x0c` | Predecessor metadata, read only when result state is non-null and the relevant predecessor boundary is selected |

The search compares input bounds word 0 against `C0 + current entry threshold`, using a strict less-than comparison.

- If the input is not less, the lower search index becomes one past the midpoint.
- If it is less and the midpoint is zero, the current entry’s payload is selected with a zero start adjustment.
- Otherwise the predecessor entry is fetched. If `C0 + predecessor threshold` is less than or equal to the input, the current entry’s payload is selected, and the predecessor threshold becomes the start adjustment.
- If that predecessor boundary is selected and result state is non-null, the predecessor metadata is written to result offset `0x34`.
- If the predecessor boundary is still above the input, the upper search index becomes one before the midpoint.

The search structure suggests an intended ascending threshold order. The function does not verify that order.

A selected payload of zero is treated as **no effective replacement**, even when the search found a boundary. A predecessor metadata write can therefore occur without replacement becoming active.

When the selected payload is nonzero:

- R0 is increased by the selected start adjustment.
- If input bounds word 2 is less than C2, R2 is reduced by `C2 − input bounds word 2`.
- The selected payload replaces the primary payload for the upcoming call.
- The collection argument passed to that call becomes null.

Without a nonzero selection, the primary payload and original collection argument are retained. A disabled selection request does not suppress forwarding a non-null collection.

The function then calls F-26ac244eb347 with the service, effective payload, working-region address, operation values A and B, effective collection, and result state.

If result state is non-null, F-8139900602e2 is called afterward with that object. No such finalization occurs when result state is null.

### Position-oriented path: bit 27 clear

F-263ec605ff37 and F-b38e44b3c7fb are called, in that order, with the service. Their signed results are named **A** and **B** here.

A position is calculated with the following precedence:

| Context flags | Position before argument-7 adjustment |
|---|---|
| Bit 24 set | `R0 + (R2 − R0 − (A + B)) / 2 + A` |
| Bit 24 clear, bit 25 set | `R0 + (R2 − R0 − A) / 2 + A` |
| Bits 24–25 clear, bit 26 set | `R2 − B` |
| Bits 24–26 clear | `R0 + A` |

Each division truncates toward zero.

F-e5e685741bac then receives the service, R1, and the computed position minus argument 7. This call occurs even for modes that perform no subsequent primary operation.

The function initializes the operation result quantity to zero and dispatches on the low three operation-flag bits:

- Mode 0: call F-443fbd28f515 with the service, primary payload, `R3 − R1`, and operation value A. Retain its signed result.
- Mode 1 or 4: call F-4a510ca8d13f with the service, primary payload, current global operation word, `R3 − R1`, operation value A, and selector 1 or 4. Retain its signed result.
- Modes 2, 3, 5, 6, and 7: make neither operation call; the result quantity remains zero.

Operation value B, indexed selection, and result state are not otherwise used on this path.

#### Optional bit-6 sequence

If operation flag bit 6 is set, the following occurs regardless of whether a primary operation was dispatched:

1. Call F-a0576cc39408 with the service and an address of static data; retain its signed result.
2. Call F-263ec605ff37 again.
3. Call F-b38e44b3c7fb again.
4. Construct a separate four-word region containing, in order, zero, zero, the sum of the two newly obtained results minus one, and one.
5. Call F-d156b0b83eb3 with this region’s address, the primary operation result minus the F-a0576cc39408 result, and zero.
6. Call F-d156b0b83eb3 again with the same region address, R1, and `R0 + argument 7 + 2`.
7. Read a global enable word through its stored address.

The repeated metric calls are new observations, not reuse of earlier values.

If the enable word is nonzero:

- A temporary 32-bit word is set to signed value −12,369,085.
- F-ccb5d9751852 receives the service and that word’s address.
- F-33ca0546dba1 receives the service and zero.
- F-92df7be8afd3 receives the service and the separate region’s address.

If the enable word is zero, these last three calls are skipped. The earlier bit-6 calls have already occurred.

## Side effects

Direct caller-visible writes are limited to the specified result-object fields on the region-oriented, mode-zero path. Working regions and temporary words are also written locally and passed by address to dependencies.

No direct allocation, deallocation, ownership transfer, or release operation is identifiable. Pointer forwarding does not establish ownership transfer. Dependencies may allocate, release, retain addresses, or modify shared state; their implementations are unavailable.

Dependencies execute in the stated order. Their unknown effects must not be treated as absent. In particular:

- Context-region words are read after earlier dependency calls, unlike the initial context flags.
- Global operation words are read at the dispatch that uses them.
- The optional enable word is read after both region-adjustment calls.
- Result finalization occurs only after F-26ac244eb347 returns.

There is no visible error return, exception handler, rollback, or restoration of service state. Invalid accesses, arithmetic overflow, and dependency failures have no established recovery behavior.

All visible arithmetic is 32-bit. Signed additions, subtractions, midpoint construction, and metric sums are not visibly widened or checked. Mathematical formulas in this contract apply when their intermediates are representable. Overflow behavior is documented_only; portable wrapping semantics are not established.

## Dependencies

| Opaque ID | Role established by this caller |
|---|---|
| F-88dd67d704e5 | Initial context/service call |
| F-412e4635fdb0 | Receives context and working-region address |
| F-7915d17c3515 | Supplies region-path measurement |
| F-59908d670f34 | Receives working region and alignment adjustment |
| F-0194b051f9cb | Region operation for modes 1 and 4 |
| F-38583c8dfd31 | Receives result subobject at offset `0x38` |
| F-26ac244eb347 | Region mode-zero operation |
| F-8139900602e2 | Post-operation result call |
| F-263ec605ff37 | Supplies first position metric |
| F-b38e44b3c7fb | Supplies second position metric |
| F-e5e685741bac | Receives service and calculated position pair |
| F-443fbd28f515 | Position mode-zero operation; returns quantity |
| F-4a510ca8d13f | Position mode-1/4 operation; returns quantity |
| F-a0576cc39408 | Supplies quantity using static-data address |
| F-d156b0b83eb3 | Receives a region and two adjustment arguments |
| F-ccb5d9751852 | Receives service and temporary-word address |
| F-33ca0546dba1 | Receives service and zero |
| F-92df7be8afd3 | Receives service and optional-operation region |

The indexed accessor is an additional indirect dependency without an assigned opaque ID. Its exact implementation and ownership rules are unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Region mode 2:** With context bit 27 set and low operation bits equal to 2, initial setup and any requested alignment occur, followed by return. Result initialization and indexed lookup do not occur.

2. **Higher operation bits:** Low bits equal to 1 still select mode 1 when unrelated higher bits are present. On the region-oriented path, bit 6 does not initiate the optional sequence.

3. **Position alignment precedence:** Given working words R0 = 10 and R2 = 110, metrics A = 7 and B = 3, and adjustment 2:
   - Bit 24 produces position argument 60.
   - Bit 25 alone produces 61, because `93 / 2` truncates to 46.
   - Bit 26 alone produces 105.
   - No alignment bits produce 15.
   
   F-e5e685741bac receives R1 as its other position argument.

4. **Indexed boundary equality:** With C0 = 100 and thresholds 20, 40, and 60, input 140 does not select the threshold-40 entry through its strict upper comparison. Under the intended sorted ordering, the threshold-60 entry is selected using predecessor threshold 40.

5. **Zero candidate payload:** If that selected entry’s payload is zero, payload replacement, R0 adjustment, R2 clipping, and collection clearing do not occur. A predecessor metadata write may nevertheless have occurred.

6. **Result clipping:** With R2 = 100, C2 = 90, and input bounds word 2 = 80, result offset `0x2c` is initially written as 90. Without effective indexed replacement, R2 itself remains 100 before F-26ac244eb347, subject to dependency effects.

7. **Unsupported position mode with bit 6:** For low bits 3 and bit 6 set, neither primary position operation is called. The optional sequence uses zero as the primary result quantity.

8. **Disabled optional final operation:** A zero global enable word suppresses only the final three optional calls, not the metric calls or the two F-d156b0b83eb3 calls.

## Uncertainties

The following are documented_only:

- Application meaning of the context, service, payloads, regions, metrics, static data, and global words.
- Dependency return conventions beyond the values visibly consumed.
- Exact mutations performed by region-adjustment and result helpers.
- Ownership, retention, allocation, and release behavior inside dependencies.
- Required collection ordering and behavior for malformed collections.
- Signed-overflow behavior and malformed-count behavior.
- Aliasing restrictions, concurrency guarantees, and reentrancy.
- Whether the displayed signedness and address interpretation fully match the original interface.

Inferred intent is limited to alignment-like arithmetic, boundary-based payload selection, and an optional secondary operation. None establishes stronger dependency semantics.

## Evidence

The supplied static listing is the sole evidence for this contract. It supports the branch structure, widths, offsets, arithmetic expressions, direct writes, call ordering, and argument forwarding described above.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed. Acceptance examples are deductions conditional on the stated inputs and dependency behavior, not measured outcomes.
