<!-- Trusted provenance: F-ca9ad2c08e4d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca9ad2c08e4d

## Purpose

F-ca9ad2c08e4d requests a 12-byte header and a header-sized payload, then selects between delivery through a temporary conversion buffer, delivery through a context-associated conversion buffer, and delivery from the captured payload address.

Selection is not solely a function of the type initially observed after payload input. For an initially observed type of 2, an enabled payload-transformation call is followed by fresh type reads; a disabled transformation instead leads directly to the mode check for context-buffer conversion.

“Input,” “normalization,” “conversion,” “allocation,” and “delivery” describe apparent dependency roles. The established contract is the calls, memory accesses, comparisons, and state changes below; the data formats and dependency semantics remain unproven.

The function has no explicit return value. Either input operation reporting a nonzero result, or a required allocation returning zero, causes an early return without a local error-reporting call.

## Interface

### Arguments

The function accepts five arguments in a 32-bit address and value model.

| Argument | Established use |
|---|---|
| Context address | Supplies the captured payload address, control byte, and reusable-buffer state. Passed to both input calls. |
| Mode | Compared with 2 for temporary-buffer conversion and with 1 for context-buffer conversion. |
| Unused argument | A 32-bit value neither inspected nor forwarded by this function. |
| First delivery argument | A 32-bit value forwarded unchanged to either delivery dependency. |
| Second delivery argument | A 32-bit value forwarded unchanged to either delivery dependency. |

No explicit validation checks the context, payload destination, mode, or delivery arguments.

### Context fields

| Byte offset | Width | Established role |
|---|---:|---|
| +4 | 4 bytes | Payload destination address, captured before header input. |
| +13 | 1 byte | Nonzero enables header normalization and, following an initial type-2 observation, payload normalization. |
| +32788 | 4 bytes | Unsigned reusable-buffer capacity. |
| +32792 | 4 bytes | Reusable-buffer address. Zero has special significance during replacement. |

Only these accessed fields are established, not the full context layout.

The payload address is cached once. Later changes to the field at +4 do not replace that cached address for this invocation. Other context fields are read at the stages described below.

### Header and local storage

The first input call receives the address of a local header region and a requested size of 12 bytes. Subsequent direct interpretation uses:

- Bytes 0–3: unsigned 32-bit payload length.
- Byte 4: one-byte type discriminator.

No direct interpretation of bytes 5–11 is visible. Header byte order is not independently established.

Additional local storage consists of:

- A 512-byte temporary conversion buffer.
- A 32-bit output-count slot passed by address to F-c8bd0cbe99d3.

Neither the temporary buffer nor the output-count slot is explicitly initialized before its conversion call.

A separate 32-bit global supplies the conversion-bound values. It is not passed as an argument. Its value, initialization, and stability are unknown.

### Preconditions and lifetime

Safe execution requires valid storage for every access actually performed. In particular, the captured payload destination must accommodate the complete length requested by the second input call. This function does not establish that capacity.

The header, temporary buffer, and output-count slot have invocation-local lifetimes. Dependencies receive their addresses, but no retention agreement is established. A dependency that previously received the header address cannot be assumed incapable of changing the header during a later call in the same invocation.

Additional dependency-specific preconditions are documented_only.

## Behavior

### 1. Capture the destination and request the header

Before either input call, the function reads the 32-bit payload address at context offset +4.

It calls F-c02571304c5c with the context, the local header address, and size 12.

- A nonzero result ends this invocation immediately.
- A zero result permits subsequent processing.
- There is no independent check that all 12 requested bytes were populated.

On early return, this function performs no rollback of dependency effects.

### 2. Optionally transform the header

Following successful header input, the function reads the control byte at context offset +13.

If nonzero, it calls F-9505d5cd8135 with the header address. No result is checked. Otherwise, the call is skipped.

Later accesses use the header contents then present. The dependency’s write extent is unknown: it must not be assumed to modify only the length or only its byte order.

### 3. Request the payload

The function calls F-c02571304c5c again, supplying:

1. The context.
2. The payload address captured before header input.
3. The current unsigned 32-bit header length.

A nonzero result causes immediate return. No payload transformation, conversion, or delivery follows.

The request uses the full length available at this point. A later conversion-bound operation neither reduces this request nor retroactively changes it. A zero length still produces an input call with size zero.

The input dependency can have effects beyond writing the payload; header or context preservation is not established.

### 4. Dispatch, including conditional payload transformation

After successful payload input, the function initially reads the type discriminator.

**If that initial value is 2**, it reads the context control byte again:

- **Control zero:** no payload-transformation call occurs. Execution proceeds directly to the mode check for context-buffer conversion, without the type rereads described below.
- **Control nonzero:** F-370c54f34dce receives the captured payload address and the then-current unsigned header length shifted right by one bit. After it returns, execution enters the type-redispatch sequence below.

The transformation count is therefore the length divided by two, rounded down. It precedes conversion-bound processing and is not reduced by it. No result from F-370c54f34dce is checked.

**If the initial type is not 2**, execution also enters the following type-redispatch sequence, without calling F-370c54f34dce.

At entry to that sequence, the function captures the current header length as a possible general-delivery size. It then:

- Reads the type and checks for 1.
  - If 1 and mode is 2, selects temporary-buffer conversion.
  - If 1 and mode is not 2, selects general delivery using the captured size.
- Otherwise, reads the type again and checks for 2.
  - If not 2, selects general delivery using the captured size.
  - If 2, proceeds to the mode check for context-buffer conversion.

Consequently, an initially observed type 2 followed by enabled payload transformation does not irrevocably select the context-buffer path. A dependency-induced change to type 1 can select temporary-buffer conversion when mode is 2. A change to another type can select general delivery.

On reaching the context-buffer mode check, the function captures the current header length as the general-delivery size. Mode other than 1 selects general delivery immediately. Mode 1 continues below.

These captures and rereads are distinct accesses; the contract does not replace them with a single immutable type-and-length pair.

### 5. Temporary-buffer conversion and delivery

When dispatch selects temporary-buffer conversion, F-ba5b7e810055 receives:

- The captured payload address.
- The header length read for this call.
- The 512-byte local buffer.
- The numeric value 255.
- The numeric value 0.

Its result is not checked. F-b9f19a3ec83a then receives the first delivery argument, the local-buffer address, and the second delivery argument. The function returns after that call; it does not call F-7834b08494a4 on this path.

The numeric argument 255 does not establish a byte count, termination rule, or write bound. The actual output layout and failure behavior are documented_only.

### 6. Context-buffer conversion: bound processing

This path first compares the current unsigned header length against a read of the 32-bit conversion-bound global.

If the header length exceeds that comparison value, the function reads the global again and assigns this second value to the header length. Otherwise, it performs no bound assignment.

The two global reads must be distinguished:

- The first supplies the comparison value.
- The second occurs only when the comparison succeeds and supplies the assignment value.

Their equality is not established. Thus, this is not unconditionally equivalent to a conventional clamp against one stable bound. If the second value is larger than the original length, the assignment can increase the length.

No second comparison validates the assigned value.

### 7. Calculate and, if necessary, obtain capacity

After bound processing, the function reads the header length for capacity sizing. It divides that unsigned value by two, rounding down, then multiplies the result by three using unsigned 32-bit arithmetic.

The multiplication is unchecked and wraps modulo \(2^{32}\). An adequately small sizing length would avoid overflow, but neither a suitable bound nor stable bound reads are established.

The function compares this calculated requirement with the unsigned capacity at context offset +32788.

**If capacity is sufficient:**

- No replacement call occurs.
- The buffer address is not independently checked for nonzero.

**If capacity is insufficient:**

1. The function reads the buffer address at +32792.
2. If nonzero, it calls F-1137792fba7d without explicit arguments.
3. After that call returns, it writes zero to the buffer-address field, then zero to the capacity field.
4. It calls F-f195b74c9758 with the previously calculated requirement.
5. It stores the returned 32-bit value in the buffer-address field.
6. A nonzero result causes the calculated requirement to be written to the capacity field.
7. A zero result causes an early return without conversion or delivery.

When the old buffer address is zero, both the no-argument call and the two clearing writes are skipped. An allocation failure in that case does not itself clear the previous capacity field.

The implementation selects an internal failure value of −108 for allocation failure, but does not return or explicitly report that value.

The sizing requirement remains the previously calculated value across replacement calls. It is not recalculated if those calls change the header.

### 8. Convert and deliver from context-associated storage

Once capacity is considered sufficient, F-c8bd0cbe99d3 receives:

- The captured payload address.
- The header length read at this later point, shifted right by one bit.
- The reusable-buffer address read from +32792.
- The reusable-buffer capacity read from +32788.
- The address of the local output-count slot.

The element count is recomputed from the then-current header length. It is not necessarily the count used to calculate the capacity requirement. In particular, F-1137792fba7d and F-f195b74c9758 can run between those length reads; their preservation of the header, including through a previously retained address, is unproven.

No converter result is checked. After conversion, the function reloads the reusable-buffer address from the context and reads the local output count. It calls F-7834b08494a4 with the first delivery argument, that reloaded address, that count, and the second delivery argument.

There is no visible validation that conversion wrote the count, that the count fits capacity, or that the delivered address matches the address supplied to conversion.

The relative evaluation order of the separate arguments to a call is not established here.

### 9. General delivery

All dispatch routes selecting unconverted delivery call F-7834b08494a4 with:

- The first delivery argument.
- The original captured payload address.
- The length captured at the applicable dispatch stage.
- The second delivery argument.

“Unconverted” excludes the two dedicated conversion paths, not all prior mutation: header and payload normalization may already have occurred.

## Side effects

Directly established effects include input requests, conditional transformation and conversion calls, conditional reusable-buffer field writes, and at most one delivery call.

No rollback is visible. Partial input, dependency state changes, or loss of a previous buffer reference can survive early return.

The reusable-buffer address and capacity remain context state after successful allocation. This function does not explicitly release that buffer on normal return. The no-argument F-1137792fba7d call does not establish which resource, if any, is released or how it is identified.

No dependency is assumed to preserve globals, the context, or previously exposed local storage. Conversely, arbitrary mutations are not claimed to occur: preservation is simply outside the supplied evidence.

Aliasing among payload storage, context fields, and reusable storage is not checked. Safe disjointness, ownership transfer, synchronization, and reentrancy guarantees are documented_only.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-c02571304c5c | Context, destination, requested size; nonzero terminates processing. | Input source, completeness, partial writes, error encoding, other mutations. |
| F-9505d5cd8135 | Conditionally receives header address. | Transformation, write extent, retention, other effects. |
| F-370c54f34dce | Conditionally receives captured payload address and half-length count. | Data format, transformation, failure behavior, header preservation. |
| F-ba5b7e810055 | Receives payload, current length, local buffer, 255, and 0. | Bounds, output representation, termination, errors. |
| F-b9f19a3ec83a | Receives delivery arguments around the local-buffer address. | Consumption, retention, ownership, reporting. |
| F-1137792fba7d | No explicit arguments; called before clearing nonzero buffer state during growth. | Resource identity, release semantics, header/context/global effects. |
| F-f195b74c9758 | Receives calculated capacity; zero result ends processing. | Allocation semantics, ownership, initialization, alignment, other mutations. |
| F-c8bd0cbe99d3 | Receives source, later half-length count, destination, capacity, output-count address. | Conversion semantics, output guarantees, bounds, errors, other mutations. |
| F-7834b08494a4 | Receives delivery arguments around an address and 32-bit size. | Consumption, retention, ownership, reporting. |

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Except for expressly stated mutations, assume dependencies return normally and leave unrelated state unchanged.

1. **Header failure.** A nonzero first input result causes immediate return, with no second input or delivery.

2. **Payload failure.** A nonzero second input result prevents payload normalization and delivery. Header normalization may already have occurred.

3. **Ordinary temporary conversion.** With stable type 1 and mode 2, F-ba5b7e810055 receives the current length and local buffer; F-b9f19a3ec83a performs the selected delivery.

4. **Enabled transformation redirects dispatch.** Initially observed type 2, nonzero control, and mode 2 cause F-370c54f34dce to run. If that call changes the header type to 1, subsequent redispatch selects temporary-buffer conversion.

5. **Disabled transformation bypasses type rereads.** Initially observed type 2 and zero control proceed directly to the context-buffer mode check. Mode 1 proceeds to bound processing without the post-transformation type checks.

6. **Payload transformation without context-buffer conversion.** With stable type 2, length 7, nonzero control, and mode other than 1, F-370c54f34dce receives count 3. General delivery then uses length 7.

7. **Stable bound.** With length 20 and both bound reads returning 8, input still requests 20 and enabled payload transformation receives count 10. Capacity sizing uses length 8, giving count 4 and requirement 12.

8. **Changing bound increases the assigned length.** With length 20, comparison-bound value 8, and assignment-bound value 30, the header becomes 30. With no later header mutation, the requirement is 45 and conversion receives count 15. No local check reconciles that count with the earlier 20-byte input request.

9. **Replacement separates sizing from conversion count.** Suppose sizing uses length 10, producing requirement 15, and existing capacity is insufficient. If a replacement dependency changes the header length to 18 through a retained header address, successful allocation still records requirement 15, but conversion receives count 9. No capacity recalculation occurs.

10. **Replacement failure with an existing address.** The no-argument dependency runs, then address and capacity are cleared in that order. A zero allocation result is stored in the address field, and processing returns without delivery.

11. **Zero-length context conversion.** With stable zero length, the requirement is zero. Growth is skipped even if the stored buffer address is zero. Conversion still receives count zero and the stored buffer state; its acceptance of those arguments is documented_only.

12. **Converter changes the delivery address.** If F-c8bd0cbe99d3 changes the context buffer field and writes output count 6, final delivery uses the changed address and size 6, not necessarily the conversion destination.

## Uncertainties

The following remain documented_only:

- Header byte order, payload encoding, type meanings, and delivery targets.
- The conversion-bound global’s values and stability across its separate reads.
- Whether dependencies retain the header address or mutate the header during later calls.
- Completeness guarantees following successful input.
- Conversion bounds, termination, output-count guarantees, and error signaling.
- The allocation-like and no-argument dependencies’ ownership relationship.
- Memory validity when arithmetic wraps, counts change after sizing, or stored capacity and address disagree.
- Aliasing, concurrent mutation, synchronization, reentrancy, and address-retention contracts.

These unknowns do not justify assuming stable state, successful conversion, or library-style ownership conventions.

## Evidence

Established static evidence consists of the supplied body’s access widths and offsets, call arguments, branch structure, separate bound reads, type rereads, unsigned arithmetic, early returns, and context-field writes.

Apparent operational roles are inferred intent, not verified dependency semantics. Dependency implementations and runtime traces were not supplied. No runtime tests have been performed.
