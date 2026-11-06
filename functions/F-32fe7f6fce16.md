<!-- Trusted provenance: F-32fe7f6fce16; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-32fe7f6fce16

## Purpose

F-32fe7f6fce16 requests a memory region from F-d34fa71e963d, checks whether a signed query lies within an interval described by that region, and, when eligible, computes a signed position by examining 32-bit values at descending memory addresses.

The established behavior is a bounded directional scan, not a general-purpose search with validated ordering. Interpreting the result as a predecessor position requires additional assumptions about the stored values.

The function returns a 32-bit status: zero for the completed scan path, or three for dependency failure, interval rejection, or an unsupported selector encountered after interval acceptance. A zero status does not guarantee a nonnegative output position.

## Interface

The function has five inputs, described here by role:

| Input | Representation | Role |
|---|---|---|
| Context address | 32-bit address-bearing value | Supplies two 32-bit fields used in the dependency call. |
| Request value | Opaque 32-bit value | Forwarded to the dependency without local interpretation. |
| Query | Signed 32-bit integer | Used in the interval test and comparisons against scanned values. |
| Selector | Signed 32-bit integer | Initially supplied through a mutable dependency argument; its original value also selects the scan layout. |
| Result destination | Address of a writable 32-bit integer | Receives zero and, on scan paths, subsequent signed position updates. |

The return value is a 32-bit status with locally established values zero and three. No distinct status identifies the individual failure causes.

The context must permit 32-bit reads at byte offsets `0x1060` and `0x1394`. Those values become the first and fourth arguments to F-d34fa71e963d.

The dependency receives four arguments:

1. The context field at offset `0x1060`.
2. A reference to a local 32-bit slot initially containing the selector.
3. The opaque request value.
4. The context field at offset `0x1394`.

On a zero dependency return, the caller interprets the current contents of the mutable slot as a region address. It does not verify that the dependency changed that slot.

For ordinary successful use, the dependency must therefore leave a usable region address in the slot. Merely returning zero is insufficient to establish memory safety.

No input-pointer validation, region-size validation, alignment validation, or overlap rejection is established.

## Behavior

### Dependency invocation

The function first initializes its local mutable slot from the selector and invokes F-d34fa71e963d.

The original selector remains the basis for later layout selection. Changing the mutable slot changes the region address used by the caller; it does not change which selector value the caller subsequently recognizes.

If the dependency returns any nonzero value, the function returns three. It does not propagate the dependency’s particular status and performs no direct write to the result destination on this path.

The two context fields are read for the call. Their relative evaluation order is not established by the supplied representation. Both reads precede entry into the dependency.

### Region metadata and interval gate

After a zero dependency return, the function reads three signed 32-bit quantities from the region:

| Byte offset from region address | Meaning in this contract |
|---|---|
| `0x7FFF0` | Interval extent |
| `0x7FFF4` | Interval origin |
| `0x7FFF8` | Scan limit |

The represented read order is extent, origin, then scan limit.

After all three reads, the function writes zero to the result destination. This write occurs before interval rejection or selector rejection. Consequently, a valid result destination is required even when no scan will occur after a successful dependency call.

The query is accepted by the interval gate only when both conditions hold:

- The query is strictly less than the sum of origin and extent.
- The origin is less than or equal to the query.

The upper-bound comparison is evaluated first. The lower-bound comparison is evaluated only if the upper-bound comparison succeeds.

This is a lower-inclusive, upper-exclusive interval when the sum is representable under ordinary signed 32-bit arithmetic. The function does not widen the sum or explicitly detect overflow. Overflow-boundary behavior is documented_only rather than assigned an unsupported arithmetic model.

If the interval gate fails, the function returns three, with its direct output write having set the result to zero.

### Selector handling and scan layout

The selector is checked only after interval acceptance.

| Original selector | First scanned value offset | Distance between successive scanned values |
|---|---:|---:|
| Zero | `0x7FFEC` | 12 bytes toward lower addresses |
| One | `0x7FFE8` | 16 bytes toward lower addresses |

Any other selector returns three after the result has been initialized to zero. On that path, no scan-value read is performed.

Each scanned value is read as a signed 32-bit integer. The spacing establishes the addresses used by this function, but does not establish the contents or purpose of intervening bytes. In particular, this function does not read an entire record at each position.

The first scan-value read occurs immediately after layout selection. It is not guarded by a positive scan limit.

### Position computation

The output position starts at zero. The first scanned value corresponds to position zero.

While the current scanned value is strictly less than the query, the function advances the output position by one. Immediately after that write, it compares the new position against the previously captured scan limit.

If the new position is greater than or equal to the scan limit:

- The function does not read another scan value.
- It decreases the output position by one.
- It returns zero.

Otherwise, it moves to the next lower scan address, reads that 32-bit value, and repeats the comparison.

When a current value is not less than the query:

- Equality leaves the output position unchanged.
- A value strictly greater than the query causes the output position to decrease by one.

The function then returns zero.

Thus, for a positive scan limit and nondecreasing values in scan order, the output identifies the position immediately before the first value greater than the query, or the first encountered equal position. If every examined value is less than the query, exhaustion leaves the last permitted position.

This interpretation does not imply selection of the last equal value among duplicates: equality stops the scan immediately.

### Boundary cases

The following behavior is established without assuming sorted values:

- A first scanned value equal to the query produces result zero.
- A first scanned value greater than the query produces result negative one.
- A first scanned value less than the query triggers an increment before the scan-limit test.
- With a scan limit of zero or a negative value, that initial read still occurs. If the first value is less than the query, the increment reaches one, the limit test stops the scan, and the final decrement restores result zero.
- No explicit error is returned for a nonpositive scan limit.
- The function does not inspect later values after an equality, a greater value, or a limit-triggered stop.
- Unsorted values are not rejected. They can cause the scan to stop before a later value that might otherwise be considered a better match.

For a positive scan limit, the scan-limit comparison prevents a read at the position equal to that limit. Position zero is nevertheless read unconditionally once the scan path is entered.

## Side effects

### Direct reads and writes

The directly represented access sequence is:

1. Read the two context fields needed by the dependency, with their relative order unspecified.
2. Invoke F-d34fa71e963d.
3. On dependency success, read extent, origin, and scan limit from the returned region.
4. Write zero to the result destination.
5. On interval acceptance and a supported selector, read the first scan value.
6. For each advance, read the output position and write its incremented value.
7. If the increment remains below the scan limit, move to the next lower scan address and read its value.
8. On overshoot or limit exhaustion, read the output position and write its decremented value.

The region metadata is captured before the initial output write. The metadata is not reloaded during the scan.

The output destination is used as working state, not merely as a final return location. Intermediate positions are written there. The source representation establishes those writes; it does not establish atomicity, synchronization, or visibility guarantees for concurrent observers.

### Aliasing

No non-aliasing requirement is enforced.

If the result destination overlaps a scanned value, initializing or advancing the result can alter data that the scan subsequently reads. The ordinary interpretation as a search over unchanged values therefore requires a result destination that does not overlap values the scan will read.

If the result destination overlaps region metadata, the initial zero write occurs after all three metadata values have already been captured. Such a write can change the underlying region without changing the captured values used by this invocation.

### Ownership and lifetime

No direct allocation, deallocation, region release, or ownership-transfer operation is visible in this function. It passes a reference to a local slot to the dependency, consumes the resulting region address, and performs no explicit release afterward.

The dependency’s ownership conventions, lifetime guarantees, retained references, and effects on global state are unknown. None should be inferred from this caller alone.

The function does not locally recover from inaccessible addresses or other invalid-memory conditions. Such conditions are outside the established status-three error contract.

## Dependencies

### F-d34fa71e963d

This is the only visible called dependency.

Established caller-facing behavior:

- It receives two context-derived 32-bit values, the opaque request value, and a reference to a mutable 32-bit slot.
- The slot initially contains the selector.
- A zero return causes the caller to interpret the slot’s current value as a region address.
- Any nonzero return causes the caller to return three.
- The caller does not preserve the dependency’s error code.
- The caller does not verify the resulting region address before reading from it.

Documented_only aspects include the dependency’s own validation, memory access, allocation or release behavior, global-state mutations, failure-side slot changes, and whether it retains the slot reference.

Statements that this function “does not write the result” on dependency failure concern only its direct writes. They do not establish absence of indirect dependency effects on overlapping or globally reachable memory.

## Acceptance examples

These are conditional static examples, not runtime test results. Unless stated otherwise, assume valid readable memory, a writable non-overlapping result destination, a zero dependency return, and representable address and interval arithmetic.

For the scan examples, let the origin be 100 and the extent be 100, giving the accepted interval from 100 inclusive to 200 exclusive. Let selector zero use values at offsets `0x7FFEC`, `0x7FFE0`, and `0x7FFD4`. With selector one, the corresponding first three offsets are `0x7FFE8`, `0x7FFD8`, and `0x7FFC8`.

| Conditions | Required result |
|---|---|
| Dependency returns any nonzero status | Return three; no direct result-destination write by this function. |
| Query is 99 | Return three; result zero. No scan-value read. |
| Query is 200 | Return three; result zero. No scan-value read. |
| Query is 150, selector is two | Return three; result zero. Metadata is read, but scan values are not. |
| Query is 100, first value is 100 | Return zero; result zero. |
| Query is 110, first value is 120 | Return zero; result negative one. |
| Query is 150, limit is three, values in scan order are 110, 140, 170 | Return zero; result one after reading all three values. |
| Query is 140, limit is three, values are 110, 140, 170 | Return zero; result one. The third value is not read. |
| Query is 180, limit is three, values are 110, 140, 170 | Return zero; result two. No fourth value is read. |
| Query is 150, limit is one, first value is 110 | Return zero; result zero. No second value is read. |
| Query is 150, limit is zero, first value is 110 | Read the first value, then return zero with result zero. |
| Query is 150, limit is negative, first value is 170 | Read the first value, then return zero with result negative one. |

For duplicate values, query 140 with scan-order values 110, 140, 140 stops at position one; it does not search onward for another equal value.

For an unordered sequence such as 110, 170, 140 and query 150, the function stops at the second value and reports position zero. It does not examine the later 140.

## Uncertainties

The following remain documented_only:

- The application-level meaning of the context, request, selectors, region, and statuses.
- Whether the region describes records, boundaries, offsets, or another structure.
- Whether producers guarantee nondecreasing scan values or a positive scan limit.
- The exact overflow semantics of the signed interval sum and address calculations.
- Required alignment and the underlying byte order.
- The validity and lifetime of the dependency-provided region.
- Dependency side effects, including global mutation and memory ownership.
- Behavior under concurrent mutation of the region or result destination.
- Whether supported callers intentionally permit aliasing.

“In-range predecessor lookup” is inferred intent under suitable ordering and storage assumptions, not an unconditional property. The directly established contract is the access, comparison, update, and return behavior described above.

## Evidence

The supplied static function representation establishes the dependency call, context offsets, metadata offsets, signed comparisons, output-write placement, selector-dependent scan addresses, stopping conditions, and return values.

The scan addresses and strides are derived from the supplied address expressions. No additional record fields or data-format semantics are inferred from their spacing.

There is no implementation evidence for pointer validation, sortedness validation, positive-limit validation, ownership management, or detailed error reporting in this function.

No runtime tests have been performed. All acceptance examples are conditional deductions from the supplied static representation, not observed executions.
