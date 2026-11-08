<!-- Trusted provenance: F-fbaf5dc07004; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbaf5dc07004

## Purpose

F-fbaf5dc07004 conditionally replaces the logical contents of a destination record with contents obtained from a source record, returning the destination record’s address.

The supplied static evidence shows three operational cases: replacement using newly allocated storage, reuse of existing storage with an additional tail transfer, and reuse of existing storage followed by contraction of recorded bounds. An identical source and destination address causes an immediate no-op.

The apparent intent is assignment between contiguous collections of 24-byte elements. That interpretation is inferred from helper usage and address arithmetic; the meanings of the helpers and the elements are not independently established.

## Interface

The function accepts four arguments:

| Argument | Established representation | Role |
|---|---|---|
| Destination | Address of a record accessed through 32-bit words | Record that may be modified; also the return value |
| Source | Address of a record accessed through 32-bit words | Record supplying range boundaries and helper-derived counts |
| Auxiliary value A | Unsigned 32-bit value | Initially stored in scratch storage, but overwritten before any consequential use |
| Auxiliary value B | Unsigned 32-bit value | Initially stored in scratch storage, but overwritten before any consequential use |

No observable result in the supplied body depends on either auxiliary value.

The visible destination layout is:

| Byte offset | Width | Observed use |
|---|---|---|
| 0 | 32 bits | Beginning address of element storage |
| 4 | 32 bits | End address of the recorded active range |
| 8 | 32 bits | Additional storage boundary, apparently the capacity boundary |

The source is directly read at offsets 0 and 4. Its complete required layout is unknown because dependencies also receive its address.

All visible element offsets use a stride of 24 bytes. Counts returned by F-77052f875b09 and F-c4290d3b80f6 participate in unsigned 32-bit comparisons or arithmetic. Storage addresses and address differences are represented through 32-bit quantities in the supplied evidence. This does not establish portability to a different pointer width.

The function returns exactly the destination argument on every explicit normal-return path. It has no explicit failure return, status output, or error-reporting branch.

For defined ordinary operation, the records, storage ranges, and helper contracts must support every access described below. Null-pointer validation, allocation-failure handling, range validation, and arithmetic-overflow rejection are not present in this body.

## Behavior

### 1. Identity check

The function first compares the source and destination record addresses.

If they are equal:

- It does not read their fields.
- It invokes no dependency.
- It performs no record or element write.
- It returns the destination address.

This check concerns record identity only. Distinct records whose storage overlaps are not detected as aliases.

### 2. Initial storage decision

For distinct record addresses, the function:

1. Obtains a source count from F-77052f875b09.
2. Obtains a destination quantity from F-c4290d3b80f6.
3. Compares the destination quantity against the source count using an unsigned comparison.

The apparent meanings are source length and destination capacity. Those meanings remain inferred.

If the destination quantity is smaller, execution takes the allocation path. Otherwise, execution takes the reuse path.

Helper results are not generally cached across later operations. Repeated calls are observable and may produce different results. The contract therefore does not assume that counts remain stable or that helpers preserve records or global state.

### 3. Allocation path

#### Selecting a requested count

After deciding that additional storage is required, the function separately obtains:

1. Another source count from F-77052f875b09.
2. A destination count from F-77052f875b09.

Using that destination count, it computes two unsigned 32-bit growth candidates:

- The count plus 32.
- The count plus its logical right shift by one, plus its logical right shift by three.

For a count whose additions do not wrap, the second candidate corresponds to the count plus the integer parts of one-half and one-eighth of that count.

The function chooses the larger candidate, then chooses the larger of that result and the newly obtained source count. Equality does not change the selected numeric value.

All these additions are unsigned 32-bit operations. The growth rule is therefore a comparison of wrapped candidates when overflow occurs, not an unbounded mathematical growth calculation.

#### Acquiring and populating storage

The selected count is multiplied by 24 in unsigned 32-bit arithmetic. The resulting value and a second argument of zero are passed to F-871e64ddc6de.

The returned 32-bit value is used as the address of new storage. There is no visible check for a null result, an error sentinel, or a request that wrapped during multiplication.

Next, F-1202cf471ac6 is called with:

- The source’s currently read beginning address.
- The source’s currently read end address.
- The returned storage address.
- The destination record address.

This occurs before the function explicitly replaces the destination’s beginning field. The dependency nevertheless receives the destination record and may have effects not visible in this body.

#### Traversing and releasing the previous storage

After the transfer call returns, the function reads and saves the destination’s current beginning field. Thus, the address selected for subsequent release is the destination beginning **after** that dependency, not necessarily the beginning observed at function entry.

Starting at that saved address, an empty-body traversal advances by 24 until its cursor equals the destination’s end field. The loop contains no explicit element operation or destructor call. Its end field is evaluated repeatedly by the visible loop condition.

This traversal completes only if successive stride advances reach the compared boundary. An inconsistent or changing boundary can prevent normal completion.

After the traversal:

1. F-c4290d3b80f6 is called on the destination.
2. F-ff1bdc376dcd is called with the saved beginning address, that helper’s result, and zero.

The second argument to F-ff1bdc376dcd is passed without visible conversion to bytes. Its units and the dependency’s ownership semantics are unknown.

#### Publishing the new boundaries

After the release-like dependency returns, the function explicitly:

1. Writes the allocation result into the destination’s beginning field.
2. Calls F-77052f875b09 on the source again.
3. Writes the destination end as its currently read beginning plus 24 times that new source count.
4. Computes a boundary from the destination’s currently read beginning plus 24 times the earlier selected allocation count.
5. Writes that boundary at destination offset 8.

The source count used for the final active end is therefore obtained **after** the release-like call. It is not necessarily the count used to select the allocation size.

There is no visible rollback if a dependency fails after storage acquisition or after any intermediate mutation.

### 4. Reuse path: destination count smaller than source count

When the initial destination capacity-like result is sufficient, the function separately obtains a destination count and then a source count from F-77052f875b09.

If the destination count is smaller:

1. It obtains the destination count again.
2. It calls F-8d47ad349e29 for a source prefix ending at the source beginning plus 24 times that latest destination count. The destination beginning is the output location.
3. It ignores that dependency’s return value.
4. It obtains the source count again.
5. It writes the destination end as the currently read destination beginning plus 24 times that source count.
6. It calls F-1202cf471ac6 for the source tail, using the previously saved destination count to form both the source-tail beginning and destination-tail beginning.
7. It returns immediately.

The explicit end-field update occurs **before** the tail-transfer dependency. This ordering is observable to that dependency.

The function performs no explicit write to destination offset 8 in this branch. This does not establish that the field remains unchanged: the tail-transfer dependency receives the destination record address.

The source and destination beginning fields and the source end are read again when constructing later calls. They are not guaranteed to retain their earlier values.

### 5. Reuse path: destination count at least source count

If the destination count is not smaller than the source count:

1. F-8d47ad349e29 is called with the source’s currently read beginning and end, and the destination’s currently read beginning.
2. Its returned value becomes a traversal cursor.
3. An empty-body traversal advances that cursor by 24 until it equals the destination end.
4. F-77052f875b09 is called on the source again.
5. A new boundary is computed from the destination’s currently read beginning plus 24 times that source count.
6. The function writes that boundary to the destination end.
7. It writes the same boundary to destination offset 8.

The offset-8 write is significant. Under the inferred collection interpretation, this branch records capacity equal to the new active length, even if the destination previously had more storage. No allocation or release-like dependency is explicitly called in this branch.

The traversal starts at the return value of F-8d47ad349e29, not at a boundary independently computed by this function. No explicit element cleanup occurs in its body.

## Side effects

### Direct effects

Depending on the branch, the function directly writes destination offsets 0, 4, and 8. Their order is:

- **Identity:** no writes.
- **Allocation:** beginning, active end, storage boundary.
- **Reuse with extension:** active end before the tail-transfer call; immediate return afterward.
- **Reuse without extension:** active end, then storage boundary, both assigned the same computed value.

The function contains no direct element-data writes. Such writes, if any, occur in dependencies.

No direct source-record write is visible. Source immutability is not guaranteed because dependencies receive source addresses or source storage addresses.

### Ownership and lifetime

The ordering supports an inferred acquire–transfer–release–publish operation in the allocation branch. It does not establish:

- Whether F-871e64ddc6de returns exclusively owned memory.
- Whether either transfer helper copies, moves, constructs, or otherwise transforms elements.
- Whether transfer helpers acquire or relinquish ownership.
- Whether F-ff1bdc376dcd releases storage, elements, both, or neither.
- Whether any dependency changes globals or either record.

The empty traversals are not evidence of element destruction.

### Errors and nontermination

There are no explicit checks for failed allocation, invalid boundaries, overlapping storage, insufficiently sized storage, or count overflow.

Both empty traversals require a boundary reachable by repeated 24-byte increments. Their bodies provide no independent escape condition.

Address calculations combine 32-bit quantities with integer-valued storage addresses. Defined behavior for out-of-range addresses, signed intermediate overflow, or invalid resulting pointers is not established by the supplied representation. Such cases must not be accepted as successful collection operations merely because a wrapped value can be calculated.

## Dependencies

| Opaque ID | Established use | Unestablished details |
|---|---|---|
| F-77052f875b09 | Receives a record address; returns a count-like value used in comparisons and 24-byte address calculations | Exact count definition, validity requirements, mutations, failure behavior |
| F-c4290d3b80f6 | Receives the destination; supplies the initial capacity-like comparison value and, on allocation paths, an argument to the release-like call | Units, derivation, side effects, relation to offset 8 |
| F-871e64ddc6de | Receives a wrapped count-times-24 request and zero; its result becomes a storage address | Allocation guarantees, alignment, ownership, failure representation |
| F-1202cf471ac6 | Receives a source range, destination address, and destination record | Transfer semantics, cleanup, record mutation, exceptional behavior |
| F-8d47ad349e29 | Receives a source range and destination address; its return is ignored in one branch and used as a traversal cursor in another | Element semantics, return-address guarantee, overlap support |
| F-ff1bdc376dcd | Receives the saved destination beginning, a fresh F-c4290d3b80f6 result, and zero | Release semantics, units, ownership requirements, failure behavior |

Argument expressions within an individual dependency call do not establish a complete left-to-right order of field reads. The inter-call ordering and explicit field-write ordering described above are established.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Numeric examples assume stable length/capacity-like helper results, valid nonoverlapping storage, address arithmetic without overflow, and dependencies that perform the inferred transfers without unexpected record mutation.

1. **Same record address.**  
   Passing the same record as source and destination returns that address without invoking dependencies or reading fields.

2. **Allocation from an empty destination.**  
   A destination length and capacity of zero with source length 5 selects 32 elements: the additive growth candidate is 32, the fractional candidate is zero, and the source count is smaller than 32. The allocation request is 768 bytes. After successful dependencies, the recorded active end is 120 bytes beyond the new beginning, and offset 8 is 768 bytes beyond it.

3. **Fractional growth dominates.**  
   Destination length and capacity 64, source length 65: the candidates are 96 and 104. The selected count is 104, producing a 2,496-byte request. The final active end is 1,560 bytes beyond the new beginning.

4. **Source count dominates growth.**  
   Destination length and capacity 8, source length 100: the candidates are 40 and 13. The selected count is 100, producing a 2,400-byte request.

5. **Reuse with extension.**  
   Destination length 2, capacity 8, source length 5: the prefix transfer covers 48 source bytes. The function publishes an active end 120 bytes beyond the destination beginning before transferring the remaining three elements. It does not explicitly rewrite offset 8.

6. **Reuse with contraction.**  
   Destination length 5, capacity 8, source length 2: the full-source transfer is followed by the empty traversal. Assuming the transfer returns the destination address immediately after two elements, the traversal advances three strides to the old active end. The function then writes both destination boundaries to beginning plus 48 bytes.

7. **Zero-length source with a nonempty destination.**  
   This follows the non-extension reuse branch under stable helper results. If the transfer returns the destination beginning and the traversal terminates, both destination boundaries become the beginning address. No explicit release-like call occurs.

8. **Overflowing allocation multiplication.**  
   A selected count of 178,956,971 produces a 32-bit allocation argument of 8 because multiplication by 24 wraps. Acceptance must preserve that observed argument calculation; successful storage or transfer behavior is documented_only.

## Uncertainties

- **Inferred intent:** contiguous-collection assignment, length and capacity helpers, storage acquisition and release, and prefix assignment versus tail construction.
- **Documented_only:** element representation, element ownership, alignment, helper error contracts, overlap support, concurrency behavior, and dependency effects on global state.
- **Documented_only:** behavior after allocation failure, dependency failure, invalid ranges, or address overflow.
- **Documented_only:** whether changing helper results can leave the allocation request smaller than the later published active range.
- **Established limitation:** the non-extension reuse branch writes offset 8 equal to the new active end. It must not be specified as preserving prior capacity.
- **Established limitation:** auxiliary arguments do not influence the visible outcome.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established evidence consists of branch conditions, dependency-call order, 24-byte strides, unsigned growth calculations, explicit field accesses, empty traversal bodies, boundary-write ordering, and normal return values.

Collection terminology and dependency roles are interpretations, not independently verified guarantees. No dependency implementations were supplied. No runtime tests have been performed.
