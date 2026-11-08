<!-- Trusted provenance: F-c314dc6fa2fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c314dc6fa2fa

## Purpose

F-c314dc6fa2fa conditionally selects two values from an associated indexed object, applies the primary value to its receiver, writes both values into receiver state, and invokes two downstream notifications.

The established contract is an ordered sequence of checks, dependency calls, shared-word reads, and receiver writes. Explicit normal returns are 32-bit `0` or `1`:

- `0` means that an explicit eligibility condition prevented selection.
- `1` means that selection, application, the two explicit state writes, both notifications, and the region-exit call completed normally.

Neither result establishes that dependencies had no other effects.

The apparent intent is to advance through an ordered collection, optionally skipping an entry according to a threshold. This is inferred intent only. The supplied representation does not establish the domain meaning of the values, valid index ranges, monotonic progression, or a conventional end-of-collection rule.

## Interface

### Receiver and result

The sole explicit input is a pointer to a mutable receiver.

The representation uses four-byte integer words, 32-bit result values, and object references held in four-byte storage. All offsets below describe that represented layout; compatibility with another pointer width is not established.

The receiver is not checked for null. For normal completion, the receiver, accessed embedded storage, dispatch tables, associated object, and invoked targets must remain valid whenever used. The function does not itself enforce those lifetime requirements.

A dependency that faults, throws, terminates, or otherwise fails to return normally may prevent any explicit result. No exception translation, fault recovery, or abnormal-exit cleanup is shown.

### Accessed receiver layout

Offsets are byte offsets from the receiver address.

| Offset | Established use |
|---|---|
| `0x000` | Receiver dispatch-table pointer. |
| `0x014` | Embedded dispatch-bearing subobject used by the final notification. |
| `0x054` | Four-byte nullable associated-object reference. |
| `0x060` | Embedded address passed to F-b7abcf60c990. |
| `0x994` | Four-byte destination for the selected secondary value. |
| `0x998` | Four-byte destination for the selected primary value. |
| `0xA4C` | Embedded address passed to the paired entry and exit dependencies. |

For a nonzero associated reference, the indexed-object receiver is the referenced address plus four bytes. The word at that derived address supplies its dispatch-table pointer.

These observations establish accessed storage locations, not complete object definitions, allocation sizes, or ownership relationships.

### Shared inputs

Two shared 32-bit words are read:

- **Threshold:** read once, immediately after the first indexed-object mode query returns. That saved threshold is used for both possible threshold comparisons.
- **Notification argument source:** read twice on a successful selection path, after both explicit receiver-state writes.

The notification reads have distinct roles and order:

1. The first read is saved for the later embedded-subobject notification.
2. The second read supplies the argument to F-069a1d444a2d.

The two notification arguments need not be equal. Equality requires stability between the reads, which the supplied evidence does not guarantee. Changes made by F-069a1d444a2d cannot change the earlier saved value supplied to the final notification.

Initialization, permitted concurrent access, atomicity guarantees, and domain meanings of these shared words are unknown.

## Behavior

### Initial checks and region entry

The first call is F-b7abcf60c990 with receiver address plus `0x060`.

If its result is zero, the function returns `0`. It does not perform the receiver eligibility dispatch, region entry, associated-object read, or explicit state writes.

If that result is nonzero, the function invokes F-000000000001 on the receiver. A nonzero result causes return `0`, without region entry.

If both checks permit continuation, the function computes receiver address plus `0xA4C` and invokes F-000000000002 with that address. The corresponding F-000000000003 call occurs before every explicit normal return after entry.

The pairing suggests a protected region, but locking or resource-management semantics are not established. The embedded address used for the exit call is retained from the entry setup.

### Associated object, count, and lookup

After entry returns, the function reads the associated-object reference at receiver offset `0x054`.

- A zero reference causes F-000000000003 followed by return `0`.
- A nonzero reference produces the indexed-object receiver at referenced address plus four bytes.

The associated reference is read after entry, so changes made before that read can affect which object is selected. Once the indexed-object receiver is derived, subsequent operations use that retained receiver rather than rereading the associated-reference slot.

F-000000000004 obtains the indexed-object count. A zero count causes region exit and return `0`.

Only equality with zero is tested. Negative counts are not explicitly rejected. The count is obtained once and retained; later dependency mutations do not cause another count query.

For a nonzero count:

1. F-000000000005 obtains the receiver’s current value.
2. F-000000000006 receives that value and returns a working index.

There is no explicit check that the working index is nonnegative or within the count.

If the index equals count minus one, the function exits the region and returns `0`. This path performs no mode query and does not read the threshold.

### Initial candidate gate

If the index differs from count minus one, F-000000000007 queries the indexed-object mode. The threshold is read immediately after that call returns, even if its result is zero.

A zero mode result permits candidate construction without a gate-specific indexed-value query or current-value query.

For a nonzero mode result:

1. F-000000000008 queries the value at index plus one.
2. The function checks whether the working index equals count minus two.
3. If it does not, candidate construction is permitted.
4. If it does, F-000000000005 obtains a fresh receiver current value. Candidate construction is permitted only when the unsigned 32-bit interpretation of the queried next value minus that current value is greater than or equal to the saved threshold.

Thus, the gate-specific value query occurs for every nonzero first mode result, not only at count minus two. The fresh current-value query occurs only at that boundary.

A rejected gate causes region exit and return `0`.

The comparison is unsigned. A negative, representable signed difference becomes a large unsigned value when converted; it is not automatically treated as a disqualifying negative distance.

### Initial candidates

Once the gate permits continuation, F-000000000008 queries index plus one again. This new result becomes the primary candidate.

Any earlier value from the gate is not reused. Dependencies may be stateful, so the two queries can produce different values.

The initial secondary candidate is selected next:

- If index plus one equals count minus one, it is the 32-bit all-ones value, represented as `-1`.
- Otherwise, F-000000000008 queries index plus two, and that result becomes the secondary candidate.

The all-ones value is established as a value written on particular paths. Its meaning to downstream consumers is unknown.

### Optional skip

F-000000000007 is then called a second time. Its result is not assumed to match the first mode result.

If the second result is zero, the initial candidates remain selected. No skip-specific current-value query occurs.

If the second result is nonzero, F-000000000005 obtains another fresh current value. A skip occurs only when the unsigned 32-bit interpretation of primary candidate minus this current value is strictly less than the previously saved threshold.

When a skip occurs:

1. F-000000000008 queries index plus two; that result replaces the primary candidate.
2. The function evaluates whether index plus two, minus the retained count, equals `-2`.
3. If equal, the secondary candidate becomes the all-ones value.
4. Otherwise, F-000000000008 queries index plus three; that result replaces the secondary candidate.

The skip sentinel condition must not be replaced by a conventional “replacement primary is last” rule. In nonoverflowing arithmetic, the represented condition is equivalent to index equaling count minus four.

Consequently, index equal to count minus three does not trigger this sentinel. A taken skip then requests a secondary value at index plus three, numerically equal to count. Whether that request is valid is a dependency-level question.

### Applying and reporting the selection

After candidate selection, the established observable sequence is:

1. Invoke F-000000000009 on the receiver with the selected primary value.
2. Write the selected primary value to receiver offset `0x998`.
3. Write the selected secondary value to receiver offset `0x994`.
4. Read the shared notification word and save this first value.
5. Read the same shared notification word again as the argument supplied to F-069a1d444a2d.
6. Invoke F-069a1d444a2d with the receiver and the value from the second read.
7. Obtain the final notification’s dispatch information from the embedded subobject at receiver offset `0x014`.
8. Invoke F-00000000000a on that subobject with the saved value from the first notification-word read and an additional zero argument.
9. Invoke F-000000000003 with the retained region address.
10. Return `1`.

Application precedes both explicit state writes; the primary write precedes the secondary write.

The first notification-word read is not merely a description of the argument evaluation for F-069a1d444a2d: a separate second read supplies that call. The final notification uses the earlier saved value, not a reread after the helper.

The final dispatch lookup occurs after F-069a1d444a2d returns. No assumption that the helper preserves that dispatch information is justified.

### Arithmetic boundaries

Counts, indices, and indexed values are represented using 32-bit words. Threshold comparisons explicitly interpret the computed differences as unsigned 32-bit values.

The representation also contains signed arithmetic for count minus one, count minus two, index increments, skip-boundary subtraction, and value differences. Exact behavior where those signed operations overflow is not independently established by the supplied representation. Ordinary arithmetic descriptions and examples below assume the relevant signed operations are representable.

The function performs no explicit arithmetic-overflow rejection.

## Side effects

The function’s explicit receiver writes are the primary and secondary four-byte stores on a successful selection path. No direct allocation, deallocation, reference-count adjustment, or ownership transfer is shown.

Dependencies can have additional effects. Their implementations are unavailable, so this contract does not assume that they preserve:

- Receiver or indexed-object contents.
- Shared words.
- Dispatch pointers.
- Object lifetimes.
- External state.

Important consequences:

- Return `0` does not imply absence of side effects.
- Entry and exit are paired on the explicit normal-return paths after entry, not necessarily on abnormal exits.
- A normally returning application call is followed by both explicit state writes.
- Later notifications or the exit dependency may modify the state again; the final stored values at return are not guaranteed solely by the earlier stores.
- No rollback is shown if a later dependency fails after either state write.
- The saved notification value remains the argument selected for the final notification despite subsequent changes to its shared source.

Pointers are used as borrowed addresses in the visible operations. No stronger ownership or lifetime guarantee follows, and no dependency is assumed to retain or release an object.

## Dependencies

The supplied dependency identities are preserved below. Other opaque IDs are document-local identifiers for unresolved call targets.

| Opaque ID | Established interface and use |
|---|---|
| F-b7abcf60c990 | Receives receiver address plus `0x060`; zero prevents further processing. |
| F-000000000001 | Receiver dispatch offset `0x1A4`; receives receiver; zero permits entry. |
| F-000000000002 | Receives receiver address plus `0xA4C`; entry role inferred from pairing. |
| F-000000000003 | Receives the retained entry address before every explicit normal return after entry. |
| F-000000000004 | Indexed-object dispatch offset `0x138`; returns the count. |
| F-000000000005 | Receiver dispatch offset `0x17C`; returns a current value for lookup or comparison. |
| F-000000000006 | Indexed-object dispatch offset `0x144`; receives the current value and returns an index. |
| F-000000000007 | Indexed-object dispatch offset `0x118`; returns a zero/nonzero mode result. |
| F-000000000008 | Indexed-object dispatch offset `0x13C`; receives an index and returns its associated value. |
| F-000000000009 | Receiver dispatch offset `0x100`; receives the selected primary value before direct state writes. |
| F-069a1d444a2d | Receives receiver and the second notification-word read, after the first read has been saved. |
| F-00000000000a | Embedded-subobject dispatch offset `0x14`; receives that subobject, the saved first notification-word read, and zero. |

Ignored dependency results do not introduce additional success checks.

Repeated mode, current-value, and indexed-value calls are separate observations. They cannot be collapsed without an external equivalence guarantee. Likewise, the two notification-word reads cannot be treated as one observation without an external stability guarantee.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless a case specifies otherwise, assume valid storage, normally returning dependencies, and no signed arithmetic overflow.

### Early rejection

If F-b7abcf60c990 returns zero, the result is `0`; receiver eligibility and region entry are not called.

If it returns nonzero but F-000000000001 returns nonzero, the result is `0`, again without region entry.

### Missing or empty object

After entry, a zero associated reference causes exit and return `0`, without a count query.

With a nonzero associated reference and count zero, the function exits and returns `0`, without querying the receiver’s current value or looking up an index.

### Final-index rejection

For count `5` and lookup index `4`, the function exits and returns `0`. There is no mode query or threshold read.

### Ordinary selection

For count `5`, index `1`, and both mode results zero:

- The primary value is queried at index `2`.
- The secondary value is queried at index `3`.
- No threshold-comparison current-value query occurs.
- The threshold is nevertheless read after the first mode query.
- Application, state writes, both notification-word reads, notifications, and exit occur in the specified order.
- The result is `1`.

### Gate equality and rejection

For count `5`, index `3`, first mode nonzero, gate value `100`, fresh current value `95`, and threshold `5`, the gate permits selection because the difference equals the threshold.

If candidate construction again obtains `100`, the second mode is nonzero, and the next fresh current value is `95`, no skip occurs: the skip requires a difference strictly below `5`. The secondary candidate remains all ones.

If instead the gate’s fresh current value is `96`, the difference is `4`; selection is rejected, followed by exit and return `0`.

### Unusual skip boundary

For count `6`, index `2`, second mode nonzero, primary candidate `100`, fresh current value `98`, and threshold `5`, a skip occurs.

The replacement primary is queried at index `4`. The secondary becomes all ones because index plus two minus count equals `-2`. No replacement-secondary query at index `5` occurs.

For count `6` and index `3`, a taken skip instead queries replacement primary index `5` and replacement secondary index `6`. The latter request’s validity is unresolved.

### Unsigned difference

For a queried value of `90` and fresh current value of `100`, subtraction yields representable signed `-10`; its unsigned 32-bit interpretation is `4,294,967,286`.

Against threshold `5`, this satisfies the initial gate’s greater-than-or-equal comparison and does not satisfy the skip’s strictly-less comparison.

### Distinct notification observations

Suppose a successful path observes `17` on the first notification-word read and `23` on the second, under an environment permitting those observations.

F-069a1d444a2d receives `23`. The final embedded-subobject notification receives `17` and zero. Argument equality is not required by this function.

If F-069a1d444a2d subsequently changes the shared word to `99`, the final notification still receives `17`; it does not reread the shared word.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the receiver, count, index, selected values, mode, threshold, and notification word.
- Whether entry and exit implement locking or another protocol.
- Dependency effects, preservation rules, lifetime behavior, and failure conventions beyond visible zero/nonzero checks.
- Nonnegative-count and valid-index guarantees.
- Indexed-value behavior for negative, out-of-range, or overflow-derived indices.
- Whether all ones is a documented consumer sentinel.
- Whether the unusual skip sentinel condition is intentional.
- Concurrency, visibility, and atomicity guarantees, including whether notification-word changes between its two reads are permitted.
- Exact behavior at signed-overflow boundaries.

No unknown is resolved by assuming conventional collection, synchronization, ownership, or notification semantics.

## Evidence

This contract is based solely on static inspection of the supplied representation. No runtime tests have been performed.

Established evidence includes the explicit conditions and return values, 32-bit word uses, receiver and dispatch offsets, retained count and index, repeated dependency queries, threshold capture point, primary-before-secondary stores, unusual skip condition, and post-helper final dispatch lookup.

The notification sequence specifically establishes two ordered reads of the same shared word: the first is retained for the final notification; the second supplies F-069a1d444a2d. It does not establish equality of those observations.

Inferred intent is limited to advancement through a collection, a protected-region interpretation of the paired calls, and notification roles for the downstream calls. Dependency implementations, external object definitions, ownership policies, concurrency rules, and instruction-level overflow behavior were not supplied.
