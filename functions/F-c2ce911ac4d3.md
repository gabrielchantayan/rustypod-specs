<!-- Trusted provenance: F-c2ce911ac4d3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c2ce911ac4d3

## Purpose

F-c2ce911ac4d3 advances a cursor within a bounded region and requests extraction of up to a signed caller-supplied number of values. Each completed extraction is followed by a 16-bit destination write. The function returns the number of those writes and publishes its final cursor through the caller’s state object.

The established behavior distinguishes an initial `[` or `{` from other initial bytes:

- An initial `[` or `{` is consumed and enables repeated extraction.
- Other initial bytes permit at most one extraction.
- An initially observed zero byte can instead cause a one-byte cursor advance without extraction, subject to bounds and capacity checks.

The apparent intent is a bounded parser supporting bracketed collections and scalar input. That interpretation is not a complete behavioral contract: the visible closing-delimiter comparison uses the initially observed byte, not a newly read byte at the current cursor. Consequently, the visible implementation does not recognize closing delimiters in the ordinary way.

The meaning of an extracted value remains unresolved because the 16-bit quantity written to the destination is supplied through an auxiliary dependency output whose production is not shown.

## Interface

The function accepts four arguments:

| Argument | Established role |
|---|---|
| State pointer | Addresses a mutable array-like object with 32-bit slots. Slot 0 contains the current cursor; slot 2 contains the region endpoint. Slot 1 has no direct access in this function. |
| Capacity | A signed 32-bit integer limiting the number of destination writes. Zero and negative values prohibit extraction but do not prohibit all cursor movement or dependency calls. |
| Destination address | An address represented by a 32-bit integer expression. Successive extraction results are written as 16-bit quantities at two-byte intervals. |
| Auxiliary pointer | Accepted and initially assigned to temporary cursor storage, but overwritten before this function uses that stored value. No direct dereference of this argument is visible. |

The return value is a signed 32-bit count, initially zero and increased once for each extraction followed by a destination write.

The state layout visible to this function is:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 32 bits | Cursor representation; read after the initial dependency call and overwritten on normal return |
| 4 | 32 bits | No direct access |
| 8 | 32 bits | Endpoint representation; read for the initial dependency argument and read again after that call |

These widths follow the supplied representation. They do not independently establish the original public ABI, pointer provenance rules, alignment requirements, or byte order.

Necessary execution preconditions include:

- The state pointer permits the displayed slot reads and final slot-0 write.
- Any input byte actually read by this function is accessible.
- Each destination location actually written permits a 16-bit write.
- Dependency-specific argument requirements are satisfied.
- Address calculations and comparisons have meaningful behavior in the execution environment.

There is no explicit validation of null pointers, destination extent, destination alignment, state consistency, or arithmetic overflow.

## Behavior

### Initial dependency call and state capture

Before checking capacity or comparing the cursor with the endpoint, the function calls F-1fe3b1afff2c with:

1. The state pointer itself.
2. The endpoint value obtained from state slot 2 for that call.

The dependency’s return value is not used.

Only after that call returns does F-c2ce911ac4d3 read state slot 0 into its working cursor and read state slot 2 into its working endpoint. Therefore, mutations made by the initial dependency can affect both the cursor and endpoint used by the rest of the operation.

The initially supplied endpoint argument and the subsequently captured endpoint need not be equal if the dependency changes slot 2.

The function initializes the extraction count to zero. If the captured cursor is at or beyond the captured endpoint, it performs no direct input-byte read, makes no extraction call, and proceeds to publish the cursor and return zero. The initial dependency call has nevertheless already occurred.

### Initial byte and mode selection

When the cursor is below the endpoint, the function reads exactly one byte directly from that cursor. This byte is retained for later comparisons; no subsequent direct input-byte reload is visible.

The function selects a mode from that byte:

- For `[`, it records `]` as the expected closing byte and advances the working cursor by one byte.
- For `{`, it records `}` as the expected closing byte and advances the working cursor by one byte.
- For every other byte, it records zero as the closing-byte marker and does not immediately advance the cursor.

The opener advance occurs before the loop’s capacity check. A capacity of zero or less therefore does not prevent consumption of a directly observed opener.

### Checks before each potential extraction

Each pass performs these operations in order:

1. Compare the working cursor with the captured endpoint. If the cursor is at or beyond the endpoint, stop.
2. Call F-1fe3b1afff2c with the address of the working cursor and the captured endpoint.
3. Compare the possibly changed working cursor with the captured endpoint again. If it is at or beyond the endpoint, stop.
4. Compare the extraction count with capacity using signed ordering. If the count is greater than or equal to capacity, stop.
5. Compare the retained initial byte with the selected closing-byte marker.

This ordering is observable. In particular:

- A normalization-like dependency call can occur even when capacity is zero or negative.
- No such loop dependency call occurs when the cursor is already at or beyond the endpoint.
- The initial byte comparison does not inspect the byte at the cursor produced by that dependency.

The function does not refresh its captured endpoint from the state object during these passes.

### Closing-byte comparison

If the retained initial byte equals the selected closing-byte marker, the function advances the current working cursor by one byte and stops, without extraction or destination write.

For the explicitly recognized openers, this comparison cannot succeed: the retained byte is `[` while the selected marker is `]`, or it is `{` while the selected marker is `}`.

For non-opener input, the marker is zero. Thus the visible equality branch is reachable when the retained initial byte is zero, provided the preceding bounds and capacity checks allow it.

The byte advanced over in that case is at the cursor after F-1fe3b1afff2c returns. It is not necessarily the location of the initial zero byte, because that dependency can change the cursor.

### Extraction and destination write

If all checks allow extraction, the function calls F-2f16e714b3bb with:

1. The address of the working cursor.
2. The captured endpoint.
3. A literal zero argument.

After the dependency returns, the function writes a 16-bit auxiliary output to the destination location corresponding to the current extraction index. The first write starts at the supplied destination address, and subsequent writes start two bytes apart.

The extraction count increases once per such call-and-write sequence. No visible branch tests a success status, error code, or primary return value from F-2f16e714b3bb before incrementing the count and writing.

The primary return value of that dependency is not the value visibly selected for the destination write. The auxiliary 16-bit output is not sufficiently defined by the supplied material to specify its numeric meaning.

### Repetition and termination

After an extraction:

- Opener mode repeats the pre-extraction checks.
- Non-opener mode stops immediately.

Accordingly, non-opener mode performs at most one extraction.

In opener mode, a later pass can call F-1fe3b1afff2c before discovering that the extraction count has reached capacity. If the cursor is still below the endpoint, this permits additional dependency-driven cursor movement after the final permitted destination write.

There is no explicit progress requirement for F-2f16e714b3bb. Even if it leaves the cursor unchanged, a finite positive capacity still bounds the number of extraction writes, assuming ordinary completion and valid arithmetic.

### Final publication

On the normal completion paths shown, the function writes its working cursor into state slot 0 and returns the extraction count.

This final write occurs even when the count is zero.

The function does not directly publish a new endpoint. If a dependency changes state slot 2 through another accessible reference, that mutation is not explicitly undone here. If a dependency changes state slot 0 independently, the final cursor write can overwrite that change.

## Side effects

Directly established effects are:

- One unconditional initial call to F-1fe3b1afff2c.
- Reads of the state fields described above.
- At most one direct input-byte read.
- Zero or more further calls to F-1fe3b1afff2c.
- Zero or more calls to F-2f16e714b3bb.
- One 16-bit destination write per counted extraction.
- A final 32-bit cursor publication to state slot 0.

No direct allocation, deallocation, ownership transfer, global-variable access, logging, or error reporting is visible. This does not establish that the dependencies lack such effects.

No dependency is assumed to preserve globals, preserve unrelated state, or own or release supplied memory.

Destination writes occur after their corresponding extraction calls and before subsequent loop checks or dependency calls. If the destination overlaps the input or state, those writes can affect memory used later. No overlap protection is visible. The final cursor publication can also overwrite an overlapping destination result.

The signed count itself does not need to exceed the largest positive signed 32-bit value before capacity stops further extraction. However, the doubled destination index can exceed the signed 32-bit range beginning at extraction index 1,073,741,824. Adding the destination base can also overflow independently. No checked arithmetic is shown; overflow behavior is documented_only rather than assigned portable semantics.

## Dependencies

### F-1fe3b1afff2c

Two invocation forms are visible:

- Initial invocation: state pointer and the endpoint read from state slot 2.
- Loop invocation: address of the working cursor and the captured endpoint.

Its return value is ignored in both forms.

Its exact scanning rules, memory accesses, state layout expectations, side effects, and failure behavior are unknown. Describing it as whitespace skipping or normalization is inferred intent, not established behavior.

### F-2f16e714b3bb

Receives the working cursor by address, the captured endpoint, and zero.

It can affect the cursor through the supplied address. A 16-bit auxiliary output is used for the subsequent destination write, but its definition is unresolved.

Its parsing grammar, consumption rules, error signaling, progress guarantees, and memory ownership effects are unknown. No visible error-result handling surrounds it.

## Acceptance examples

These examples are conditional behavioral obligations, not executed tests. Except where stated otherwise, they assume valid non-overlapping storage, returning dependencies, and arithmetic within range.

1. **Empty region after the initial dependency**  
   If the captured cursor equals the captured endpoint, the result is zero. No direct input byte is read, no extraction occurs, and the captured cursor is written back. The initial dependency call still occurs.

2. **Zero capacity with an ordinary initial byte**  
   If the initial byte is nonzero and not an opener, the cursor is below the endpoint, and the loop dependency leaves it unchanged, the function returns zero without destination writes. That loop dependency is called before capacity stops processing.

3. **Zero capacity with an opener**  
   If the initial byte is `[` and at least one byte remains afterward, the opener is consumed before capacity is checked. The loop dependency is then called. With no further dependency movement, the final cursor is one byte past the opener and the return value is zero.

4. **Positive capacity with a non-opener**  
   If the initial byte is nonzero and not an opener, the loop dependency leaves the cursor in bounds, and capacity is positive, exactly one extraction call and one 16-bit destination write occur. The return value is one, even if capacity is larger.

5. **Retained zero byte**  
   If the initial byte is zero, capacity is positive, and the loop dependency leaves the cursor in bounds, no extraction occurs. The current cursor advances one byte and the function returns zero.

6. **Closing delimiter encountered after an opener**  
   If the initial byte is `[` and the cursor later reaches a `]`, this function does not directly recognize that current byte as a closing delimiter. If bounds and capacity permit, F-2f16e714b3bb is called. Its response to `]` is unknown.

7. **Capacity reached while input remains**  
   In opener mode, after the final allowed extraction, if the cursor remains below the endpoint, F-1fe3b1afff2c is called once more before capacity terminates the loop. Any cursor change from that call is included in the final published cursor.

8. **Extraction dependency makes no progress**  
   In opener mode with positive capacity, if both dependencies repeatedly leave an in-bounds cursor unchanged, the function still performs the capacity-limited number of extraction writes. It does not stop merely because the cursor failed to advance.

## Uncertainties

The following remain documented_only:

- The numeric meaning and production of the auxiliary 16-bit output.
- Both dependencies’ full contracts and failure behavior.
- Whether the retained-byte comparison reflects intended behavior or an incomplete recovered representation.
- Pointer ordering semantics, byte order, and alignment constraints of the original environment.
- Behavior after invalid memory access, arithmetic overflow, dependency non-return, or dependency-induced memory invalidation.
- Any intended significance of the fourth argument beyond its unused initial assignment.

These unknowns must not be replaced with assumptions that values are valid, closing delimiters are consumed correctly, or failures are represented by a particular return value.

## Evidence

Established static evidence consists of the supplied function representation: argument and storage widths, dependency call sites, state accesses, branch conditions, cursor updates, destination-store width and spacing, and return-count behavior.

Collection parsing, normalization, and element decoding are inferred intent only.

No dependency implementations, independent ABI documentation, or runtime observations were supplied. No runtime tests have been performed.
