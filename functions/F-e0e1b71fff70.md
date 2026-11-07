<!-- Trusted provenance: F-e0e1b71fff70; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e0e1b71fff70

## Purpose

F-e0e1b71fff70 searches a linked collection for a record matching two 32-bit values. It distinguishes an existing match from a newly assigned record through its return value:

- **1:** an existing matching record was found.
- **0:** no existing match was found, whether a replacement record was assigned or no usable record existed.

The output slot receives a record pointer when a record is selected, or a zero word on the explicit no-capacity path.

The visible selection policy prefers the last encountered record whose first key word is zero. If no such record exists, it may reuse an occupied record with a zero usage count, choosing the smallest eligible unsigned timestamp.

This resembles a reference-counted cache or registry with an age-based replacement policy. That interpretation is inferred intent; the observable contract is the lookup, selection, writes, calls, and return behavior described below.

## Interface

### Inputs

The function accepts three arguments:

1. **Output-slot address.** An address at which the function writes one 32-bit word containing either a record pointer or zero.
2. **First key.** A 32-bit value passed to the initial lookup dependency and compared with the record’s first key word.
3. **Second key.** A 32-bit value compared with the record’s second key word.

The two keys are tested for equality, not ordered numerically. Their signed interpretation therefore does not affect ordinary matching.

The output-slot address must identify writable storage whenever execution reaches an explicit output write. No null check, size check, or alignment check is visible.

### Result

The result is a 32-bit value, explicitly set to either 0 or 1 on every visible normal-return path.

A result of 0 is **not sufficient to establish failure**. The output word distinguishes a miss that assigned a record from the explicit no-capacity outcome.

### Record layout

Each traversed record is accessed as a sequence of 32-bit words:

| Byte offset | Visible use |
|---|---|
| 0 | Link to the next record; zero terminates traversal |
| 4 | Timestamp or generation word |
| 8 | First key; zero also marks a record as free |
| 12 | Second key |
| 16 | Usage or reference-count word |

The pointer-bearing fields are represented as single 32-bit words in the supplied representation. No additional record fields are inspected or modified by this function.

Records must be readable at the offsets used by their particular path. Selected records must also be writable at the modified offsets. The traversal performs no structural validation.

### Shared state

The function accesses a shared 32-bit counter through a shared pointer. It:

- Reads the shared pointer after calling F-4c6d30ac2311.
- Retains that pointer for a possible later counter read on a hit.
- Reads the counter and writes its incremented value before beginning traversal.
- Retains the pre-increment value for computing the timestamp assigned on a miss.

No validation of the shared pointer is visible.

## Behavior

### Entry and operation ordering

The visible sequence begins with F-f48e102f05c9, receiving the first key. Its result supplies the initial record pointer.

Only after that call returns does the function call F-4c6d30ac2311. It then captures the shared-counter pointer, reads the counter, and stores its incremented value.

Consequently:

- Obtaining the initial record pointer precedes F-4c6d30ac2311.
- Counter advancement occurs even if the initial record pointer is zero.
- Counter advancement occurs on hits, successful assignments, and the explicit no-capacity path.
- Traversal begins after the counter write.

F-4c6d30ac2311 and F-cc60a532ba6c appear positioned like entry and exit synchronization operations, but their actual behavior is not established.

### Traversal and matching

Traversal follows the link at offset 0 until a matching record is found or the current pointer becomes zero.

For each nonzero current pointer, the function first reads the first key at offset 8.

If that word is zero, the record is treated as free. It is not tested for a match, even when the requested first key is also zero. The record becomes the preferred free candidate, replacing any earlier free candidate.

If the first key is nonzero:

- It is compared with the requested first key.
- The second key is read for matching only when the first keys are equal.
- Equality of both keys ends traversal immediately.

A later duplicate match is never considered after an earlier match has been accepted.

For an occupied record that does not match, the function considers replacement eligibility:

- The usage word at offset 16 must equal zero.
- The timestamp at offset 4, interpreted as an unsigned 32-bit value, must be strictly less than the current replacement threshold.

The threshold begins at 4,294,967,295. Therefore an occupied record with timestamp 4,294,967,295 is never selected by this replacement comparison, even if its usage word is zero and no other candidate exists.

Whenever an occupied candidate qualifies, its timestamp becomes the new threshold. Equal timestamps do not replace the existing candidate, so the first encountered record wins a tie.

After processing a nonmatching record, traversal reads its link at offset 0 and proceeds.

### Existing-match path

On finding a matching occupied record, the function performs these explicit operations in order:

1. Writes the matching record pointer to the output slot.
2. Reads the counter through the shared-counter pointer retained at entry and writes that value to the record’s timestamp.
3. Reads the record’s usage word, increments it, and writes the result back.
4. Calls F-cc60a532ba6c.
5. Returns 1 if that dependency returns normally.

The hit timestamp uses a fresh counter read. It is not necessarily the value written by the entry increment. Equality holds only if nothing has changed the counter in the intervening execution.

A matching record with usage zero is still a hit. Its usage word is incremented; it is not reassigned through the miss path.

The record’s keys and link are not explicitly rewritten on this path.

### Miss with an available record

When traversal reaches a zero pointer, selection follows this priority:

1. The last free record encountered, if any.
2. Otherwise, the occupied replacement candidate with the smallest eligible unsigned timestamp.
3. Otherwise, the no-capacity path.

Free records always take precedence over occupied replacement candidates, regardless of their timestamps or usage words. The function does not inspect those fields to determine whether a zero-key record is free.

For the selected record, the function explicitly writes, in order:

1. Offset 4: the entry counter’s original value plus one.
2. Offset 8: the requested first key.
3. Offset 12: the requested second key.
4. Offset 16: the value 1.

It then calls F-cc60a532ba6c, writes the selected record pointer to the output slot, and returns 0.

The link at offset 0 is not explicitly changed. No new record is allocated by the visible body.

Unlike a hit, the miss path writes the output pointer **after** F-cc60a532ba6c returns.

### Miss without an available record

If neither a free record nor an eligible occupied replacement candidate exists, the function:

1. Calls F-cc60a532ba6c.
2. Calls F-3ba698310ee1 with the numeric argument 6.
3. Writes zero to the output slot.
4. Returns 0.

This ordering is significant. The output slot is not explicitly cleared before either dependency call. If a dependency does not return normally, the later output write and normal return are not established.

The meaning of the argument 6 is unknown. It is an observed argument, not independently established as an allocation error, capacity error, or any particular external error identifier.

### Zero first key

Zero has two simultaneous roles: it is an accepted input value and the record-free marker.

A request whose first key is zero cannot hit any record under the visible matching rules. A selected record can nevertheless be assigned that key and usage 1. On a later traversal, that record is again treated as free because its first key remains zero.

There is no visible rejection or normalization of this input.

### Arithmetic and ordering boundaries

Timestamps are ordered using unsigned 32-bit comparisons. This is a numeric minimum policy, not a demonstrated wrap-aware age comparison.

The counter increment and usage increment have no visible saturation, overflow check, or error branch. For values whose increment is representable under the displayed signed-counter interpretation, the result is unambiguous.

Modulo-2³² behavior is consistent with the apparent word operations, but the supplied representation alone does not independently establish every overflow case for the signed counter expression. Overflow-dependent behavior remains documented_only.

The usage word is incremented without an eligibility check on the hit path. A maximum-value usage word is not rejected before that operation.

## Side effects

### Explicit writes

Subject to reaching the relevant operations, the function explicitly changes:

- The shared counter, once near entry.
- On a hit: the output slot, selected timestamp, and selected usage word.
- On assignment: the selected timestamp, both key words, usage word, and output slot.
- On no capacity: the output slot to zero.

It does not explicitly change any link word.

These are statements about the function’s own accesses. Dependencies may have additional effects that are not known.

### Ownership and lifetime

The visible body contains no direct allocation or deallocation operation. It selects pointers obtained through the initial dependency and the existing links.

It does not establish whether the returned pointer is borrowed, owned, pinned, or otherwise protected. The usage increment and initialization to 1 suggest a reference-acquisition convention, but no release operation or lifetime rule is supplied.

In particular, F-cc60a532ba6c must not be assumed to preserve the selected record, its contents, the output slot, or shared state. The body uses the selected pointer after that call on the assignment path, but this usage does not itself prove the dependency’s lifetime guarantees.

### Aliasing and concurrent mutation

The ordinary field descriptions assume that the output slot, counter, and record fields do not overlap in ways that interfere with one another. The function does not enforce that separation.

Aliasing can change observable behavior because write order differs between paths. For example, the hit path writes the output slot before reading the counter and updating the record’s remaining fields.

No concurrency guarantee can be derived merely from the placement of the two apparent synchronization dependencies. If records or the counter change during traversal, later reads may observe those changes. A stable, valid list is needed for the straightforward sequential interpretation above.

A cycle without a matching record can cause traversal not to terminate. No cycle detection or traversal limit is visible.

## Dependencies

| Opaque function ID | Observed role |
|---|---|
| F-f48e102f05c9 | Called first with the first key. Its returned pointer starts traversal. |
| F-4c6d30ac2311 | Called after obtaining the initial pointer and before accessing the shared counter. Its return value is not used. |
| F-cc60a532ba6c | Called once on each visible terminating path, with path-specific ordering relative to record and output writes. Its return value is not used. |
| F-3ba698310ee1 | Called with 6 only when no record can be selected, after F-cc60a532ba6c and before clearing the output slot. Its return value is not used. |

No contracts for these dependencies are supplied. Their effects on globals, storage ownership, synchronization, error state, and control flow are unknown.

## Acceptance examples

These examples are conditional on valid, noninterfering storage; stable traversal data; normally returning dependencies; and arithmetic that does not encounter an unresolved overflow boundary. They describe explicit function behavior, not runtime observations.

1. **Existing match**

   The entry counter is 40. A record matches both keys and has usage 2. Nothing changes the counter after the function advances it.

   The function writes the record pointer to the output slot, sets its timestamp to 41, changes usage to 3, calls F-cc60a532ba6c, and returns 1.

2. **Last free record wins**

   Traversal encounters a free record, then an occupied replacement candidate, then another free record. No match exists.

   The second free record is assigned the requested keys, timestamp, and usage 1. The earlier free record and occupied candidate are not selected.

3. **Unsigned minimum and stable ties**

   With no free records or match, three zero-usage occupied records have timestamps 12, 7, and 7.

   The first record with timestamp 7 is selected. The later equal timestamp does not displace it.

4. **Maximum timestamp is excluded**

   The only record is occupied, does not match, has usage zero, and has timestamp 4,294,967,295.

   No replacement candidate is selected. The function calls F-cc60a532ba6c, then F-3ba698310ee1 with 6, clears the output slot, and returns 0.

5. **Empty collection**

   F-f48e102f05c9 returns zero.

   The function still calls F-4c6d30ac2311 and advances the counter before taking the no-capacity path.

6. **Zero-key request**

   The requested first key is zero and a free record is available.

   The function assigns that record and returns 0 with its pointer in the output slot. A later identical request cannot recognize it as a hit through this function’s matching rule.

7. **Hit overrides replacement availability**

   A free record appears before a later matching occupied record.

   The matching record is returned with result 1. The free candidate is not assigned.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the collection, key values, timestamps, and usage count.
- The dependencies’ synchronization and memory-lifetime guarantees.
- The meaning and externally observable effects of the argument 6.
- Whether dependencies return normally under all inputs.
- The intended legality of zero as a first key.
- Exact counter behavior at signed-overflow boundaries.
- Record ownership and any required release protocol.
- Guarantees under aliasing, concurrent mutation, malformed links, or invalid storage.
- Any allocation, release, or additional state changes performed inside dependencies.

The apparent age-based replacement and reference-acquisition roles are inferred intent, not substitutes for these missing contracts.

## Evidence

Established static evidence consists solely of the supplied function representation: its 32-bit word accesses, branch conditions, unsigned timestamp comparison, dependency calls, explicit write order, and return constants.

The record layout and selection rules follow from those accesses. Cache, reference-count, and synchronization terminology describes plausible intent and is qualified accordingly.

No implementation outside the supplied body was inspected. No runtime tests have been performed.
