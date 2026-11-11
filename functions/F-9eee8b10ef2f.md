<!-- Trusted provenance: F-9eee8b10ef2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9eee8b10ef2f

## Purpose

F-9eee8b10ef2f mutates links and ranking metadata in a collection of fixed-stride records. Its observable work has three phases:

1. Compare eligible record pairs and independently update each endpoint’s preferred link.
2. Examine reciprocal links and resolve certain enclosing or competing pairs, either clearing links or redirecting incoming links into a secondary field.
3. Count incoming preferred links and remove nonreciprocal preferred links, conditionally retaining an indirect link in the secondary field.

These are behavioral descriptions, not established application meanings. Interpreting the coordinates as geometric positions, the preferred links as matches, or the final counter as a reference count is inferred intent.

The visible interface is incomplete: four incoming machine-state values are consumed without being initialized by the function. They are necessary inputs to the contract, not safe defaults.

## Interface

### Explicit inputs and output

The visible entry has three arguments:

| Input | Representation | Use |
|---|---|---|
| Distance threshold | Signed 32-bit integer | Controls reciprocal-pair processing and final nonreciprocal-link filtering. |
| Initial first-record address | Unsigned 32-bit address | First endpoint used by the initial comparison phase. |
| Initial second-record address | Unsigned 32-bit address | Second endpoint used by the initial comparison phase. |

There is no explicit return value. Results consist of writes to records and to records reached through their links.

### Required incoming machine state

Four additional values are read from incoming execution state:

| Implicit input | Representation | Use |
|---|---|---|
| Collection end | Unsigned 32-bit address | Exclusive upper bound for collection traversal and pair advancement. |
| Collection base | Unsigned 32-bit address | Restart address for second-endpoint scans and start of later collection-wide scans. |
| Minimum overlap | Signed 32-bit integer | Lower bound for accepting an overlap during pair comparison. |
| Selection tag | Unsigned 32-bit value | Compared with the promoted byte at offset 1 when selecting subsequent first endpoints. |

How callers establish these values is **documented_only**. The visible argument list alone is insufficient to invoke the behavior reliably.

### Record layout

Collection traversal uses a stride of 48 bytes. The following fields are accessed:

| Byte offset | Width | Behavioral role |
|---|---:|---|
| 1 | 8 bits | Tag used in pair eligibility and subsequent first-endpoint selection. |
| 2 | Signed 16 bits | Coordinate used for ordering and distance calculation. |
| 4 | Signed 16 bits | Lower extent used in overlap calculation. |
| 6 | Signed 16 bits | Upper extent used in overlap calculation. |
| 16 | 32 bits | Primary link, interpreted as an address; zero means absent. |
| 20 | 32 bits | Secondary link or redirected address. |
| 24 | 32 bits | Counter, incremented and sometimes decremented. |
| 28 | Signed 32 bits, also raw 32-bit reads | Stored distance or ranking value. |
| 32 | Signed 32 bits | Stored overlap or ranking value. |
| 36 | 32 bits | Selection field compared for equality with offset 40. |
| 40 | 32 bits | Selection field compared for equality with offset 36. |

The byte’s signedness is not established by the supplied representation. Reads through plain character types may promote differently depending on the target’s character convention. This affects both the zero-sum tag test and the comparison with the selection tag.

Byte order and alignment requirements are not established here.

### Preconditions and ownership

For defined, terminating execution:

- Every accessed field must be readable, and every written field must be writable.
- Every nonzero primary link that is dereferenced must designate storage supporting the accessed fields.
- The collection and its 48-byte progression must be consistent with the intended record layout.
- Address advancement must not wrap before traversal reaches its bound.
- The initial endpoint addresses must be valid even if the collection bounds describe an empty range: the initial pair is examined before a bounds check protects it.

The function does not visibly allocate, release, acquire, or transfer ownership of memory. Writing a link is not evidence of ownership transfer. It does not validate pointers or record contents.

## Behavior

### 1. Initial and subsequent pair comparisons

The first pair is examined immediately, without first checking whether the endpoints are equal or within the collection bounds.

A pair proceeds to ranking only if:

- The promoted tags at offset 1 sum to zero.
- The second endpoint’s signed 16-bit coordinate minus the first endpoint’s signed 16-bit coordinate is nonnegative.

The coordinate difference lies between −65,535 and 65,535 before the nonnegative restriction.

For an eligible ordered pair, the overlap is:

- The smaller of the two signed upper extents,
- Minus the larger of the two signed lower extents.

There is no clamp to zero. Disjoint extents can therefore produce a negative overlap. That overlap remains eligible if it is at least the incoming minimum overlap.

Each endpoint is then considered independently for a ranking update. For a given endpoint, let the candidate distance be the coordinate difference, and let its stored distance and overlap be the values at offsets 28 and 32.

An update occurs only when both conditions hold:

- Eight times the candidate distance is strictly less than nine times the stored distance.
- Either eight times the candidate distance is strictly less than seven times the stored distance, or the candidate overlap is strictly greater than the stored overlap.

When accepted, writes occur in this order:

1. Set the endpoint’s primary link to the other endpoint.
2. Store the candidate distance at offset 28.
3. Store the candidate overlap at offset 32.

The initial first endpoint is considered before the initial second endpoint. The second decision reads its fields after the first endpoint’s writes. The function does not stage both decisions against a snapshot. If endpoints alias, this ordering can affect the second decision.

No counters or secondary links are changed by this ranking step.

### 2. Pair traversal and first-endpoint selection

After each comparison attempt, the second-endpoint address advances by 48 bytes.

If it remains below the collection end, it is used for the next comparison unless it equals the first-endpoint address. Equal endpoints encountered during this advancement are skipped. This does not retroactively prevent an equal initial pair from being examined.

When the second endpoint reaches or passes the end, the function advances the first endpoint by 48 bytes. A newly advanced first endpoint is selected only when:

- Its 32-bit fields at offsets 36 and 40 differ.
- Its promoted tag equals the incoming selection tag.

For this selection, the fields are examined in that order. The tag is not read when the two selection fields are equal. When those fields differ, the second-endpoint address is reset to the collection base as part of the tag-selection path.

The initial first endpoint bypasses these selection checks.

When first-endpoint advancement reaches or passes the collection end, pair comparison ends. Subsequent processing begins at the collection base.

### 3. Reciprocal-pair processing

Records are visited in ascending address order, in 48-byte increments.

A record is considered for this phase only when:

- Its primary link is nonzero.
- The linked record’s primary link points back to it.
- Its signed coordinate is strictly less than the linked record’s coordinate.
- Its stored distance is strictly less than the explicit distance threshold.

For such a reciprocal pair, the function scans the collection from its base for a competing reciprocal pair. A competitor qualifies only if:

- Its first coordinate is no greater than the current record’s coordinate.
- Its first endpoint is not the current record.
- It has a nonzero primary link, and that link is reciprocal.
- Its linked endpoint’s coordinate is at least the current pair’s linked endpoint coordinate.
- At least one of the two endpoint coordinates differs between the pairs.
- The current pair’s stored distance is strictly less than the competitor’s stored distance.
- The competitor’s stored distance is strictly less than four times the current pair’s stored distance.

For each qualifying competitor, the overlap comparison determines the mutation.

**Clearing case:** If the current pair’s stored overlap is strictly less than three times the competitor’s stored overlap:

1. Clear the primary link on the current pair’s linked endpoint.
2. Clear the primary link on the current record.
3. Stop scanning competitors for that current record.

Secondary links, counters, and stored ranking fields are not cleared by this case.

**Redirection case:** Otherwise, scan every collection record in address order:

- If its primary link equals the competitor’s first endpoint, write the current pair’s linked endpoint into its secondary link, then clear its primary link.
- Otherwise, if its primary link equals the competitor’s linked endpoint, write the current record into its secondary link, then clear its primary link.

The first condition has precedence. Secondary links are overwritten without an emptiness check.

This scan includes the competitor endpoints themselves and can therefore break their reciprocal relationship. After redirection, scanning for additional competitors continues.

All scans operate on live mutable fields. Earlier changes can affect later eligibility; there is no preserved graph snapshot.

### 4. Final counting and nonreciprocal-link filtering

After reciprocal-pair processing reaches the end, a final collection-wide pass begins at the base.

For each record, in address order:

1. Read its primary link.
2. If nonzero, increment the linked record’s 32-bit counter at offset 24.
3. Read that linked record’s primary link after the increment.
4. If it points back to the current record, retain the current primary link.
5. Otherwise, clear the current primary link before making the remaining decision.

For a cleared nonreciprocal link, read the linked record’s stored distance.

Two signed comparisons determine whether the counter increment is undone:

- If the linked distance is below the explicit threshold, undo the increment.
- Otherwise, compare the current record’s stored distance with four times the linked distance. Undo the increment if the current distance is at least that product.

If neither condition causes the increment to be undone, copy the linked record’s then-current primary link into the current record’s secondary link. This copied value can be zero.

The counter is not initialized by this function. Its final value is relative to its incoming value and all preceding mutations. A failed retention decision does not clear an existing secondary link; it only clears the primary link and undoes the counter increment.

### Arithmetic and comparison boundaries

Address bounds and traversal comparisons use unsigned 32-bit values. Coordinates and ranking comparisons use signed values as described above.

Strictness matters:

- Minimum overlap accepts equality.
- Ranking inequalities are strict.
- The reciprocal-pair threshold comparison is strict.
- Competing distances must be strictly between the current distance and four times that distance.
- Equality in the overlap-times-three comparison selects redirection, not clearing.

The candidate distance multiplied by eight fits within signed 32 bits. Products involving stored 32-bit distances or overlaps need not.

The final filtering decision explicitly uses subtraction-overflow information. Its comparison against four times the linked distance applies to the 32-bit scaled result, not necessarily to an unbounded mathematical product. Ordinary mathematical interpretations of the other scaling comparisons are safe only when their intermediate products are representable. Overflow behavior outside that domain is **documented_only**.

## Side effects

The function may write:

- Primary links at offset 16.
- Secondary links at offset 20.
- Counters at offset 24.
- Stored distances at offset 28.
- Stored overlaps at offset 32.

It does not visibly write coordinates, extents, tags, or the selection fields.

Writes can extend beyond the collection because nonzero primary links are followed without range validation. No visible synchronization, rollback, error reporting, or external I/O is present.

Observable sequencing includes:

- Primary-link writes before ranking metadata writes.
- Clearing the linked endpoint before the current endpoint in the reciprocal clearing case.
- Secondary-link writes before primary-link clearing in redirection.
- Counter increment before reciprocal-link inspection in the final pass.
- Primary-link clearing before the final nonreciprocal retention decision.

## Dependencies

No called functions are visible. No additional opaque dependency IDs can be established from the supplied evidence.

The effective dependencies are the incoming machine state, the record memory, linked-record memory, and the target’s data representation and arithmetic semantics. There is no evidence permitting assumptions about external initialization, exclusive access, pointer ownership, or counter meaning.

## Acceptance examples

These examples are conditional static acceptance criteria. No runtime tests have been performed. Assume valid memory, signed values within the nonoverflowing arithmetic domain, and no unintended aliasing unless stated otherwise.

### Independent endpoint ranking

Suppose a pair has zero-sum tags, coordinates 10 and 20, and extents [0, 20] and [5, 25]. Its distance is 10 and overlap is 15.

If an endpoint initially stores distance 12 and overlap 10, it accepts the candidate: 80 is less than 108, and also less than 84. Its primary link is written first, followed by distance 10 and overlap 15.

The other endpoint is evaluated separately against its own stored values. Acceptance by one endpoint does not imply acceptance by the other.

### Ranking equality boundaries

For candidate distance 9 and stored distance 8, the first ranking comparison is 72 less than 72, which is false. No update occurs, regardless of overlap.

For candidate distance 7 and stored distance 8, the first comparison succeeds, but the second distance comparison is 56 less than 56, which is false. Acceptance then requires a strictly larger candidate overlap.

### Negative overlap

For extents [0, 3] and [5, 8], overlap is −2. A minimum overlap of −2 permits ranking evaluation; a minimum overlap of −1 rejects it.

### Enclosing competitor clears a pair

Suppose a reciprocal pair has coordinates 10 and 20, stored distance 10, and stored overlap 20. The threshold is 11. A reciprocal competitor has coordinates 5 and 25, stored distance 20, and overlap 7.

The coordinate and distance conditions qualify. Since 20 is less than 21, the current pair’s two primary links are cleared, linked endpoint first.

### Overlap equality redirects instead

With the preceding setup but current overlap 21, the clearing comparison fails at equality. Records pointing to the competitor’s first endpoint receive the current pair’s linked endpoint as a secondary link. Records pointing to the competitor’s linked endpoint receive the current record instead. Each affected primary link is cleared after its secondary write.

### Final nonreciprocal retention

Suppose a current record points to a target that does not point back. The threshold is 10 and the target’s stored distance is 12.

The target counter is incremented, and the current primary link is cleared.

- With current stored distance 47, the increment remains and the target’s current primary link is copied into the current secondary link.
- With current stored distance 48, the increment is undone and the existing secondary link remains unchanged.

### Initial-pair asymmetry

Equal endpoint addresses supplied initially are not skipped before comparison. Equality reached through later second-endpoint advancement is skipped. Callers must not assume those cases are equivalent.

## Uncertainties

The following remain **documented_only**:

- The calling convention supplying the four implicit inputs.
- The target’s plain-character signedness.
- Byte order and alignment rules.
- Application meanings and initialization rules for fields.
- Whether callers guarantee collection membership for links.
- Behavior under invalid addresses, address wraparound, unsupported aliasing, or concurrent mutation.
- Full overflow semantics for ordinary scaled signed comparisons.
- Whether any externally meaningful return-state convention exists beyond the visible absence of a return value.

## Evidence

Established static evidence consists of the supplied function body: field widths and offsets, 48-byte traversal, incoming-state reads, branch predicates, write order, link-following behavior, and absence of visible calls or allocation operations.

Geometric matching, pair competition, and incoming-link counting are inferred interpretations of those operations, not independently established domain requirements.

No implementation execution, runtime tests, caller inspection, external dependency inspection, or target-platform validation has been performed.
