<!-- Trusted provenance: F-a3f49b3cb3da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3f49b3cb3da

## Purpose

F-a3f49b3cb3da derives a position-and-count pair from two signed coordinates and selected fields of a caller-provided object. It publishes the pair through two output destinations and returns no value.

Its established contract is numerical selection, not interval validation. It permits reversed coordinates, negative object fields, and negative counts. It does not establish that the selected position identifies accessible storage.

The field names **anchor**, **limit**, and **terminal** below are descriptive labels, not established application semantics. An interpretation involving portions of a stored range is plausible but remains inferred intent.

This specification separates statically established behavior from target-dependent details and application-level unknowns. No runtime tests have been performed.

## Interface

### Arguments and results

The five arguments, in order, are:

| Argument | Meaning |
|---|---|
| Object reference | Base address representation used to locate selected integer fields. |
| First coordinate | Signed integer governing the principal selection case. |
| Second coordinate | Signed integer used for eligibility and, in one case, trailing selection. |
| Position destination | Address receiving one signed integer. |
| Count destination | Address receiving one signed integer. |

There is no direct return value or separate success indicator. Every ordinarily completing invocation writes both outputs.

The supplied representation carries the object reference in an integer-valued argument, rather than establishing a typed object-pointer interface. Its address interpretation is consequently a target-dependent prerequisite.

### Integer widths and object layout

The coordinates, accessed fields, arithmetic results, and output elements share the representation’s signed integer type. The supplied material does not independently establish that type’s bit width, its storage size, the address width, or the calling convention.

The object contains the following accessed locations:

| Byte displacement from the object base | Contract label | Role |
|---|---|---|
| 704 | Limit | Bounds a forward selection or supplies a count directly. |
| 716 | Anchor | Separates the selection cases and can supply the position. |
| 724 | Terminal | Determines a trailing count. |

Each access covers one interface integer. If the target uses four-byte integers, these locations occupy four bytes each, and the last field extends through byte displacement 727. Neither that integer size nor a complete object allocation layout is independently established.

No other object fields are explicitly accessed.

### Preconditions

Ordinary completion requires:

- Output storage suitable for one integer store at each destination.
- Readable object storage for every field reached by the selected case.
- Valid address formation for those reached fields, including the fixed byte displacements.
- Alignment and aliasing compatible with the target’s memory rules.
- Arithmetic semantics sufficient to evaluate the selected calculations.

The function does not validate these conditions. Where signed overflow or invalid memory has no defined behavior in the target environment, this contract supplies no replacement result.

An object field that is not reached need not be readable for this function’s explicit access pattern. Output storage remains necessary even when no object fields are read.

## Behavior

### Selection relation

The following table defines the result for stable object fields and representable arithmetic. The comparison terms “below,” “equal,” and “above” are signed comparisons.

| Input condition | Selected position | Selected count |
|---|---|---|
| Either coordinate is negative | −1 | 0 |
| Both coordinates are nonnegative; first coordinate equals anchor | −1 | 0 |
| Both coordinates are nonnegative; first coordinate is above anchor | Anchor | Lesser of limit and the distance from anchor to first coordinate |
| Both coordinates are nonnegative; first coordinate is below anchor; second coordinate is below anchor | Anchor | Limit |
| Both coordinates are nonnegative; first coordinate is below anchor; second coordinate equals or exceeds anchor | Successor of second coordinate | Terminal minus second coordinate, subject to the intermediate-arithmetic qualification below |

No relationship between the two coordinates is required beyond the conditions in the selected row. In particular, the first coordinate may exceed the second.

The default pair, **(−1, 0)**, is a normal result for rejected coordinates and anchor equality. It is not a general error result.

### Eligibility and field access

Coordinate eligibility is evaluated before any object access. The first coordinate’s nonnegative condition precedes the second coordinate’s condition; a negative first coordinate short-circuits that eligibility evaluation.

For eligible inputs, the anchor is obtained once. Subsequent case selection and any position derived from the anchor use that captured value. There is no later anchor refresh.

The explicit object-access requirements are:

| Selected condition | Ordered object reads |
|---|---|
| Either coordinate negative | None |
| First coordinate equals anchor | Anchor |
| First coordinate above anchor; first limit value does not fall below the forward distance | Anchor, limit |
| First coordinate above anchor; first limit value falls below the forward distance | Anchor, limit, limit again |
| First and second coordinates both below anchor | Anchor, limit |
| First coordinate below anchor; second coordinate at or above anchor | Anchor, terminal |

These are accesses established by the supplied representation, not guaranteed processor instruction counts or hardware transactions.

Once the second coordinate has passed eligibility, it has no further effect when the first coordinate equals or exceeds the anchor.

### Forward-count qualification

For the forward-selection case, the mathematical “lesser of” description assumes a stable limit.

The first limit read determines whether the limit is below the forward distance. When it is not below that distance, the selected count is the distance and no second limit read occurs. Equality therefore does not cause a reread.

When the first limit value is below the distance, a second limit read supplies the selected count. The first value is not reused as the result. If the two reads return different values under an execution model permitting that observation, the selected count need not be the lesser of the distance and either a stable or final limit.

No sign check is applied to the limit. A negative limit can become a negative count.

### Trailing-count qualification

In the trailing-selection case, the selected position is one greater than the second coordinate.

Although the mathematical count simplifies to terminal minus second coordinate, the established calculation involves the successor of the second coordinate, subtraction of that successor from the terminal, and a subsequent increment. Every required intermediate must be supported by the target’s arithmetic semantics.

A representable simplified result does not establish that those intermediate calculations are representable. This distinction matters at signed boundaries.

The terminal is not checked against either coordinate or the selected position. Zero and negative trailing counts are allowed outcomes when the arithmetic is otherwise valid.

### Output publication

The selected position is stored first. The selected count is stored second.

All explicit object reads needed for selection precede both output stores. There are no subsequent object reads.

Consequences, subject to valid target aliasing rules:

- If the destinations identify the same integer location, its final value is the selected count.
- If an output overlaps an accessed object field, the store can change that field, but cannot change the already selected result for this invocation.
- Other observers may distinguish the two stores; publication is not established to be atomic or transactional.
- A failure during the second store has no established rollback behavior for the first store.

Partial overlaps and incompatible storage interpretations have no portable behavior established here.

## Side effects

The only explicit writes are the position store and the count store, in that order.

For valid, non-overlapping output storage, the object is read-only. Output aliasing can instead make either store modify object storage.

The supplied body establishes no:

- Allocation, release, or ownership transfer.
- Resource acquisition or cleanup.
- External routine or callback invocation.
- Logging or explicit error reporting.
- Direct global-state access.
- Synchronization or atomic publication mechanism.

The caller supplies the object and output storage. No ownership policy is established beyond the absence of visible acquisition or release by this function.

Invalid addresses, unsupported arithmetic, or incompatible aliasing do not select a defined error pair. Their consequences depend on the target environment and remain unresolved.

## Dependencies

No dependency calls are present. There are no additional opaque function IDs to enumerate.

Execution nevertheless depends on external interface conditions:

- The calling convention and integer/address representations.
- The validity and alignment of reached storage.
- The target’s signed-arithmetic rules.
- Any rules permitting or prohibiting overlapping destinations.
- Any synchronization governing changes to object fields.

No initializer, validator, allocator, or cleanup dependency may be assumed. There is no unknown callee whose global preservation or memory-ownership behavior needs to be inferred.

## Acceptance examples

These are conditional predictions from static evidence, not observed test results. Unless an example says otherwise, assume stable fields, valid non-overlapping storage, valid address formation, and representable arithmetic. Results are shown as **(position, count)**.

### Coordinate eligibility

| First coordinate | Second coordinate | Expected result | Object access |
|---|---|---|---|
| −1 | 20 | **(−1, 0)** | None |
| 20 | −1 | **(−1, 0)** | None |
| −5 | −7 | **(−1, 0)** | None |

An object reference that cannot support field reads is not dereferenced on these paths. Both output destinations must still support their stores.

### Forward selection

For anchor 10, first coordinate 16, and any nonnegative second coordinate:

| Limit | Expected result | Limit reads |
|---|---|---|
| 20 | **(10, 6)** | One |
| 6 | **(10, 6)** | One |
| 4 | **(10, 4)** | Two |
| −3 | **(10, −3)** | Two |

The terminal is irrelevant and is not accessed.

If the first limit read returns 4 and the required second read returns 9, the expected result is **(10, 9)**. This example is conditional on an execution model allowing those observations; it does not establish that unsynchronized concurrent mutation is supported.

### Anchor equality

For anchor 10 and first coordinate 10, every nonnegative second coordinate produces **(−1, 0)**.

Only the anchor is read. Limit and terminal storage need not be accessed on this path.

### Selection below the anchor

For anchor 10 and limit 5:

- Coordinates 2 and 8 produce **(10, 5)**.
- Coordinates 9 and 1 also produce **(10, 5)**.

The second example demonstrates that reversed coordinate ordering is not rejected. The terminal is not accessed.

### Trailing selection

For anchor 10 and first coordinate 2:

| Second coordinate | Terminal | Expected result |
|---|---|---|
| 10 | 14 | **(11, 4)** |
| 14 | 14 | **(15, 0)** |
| 15 | 14 | **(16, −1)** |

These cases do not access the limit.

### Aliasing and signed boundaries

If both output destinations identify the same valid integer location, a selection of **(10, 4)** leaves 4 in that location after ordinary completion.

For anchor −1, first coordinate 0, any nonnegative second coordinate, and limit 5, the expected result is **(−1, 1)**. A position of −1 therefore does not by itself identify the default result.

Under a signed 32-bit interpretation:

- A second coordinate of 2,147,483,647 cannot have a representable signed successor. A trailing-selection case using that value has no portable expected output under ordinary signed arithmetic.
- Anchor −2,147,483,648 and first coordinate 0 require an unrepresentable forward distance.
- Anchor 1, first coordinate 0, second coordinate 1, and terminal −2,147,483,647 yield a mathematically simplified count of −2,147,483,648, but the established intermediate subtraction requires −2,147,483,649. The simplified result does not justify accepting this case as representable.

These boundary cases require target-specific evidence before assigning concrete outputs.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, integer storage size, address width, and calling convention.
- Signed representation and overflow behavior.
- Integer-to-address interpretation and the semantics of address-displacement arithmetic.
- Required alignment and permitted aliasing arrangements.
- Whether the fields describe a buffer, interval, capacity, window, or another abstraction.
- Caller guarantees concerning coordinate ordering and field consistency.
- Whether object fields can change during execution and what memory model governs such changes.
- Whether −1 has an application-defined sentinel meaning.
- Fault handling and externally visible state after an invalid read or failed store.
- Machine-level access widths, compiler transformations, and hardware atomicity.

These unknowns do not justify adding clamping, normalization, bounds validation, retry behavior, or recovery semantics.

## Evidence

The supplied static representation establishes the argument roles, signed comparisons, fixed field displacements, selection conditions, arithmetic dependencies, conditional repeated limit read, absence of dependency calls, and ordered output stores.

The behavioral tables express that evidence as input/output obligations and observable access requirements rather than an implementation procedure.

The descriptive field labels and possible range-selection purpose are inferred intent only. Exact ABI properties, application invariants, and target-specific failure behavior are not established.

No executable observation, caller contract, ABI specification, or dependency implementation was supplied. No runtime tests have been performed.
