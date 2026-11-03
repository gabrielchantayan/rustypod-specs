<!-- Trusted provenance: F-c4fb8fbff3d6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4fb8fbff3d6

## Purpose

F-c4fb8fbff3d6 performs a comparison-driven search over positions in an externally supplied table. It returns a 32-bit success indicator and writes a signed 32-bit position through an output pointer.

The established success condition is a zero result from dependency F-4990728d74c1. A negative comparison moves the lower search bound upward; a positive comparison moves the upper search bound downward.

The control flow resembles binary search, but it does **not** establish a conventional complete search or insertion-point contract. In particular, the loop stops when its bounds are equal without comparing that remaining position. Consequently:

- An existing match can be missed.
- A failure position can refer to an entry that was never compared.
- A failure position is not established to be a correct insertion point.
- The meaning of the second argument—count, upper position, or another bound—is not resolved by the supplied evidence.

Consumers must use the behavior below rather than assuming standard library search semantics.

## Interface

The function accepts five arguments:

| Argument | Contract |
|---|---|
| Object address | A 32-bit address-like value. During each comparison iteration, the function reads a 32-bit table-base value at byte offset 12 from this address. |
| Initial upper bound | A signed 32-bit integer used directly as the initial upper search bound. It is not adjusted before the first loop condition. |
| First comparison operand | An opaque 32-bit value forwarded unchanged to F-4990728d74c1 on every comparison. |
| Second comparison operand | Another opaque 32-bit value forwarded unchanged to the same dependency. |
| Position output | An address at which the function writes one signed 32-bit position on normal completion. |

The return value is a 32-bit value:

- **1:** A comparison returned zero; the output is the position supplied to that comparison.
- **0:** No comparison returned zero before the search loop ended; the output is the final lower bound.

The return value is not a comparison result and does not encode negative or positive ordering.

### Referenced layout

Only the following object layout is established:

| Byte offset | Width | Meaning used here |
|---|---:|---|
| 12 | 4 bytes | Base address used to construct comparison-entry addresses |

The comparison-entry address is the loaded table base plus eight times the selected position. This establishes an **eight-byte addressing stride**, not the fields, validity, or ownership of an eight-byte record. This function does not itself read the contents of the selected entry.

No other object fields are explicitly read or written.

### Preconditions for defined, ordinary operation

- The position output must permit a four-byte write whenever the function completes normally.
- If at least one comparison iteration occurs, the object field at offset 12 must be readable.
- Every generated entry address must meet the requirements of F-4990728d74c1.
- The two opaque operands must meet that dependency’s requirements.
- Bound arithmetic and address construction must remain within the arithmetic and addressing behavior supported by the actual execution environment.

There is no explicit argument validation. A nonpositive upper bound avoids the object-field read but does not avoid the output write.

## Behavior

### Search state and initial condition

The lower bound begins at zero. The upper bound begins at the supplied signed 32-bit bound.

Comparison iterations occur only while the lower bound is strictly less than the current upper bound. Thus, an initial upper bound of zero or any negative value produces no comparisons.

For that no-comparison case, the function writes zero to the position output and returns zero.

### Position selection

On each iteration, the function selects a position by adding the current lower and upper bounds and shifting that signed result right by one bit.

For nonnegative bounds whose sum is representable as a signed 32-bit integer, this is the floor of their arithmetic mean. It is not an overflow-avoiding midpoint calculation.

The widths matter:

- Bounds and selected positions use signed 32-bit arithmetic.
- The bound sum can exceed the signed 32-bit range even when both individual bounds are nonnegative.
- Entry addressing multiplies the selected position by eight before combining it with the table base.
- A position fitting in 32 bits does not establish that its eight-byte displacement or final address is valid.

The evidence does not establish a portable result for overflowing arithmetic or a negative shifted sum. Acceptance conditions below exclude those cases.

### Observable comparison sequence

For each iteration, the function:

1. Selects the position from the current bounds.
2. Reads the table-base field at object offset 12.
3. Constructs the selected entry address using the eight-byte stride.
4. Calls F-4990728d74c1 with that address and the two original opaque operands.
5. Uses the sign of the returned signed comparison result to choose the next action.

The table base is read anew for each comparison. It is not established as a single snapshot taken at function entry.

A dependency that changes that field can therefore cause later iterations to address a different table base. Such mutation is not claimed to be intended or safe; the established point is that this function performs another field read before each later comparison.

### Comparison outcomes

**Zero result**

A zero comparison ends the search immediately. The function writes the selected position to the output and returns one.

It performs no further comparison and no further explicit object-field read.

**Negative result**

Any negative comparison result moves the lower bound to one greater than the selected position. The upper bound remains unchanged.

**Positive result**

Any positive comparison result moves the upper bound to one less than the selected position. The lower bound remains unchanged.

Only the sign matters. Results such as −1 and −7 have the same effect; results such as 1 and 7 have the same effect.

### Unsuccessful termination

After a nonzero comparison, the bounds are checked again. If the lower bound is no longer strictly below the upper bound, the search ends.

The function then writes the final lower bound and returns zero.

It does not compare a final singleton position merely because the bounds have become equal. It also does not clamp the output to an established table size.

For example, with an initial upper bound of one, a negative comparison at position zero produces failure with output one. No entry at position one is examined.

### Search completeness

These transitions do not provide a complete conventional binary search.

With an initial upper bound of two, the first selected position is one. A positive comparison changes the upper bound to zero. Since the lower bound is also zero, the loop ends without examining position zero.

Accordingly, even a stable, correctly ordered table and a consistent comparison dependency do not by themselves guarantee that an existing match will be found.

## Side effects

### Explicit memory operations

On normal completion, the function explicitly writes exactly one signed 32-bit position to the supplied output address.

Before that final write:

- Each comparison iteration explicitly reads the four-byte table-base field at object offset 12.
- Each iteration invokes F-4990728d74c1 once.
- The function does not explicitly write the object field or table entries.

The output write occurs after the final dependency call has returned. There is no explicit early initialization of the caller’s output storage.

### Aliasing

No non-aliasing requirement is established by the supplied code.

If the output overlaps the table-base field, an entry, or another caller-visible object, the final four-byte write changes that overlapping storage. This function performs no subsequent comparison after that write.

This observation does not establish that such aliasing satisfies the dependency’s contract or is an intended supported use.

### Ownership and lifetime

This function contains no explicit allocation, release, retention, or ownership-transfer operation. It forwards an entry address and two opaque values to the dependency.

The evidence does **not** establish that F-4990728d74c1:

- Preserves global state.
- Leaves the object or table unchanged.
- Avoids allocating or releasing memory.
- Retains no supplied address.
- Preserves the lifetime of any referenced storage.

Those properties must come from a separate dependency contract.

### Errors and abnormal completion

No distinct error return, pointer check, bound check, or recovery path is visible.

An invalid object-field access, invalid output write, invalid generated entry address, arithmetic problem, or abnormal dependency exit is not converted here into a documented failure result. Returning zero describes unsuccessful search completion, not a validated input-error condition.

If a dependency call does not return normally, this function provides no established final output write.

## Dependencies

### F-4990728d74c1

Each comparison call receives, in order:

1. The selected entry address.
2. The first original opaque 32-bit operand.
3. The second original opaque 32-bit operand.

Its result is consumed as a signed comparison value:

| Result | Effect in F-c4fb8fbff3d6 |
|---|---|
| Negative | Increase the lower bound past the selected position |
| Zero | Return success with the selected position |
| Positive | Decrease the upper bound below the selected position |

The comparison’s semantic direction is unknown. In particular, the evidence does not establish whether a negative result means “entry precedes key,” “key follows entry,” or another application-specific relation.

Any ordering requirement, operand interpretation, entry layout, memory-access extent, or side-effect restriction remains part of the dependency’s undocumented contract.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless specified otherwise, arithmetic and memory accesses are valid, dependency calls return normally, and the table base remains stable.

### Nonpositive initial bound

For an initial upper bound of zero or −3:

- No comparison occurs.
- No table-base field is read.
- The output becomes zero.
- The return value is zero.

The object address need not be dereferenced on this path. The output must still be writable.

### Immediate success

For an initial upper bound of six, the first selected position is three.

If that comparison returns zero:

- Exactly one comparison occurs, using the table base plus 24 bytes.
- The output becomes three.
- The return value is one.

### One-position bound, positive result

For an initial upper bound of one:

- Position zero is compared.
- A positive result changes the upper bound to −1.
- The output becomes zero.
- The return value is zero.

### One-position bound, negative result

For the same initial bound, a negative result at position zero raises the lower bound to one.

The function returns zero with output one. Position one is not compared.

### Existing match skipped

For an initial upper bound of two, suppose position zero would compare equal, but position one compares positive.

Only position one is compared. The function returns zero with output zero. The hypothetical equality at position zero does not affect this execution because that comparison never occurs.

### Multiple comparisons followed by success

For an initial upper bound of four:

- The first selected position is two.
- If its comparison is positive, the upper bound becomes one.
- The next selected position is zero.
- If that comparison is zero, the function returns one with output zero.

The compared entry displacements are 16 bytes, then zero bytes.

### Final singleton skipped after a negative result

For an initial upper bound of three:

- The first selected position is one.
- A negative result raises the lower bound to two.
- The next selected position is two.
- A positive result reduces the upper bound to one.

The function returns zero with output two. The output is the final lower bound, not a success indication.

### Changed table base between comparisons

For the two-comparison success path above, suppose the first dependency call changes the object’s table-base field to another valid base.

The second comparison uses the newly read base at position zero. This function does not require both calls to use the original base.

## Uncertainties

### Inferred intent

The bound narrowing and sign-based comparison strongly suggest an intended ordered-table search. The eight-byte stride suggests fixed-stride entries.

Neither inference establishes complete search behavior, entry contents, insertion semantics, or an ordering convention.

### documented_only

The following remain unresolved:

- Whether the initial upper bound represents a count or an inclusive endpoint.
- Whether skipped candidates are intentional or a defect.
- Exact entry layout and dependency read extent.
- Meanings of the two opaque operands.
- Dependency side effects, ownership rules, and lifetime guarantees.
- Required alignment, concurrency constraints, and memory visibility guarantees.
- Execution-environment behavior for overflowing signed arithmetic, overflowing displacement construction, or invalid addresses.
- Whether callers impose additional bounds that make those arithmetic cases unreachable.

No acceptance requirement should silently replace these unknowns with conventional binary-search assumptions.

## Evidence

**Established static evidence:** The supplied function body establishes the initial lower bound, strict loop condition, midpoint expression, repeated object-field read, eight-byte entry stride, dependency argument order, sign-based bound updates, final output writes, and zero/one return behavior.

**Static deductions:** The skipped-candidate behavior and conditional examples follow from those control-flow and arithmetic rules. They are not observations from an executing program.

**Inferred intent:** Ordered search and fixed-stride table interpretation are plausible purposes, not independently established dependency semantics.

**Verification limit:** No runtime tests have been performed. No external caller contract, dependency implementation, or runtime memory layout was available to resolve the documented unknowns.
