<!-- Trusted provenance: F-ebbb7a35dd31; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebbb7a35dd31

## Purpose

F-ebbb7a35dd31 performs an insertion-shaped update on a descriptor for contiguous, 16-byte units. It accepts a destination position, an unsigned insertion count, and an opaque 32-bit value supplied to the operations that populate the inserted region.

Two storage paths are established:

- If the computed capacity is insufficient, allocate replacement storage, process the prefix, populate the insertion region, process the suffix, clean up the previous range, release the previous allocation, and publish replacement descriptor fields.
- Otherwise, extend the descriptor’s logical end first and rearrange or populate the existing storage.

A zero insertion count bypasses all descriptor accesses and dependency calls.

“Container,” “insertion,” “capacity,” and “element” describe the apparent intent. The supplied evidence does not establish the element type, whether helper operations copy or move objects, or the ownership represented by the opaque value.

## Interface

The function has four arguments and no returned value:

| Argument | Established representation | Role |
|---|---|---|
| Descriptor | Address of at least three consecutive 32-bit words | Holds the storage start, logical end, and allocation limit used by this function |
| Position | 32-bit address-like quantity | Boundary at which the insertion-shaped operation occurs |
| Count | Unsigned 32-bit integer | Number of 16-byte units requested |
| Value | Unsigned 32-bit quantity | Forwarded unchanged to population dependencies |

The descriptor’s first three words have the following observable uses:

| Byte offset | Meaning used by this function |
|---|---|
| 0 | Storage start |
| 4 | Logical end |
| 8 | Allocation limit |

Only these three words are directly read or written by this function. Several dependencies receive the descriptor address and may access additional state; their effects are not established here.

All address and size calculations use the supplied program’s 32-bit model. Element stride is exactly 16 bytes. This is a storage stride, not evidence that an element consists of four independent integers.

For meaningful insertion behavior, the apparent preconditions are:

- The descriptor and affected storage are valid for the accesses and helper operations.
- The position denotes an appropriate boundary within the original logical range.
- Differences between relevant boundaries are multiples of 16.
- Size and address arithmetic does not overflow.
- Dependencies meet the range, lifetime, and allocation requirements expected by their callers.

These are inferred operational preconditions, not validations performed by this function. There is no explicit rejection of an out-of-range position, misalignment, excessive count, or invalid descriptor.

For a zero count, the body does not dereference the descriptor. No validity requirement for the descriptor can therefore be inferred from this body alone on that path.

## Behavior

### Zero-count path

If Count is zero, the function returns immediately.

It does not read descriptor fields, allocate storage, call a dependency, or write storage. Position and Value are unused on this path.

### Capacity decision

For a nonzero Count, the function first calls F-f40dec111e7a with the descriptor. Its result is used as a logical-size-like quantity.

It then obtains a capacity-like quantity from the difference between the allocation limit and storage start, shifted right by four bits, and compares that quantity as unsigned against the helper result plus Count.

The addition is a 32-bit unsigned computation. Overflow is not checked. Consequently, a mathematically excessive requested size can wrap before the comparison and select a path that would not be selected by unbounded arithmetic.

The storage difference is represented as signed 32-bit subtraction followed by a signed right shift in the supplied representation, then converted to unsigned for this comparison. For valid, ordered boundaries whose byte difference fits the intended signed range, this behaves as a count of 16-byte units. Behavior for larger, reversed, or otherwise invalid differences must not be normalized into an ordinary nonnegative capacity without additional evidence.

The size helper is called again in the replacement-storage path. Its first result is not reused there.

### Replacement-storage path

This path is selected when the computed capacity is less than the computed requested size.

#### Selecting the allocation size

The function makes two further, separate calls to F-f40dec111e7a:

1. The first result, plus Count, becomes the requested total.
2. The second result becomes the basis for a growth candidate.

For the second result, treated as unsigned, the growth candidate is the larger of:

- That result plus 32 units.
- That result plus its unsigned right-shift-by-one value plus its unsigned right-shift-by-three value.

The final allocation count is the larger of the growth candidate and the requested total.

All additions use 32-bit arithmetic. The selected allocation count is shifted left by four bits to produce the byte-size argument to F-871e64ddc6de. Neither element-count overflow nor loss of high bits in the byte-size conversion is checked.

The two size-helper results need not be assumed equal. Their equality requires a dependency contract or an additional stability assumption.

#### Allocating and processing ranges

F-871e64ddc6de receives the computed byte size and a second argument of zero. Its return value becomes the replacement-storage base.

There is no explicit check of that return value before subsequent dependency calls.

The following operations occur in order:

1. F-e0c21befec47 receives the current storage start, Position, the replacement base, and the descriptor.
2. F-717822f2f02f receives:
   - A destination computed by adding the replacement base to the low-four-bits-cleared difference between Position and the storage start read after the preceding call.
   - Count.
   - Value.
   - The descriptor.
3. F-e0c21befec47 receives:
   - Position.
   - The current logical end.
   - A destination at the replacement base plus 16 times the sum of Count and the signed-shifted position offset from the current storage start.
   - The descriptor.

Under stable descriptor fields and aligned position boundaries, these calls correspond to processing the original prefix into replacement storage, populating the inserted region, and processing the original suffix after that region.

The source expression for the insertion destination clears the lowest four bits of the position difference. It does not independently validate alignment. For an unaligned position, this calculation must not be described as preserving the exact byte offset.

The descriptor is passed to each of these dependencies. Storage-start and logical-end values used after a call are not established to remain equal to their earlier values.

#### Cleaning up and publishing

After the three range-processing calls:

1. F-f40dec111e7a is called again, and its result is saved for the eventual logical-end calculation.
2. F-660635c2f259 receives the descriptor and its then-current storage-start and logical-end fields.
3. F-ff1bdc376dcd receives:
   - The storage start read after that cleanup call.
   - The allocation-limit-minus-storage-start difference shifted right by four.
   - Zero.

Only after these calls does the function directly publish the replacement descriptor:

1. Storage start becomes the replacement base.
2. Logical end becomes the replacement base plus 16 times the saved size-helper result, plus 16 times Count.
3. Allocation limit becomes the replacement base plus 16 times the selected allocation count.

These are ordered field writes, not an atomic descriptor replacement.

The saved size used for the new logical end is obtained after range processing but before cleanup and release. It is not necessarily the initial size-helper result.

### Existing-storage path

This path is selected when the computed capacity is not less than the computed requested size.

The function reads the current logical end as the original end for this path. It computes Position plus 16 times Count, and then immediately writes the descriptor’s logical end as the original end plus 16 times Count.

This logical-end extension occurs before any subsequent range-processing or per-address dependency call. Dependencies receiving the descriptor therefore can observe the extended end.

The next branch uses an unsigned comparison between the computed post-insertion boundary and the original end.

#### Post-insertion boundary below the original end

When Position plus 16 times Count is strictly below the original end:

1. A boundary is computed at the original end minus 16 times Count.
2. F-e0c21befec47 receives that boundary, the original end, the original end as destination, and the descriptor.
3. F-577be948cd9a is called repeatedly on descending addresses.

In that descending sequence, the first address is the original end minus 16. Each subsequent address is 16 bytes lower. For valid aligned ranges, the final address is Position plus 16 times Count.

The number of such calls is the number of original units after Position minus Count. This sequence is not a call on each newly appended slot: its addresses are below the original end.

The loop terminates by exact equality between its descending boundary and Position. Invalid alignment or an invalid position can therefore prevent the intended termination; the function does not guard against that condition.

Finally, F-da66c997efc2 receives Position, Position plus 16 times Count, and Value.

#### Post-insertion boundary at or above the original end

Otherwise:

1. F-717822f2f02f receives:
   - The original end as destination.
   - Count minus the signed-shifted difference between the original end and Position.
   - Value.
   - The descriptor.
2. F-e0c21befec47 receives Position, the original end, Position plus 16 times Count as destination, and the descriptor.
3. F-da66c997efc2 receives Position, the original end, and Value.

For a valid position and aligned boundaries, the population count in the first call is Count minus the number of original units after Position.

Equality follows this branch. When Count exactly equals the original suffix length, F-717822f2f02f is still called with a zero count. At an end position with a positive Count, F-e0c21befec47 and F-da66c997efc2 are still called with empty original ranges. This function does not suppress those calls.

## Side effects

Directly established effects are:

- Replacement storage may be requested.
- Dependencies may read or modify element storage.
- Dependencies may inspect or modify the descriptor.
- The existing-storage path writes the logical end before performing element operations.
- The replacement-storage path writes all three descriptor fields after cleanup and release calls.
- Value is forwarded unchanged; this function does not directly dereference it.

The function does not directly release storage through a language-level primitive. It delegates the apparent old-storage release to F-ff1bdc376dcd.

Ownership of the allocation returned by F-871e64ddc6de, and the precise lifetime effects of the cleanup dependencies, remain dependency-contract questions. Successful execution appears intended to replace the descriptor’s old allocation with the new allocation, but this body alone does not establish allocator compatibility or ownership transfer rules.

There is no explicit error result, rollback, allocation-failure branch, or recovery handler. If a dependency fails to return normally, later operations are not reached. In particular, an abnormal exit during the existing-storage path can occur after the logical end has already been extended. No strong failure guarantee is established.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-f40dec111e7a | Takes the descriptor; returns a value used in size, growth, and final-end calculations | Purity, stability, exact size semantics |
| F-871e64ddc6de | Takes a 32-bit byte-size calculation and zero; returns a replacement base | Failure convention, alignment guarantee, ownership rules |
| F-e0c21befec47 | Takes source boundaries, a destination, and the descriptor | Copy versus move, construction versus assignment, overlap safety, descriptor mutations |
| F-717822f2f02f | Takes destination, count, Value, and descriptor | Population semantics, Value interpretation, zero-count behavior |
| F-660635c2f259 | Takes descriptor and range boundaries before apparent release | Destruction semantics, descriptor changes, ownership effects |
| F-ff1bdc376dcd | Takes storage base, a count-like size, and zero | Exact release semantics and size requirements |
| F-577be948cd9a | Takes one address per descending iteration | Whether it destroys, resets, or otherwise modifies an element |
| F-da66c997efc2 | Takes destination boundaries and Value | Assignment semantics, empty-range behavior, overlap and aliasing rules |

No dependency is assumed to preserve globals, the descriptor, or other reachable state unless that property is supplied separately.

## Acceptance examples

These examples are conditional: descriptor boundaries are valid and aligned, arithmetic does not overflow, the size helper returns the original logical size consistently, and dependencies preserve descriptor fields unless explicitly changed by this function. They describe required call structure, not runtime-tested outcomes.

1. **Zero count**

   For Count zero, no dependency calls or descriptor writes occur, regardless of Position or Value.

2. **Growth from ten units to fifteen**

   Suppose logical size and capacity are both ten, Position is three units after the storage start, and Count is five.

   The requested total is fifteen. The growth candidates are forty-two and sixteen units, so the allocation request is for forty-two units, or 672 bytes.

   The range-processing calls target the replacement prefix, then five units beginning at replacement offset 48 bytes, then the original suffix beginning at replacement offset 128 bytes. After cleanup and release return normally, the descriptor identifies fifteen logical units and forty-two allocated units.

3. **Existing storage with a suffix longer than Count**

   Suppose size is ten, capacity is twenty, Position is two units from the start, and Count is three.

   The logical end is first extended to thirteen units. F-e0c21befec47 processes the original range from unit seven to unit ten into destination unit ten. F-577be948cd9a is then called at original unit indices nine, eight, seven, six, and five, in that order. F-da66c997efc2 receives the range from unit two to unit five.

4. **Existing storage with a shorter suffix**

   Suppose size is ten, capacity is twenty, Position is eight units from the start, and Count is four.

   The logical end is first extended to fourteen units. F-717822f2f02f receives original end and count two. F-e0c21befec47 receives the original range from unit eight to unit ten with destination unit twelve. F-da66c997efc2 receives the original range from unit eight to unit ten.

5. **Exact suffix-length boundary**

   Suppose Position is six units from the start of a ten-unit range and Count is four, with sufficient capacity.

   The at-or-above branch is selected. F-717822f2f02f is called with count zero; the call is not omitted.

6. **Insertion at the logical end**

   With sufficient capacity and Position equal to the original end, the population dependency receives the full Count. The subsequent range-processing and final population calls receive empty original ranges.

## Uncertainties

The following are **documented_only**:

- The actual element type and interpretation of Value.
- Whether dependencies implement copying, moving, construction, assignment, destruction, or another operation.
- Allocation failure behavior and ownership compatibility.
- Dependency effects on descriptor fields, globals, aliases, or reachable storage.
- Whether callers enforce valid positions, alignment, nonnegative ordered differences, and overflow-free arithmetic.
- Whether Value may alias storage affected or released during the operation.
- Behavior under concurrent access or reentrant dependency calls.
- Exact execution behavior for invalid signed arithmetic cases beyond the supplied 32-bit representation.

The apparent intent is insertion of Count repeated values into a contiguous sequence. That intent does not justify replacing the observed dependency-call sequence with a conventional insertion algorithm. In particular, the descending per-address calls in the long-suffix branch are established; their object-lifetime meaning is not.

## Evidence

Static evidence establishes the zero-count guard, three-word descriptor layout, 16-byte stride, arithmetic expressions, branch conditions, dependency arguments, descending call order, and descriptor-write order described above.

Interpretations involving a container, logical size, capacity, insertion, allocation, and element lifetime are inferred from those uses. Dependency implementations and caller guarantees were not supplied.

No runtime tests have been performed. The acceptance examples are conditional deductions from the supplied static evidence, not observed execution results.
