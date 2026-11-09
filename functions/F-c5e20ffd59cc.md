<!-- Trusted provenance: F-c5e20ffd59cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c5e20ffd59cc

## Purpose

F-c5e20ffd59cc performs a stateful lookup using a caller-supplied 32-bit position and a table of inclusive ranges. It returns a 32-bit result and, on a successful return path, writes the state’s current position back to the caller.

Two paths are established:

- An active-state path invokes F-008865c7563b when the caller’s position equals the state’s current position.
- A table-search path searches for the caller’s position plus one, records a selected table index and position in the state, and either computes a result directly or invokes F-008865c7563b.

A zero return is used for several distinct outcomes. It does not, by itself, identify exhaustion, an inactive state, an empty table, or a dependency-produced zero result.

The apparent intent is sequential traversal through a range-backed mapping. That interpretation is inferred; the contract below describes the visible behavior without assuming what the positions or returned values represent.

## Interface

### Inputs

The function receives two address-like arguments:

1. **State address:** references a writable state object.
2. **Position address:** references a readable and, on successful paths, writable 32-bit position.

No null checks, allocation-size checks, or pointer-validity checks are visible.

### State layout

Offsets are byte offsets from the state address.

| Offset | Width | Observed role |
|---|---:|---|
| `0x10` | 32 bits in the supplied representation | Address of the range-table object |
| `0x18` | 8 bits | Active flag; zero means inactive, any nonzero byte is treated as active |
| `0x1c` | 32 bits | Current position; the all-ones value is an initial sentinel |
| `0x20` | 32 bits | Result supplied by or retained for the dependency |
| `0x24` | 32 bits | Selected table-record index |

The function explicitly writes the active flag as exactly one on the table-search selection path. It does not normalize a nonzero active flag on the active-state path.

State fields and the caller’s position use ordinary target-native 32-bit accesses. Their byte order is not independently established by the excerpt.

### Table layout

The table contains a header followed by fixed-size records:

| Table-relative offset | Width | Meaning |
|---|---:|---|
| `0x0c` | 4 bytes | Unsigned record count, encoded big-endian |
| `0x10 + 12 × index` | 4 bytes | Unsigned inclusive range start, encoded big-endian |
| `0x14 + 12 × index` | 4 bytes | Unsigned inclusive range end, encoded big-endian |
| `0x18 + 12 × index` | 4 bytes | Unsigned mapping base, encoded big-endian |

Only these header and record fields are read directly. No table-format tag or record-validity check is visible.

### Output

The return value is a 32-bit quantity represented as a signed integer in the supplied source. A nonzero bit pattern is accepted as a result, including patterns that represent negative signed values.

The caller’s position is written only on the explicitly successful paths described below. The function does not initialize it on failure.

### Preconditions and ownership

For ordinary, non-aliasing use:

- The state must remain accessible for all fields reached by the selected path.
- The position address must support the reads and writes reached by that path.
- The table must contain accessible header and record bytes for every index examined.
- F-008865c7563b must be callable with the state address and must satisfy any additional requirements of its own.

No allocation, deallocation, ownership transfer, or reference-count operation is directly visible. The dependency’s ownership behavior is unknown.

Aliasing between the position storage, state fields, and table storage is not checked. Such aliasing can make the ordered writes observable in ways that differ from the ordinary distinct-storage interpretation.

## Behavior

### 1. Initial sentinel check

The first relevant read is the state’s current-position field.

If its 32-bit value is all ones, interpreted as signed minus one, the function returns zero immediately.

This path does not read the active flag, dereference the caller’s position, read the table address, or call the dependency. It performs no direct writes.

The sentinel test applies only at entry. A later dependency change to the current position is not followed by another sentinel test.

### 2. Active-state continuation

If the initial sentinel is absent, the function reads the active flag.

If the flag is nonzero, it compares the state’s current position with the caller’s position. Equality is a comparison of the same 32-bit values; signed interpretation does not change the equality result.

When they match:

1. F-008865c7563b is called with the state address.
2. The active flag is read again.
3. If the flag is now zero, the function returns zero without directly writing the caller’s position.
4. Otherwise, the result field is read and retained.
5. If that result is nonzero, the current-position field is read and copied to the caller’s position.
6. The retained result is returned, including zero.

There is no table search on this path. The dependency can change the position ultimately copied to the caller.

A zero result with an active flag still set does not cause a caller-position write. Dependency changes to other memory are not undone.

If the flag is initially zero, the equality comparison and its caller-position read are skipped. If the flag is nonzero but the positions differ, execution proceeds to table search without first calling the dependency.

### 3. Table setup

The table address is read from the state. The record count is assembled from four bytes in big-endian order.

If the count is zero, the function returns zero. It has made no direct state or caller-position writes. On an initially inactive path, this exit can occur without any read of the caller’s position.

For a nonempty table, the search target is formed from the caller’s position plus one and held as an unsigned 32-bit value. The search comparisons use unsigned ordering.

There is an arithmetic qualification: the supplied representation expresses the addition using a signed position operand before assigning to unsigned storage. The behavior at signed maximum therefore cannot be settled solely by treating the excerpt as portable signed-language arithmetic. A modulo-32-bit machine interpretation is plausible, but its status is documented under uncertainties.

### 4. Range search

The search maintains a lower index of zero and an exclusive upper index equal to the record count.

While the lower index is below the upper index, it selects a midpoint using an unsigned 32-bit sum followed by a one-bit right shift. Consequently, this midpoint calculation is not a guaranteed overflow-free average for arbitrary counts.

For each examined record:

- Its start is read and decoded.
- If the target is below that start, the upper bound becomes the midpoint.
- Its end is read and decoded even when the target is below the start.
- If the target is below the start, the search continues without reading that record’s mapping base.
- Otherwise, if the target is at most the end, the record is accepted as containing the target.
- Otherwise, the lower bound becomes one past the midpoint.

For a containing record, the candidate result is:

**mapping base + target − range start, reduced to 32 bits.**

This arithmetic is unsigned in the supplied representation. Overflow or underflow in the mapped value therefore wraps modulo \(2^{32}\).

Search correctness as a range lookup requires suitable table ordering and range consistency. Neither is checked. Overlapping, reversed, or unordered ranges follow the same comparisons but have no separately established semantic guarantee.

### 5. Selection after search exhaustion

If no containing record is found before the bounds meet, selection uses the most recently examined index and end value.

- If the last examined end is below the target, the selected index is incremented.
- If that increment makes the index equal to the record count, the function returns zero immediately.
- Otherwise, the selected index is retained for state publication and dependency processing.

For a conventional ordered, non-overlapping table, this can select a later record when the target lies in a gap, or the first record when the target precedes all ranges.

The function does **not** directly replace the target with the selected record’s start. It publishes the original incremented target. Any normalization to another position would have to come from F-008865c7563b.

### 6. Publishing the selection

For either a containing record or an exhaustion-based selection that did not return zero, direct state writes occur in this order:

1. Active flag becomes one.
2. Selected record index is written.
3. Current position becomes the search target.

These writes occur before any dependency call on this path.

The next action depends on the candidate result:

- **Nonzero computed candidate:** the candidate is written to the result field. The dependency is not called.
- **Zero candidate, or no containing record:** F-008865c7563b is called.

A containing range whose arithmetic result is zero therefore does not return directly through the computed-result path. Zero triggers the same dependency path used for a gap selection.

After this dependency call:

1. The active flag is read.
2. If zero, the function returns zero.
3. Otherwise, the result field is read.
4. If zero, the function returns zero.
5. Otherwise, that result is retained for return.

On either successful branch, the function then reads the state’s current position, writes it to the caller’s position, and returns the retained nonzero result.

The final caller-position value is therefore the state position at that read, not necessarily the original search target.

## Side effects

### Direct writes

The function can directly modify:

- The active flag.
- The selected index.
- The current position.
- The result field, but only for a nonzero directly computed candidate.
- The caller’s position, only on a successful path.

No direct table writes are present.

### Failure is not transactional

Several zero-return paths occur after state publication or a dependency call. A zero return does not imply that the state is unchanged.

In particular, dependency processing after a search selection occurs after the active flag, index, and current position have already been written. These writes are not rolled back if the dependency clears the active flag or leaves a zero result.

“Caller position is not directly written” does not imply that its storage is unchanged: the dependency may modify it indirectly, or it may alias a state field written earlier.

### Read and write ordering limits

The sequencing described above is supported by the visible control flow. Individual byte reads inside a single big-endian decoding expression are not assigned an additional guaranteed order here. Concurrent mutation, volatile-access semantics, memory barriers, and synchronization guarantees are not established.

## Dependencies

### F-008865c7563b

**Input:** the state address.

**Return value:** not used by F-c5e20ffd59cc.

**Call sites:**

- Active flag nonzero and state position equal to caller position.
- A table selection with no nonzero directly computed candidate.

**Observed post-call decisions:**

- Read the active flag.
- If active, read the result field.
- For a nonzero result, later read the current position for caller-position publication.

The dependency’s internal behavior is unavailable. It must not be assumed to preserve the state, table storage, unrelated globals, caller-visible memory, or ownership relationships. Its apparent role in advancing or resolving a selection is inferred from these call sites, not established independently.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, storage is distinct, tables are accessible and conventionally ordered, and no arithmetic-addressing overflow occurs.

1. **Entry sentinel bypasses all later work.**  
   If the state’s current position is all ones, the result is zero. The caller-position storage and table need not be accessed on this path. No dependency call occurs.

2. **Active continuation succeeds with dependency-updated state.**  
   Suppose the active flag is nonzero and both positions are 40. If F-008865c7563b leaves the state active, sets its position to 43, and sets its result to 900, the function writes 43 to the caller’s position and returns 900.

3. **Active continuation becomes inactive.**  
   Under the same initial matching conditions, if the dependency clears the active flag, the function returns zero without a direct caller-position write. Any dependency changes remain observable.

4. **Empty table exits before incrementing the position.**  
   If the state is initially inactive and the table count is zero, the function returns zero without reading the caller’s position or calling the dependency.

5. **Direct inclusive-range mapping.**  
   Let the table contain one record with start 10, end 12, and mapping base 100. With a caller position of 10 and no active-state match, the target is 11. The function sets the active flag to one, index to zero, current position to 11, and result to 101. It writes 11 to the caller’s position and returns 101 without calling the dependency.

6. **Inclusive upper boundary.**  
   With the same record and a caller position of 11, target 12 is contained. The directly computed result is 102.

7. **Past the final range.**  
   With the same single record and a caller position of 12, target 13 lies beyond the end. Selection advances to the count and returns zero before publishing search state. No dependency call occurs.

8. **Gap selection preserves the incremented target before the call.**  
   Suppose records cover 10–12 and 20–22, and the caller position is 14. The target is 15. The search selects the second record, publishes index one and position 15, then calls the dependency. A return value or final caller position cannot be specified without the dependency’s behavior.

9. **A zero mapped value invokes the dependency.**  
   A containing record with start 10 and mapping base zero produces a zero candidate for target 10. The function publishes the selection and calls the dependency instead of directly returning the mapped zero.

10. **Nonzero wrapped mapping remains a result.**  
    For a range starting at 10, a mapping base of `0xffffffff`, and target 12, unsigned mapping arithmetic produces one. Subject to valid table bounds, the direct path returns one and does not invoke the dependency.

## Uncertainties

- **documented_only — Dependency semantics:** its position advancement, result generation, memory effects, errors, and ownership behavior are unavailable.
- **documented_only — Signed increment boundary:** the supplied representation does not establish portable behavior for adding one to a signed position of `0x7fffffff`. Exact machine arithmetic requires additional evidence.
- **documented_only — Address arithmetic:** pointer width, address wrapping, alignment requirements, and behavior on invalid addresses are not fully established. The supplied representation uses 32-bit address-like fields.
- **documented_only — Table invariants:** ordering, non-overlap, maximum record count, and start/end validity are not checked here.
- **documented_only — Very large counts:** midpoint addition and record-offset arithmetic may overflow. No safe maximum is established by validation in this function.
- **documented_only — Aliasing and concurrency:** no non-aliasing guarantee or synchronization contract is supplied.
- **Inferred intent:** the active flag, index, current position, and result appear to form an iteration or lookup cache. Their domain-specific meaning is unknown.

## Evidence

The supplied static function body supports the field offsets, widths, big-endian decoding, unsigned comparisons, branch conditions, dependency call sites, and direct write ordering described above.

The dependency’s body, callers, allocation contracts, and table producer are not supplied. No runtime tests have been performed. The acceptance examples are deductions from the visible control flow under their stated conditions, not observations of execution.
