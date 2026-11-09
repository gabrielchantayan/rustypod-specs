<!-- Trusted provenance: F-6fdb28c2fbf3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fdb28c2fbf3

## Purpose

F-6fdb28c2fbf3 computes and requests an index change using comparisons between values obtained at selected positions in an associated object.

The established behavior is a two-stage search: a coarse probe sequence followed by a finer scan. A negative movement also permits an initial comparison against the immediately preceding index. The selected destination is passed to an indirect method after the associated object's saved cursor has been restored.

The return value reports whether the index observed after that indirect call differs from the saved associated-object cursor. It does **not** directly report whether a matching value was found, whether the requested destination was accepted, or whether the final index differs from the initial index-query result.

A plausible intent is movement toward a boundary between groups of comparison-equivalent entries. That interpretation is inferred; the meaning of the retrieved values and the indirect method is not established.

## Interface

### Inputs

The function takes two inputs:

- **Receiver:** an address used as a structured object with several four-byte fields.
- **Movement:** a signed 32-bit integer. Zero disables all work. Positive and negative values select different endpoint and destination rules; the magnitude also affects search spacing.

The receiver layout visible to this function is:

| Byte offset | Observed use |
|---|---|
| `0x000` | Address of a dispatch table. |
| `0x0E8` | Address of an associated object containing the working cursor. |
| `0x100` | Signed 32-bit inclusive lower bound. |
| `0x104` | Signed 32-bit exclusive upper bound. |
| `0x1A0` | Four-byte argument passed to the value-lookup dependency. |

The associated object has a signed 32-bit cursor field at byte offset `0x018`.

The indirect destination method is obtained from byte offset `0x178` of the dispatch table. Its observed arguments are the receiver and one signed index. Its return value, if any, is not used.

These widths and offsets follow the supplied representation, which uses four-byte object words and addresses. The complete ABI, including the physical representation of a Boolean return, is not independently established.

### Output

The logical return is Boolean:

- Movement zero returns `false`.
- Otherwise, after the destination method completes, the function queries the receiver's index and returns whether that result differs from the saved associated-object cursor.

### Preconditions and validity limits

For nonzero movement, defined execution requires:

- Valid access to every receiver field, associated-object cursor, dispatch-table slot, and dependency argument actually used.
- A callable indirect destination method.
- Dependencies that return normally if normal completion is expected.
- Arithmetic results for which the target's signed-integer behavior is defined.

There is no visible null check, receiver validation, interval validation, allocation-failure handling, or arithmetic-overflow check.

An empty or reversed interval is not rejected. It can still lead to an indirect destination request, including an endpoint outside any nonempty valid interval.

## Behavior

### Zero movement

When movement is zero, the function returns `false` immediately.

This path does not dereference the receiver, call dependencies, create temporary handles, change the cursor, or invoke the destination method. Consequently, this path does not require the receiver to be readable.

### Initial value and state capture

For nonzero movement, the function first obtains a reference value:

1. It calls F-5710a8628cba using the receiver's associated-object address, an opaque four-byte lookup token read from shared storage, and the receiver's field at `0x1A0`.
2. It supplies that result to F-7afe554bcf62, with an eight-byte temporary storage area as the destination.
3. It calls F-9e5fecb2740f on that temporary and retains the returned four-byte value as the comparison reference.

The eight-byte temporary remains active until the final cleanup.

Only after these calls does the function read and save the associated object's cursor. Thus, the saved cursor is not necessarily the cursor present at function entry: the earlier dependencies may have affected it.

The function then calls F-b3f4a7a52609 to obtain the initial search index. After that call, it reads the lower bound and then the upper bound from the receiver. These bounds are retained for the search and final destination decision.

The saved cursor and initial search index are distinct observations and need not be equal.

### Optional preceding-position adjustment

For negative movement, the function considers the index immediately before the initial search index. It performs this adjustment when that preceding index is at least the cached lower bound.

There is no corresponding upper-bound check on this preliminary position.

When the adjustment is taken:

- The associated object's cursor is written with the preceding index.
- A value is obtained through F-5710a8628cba, F-7afe554bcf62, and F-9e5fecb2740f.
- F-b2840d22ea51 compares that value against the current comparison reference.

A zero comparison result has two effects:

- F-9e5fecb2740f is called **again** on the same temporary, and this second result replaces the comparison reference.
- The search anchor moves back by exactly one index.

The second extraction is observable and must not be replaced with reuse of the first extraction unless dependency behavior independently permits that change.

A nonzero comparison result leaves the search anchor and reference unchanged.

In either case, F-289ec4311c64 is called on the preliminary temporary before searching continues. If the comparison reference was replaced, it remains the second extracted value even after that temporary is released. Whether that value remains valid is a dependency-lifetime question, not something established by this function.

### Coarse probing

The first coarse candidate is the current search anchor plus movement.

A candidate is admissible only when it lies in the cached half-open interval: at least the lower bound and strictly below the upper bound.

For each admissible coarse candidate, the function:

- Writes that candidate to the associated object's cursor.
- Retrieves and materializes a value using the same lookup and temporary-handle sequence.
- Extracts a four-byte value and compares it against the reference.

If the comparison returns zero, the temporary is released and coarse probing stops. The fine scan begins from the existing anchor, not from this matching candidate.

If the comparison returns nonzero, the temporary is released, that candidate becomes the new anchor, and the next candidate is formed by adding **32 times movement** to it.

Thus, the first coarse displacement is movement, while subsequent coarse displacements are 32 times movement. Movement is not normalized to positive or negative one.

If a coarse candidate is outside the interval, it is not written to the cursor and no candidate value is materialized for it. The function proceeds to the fine scan from the current anchor.

### Fine scanning

The fine scan starts at the anchor left by initialization, the optional preceding-position adjustment, and coarse probing.

At each fine-scan position:

- An out-of-range position ends the scan without a value lookup at that position.
- An in-range position is written to the associated object's cursor.
- A temporary value is created, extracted, and compared against the reference.
- A zero result ends the scan after releasing that temporary.
- A nonzero result causes temporary release followed by advancing the fine-scan position by movement.

The final fine-scan position therefore identifies either a position whose comparison returned zero or a position outside the cached interval. It is not necessarily adjacent to the saved cursor or to the last coarse probe.

The function treats every nonzero comparator result identically. No ordering interpretation of positive versus negative comparator results is used.

### Restoration, destination selection, and result

After the fine scan stops, the function writes the saved cursor back through the receiver's then-current associated-object address.

It then obtains the indirect destination method and selects its index argument:

| Fine-scan stopping condition | Movement | Requested destination |
|---|---|---|
| Outside cached interval | Positive | Upper bound minus one |
| Outside cached interval | Negative | Lower bound |
| In range, comparison returned zero | Positive | Fine-scan position |
| In range, comparison returned zero | Negative | Fine-scan position plus one |

The function invokes the indirect method with the receiver and selected destination.

It then calls F-b3f4a7a52609 again, retains the resulting index, releases the initial eight-byte temporary through F-289ec4311c64, and returns whether the retained index differs from the saved cursor.

The final cleanup occurs **after** the final index query. Effects of that cleanup on object state therefore do not change the already-obtained index used for the return comparison.

## Side effects

### Cursor writes

The associated-object cursor can be written repeatedly during:

- The optional preceding-position adjustment.
- Every in-range coarse probe.
- Every in-range fine-scan probe.
- Restoration immediately before destination dispatch.

Restoration does not occur between probes. Dependencies called after a probe write can observe that probe's cursor.

The function reloads the receiver's associated-object address for later cursor accesses and lookups rather than retaining a guaranteed fixed associated-object identity. If a dependency changes that field, a subsequent write—including restoration—can affect a different associated object.

The normal path contains a restoration write, but this is not an unconditional promise that the original object or final object state is restored. Dependencies can change pointers or state, and no exceptional-exit cleanup is visible.

### Cached and refreshed state

The function retains:

- The saved cursor.
- The lower and upper bounds.
- The comparison reference, except for the explicit negative-movement replacement.
- The current search positions.

It does not reread the bounds after dependency calls. A dependency that changes the receiver's bounds does not thereby change the interval used by this invocation.

Lookup operations use the receiver's associated-object and auxiliary fields again and read the shared lookup token again. The supplied representation also places token reads before range checks in the search phases; an out-of-range stopping position does not imply that every shared-state read is skipped. No synchronization or volatile semantics are established.

### Temporary ownership and cleanup

Two eight-byte temporary storage areas are visible:

- One for the initial reference handle, retained across the search and destination call.
- One reused for preliminary and search probes.

Every visible normally completed probe construction is paired with F-289ec4311c64 before that storage is reused or before destination dispatch. The initial temporary is released once, after the final index query.

This establishes call pairing, not the underlying resource model. The function does not establish whether these dependencies allocate, retain, borrow, copy, or free memory, nor whether extraction returns an owned value or a borrowed address.

No explicit heap allocation or deallocation is otherwise visible.

### Errors and abnormal completion

There is no explicit error result other than the Boolean described above. Lookup failure, extraction failure, comparator failure, invalid dispatch, memory faults, and abnormal dependency exits have no visible local recovery path.

If a dependency fails to return normally, later releases and cursor restoration are not established.

## Dependencies

| Opaque function ID | Established use | Not established |
|---|---|---|
| F-5710a8628cba | Called with an associated-object address, a shared four-byte token, and a receiver-supplied four-byte argument. Returns the input used for temporary construction. | Lookup semantics, failure representation, ownership, global preservation. |
| F-7afe554bcf62 | Receives an eight-byte temporary destination and a lookup result. | Exact handle layout, allocation behavior, resource ownership. |
| F-9e5fecb2740f | Extracts a four-byte value from a temporary; called twice on the preliminary temporary when its comparison returns zero. | Value type, stability across calls, validity after temporary release. |
| F-b2840d22ea51 | Compares candidate-derived value first and reference second. Zero stops a search or accepts the preliminary adjustment; nonzero continues. | Whether zero means equality in a broader semantic sense; comparator side effects. |
| F-289ec4311c64 | Called to release or finalize each constructed temporary on normal paths. | Whether resources are freed and what other state changes occur. |
| F-b3f4a7a52609 | Queries an index before searching and again after destination dispatch. | Relationship to the associated-object cursor, range guarantees, side effects. |

The indirect destination method has no supplied opaque function ID. Its established identity is only its dispatch-table slot and observed argument shape.

No dependency is assumed to preserve globals, receiver fields, cursor state, or extracted-value lifetime.

## Acceptance examples

These examples are conditional static acceptance cases, not observed executions. Unless stated otherwise, dependencies return normally, relevant fields remain stable, arithmetic is representable, and comparisons return the specified results.

### Zero movement

With movement zero, the result must be `false`, with no receiver reads and no dependency calls. This remains true even if the receiver would be invalid to dereference.

### Positive movement with a coarse match

Assume bounds `[0, 100)`, initial search index 10, and movement 1.

If the comparison at 11 returns zero, coarse probing stops at 11. Fine scanning starts at 10. If comparison at 10 is nonzero and comparison at 11 is zero, the requested destination is 11.

The match at 11 can therefore be looked up twice. It does not bypass fine scanning.

### Coarse stride after a mismatch

With the same bounds, initial index, and movement, a nonzero comparison at coarse position 11 makes the next coarse candidate 43, not 12.

If 43 compares as zero, fine scanning starts at 11 and advances by one until it finds a zero comparison or leaves the interval.

### Movement magnitude greater than one

Assume bounds `[0, 200)`, initial index 10, and movement 2.

The first coarse candidate is 12. After a nonzero comparison there, the next candidate is 76. Fine scanning advances by two and need not inspect intervening odd indices.

### Negative preliminary adjustment

Assume bounds `[0, 20)`, initial index 8, and movement −1.

The preliminary lookup is at 7. If its comparison returns zero, extraction from that temporary occurs a second time, that result becomes the reference, and the anchor becomes 7. The first coarse candidate is then 6.

If the fine scan eventually stops at an in-range position 5 whose comparison is zero, the requested destination is 6.

### Interval escape

For bounds `[3, 9)`, an out-of-range fine-scan stop requests:

- 8 for positive movement.
- 3 for negative movement.

For empty bounds `[3, 3)`, the corresponding requests are 2 and 3. No special empty-interval rejection is visible.

### Return value is not match success

Assume the saved cursor is 7. If the destination request is 12 but the final index query returns 7, the function returns `false`.

Conversely, if the initial index query returned 7 but the saved cursor was 4, a final index-query result of 7 produces `true`, even though that queried index did not change.

## Uncertainties

- **documented_only — Domain meaning:** The grouping, navigation, and lookup semantics are unknown. Boundary navigation is inferred intent.
- **documented_only — Integer overflow:** Arithmetic includes subtraction by one, addition of movement, multiplication of movement by 32, and negative-direction destination adjustment by one. No overflow guards are visible. Signed wraparound, trapping, and other overflow behavior cannot be selected from the supplied representation alone.
- **documented_only — Termination:** Termination is not established for overflow cases or dependencies with arbitrary side effects.
- **documented_only — Extracted-value lifetime:** In particular, the negative preliminary adjustment can retain a value extracted from a subsequently released temporary.
- **documented_only — Dependency mutations:** Dependencies may modify globals, receiver fields, associated-object identity, or cursor state. No preservation contract was supplied.
- **documented_only — Failure and unwinding:** Cleanup and restoration after abnormal exits are unknown.
- **documented_only — Dispatch semantics:** The destination method may reject, clamp, transform, or otherwise handle the requested index. Only its invocation and the subsequent query are established.
- **documented_only — ABI and concurrency:** Calling convention, alignment guarantees, atomicity, synchronization, and precise machine behavior beyond the displayed four-byte representation are not independently verified.

## Evidence

The established behavior above derives solely from static inspection of the supplied function representation: its branches, field accesses, dependency calls, temporary cleanup calls, arithmetic, and indirect dispatch.

No dependency implementations or independent type definitions were supplied. Intent and lifetime interpretations are separated from the directly visible operations. No runtime tests have been performed.
