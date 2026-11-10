<!-- Trusted provenance: F-0ad008decd44; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ad008decd44

## Purpose

F-0ad008decd44 updates an object’s provider-backed selection state, conditionally invokes a refresh dependency, and then issues five virtual notifications.

The established behavior is a sequence of dependency calls, field updates, reference-count operations, selection validation, and notifications. Interpreting the provider’s entries as artwork is inferred intent, supported by the lookup label `GeniusMixArtwork`; the excerpt does not establish the meaning of the selected values or notifications.

The function attempts to preserve an existing selection when its index is eligible and its stored value passes a validity query. Otherwise, it selects entry zero when entries exist, or installs an empty-like value when no entries exist. The exposed entry count is capped at twelve.

## Interface

**Inputs**

- A pointer to a mutable destination object.
- A pointer to a four-byte provider-handle slot.

The visible representation uses four-byte integer and pointer-sized slots. The destination fields accessed directly are:

| Byte offset | Width | Role established by use |
|---|---:|---|
| `0x00` | 4 bytes | Virtual dispatch-table pointer |
| `0x3C` | 4 bytes | Provider entry count, subsequently capped |
| `0x40` | 4 bytes | Selection index; `−1` is treated as a sentinel |
| `0x44` | Not established | Beginning of the stored selection value |
| `0x1FC` | 4 bytes | Provider-handle slot |
| `0x200` | 4 bytes | Result of the named lookup |

The stored selection is operated on through dependencies. Its complete size, representation, and relationship to adjacent fields are not established.

The second input may point exactly to the provider-handle slot at destination offset `0x1FC`. That exact-address alias is explicitly handled. No general guarantee is established for other overlapping inputs.

**Output**

There is no explicit return value. Results are conveyed through destination state, provider reference-count changes, dependency side effects, and virtual notifications.

**Preconditions**

The destination must support all listed accesses and virtual calls. The source handle slot must be readable unless it is the explicitly handled self-alias. Any nonzero provider handle used by the visible reference-management sequence must support four-byte accesses at offsets `+4` and `+8`.

Provider resolution must yield an object supporting dispatch-table offsets `0x280` and `0x284` on paths that reach those calls. The destination must support dispatch-table offset `0x58`.

The function does not visibly validate pointers, dispatch tables, provider availability, or dependency success. A zero source handle is accepted by the copying sequence, but later provider resolution still occurs. Thus zero is not established as a safe provider value for the complete operation.

## Behavior

### 1. Perform the named lookup

The function first calls F-763286f35d31. Its result is passed to F-49d04c9bba7a together with the literal label `GeniusMixArtwork`.

The four-byte lookup result is written to destination offset `0x200`.

This happens before provider replacement or selection processing. Consequently, it also occurs when the second input is exactly the destination’s existing provider-handle slot.

The stored result is later tested for nonzero, but that later test reads the destination field again. It is not necessarily a test of the original lookup result if an intervening dependency changes that field.

### 2. Replace the provider handle unless the slots are identical

If the source slot and destination slot at offset `0x1FC` have the same address, the entire visible handle-replacement sequence is skipped.

Otherwise, the order is:

1. Call F-561858688f94 on the destination provider-handle slot.
2. Read the four-byte handle from the source slot.
3. Write that handle into the destination slot.
4. If the copied handle is nonzero, inspect its four-byte field at offset `+8`.
5. If that field is nonzero, call F-338d16f9ce72.
6. Increment the handle object’s four-byte field at offset `+4`.
7. Read the field at offset `+8` again.
8. If this second read is nonzero, call F-1333331b2da1.

The source handle is read **after** F-561858688f94 returns. It is not captured before that call.

The two tests of the field at `+8` are separate reads. The function does not guarantee that the two conditional dependencies are either both called or both omitted. Their pairing requires the relevant state to remain unchanged.

The increment is a four-byte operation with no visible saturation or overflow check. Its practical ownership meaning is inferred to be reference retention. The excerpt does not establish the counter’s valid range or a signed-language overflow contract.

F-561858688f94 may release or reset the old handle, but its actual ownership behavior is unknown here.

### 3. Query and cap the count

The function resolves the destination provider-handle slot through F-7d964f24a044 and invokes the resolved object’s virtual operation at dispatch-table offset `0x280`.

The resulting 32-bit value is written to destination offset `0x3C`.

The cap decision uses an **unsigned** comparison:

- Raw values `0` through `12` are retained.
- Every raw value from `13` through `0xFFFFFFFF` is replaced with `12`.

A raw result whose signed interpretation is negative therefore becomes twelve, not zero and not an error.

The count subsequently drives either empty-state handling or nonempty selection processing. The cap limits the indices this routine treats as eligible; it does not establish the provider’s actual number of entries.

### 4. Handle a zero count

For a raw count of zero, the function:

1. Writes `−1` to the selection-index field at offset `0x40`.
2. Calls F-1c06d0a5910b with temporary storage and two zero arguments.
3. Passes the returned pointer to F-047d22db5c73 as the source for updating the stored selection at offset `0x44`.
4. Calls F-c1466ea05103 on the temporary storage.

The constructor-like dependency’s returned pointer is used for assignment, whereas cleanup targets the supplied temporary storage. Their identity is not established by the excerpt.

This branch does not retrieve provider entry zero.

Calling the resulting value “empty” describes inferred intent. The established fact is construction with two zero arguments followed by assignment into the stored selection.

### 5. Decide whether to preserve an existing index

For a nonzero capped count, the function first reads the selection index.

If it equals `−1`, preservation is bypassed.

Otherwise, F-b7abcf60c990 is called on the stored selection. Its visible return value is 64 bits, but only the low 32 bits affect the preservation decision.

If those low 32 bits are zero, preservation is bypassed.

If they are nonzero, the function reads the selection index and count and tests whether the index is less than the count using a signed comparison. Although the recovered expression uses subtraction and overflow information, its relevant meaning is signed less-than, not a comparison against an arbitrarily wrapped subtraction result.

There is no explicit nonnegative-index check. A negative index other than `−1` can therefore reach the preservation path.

The initial `−1` test precedes F-b7abcf60c990. That dependency is not established to preserve destination state. Under mutation by that call, the index used for the subsequent comparison can differ from the one tested against `−1`.

### 6. Retrieve and reconcile a preserved selection

When the preservation condition succeeds:

1. Resolve the provider again through F-7d964f24a044.
2. Invoke its virtual operation at offset `0x284`, supplying temporary result storage, the resolved provider, and the selection index read at that point.
3. Call F-d3b5bb23607f on the stored selection.
4. Call F-d3b5bb23607f on the retrieved temporary.
5. Compare the two returned 64-bit values.

If the two 64-bit values differ, assign the retrieved temporary to the stored selection through F-047d22db5c73.

If the two values are equal:

1. Call F-961dd3c5e67e to produce a comparison representation of the stored selection.
2. Call F-961dd3c5e67e to produce a comparison representation of the retrieved value.
3. Call F-d45d65316ed3 with the retrieved representation first and the stored representation second.
4. Assign the retrieved value only if that dependency returns zero.

Thus a nonzero comparison result suppresses assignment when the 64-bit values match. The excerpt does not establish whether this result denotes equality, compatibility, or something else.

The retrieved temporary is then passed to F-c1466ea05103, regardless of whether assignment occurred.

This path does not directly rewrite the selection-index field. Dependencies may still change it. Provider resolution also occurs between the range decision and the read of the index supplied to entry retrieval; no snapshot consistency is established across those calls.

No explicit cleanup calls for the two comparison representations are visible. Their ownership cannot be inferred from that absence.

### 7. Fall back to entry zero

For a nonzero count, preservation is bypassed when:

- The initial index is `−1`.
- F-b7abcf60c990 returns zero in its low 32 bits.
- The signed index comparison fails.

The function then:

1. Writes zero to the selection-index field.
2. Resolves the provider again.
3. Retrieves an entry through virtual offset `0x284`, using the selection-index field read for that call.
4. Assigns the retrieved temporary to the stored selection.
5. Cleans up the temporary through F-c1466ea05103.

Without intervening mutation, the retrieved index is zero. Because provider resolution occurs after the zero write, an unknown side effect of that dependency could change the index before retrieval.

### 8. Conditionally refresh and issue notifications

After selection handling and temporary cleanup, the function reads destination offset `0x200`. If nonzero, it calls F-41552b023513 with the destination object.

It then performs five destination virtual calls through dispatch-table offset `0x58`.

For clarity, the fixed notification data locations are designated:

- First shared value, with first and second companion values.
- Second shared value, with third and fourth companion values.
- Final value, with fifth companion value.

The call order is:

1. First shared value and first companion.
2. A previously captured first shared value and second companion.
3. Second shared value and third companion.
4. A previously captured second shared value and fourth companion.
5. Final value and fifth companion.

The function captures the first shared value before the first call. The first call itself uses the corresponding global value as an argument; the second call uses the captured value. The second shared value follows the same pattern for calls three and four.

Consequently, a notification callback that changes a shared global does not change the already captured value used by the following paired call. Later companion values are not visibly captured at the same earlier point.

The destination dispatch table is consulted for each notification. The five calls are sequential, not an established atomic batch.

## Side effects

Direct destination writes include:

- The named lookup result at `0x200`.
- The provider handle at `0x1FC`, except for exact slot self-alias.
- The count at `0x3C`, including a possible second write imposing the cap.
- The index at `0x40` on empty and fallback paths.

Stored-selection changes occur through F-047d22db5c73. The function may leave that selection unassigned on the successful preservation path.

For a copied nonzero provider handle, a four-byte field at handle offset `+4` is incremented.

Additional effects may arise from every dependency and virtual call. No callee is established to preserve globals, destination fields, handle state, or memory ownership.

There is no visible rollback, error return, retry, or failure translation. Normal continuation after a dependency call assumes that it returns. Exceptions, traps, allocation failures, and nonlocal exits are documented_only.

## Dependencies

| Opaque ID or dispatch slot | Visible use |
|---|---|
| F-763286f35d31 | Supplies the first argument for the named lookup |
| F-49d04c9bba7a | Performs the lookup using `GeniusMixArtwork` |
| F-561858688f94 | Operates on the old destination provider slot before replacement |
| F-338d16f9ce72 | Conditionally called before the counter increment |
| F-1333331b2da1 | Conditionally called after the counter increment |
| F-7d964f24a044 | Resolves the destination provider slot |
| Provider virtual `0x280` | Returns the 32-bit count |
| Provider virtual `0x284` | Produces an indexed temporary value |
| F-b7abcf60c990 | Supplies the low-word selection-validity gate |
| F-d3b5bb23607f | Produces a 64-bit selection comparison value |
| F-961dd3c5e67e | Produces comparison representations |
| F-d45d65316ed3 | Compares retrieved and stored representations |
| F-047d22db5c73 | Updates the stored selection |
| F-1c06d0a5910b | Constructs the zero-argument-pair temporary |
| F-c1466ea05103 | Cleans up the selected temporary |
| F-41552b023513 | Conditional post-selection operation |
| Destination virtual `0x58` | Receives five ordered notifications |

These are call-site contracts only. In particular, assignment does not establish deep copying, cleanup does not establish deallocation, and provider-slot replacement does not establish complete ownership transfer.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, dependencies return normally and do not unexpectedly mutate state relevant to the example.

1. **Exact provider-slot alias.** If the source slot is the destination slot at `0x1FC`, the lookup still runs, but F-561858688f94 and the visible counter increment are skipped. Count and selection processing continue.

2. **Different slots containing the same handle.** Equal stored handle values do not trigger the alias shortcut. The old-slot operation occurs before the source handle is read, followed by the nonzero-handle increment sequence. Safety requires compatible dependency and ownership behavior.

3. **Unsigned count cap.** Counts of `13`, `100`, and `0xFFFFFFFF` all yield a stored count of twelve. A raw count of twelve remains twelve.

4. **No entries.** A zero count sets the index to `−1`, assigns the value constructed with two zero arguments, and cleans up its temporary. No indexed retrieval occurs.

5. **Sentinel fallback.** With a positive count and initial index `−1`, the validity query is skipped. The index becomes zero and entry zero is retrieved, assigned, and cleaned up.

6. **Validity gate fallback.** With count eight, index three, and a validity result whose low 32 bits are zero, entry zero is selected. Nonzero upper 32 bits do not enable preservation.

7. **Preserved index, changed 64-bit value.** With count eight, index three, and a nonzero validity low word, entry three is retrieved. If its 64-bit comparison value differs, assignment occurs without the secondary comparison.

8. **Preserved value without assignment.** Under the same eligible index, equal 64-bit values lead to the secondary comparison. A nonzero result suppresses assignment; cleanup still occurs.

9. **Negative non-sentinel index.** With count eight, index `−2`, and a nonzero validity low word, the signed range test succeeds. Retrieval receives `−2`. Acceptance by the provider is not established.

10. **Paired notification capture.** If the first notification changes the first shared global, the second notification still receives the value captured before the first call. All five notifications follow selection cleanup and any conditional refresh.

## Uncertainties

**Established static evidence:** field offsets and four-byte operations; unsigned count capping; signed index comparison; dependency ordering; assignment conditions; explicit temporary cleanup; five ordered notifications.

**Inferred intent:** provider ownership management, reference counting, selection validity, empty-value construction, artwork-related state, and refresh semantics.

**Documented_only unknowns:**

- Complete object layouts, endianness, and calling convention.
- The count operation’s exact receiver-passing convention.
- Dependency ownership, allocation, synchronization, and mutation behavior.
- Whether negative indices are supported by the provider.
- The semantic meaning of both comparison stages.
- Whether comparison representations require cleanup.
- Notification argument values and meanings.
- Thread safety, reentrancy, and consistency under callback mutation.
- Behavior for invalid pointers, failed resolution, counter overflow, and dependency failure.

## Evidence

This contract derives solely from the supplied static function excerpt and the supplied opaque dependency mapping. It distinguishes visible operations from inferred semantics and does not incorporate dependency implementations.

No runtime tests, execution traces, or independent implementation inspection have been performed. The acceptance examples specify conditional behavior; they are not verification results.
