<!-- Trusted provenance: F-f2d9a0ba537a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2d9a0ba537a

## Purpose

F-f2d9a0ba537a conditionally replaces destination storage, prepares an embedded coordinate collection, invokes a context-supplied callback, and performs post-callback coordinate adjustment. A later zero-status check enables updates to the target’s discriminator and two position fields.

Rendering coordinates into a bitmap-like destination is inferred intent. The established contract is the sequence of comparisons, arithmetic, memory accesses, dependency calls, and status checks below.

The operation is not transactional. An existing owned buffer may be submitted for release before allocation fails. Destination metadata is written before allocation status is checked. Callback failure does not itself release the replacement buffer.

Two local storage areas escape to dependencies: the four-word bounds storage and the status word. Their values must not be treated as immutable snapshots after those calls. Later dependencies may modify them through retained addresses; no non-retention guarantee is supplied.

## Interface

The function accepts seven arguments and returns a 32-bit status word.

| Argument | Established use |
|---|---|
| Context | Address of a record containing a comparison value, resource handle, callback context, and callback entry. |
| Target | Address of a mutable record containing destination metadata, an embedded coordinate object, and a flags-record pointer. |
| Requested value | 32-bit value compared with the fifth argument. |
| Optional translation | Null, or an address supplying two 32-bit components. |
| Required value | 32-bit value that must equal the third argument. |
| Horizontal factor | 32-bit factor; zero disables horizontal multiplication and post-callback horizontal adjustment. |
| Vertical factor | 32-bit factor; zero disables vertical multiplication and post-callback vertical adjustment. |

The supplied representation uses 32-bit addresses and ordinary 32-bit integer fields. All offsets below are byte offsets.

### Context layout

| Offset | Access | Established role |
|---|---|---|
| `0x08` | Read, 32 bits | Resource handle captured for release and allocation. |
| `0x18` | Read, 32 bits | Compared with target offset `0x48`. |
| `0x3C` | Read, 32 bits | First callback argument. |
| `0x40` | Read, 32 bits | Indirect callback entry. |

### Target layout

| Offset | Access | Established role |
|---|---|---|
| `0x48` | Read/write, 32 bits | Initial comparison field; conditionally replaced from a global data word. |
| `0x4C` | Write, 32 bits | Destination row count. |
| `0x50` | Write, 32 bits | Logical row width. |
| `0x54` | Write, 32 bits | Row stride. |
| `0x58` | Read/write, 32 bits | Destination buffer value. |
| `0x5C` | Write, 16 bits | Set to `0x0100`. |
| `0x5E` | Write, 8 bits | Set to `2`. |
| `0x64` | Write, 32 bits | Conditionally receives a shifted lower horizontal bound. |
| `0x68` | Write, 32 bits | Conditionally receives a shifted upper vertical bound. |
| `0x6C` | Passed by address | Embedded coordinate object. |
| `0x6E` | Repeated read, signed 16 bits | Coordinate-pair count for local loops. |
| `0x70` | Read, 32 bits | Coordinate-pair storage address. |
| `0x9C` | Read, 32 bits | Flags-record address. |

Each coordinate pair occupies eight bytes: a 32-bit horizontal component followed by a 32-bit vertical component. Bit 0 of the 32-bit word at flags-record offset `0x04` controls the old-buffer release sequence and is set after allocation is accepted.

The optional translation supplies its horizontal component at offset `0` and vertical component at offset `4`.

A separate global 32-bit data word supplies the successful discriminator update. Its numerical value and immutability are not established.

Required addresses, reachable storage, callback entry, and dependency-specific inputs must be valid for the accesses actually reached. The function supplies no general pointer, capacity, range, or callback-validity checks.

## Behavior

### Eligibility

The target field at `0x48` is compared with the context field at `0x18`.

- A mismatch assigns status `6` and returns without invoking dependencies or explicitly writing the target.
- Otherwise, the third and fifth arguments are compared.
- A mismatch there assigns status `19` and returns without invoking dependencies or explicitly writing the target.

The first comparison takes precedence.

### Initial translation and bounds

The embedded-object address is the target address plus `0x6C`.

If the optional translation is non-null, F-074238b57126 receives the embedded object and the two translation components. Those components are read for this call; they are not saved for final cleanup.

F-960c2eabcfd8 then receives the embedded object and the address of four consecutive 32-bit local words. Their subsequent uses identify their order as lower horizontal, lower vertical, upper horizontal, and upper vertical bounds.

The caller updates that storage in this order:

1. Clear the lower horizontal word’s low six bits.
2. Clear the lower vertical word’s low six bits.
3. Add `63` to the upper horizontal word modulo \(2^{32}\), then clear its low six bits.
4. Apply the same operation to the upper vertical word.

Each extent is the corresponding upper-minus-lower difference modulo \(2^{32}\), interpreted as signed 32-bit and arithmetically shifted right by six.

For ordinary non-overflowing inputs, this rounds bounds outward onto a 64-unit grid. Integer-limit behavior follows the stated wrapping operations; no overflow rejection is present.

The computed extents are retained separately for destination sizing. Later mutation of the escaped bounds storage does not, by itself, recompute those extents. Conversely, subsequent normalization, restoration, and position writes read the bounds storage again and can observe changed values.

### Old-buffer release

After bounds alignment and extent calculation, the resource handle is read from context offset `0x08` and retained for the release and allocation calls.

The ownership bit is examined through the target’s flags-record pointer. If set:

1. F-bf7242c98d06 receives the retained handle and current buffer value.
2. Target offset `0x58` is written as zero.
3. The flags-record pointer and word are accessed again, and bit 0 is cleared while preserving the other bits then read.

The release dependency is not assumed to preserve the flags pointer, flags word, bounds storage, or other reachable state. The post-call clear therefore need not affect the same flags record initially inspected.

If the ownership bit is clear, no release call or explicit buffer-zeroing occurs, even when the buffer value is nonzero. Allocation will still overwrite that value.

### Destination sizing and allocation

Initially, logical width and stride both equal the computed horizontal extent.

For a nonzero horizontal factor:

- Logical width becomes the low 32 bits of factor times horizontal extent.
- Stride becomes logical width plus `3`, modulo \(2^{32}\), with its low two bits cleared.

A zero horizontal factor leaves both values equal to the extent; it does not produce zero width or request four-byte stride alignment.

The caller next writes byte `2` at target offset `0x5E`, followed by the 16-bit value `0x0100` at `0x5C`.

Row count is the computed vertical extent when the vertical factor is zero. Otherwise, it is the low 32 bits of their product.

The destination fields are then written in this order: stride at `0x54`, row count at `0x4C`, logical width at `0x50`.

F-3ce007b692cc receives:

- The retained resource handle.
- The low 32 bits of row count times stride.
- The address of the local 32-bit status word.

Its returned buffer value is stored at target offset `0x58` before status is checked, regardless of that status.

The status currently in local storage determines entry into processing:

- Nonzero skips normalization, coordinate passes, callback invocation, and the success-field block, proceeding to final optional-translation cleanup.
- Zero enters processing.

There is no explicit rejection of zero dimensions, negative factors or extents, multiplication overflow, or a zero buffer value accompanied by zero status. The allocator’s interpretation of the size bit pattern is unknown.

### Preparation and callback

On entry into processing, bit 0 is set in the flags word reached through the then-current flags pointer, preserving its other observed bits.

F-074238b57126 receives the embedded object and the 32-bit modular negations of the lower horizontal and vertical words currently in the escaped bounds storage. These need not equal the aligned values originally computed if an intervening dependency changed that storage.

The callback argument record’s third consecutive 32-bit entry is set to `1`.

If the horizontal factor is nonzero, a complete horizontal multiplication pass follows. If the vertical factor is nonzero, a separate vertical multiplication pass follows it. Each component product retains its low 32 bits.

Each enabled pass loads the coordinate-storage address once at its start. Its loop condition repeatedly reads the signed 16-bit count at target offset `0x6E`; zero or negative counts cause no iterations. The cursor advances eight bytes per visited pair. Horizontal operations use the first word; vertical operations use the second.

After these passes, the first two callback-record entries are initialized:

| Entry | Value |
|---|---|
| 0 | Address of the destination descriptor at target offset `0x4C`. |
| 1 | Address of the embedded object at target offset `0x6C`. |
| 2 | Integer `1`, initialized before the multiplication passes. |

No larger initialized record is established.

The callback entry and callback context are obtained from the context at this stage. The callback receives that context value and the stack-record address.

Its returned 32-bit value is assigned to the same local status word whose address was passed to allocation. This assignment replaces any earlier contents, including modifications made before the callback returned. It does not prevent later dependencies from modifying the word through a retained address.

### Post-callback adjustment and success fields

Post-callback adjustment occurs without first testing callback status.

For each enabled axis, the function starts a new pass using the then-current coordinate-storage address. Every visited component is passed with its factor to F-870c4fcf7295, whose returned 32-bit value is written back to that visited address. The horizontal pass precedes the vertical pass.

Each pass repeatedly reads the current signed count. A dependency may therefore change the count used by later iterations. Changing the target’s storage pointer during a pass does not redirect that pass’s already-established cursor, though a later pass loads the pointer again.

F-074238b57126 is then called with the embedded object and the lower horizontal and vertical words currently in the local bounds storage.

Only after these adjustment calls and this translation call is the local status word tested again. The test is not necessarily a test of the callback’s original return value.

If the status currently stored is zero, the caller performs these operations in order:

1. Read the global 32-bit data word and store its value at target offset `0x48`.
2. Read the current lower horizontal bounds word, interpret it as signed, arithmetically shift it right by six, and store it at `0x64`.
3. Read the current upper vertical bounds word, apply the same signed shift, and store it at `0x68`.

The global read occurs after the post-callback translation and subsequent zero-status check. The source does not establish a constant discriminator or a function-entry snapshot of it.

The position writes consume local bounds at their respective reads. They are neither recomputed from the coordinate collection nor guaranteed to use the original aligned bounds. A nonzero status at this check skips all three writes.

### Final cleanup and return

After either allocation branch, final optional-translation cleanup occurs when both the computed embedded-object address and optional-translation pointer are nonzero.

F-074238b57126 receives the embedded object and modular negations of the translation components read again at this point. Changed translation storage therefore changes the cleanup arguments.

The function then returns the current local status word. This read occurs after final cleanup. No later dependency result is directly assigned as status, but a dependency may modify status through the address previously exposed to allocation.

Consequently, final status can differ from the allocator’s reported status or callback return. Cleanup may also change status after success-field writes have already occurred; those writes are not rolled back.

## Side effects

Explicit writes affect destination metadata, the buffer field, ownership flags, coordinate components, local bounds and status storage, and conditionally the discriminator and position fields.

The ownership protocol establishes only caller actions:

- An old buffer is submitted for release only when the ownership bit is observed set.
- The allocation result overwrites the buffer field before status is checked.
- Entry into processing sets the ownership bit.
- A nonzero callback return does not directly trigger buffer release or clear ownership.

Actual acquisition, release, and ownership-transfer semantics remain dependency-defined. A set ownership bit can cause a zero old-buffer value to be passed for release.

The function performs no explicit destination-buffer initialization.

Coordinate preparation and adjustment are not proven inverses. Exact restoration depends on dependency semantics, multiplication range, callback effects, changing counts or storage pointers, and preservation of both bounds and optional-translation values.

Bounds, status, and callback-record addresses refer to local storage valid during the invocation. Retention must not be assumed absent, but safe use after return is not established.

No synchronization or rollback is visible. Aliasing and dependency mutations can affect later reads; unknown callees must not be assumed to preserve globals or reachable state.

## Dependencies

| Opaque dependency | Observed contract and limits |
|---|---|
| F-960c2eabcfd8 | Receives the embedded object and four-word bounds-output address. Exact bounds computation, empty-object behavior, retention, and other effects are unknown. |
| F-bf7242c98d06 | Receives the retained resource handle and old buffer value on the ownership path. Return value is not consumed. Actual release and preservation guarantees are unknown. |
| F-3ce007b692cc | Receives handle, 32-bit size, and local status address; returns the replacement buffer value. No non-retention guarantee applies to the status address. |
| F-074238b57126 | Receives the embedded object and two 32-bit offsets. Placement suggests translation. Return value is not consumed; full effects and preservation guarantees are unknown. |
| F-870c4fcf7295 | Receives a component and nonzero factor; returns the replacement component. Placement suggests inverse scaling, but rounding, division, saturation, and exceptional behavior are unestablished. |
| Context-supplied callback | Receives context data and the local three-entry argument record. Return value is assigned to status. Identity, additional input requirements, retention, and side effects are unknown. |

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Numerical examples assume valid storage and no intervening mutation of escaped bounds or status storage except the specified outputs.

1. **Initial mismatch.** Different initial comparison fields produce `6`, with no dependency invocation. This takes precedence over a mismatch between the third and fifth arguments.

2. **Secondary mismatch.** Matching initial fields but unequal third and fifth arguments produce `19`, without translation or destination preparation.

3. **Zero factors.** Bounds `(64, 128, 192, 320)` produce horizontal extent `2` and vertical extent `3`. With both factors zero, width is `2`, stride is `2`, row count is `3`, and allocation size is `6`.

4. **Conditional stride alignment.** With those bounds, horizontal factor `1` and vertical factor zero produce width `2`, stride `4`, row count `3`, and allocation size `12`.

5. **Asymmetric scaling.** With those bounds, factors `3` and `2` produce width `6`, stride `8`, row count `6`, and allocation size `48`. All horizontal preparation writes precede vertical preparation writes.

6. **Allocation rejection.** If allocation leaves nonzero status at the immediate check, callback processing is skipped. An owned old buffer has already been submitted for release, metadata has been written, and the returned buffer value has been stored. Returning that same status additionally requires that final cleanup not modify status.

7. **Nonzero callback return.** Enabled adjustment passes and lower-bound translation still occur. If they leave status nonzero, success-field writes are skipped. If a later dependency instead changes status to zero before the success check, those writes occur despite the original nonzero callback return.

8. **Status changes after success.** A zero status at the success check permits discriminator and position writes. If final cleanup subsequently changes status to nonzero through a retained address, that nonzero value is returned without undoing those writes.

9. **Mutable bounds.** If a later dependency changes a lower bound before normalization, normalization consumes the changed word. If bounds change again before restoration or position writes, those operations consume the later values. Previously computed dimensions are not automatically recalculated.

10. **Successful positions and discriminator.** With the bounds from example 3 preserved through the relevant reads and zero status at the success check, `0x64` receives `1` and `0x68` receives `5`. Offset `0x48` receives the global word’s value when read after that check, not an established fixed constant.

11. **Nonpositive count.** A pass whose loop condition observes a nonpositive count performs no component writes while that condition remains false. This alone does not suppress allocation, translation calls, or callback invocation.

## Uncertainties

The following remain **documented_only**:

- The global discriminator word’s numerical value, initialization, and mutability.
- Complete record types and callback-record extent.
- Bounds computation and dependency behavior for empty or malformed objects.
- Actual buffer allocation, release, initialization, and ownership guarantees.
- Interpretation of allocation-size bit patterns.
- F-870c4fcf7295 arithmetic and rounding semantics.
- Whether coordinate preparation is exactly reversible.
- Retention and later mutation of escaped bounds, status, or callback-record storage.
- Dependency effects on globals and other reachable state.
- Behavior for invalid addresses, inadequate capacity, unsupported aliasing, or exceptional callback execution.

No additional validation, recovery, immutability, or state-preservation guarantee is implied.

## Evidence

Established static evidence consists of the supplied function’s comparisons, access widths and offsets, arithmetic operations, explicit write order, dependency arguments, escaped local addresses, status-check locations, global read, and final status read.

Rendering, translation, allocation, release, and inverse-scaling terminology describe inferred roles, not independently verified dependency implementations.

No runtime tests, compilation, or dependency implementation inspection have been performed.
