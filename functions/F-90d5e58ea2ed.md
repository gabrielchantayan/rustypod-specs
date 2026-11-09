<!-- Trusted provenance: F-90d5e58ea2ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-90d5e58ea2ed

## Purpose

F-90d5e58ea2ed performs an ordered, teardown-like transition on a supplied object. On ordinary completion, it returns the original object address.

Established behavior includes five global-sourced value reads, four staged writes to the object’s first word, state checks, conditional relationship operations, associated-record processing, traversal, and subordinate-object dispatch.

A destructor or finalizer role is **inferred intent**, not an established ownership contract. The visible operations do not establish that the input is freed, that dispatched operations destroy their targets, or that the returned address still identifies a usable object.

## Interface

### Input and output

The function accepts one object address. The supplied representation uses 32-bit words, including address-like values.

On ordinary completion, the result is the original input address. No separate success status is returned. Three failure routes end in F-feee9156a257, which the supplied representation marks as non-returning.

The input is not checked for null before use. Execution begins by reading a global-sourced value and writing it to the input’s first word.

### Referenced storage

All offsets below are byte offsets. These are partial layouts, not complete type definitions.

| Storage | Offset | Width | Observed role |
|---|---:|---:|---|
| Input object | `0x00` | 32 bits | Receives four staged dispatch-table-like values. |
| Input object | `0x04` | 32 bits | Optional subordinate-object address. |
| Input object | `0x08` | 32 bits | State checked within the final subordinate branch. |
| Input object | `0x0C` | 32 bits | Optional associated-record address. |
| Input object | `0xB4` | 32 bits | Second initial state guard. |
| Input object | `0xB8` | 32 bits | First initial state guard. |
| Associated record | `0x04` | 32 bits | Value retained for later calls. |
| Associated record | `0x08` | 32 bits | Optional collection-like object address. |
| Context | `0x1C` | 32 bits | First argument source for F-77781c8e0ff9. |
| Context | `0x24` | 32 bits | Reference conditionally cleared. |
| Context | `0x28` | 32 bits | Reference used for dispatch, argument passing, and equality checks. |

Indirect calls select entries at dispatch-table byte offsets `0x80`, `0x7C`, `0x48`, and `0x04`. The representation uses 32-bit address-like words to reach these tables.

Five global-sourced reads each supply a 32-bit value: four dispatch-table-like values and one notification argument. Exact values and table identities are **documented_only**. These reads are separate observations at different execution stages; the values are not established as immutable.

### Preconditions and validity

The input must permit every read and write reached on the selected path, including the initial write and initial guards. Every dereferenced record, context, dispatch table, and dispatch target must be valid when used. An earlier nonzero check does not guarantee validity after an intervening dependency call.

No general pointer validation, bounds checking, or alignment checking is visible. The complete ABI and alignment requirements are **documented_only**.

The iterator receives a 20-byte local state area and an address-sized output location. Their lifetime is limited to this invocation; any dependency retention of their addresses is not established.

## Behavior

### Global-read ordering

The five global-sourced reads occur at these points:

| Read | Position in the observable sequence | Use |
|---|---|---|
| Entry dispatch value | Before the first guard, immediately preceding the initial object-word assignment | Written to input offset `0x00`. |
| Second dispatch value | After both initial guards pass, before F-d277584d672d | Written to input offset `0x00`. |
| Third dispatch value | After F-a6fdc7050c77 returns, before F-408b56b8b7c8 | Written to input offset `0x00`. |
| Notification argument | After the first F-3fdd14ecd250 returns, before F-77781c8e0ff9 is called | Passed as that call’s second argument. |
| Final dispatch value | After associated-record processing finishes, or after that branch is skipped | Written to input offset `0x00` before the final subordinate check. |

No caching of all five values at entry is established. Intervening dependencies may change global storage. The contract therefore uses the value observed at each listed read, not a presumed startup value or constant.

For notification, both the context-field read and the global-sourced argument read occur after the context query and before the notification call. Their relative order within argument preparation is not established here.

### Entry transition and guards

The entry dispatch value is read and written to input offset `0x00`. The function then reads offset `0xB8`.

If this first guard is nonzero, F-163ea045cd81 is called, followed by F-feee9156a257. Under the supplied non-returning annotation, execution does not continue. Offset `0xB4` is not checked on this path.

If the first guard is zero, offset `0xB4` is read. A nonzero second guard produces the same two calls in the same order.

Both failure routes occur after the initial object-word write. Neither reaches the second global-sourced dispatch read.

### Relationship handling

When both guards pass, the second dispatch value is read and written to offset `0x00`. F-d277584d672d is then called with the input address.

A null result skips relationship dispatch. For a nonnull result:

- Its dispatch entry at byte offset `0x80` is invoked with that object explicitly supplied.
- The returned address is compared with the original input address.
- Equality causes an invocation of the same retained object’s entry at byte offset `0x7C`, explicitly passing that object and zero.
- Inequality skips the `0x7C` invocation.

The comparison is exact address equality. No type or content comparison is visible. The dispatch table used for the second invocation is selected after the first invocation; preservation of the object or its table is not guaranteed by this function.

F-a6fdc7050c77 is then called with the input address, zero, and one, regardless of the relationship branch.

### Intermediate transition and notification

After F-a6fdc7050c77 returns, the third dispatch value is read from its global source and written to input offset `0x00`. F-408b56b8b7c8 is called with the input address.

The function next calls F-3fdd14ecd250. After that query returns, it prepares a notification using:

1. The returned context’s 32-bit field at offset `0x1C`.
2. A 32-bit value read from the notification argument’s global source at this stage.
3. The original input address.

These arguments are passed to F-77781c8e0ff9. The second argument is not established as fixed, immutable, or preserved by the preceding context query.

Only after notification returns does the function read input offset `0x0C` to select an associated record.

### Associated-record processing

A zero associated-record field skips this entire stage. Otherwise, its observed address is retained, and the 32-bit value at that record’s offset `0x04` is saved before F-a1a48bd3fbde is called with the input address.

Later traversal and record completion use the retained record address. They do not automatically follow subsequent changes to input offset `0x0C`.

#### Conditional active-state operations

If F-a1a48bd3fbde returns nonzero, a context is obtained and its offset-`0x28` field is checked.

If that field is nonzero, another context is obtained. The second context’s offset-`0x28` reference selects an indirect dispatch entry at byte offset `0x48`, which is invoked. No explicit arguments are shown for that invocation.

The second context’s reference is not separately checked for null. Validity of the first observed reference does not establish validity of the second.

After the invocation:

- Input offset `0x0C` is reread.
- Offset `0x04` of that newly referenced record is read.
- Another context is obtained.
- F-a5514b9f24f7 receives that context’s current offset-`0x28` value and the freshly read record value.

This fresh record value need not equal the value saved from the retained record. There is no new null check on the associated-record address before the fresh dereference.

Whether or not the context-reference subbranch runs, a nonzero result from F-a1a48bd3fbde is followed by F-65528ded3450 with the input and zero, then F-a82ec4b43e2f with the originally saved record value.

A zero result skips all operations in this subsection.

#### Collection-like traversal

The retained record’s offset-`0x08` field is checked. If nonzero, that field supplies the collection-like argument to F-538044b67754, together with the local 20-byte state address and the 32-bit bit pattern `0xFFFFFFFD`.

The pattern has unsigned value 4,294,967,293 and signed two’s-complement interpretation −3. Its semantic meaning is unknown. No local arithmetic transforms it.

The field is read for the branch check and again for the initialization argument. There is no intervening explicit dependency call between those observations.

F-ebd94610a438 is repeatedly called with the iteration-state address and an output-address location:

- Zero ends iteration.
- Nonzero causes the output location to be examined.
- A null output skips element dispatch but does not end iteration.
- A nonnull output selects an indirect entry at byte offset `0x04`, which is invoked with no explicit arguments shown.

On normal loop termination, F-944b595e25e9 receives the iteration-state address.

The retained record’s offset-`0x08` field is then checked again. If nonzero, its current reference is used to invoke a dispatch entry at byte offset `0x04`. This target need not be the object supplied to iterator initialization.

No duplicate suppression, explicit iteration bound, or termination guarantee is visible. No iterator finalization is established for abnormal exits from dependencies.

#### Context references and record completion

After traversal, or after traversal is skipped:

1. A context is obtained and its offset-`0x28` reference is compared with the input address.
2. Equality causes F-a82ec4b43e2f to receive the originally saved record value.
3. Another context is obtained and its offset-`0x24` reference is compared with the input address.
4. Equality causes another context query, followed by a zero write to that returned context’s offset `0x24`.
5. F-65528ded3450 receives the input address and zero.
6. F-8349c252c168 receives the retained record address and zero.

The context whose offset `0x24` is cleared need not be the context whose field matched the input.

F-a82ec4b43e2f can consequently receive the saved value twice. F-65528ded3450 can likewise receive the input and zero twice.

### Final transition and subordinate processing

After associated-record processing returns, or after its branch is skipped, the final dispatch value is read from its global source and written to input offset `0x00`.

The function then checks input offset `0x04`.

If zero, it returns the input address without checking offset `0x08`.

If nonzero, F-c719314a16bb is called with the input. Offset `0x04` is then reread. A nonzero current reference selects an indirect entry at byte offset `0x04`, invoked with no explicit arguments shown.

The function next checks input offset `0x08`, even if F-c719314a16bb cleared the subordinate reference. A nonzero state causes F-c8eefdf96f4b, followed by the non-returning F-feee9156a257. Otherwise, the original input address is returned.

## Side effects

Direct writes to external storage are:

- Up to four assignments to input offset `0x00`, each using its stage-specific global read.
- A conditional zero write to a separately obtained context’s offset `0x24`.

The function does not directly clear input offsets `0x04`, `0x08`, or `0x0C`, or the retained record’s collection field. Dependencies may change these fields, global sources, dispatch tables, or other state; their preservation is not established.

The final dispatch assignment is the last direct write to the input’s first word by this function. Subsequent dependencies may overwrite it, so its value at return is not guaranteed solely by that assignment.

Ownership, release, and destruction effects are **documented_only**. In particular:

- Returning the input address does not prove continued lifetime.
- Repeated saved-value calls do not prove reference counting.
- Element dispatch does not prove collection ownership.
- F-8349c252c168 does not establish whether the retained record is freed.
- Passing local-state addresses does not establish whether dependencies retain them.

No rollback, synchronization, or guaranteed exception cleanup is visible.

## Dependencies

Identifiers are opaque; the roles below describe usage, not implementation.

| Dependency | Observable role |
|---|---|
| F-163ea045cd81 | Called before either initial guard’s non-returning failure call. |
| F-feee9156a257 | Annotated non-returning on all three failure routes. |
| F-d277584d672d | Supplies the optional relationship-dispatch object. |
| F-a6fdc7050c77 | Receives input, zero, and one before the third global read. |
| F-408b56b8b7c8 | Receives input after the third dispatch assignment. |
| F-3fdd14ecd250 | Supplies contexts through repeated, independent queries. |
| F-77781c8e0ff9 | Receives a context field, a newly read global-sourced argument, and input. |
| F-a1a48bd3fbde | Controls active-state operations. |
| F-a5514b9f24f7 | Receives a current context reference and freshly read record value. |
| F-65528ded3450 | Receives input and zero, once or twice within record processing. |
| F-a82ec4b43e2f | Receives the retained record value under two separate conditions. |
| F-538044b67754 | Initializes local iteration state. |
| F-ebd94610a438 | Supplies loop continuation status and an output address. |
| F-944b595e25e9 | Finalizes iteration state after normal loop termination. |
| F-8349c252c168 | Receives the retained record address and zero. |
| F-c719314a16bb | Runs between the final subordinate check and reread. |
| F-c8eefdf96f4b | Precedes the late non-returning failure call. |

Indirect dependencies are selected at byte offsets `0x80`, `0x7C`, `0x48`, and `0x04`. Full signatures and receiver-passing conventions are **documented_only** where explicit arguments are absent.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes valid reached storage and normally returning dependencies except where non-returning behavior is specified.

1. **First guard fails.** With offset `0xB8` nonzero, the entry global is read and its value written before F-163ea045cd81 and F-feee9156a257. Neither the second guard nor later global reads are reached.

2. **Second guard fails.** With offset `0xB8` zero and offset `0xB4` nonzero, the same failure calls occur after the second check. Only the entry dispatch assignment has occurred.

3. **Global changes during a dependency.** If F-a6fdc7050c77 changes the third dispatch source, the following assignment uses the value read after it returns, not a value assumed captured at entry.

4. **Notification source changes during context acquisition.** If the first F-3fdd14ecd250 changes the notification argument’s global source, F-77781c8e0ff9 receives the value subsequently read from that source.

5. **Final source changes during record processing.** If record processing changes the final dispatch source, the final assignment uses the value read after that processing finishes.

6. **Optional branches absent.** If the initial guards pass and offsets `0x0C` and `0x04` are zero when checked, record and subordinate processing are skipped. All five global-sourced reads are still reached, and the input address is returned.

7. **Relationship mismatch.** A nonnull relationship object whose `0x80` entry returns an address different from input does not receive the conditional `0x7C` invocation.

8. **Null iteration output.** A nonzero iterator result with a null output causes no element dispatch; iteration continues.

9. **Collection replacement.** If the retained record’s collection reference is replaced before the post-finalization check, that check and dispatch use the current field, not the initializer’s argument.

10. **Both saved-value conditions hold.** A nonzero active-state result and a later context offset-`0x28` equality cause two calls to F-a82ec4b43e2f with the original saved value.

11. **Subordinate cleared.** If F-c719314a16bb clears offset `0x04`, subordinate dispatch is skipped, but offset `0x08` is still checked and may cause late failure.

12. **State without subordinate entry.** If offset `0x04` is zero at the final branch check, nonzero offset `0x08` alone does not trigger late failure.

## Uncertainties

The following remain **documented_only**:

- Exact global values, dispatch-table identities, and global mutability.
- Whether intervening dependencies preserve any global source.
- Complete layouts, ABI, alignment, and indirect-call signatures.
- Whether repeated context queries identify the same storage.
- Dependency mutation, ownership, allocation, and release semantics.
- Iteration order, completeness, mutation rules, and termination.
- The mechanism behind annotated non-returning calls.
- Concurrency, reentrancy, and external mutation constraints.

Saved-register bookkeeping does not establish an additional public input or output. No arithmetic, truncation, or sign-dependent branch beyond the stated 32-bit comparisons and iterator argument is established.

## Evidence

The evidence is the supplied static function representation and dependency-ID mapping. It establishes the five global-sourced reads and their stage ordering, field widths and offsets, conditional calls, retained values, rereads, direct writes, and return expression.

Teardown intent, collection semantics, and release behavior remain interpretations. Non-returning behavior relies on supplied annotations. No dependency implementation inspection, runtime execution, or tests have been performed.
