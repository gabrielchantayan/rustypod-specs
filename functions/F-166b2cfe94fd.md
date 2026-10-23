<!-- Trusted provenance: F-166b2cfe94fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-166b2cfe94fd

## Purpose

F-166b2cfe94fd conditionally raises one caller-supplied signed integer to a stored lower limit and conditionally lowers another caller-supplied signed integer to a stored upper limit.

The value −1 acts as a protected sentinel in either caller-supplied integer: a value observed as −1 is not replaced by the corresponding limit. The stored limits themselves are not validated, and −1 is not treated specially when it appears in a limit.

The two adjustments are ordered, not simultaneous. The lower-side operation completes before the upper limit is obtained. Consequently, aliasing between the supplied locations and the object’s storage can change the second operation’s inputs or destination.

This is not a general interval-validation operation. It neither establishes that the lower limit is at most the upper limit nor guarantees that the two resulting caller values form an ordered interval.

## Interface

The represented interface has three arguments and no return value:

| Argument | Role |
|---|---|
| Object address | Identifies an object containing a reference to limit storage at byte offset 4. |
| Lower-value address | Identifies a readable signed integer location that is conditionally written. |
| Upper-value address | Identifies a readable signed integer location that is conditionally written. |

The object and limit-storage layout relevant to this operation is:

| Location | Meaning |
|---|---|
| Object address + 4 | Stored address of the limit record. |
| Limit-record address + 0x38 | Signed lower limit. |
| Limit-record address + 0x3C | Signed upper limit. |

Offsets are byte offsets. No other object fields are explicitly accessed.

**Width qualification:** the supplied representation uses integer-typed addresses and signed integer values, with adjacent limit fields four bytes apart. This supports an intended 32-bit address and signed 32-bit value model. The actual target ABI, integer width, address width, alignment requirements, and byte order are not independently established. Examples below use values representable in signed 32-bit integers and do not depend on byte order.

Under the intended 32-bit model, comparisons are signed, the protected sentinel is the integer −1, and the numerical range is −2,147,483,648 through 2,147,483,647. The operation performs no arithmetic on the caller values or limits. Its additions are address calculations for the stated offsets.

Required memory preconditions are:

- The object’s address-reference field must be readable whenever accessed.
- The referenced lower-limit and upper-limit fields must be readable when reached.
- Each caller-value location must be readable when examined.
- A caller-value location must be writable if its replacement condition is met.
- Address calculations and accesses must be valid for the target execution model.

There are no explicit null, alignment, address-range, or access-permission checks. Failure to meet these preconditions has no defined recovery behavior in the supplied evidence.

## Behavior

### Lower-side adjustment

The operation first obtains the limit-record address from the object and reads the lower limit at offset 0x38. This limit is captured before the lower caller value is examined.

It then compares the lower caller value with that captured limit using signed ordering:

- If the caller value is not less than the limit, no lower-side write occurs.
- If it is less than the limit, the caller location is examined for the −1 sentinel.
- If the sentinel check observes −1, no lower-side write occurs.
- Otherwise, the captured lower limit is written to the lower-value location.

For stable, non-overlapping storage, this means an ordinary lower caller value is raised to the limit only when it lies below that limit. Equality does not cause a write. A caller value of −1 remains unchanged, regardless of its relationship to the limit.

The sentinel check is conditional: it is reached only after the less-than comparison succeeds. The supplied representation contains another read of the caller location at that point. It does not establish that a single caller-value snapshot is used throughout the decision.

### Upper-side adjustment

After the lower-side operation completes, the operation obtains the limit-record address from the object again. It then reads the upper limit at offset 0x3C.

The second address lookup is significant. The upper limit is not necessarily taken from the same record as the lower limit if the earlier write changed the object’s reference field.

The upper caller value is compared with the captured upper limit:

- If the caller value is less than or equal to the limit, the operation returns immediately without an upper-side write.
- Otherwise, the caller location is examined for the −1 sentinel.
- If the sentinel check observes −1, no upper-side write occurs.
- Otherwise, the captured upper limit is written to the upper-value location.

For stable, non-overlapping storage, an ordinary upper caller value is lowered only when it exceeds the upper limit. Equality does not cause a write. A caller value of −1 remains unchanged.

The second upper-value read occurs only when the initial comparison finds a value greater than the limit. All normal completion paths return without a result value.

### Ordering and aliasing

The established access order at the supplied representation level is:

1. Obtain the record reference and capture the lower limit.
2. Examine the lower caller location and perform any lower-side write.
3. Obtain the record reference again and capture the upper limit.
4. Examine the upper caller location and perform any upper-side write.

The record-reference and limit reads are not skipped merely because a corresponding caller location contains −1. Each limit is obtained before its caller value is checked.

No non-aliasing requirement is expressed in the supplied representation. If the two caller addresses identify the same integer, the upper-side operation observes the result of any lower-side write. It does not operate on the original value independently.

Likewise, a lower-side write can affect the later upper-limit lookup if its destination overlaps relevant object storage. For example:

- A write to the stored record reference can redirect the subsequent upper-limit read.
- A write to the upper-limit field can change the upper limit subsequently captured.
- A write to the upper caller location can change the value subsequently compared.

Such overlapping arrangements are meaningful only where the target’s memory and aliasing rules permit them. The supplied evidence does not establish those rules for every possible overlap.

### Boundary behavior

There is no rejection of reversed limits. If the stored lower limit exceeds the stored upper limit, the two operations still run in their normal order.

Negative limits are compared normally. A stored limit of −1 can be written to a caller location containing a different value. Sentinel protection concerns the caller value observed during its sentinel check; it does not prohibit producing −1 as an output.

At signed minimum and maximum values, the same comparison rules apply. There is no caller-value increment, decrement, subtraction, or other arithmetic that could overflow.

## Side effects

The only explicit writes are conditional replacements of the two caller-supplied integer locations.

For distinct destinations, there is at most one write to each location. If both addresses identify the same location, that location can receive two successive writes. A later decision not to write does not undo an earlier write.

The operation explicitly performs no allocation, deallocation, resource acquisition, logging, callback invocation, or external function call. It does not establish ownership transfer for the object, record, or caller-value storage. It uses caller-provided storage in place and does not explicitly retain any address after returning.

Object and record storage are not ordinarily written. They can nevertheless be changed through a caller-value address that aliases them.

No atomicity or synchronization is established. The behavior should not be treated as a transaction or as a coherent multi-location snapshot. Faults during later accesses have no explicit rollback path for an earlier successful write.

These are representation-level accesses. Exact machine-load counts, compiler transformations, and memory-mapped I/O behavior are not independently established.

## Dependencies

F-166b2cfe94fd contains no explicit calls to another function. There are therefore no opaque callee IDs to list and no callee ownership or global-preservation guarantees to assume.

Its operational dependencies are the supplied memory layout, valid address interpretation, signed comparison semantics, and readable or writable storage as required by the selected path.

## Acceptance examples

The following are conditional acceptance expectations, not executed tests. Unless stated otherwise, assume valid memory, stable values, distinct caller locations, and no overlap with object or limit storage.

| Lower limit | Upper limit | Initial lower value | Initial upper value | Expected final values | Expected writes |
|---:|---:|---:|---:|---|---|
| 10 | 20 | 3 | 25 | 10, 20 | Both locations |
| 10 | 20 | 12 | 18 | 12, 18 | Neither |
| 10 | 20 | 10 | 20 | 10, 20 | Neither |
| 10 | 20 | −1 | −1 | −1, −1 | Neither |
| −10 | −5 | −20 | 0 | −10, −5 | Both locations |
| −10 | −5 | −1 | −1 | −1, −1 | Neither |
| −1 | −1 | −2 | 0 | −1, −1 | Both locations |
| 20 | 10 | 0 | 30 | 20, 10 | Both locations |

Additional ordering-sensitive expectations:

- **Shared caller location:** with limits 10 and 20 and a shared initial value of 5, the lower-side write produces 10. The upper-side comparison then sees 10 and does not write.
- **Shared location with reversed limits:** with limits 20 and 10 and a shared initial value of 0, the location is first written to 20 and then to 10. The final value is 10.
- **Sentinel created by the first write:** with lower limit −1, upper limit −5, and a shared initial value of −2, the first operation writes −1. The second operation finds that −1 exceeds −5 but preserves it as the sentinel. The final value is −1.
- **Modified upper-limit storage:** if the lower destination legally aliases the upper-limit field, any lower-side write is visible when the upper limit is subsequently captured.
- **Sentinel does not bypass record access:** even when both caller values are −1, the record reference and both limit fields must remain valid for their accesses. No sentinel-based early exit avoids those reads.

## Uncertainties

**Inferred intent:** the paired offsets and opposite comparison directions suggest lower-bound and upper-bound clamping with −1 representing an unset or exempt value. The application meaning of the values is not established.

**documented_only — ABI:** actual integer width, pointer width, byte order, alignment, and integer-to-address interpretation are not independently confirmed. The 32-bit model is supported by the representation, not by separate target metadata.

**documented_only — memory model:** legality of unusual overlap, concurrent mutation, volatile storage, device memory, and exact generated access counts is unknown. No concurrency guarantee should be inferred from the repeated reads.

**documented_only — failures:** invalid-access behavior, operating-system fault handling, and any surrounding exception mechanism are not shown. No structured error return or recovery policy is established.

**documented_only — domain invariants:** the evidence supplies no guarantee that limits are ordered, that values are nonnegative, or that caller destinations are distinct.

## Evidence

The established behavior comes solely from the supplied static representation of F-166b2cfe94fd: its argument forms, fixed-offset accesses, signed comparisons, conditional writes, return paths, and absence of explicit calls.

The representation directly supports the lower-before-upper ordering, repeated record-reference lookup, conditional sentinel checks, and lack of explicit validation or rollback. Application intent and ABI details remain qualified separately above.

No implementation was accessed, no runtime tests were performed, and no acceptance example was executed.
