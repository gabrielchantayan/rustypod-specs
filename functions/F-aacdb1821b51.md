<!-- Trusted provenance: F-aacdb1821b51; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aacdb1821b51

## Purpose

F-aacdb1821b51 performs a staged maintenance operation on a mutable object containing a table of 24-byte records.

Established behavior includes:

- An optional initialization call when the supplied starting index is zero.
- A preliminary scan that may invoke a dependency for records with a nonzero field at offset 20.
- A special early-return path for a positive starting index.
- Clearing one object flag bit.
- Invoking callbacks and clearing a record-state field for records whose field at offset 4 is zero.
- Processing records from index 2 onward, retaining records with a nonzero field at offset 4 and passing the first word of other records to a dependency.
- Requesting external operations over moved records and the unused table suffix.
- Publishing a revised record count.
- Potentially transferring the first two records into object-embedded storage and passing the former table address to a dependency.

The apparent intent is removal of inactive records, stable compaction of retained records, and migration to embedded storage when only two reserved records remain. That intent is inferred. In particular, the supplied evidence does not establish that the external operations actually copy or clear memory, or that the dependency receiving addresses releases allocations.

## Interface

### Inputs and result

The function accepts two 32-bit inputs:

| Input | Interpretation |
|---|---|
| Object address | Address of a mutable object and its associated table. |
| Starting index | Signed 32-bit index controlling the preliminary scan and early-return behavior. |

There is no defined return value in the supplied interface. Completion is observable through memory changes, dependency calls, callbacks, and return timing.

The source representation uses 32-bit words for object fields, record fields, addresses, and callback addresses. No 64-bit widening is established.

### Object layout

The following offsets are relative to the object address:

| Offset | Width | Established use |
|---|---:|---|
| 4 | 4 bytes | Record count, read using signed comparisons. |
| 8 | 4 bytes | Address of the current record table. |
| 12 | 4 bytes | Flag word. Bit value 16 is cleared during full maintenance. |
| 332 | At least 48 bytes when used | Embedded storage destination for two records. |

Other object fields are not characterized.

The embedded address is the object address plus 332. The function tests address equality against this exact location; it does not test whether the table points anywhere within an embedded-storage range.

### Record layout

Records have a stride of 24 bytes:

| Offset within record | Width | Established use |
|---|---:|---|
| 0 | 4 bytes | Value passed to F-94cb53877c15 for records removed during compaction; subsequently overwritten with zero. |
| 4 | 4 bytes | Retention discriminator. Zero selects cleanup/removal behavior; any nonzero value selects retention. |
| 8 | 4 bytes | No field-specific access established; included in whole-record transfer requests. |
| 12 | 4 bytes | Callback eligibility/state word; cleared for records whose discriminator is zero. |
| 16 | 4 bytes | Callback address, read only when the state word is nonzero. |
| 20 | 4 bytes | Preliminary-scan trigger; any nonzero value causes a dependency call. |

No narrower interpretation of these words is established. In particular, discriminator values are not restricted to zero and one.

### Preconditions and arithmetic domain

The function performs no explicit validation of the object address, table address, count, index, callback address, or ownership state.

For ordinary defined execution, callers must provide accessible storage for every reached read and write, callable nonzero callback addresses, and arithmetic that produces valid addresses and dependency arguments.

A count of at least two is an inferred structural invariant, not a checked requirement. The compaction destination index starts at two even when the recorded count is smaller.

Negative starting indices are accepted by the control flow. If a negative index is less than the current count, the function accesses a record before the table base. Such use requires an independently valid memory arrangement; the function supplies no bounds protection.

## Behavior

### 1. Optional initialization and preliminary scan

The preliminary scan begins at the supplied starting index.

If that input is exactly zero, F-b3e71981d63f is called with the object address before the first scan-bound check. Changes made by that dependency may therefore affect the first count or table access.

For each reached scan iteration:

1. The current index is compared, as a signed value, against the current object count.
2. If the index is below the count, the function obtains the current table address and reads the record word at offset 20.
3. A nonzero word causes F-45a4578120f0 to be called with no explicit arguments.
4. If the original starting index is positive, the function returns immediately.
5. Otherwise, the scan index advances by one and the next bound check occurs.

The positive-index return occurs after the trigger check regardless of whether the trigger was zero. It occurs before any flag clearing, callback sweep, compaction, or embedded-storage transition performed by this function.

A positive starting index does **not** unconditionally select early return. If it is already greater than or equal to the count at the first check, no scan iteration is entered, and full maintenance follows.

The count is reread at successive loop checks. The table address is obtained again for successive record accesses. Neither is snapshotted for the whole scan. A dependency may therefore affect which later records are visited.

### 2. Flag update and maintenance dependency

If the preliminary phase does not return early, the function reads the 32-bit flag word at object offset 12, clears bit value 16, and writes the result back to that field.

This write preserves the other bits of the value read. It does not guarantee their final values, because later dependencies may modify the object.

F-a1cc018ab664 is then called with the object address.

All following table traversal uses object state available after that call. The function does not assume, or enforce, that the dependency leaves the table address, count, flags, or record contents unchanged.

### 3. Callback and state-clearing sweep

A new scan begins at index zero. At each iteration, the current index is compared against the then-current signed count, and a record address is formed from the current table address.

For a record whose discriminator at offset 4 is nonzero, this sweep performs no callback-state write.

For a record whose discriminator is zero:

- The state word at offset 12 is read.
- If that word is zero, the callback address is not read and no callback is invoked.
- If that word is nonzero, the callback address at offset 16 is read.
- A nonzero callback address is invoked with no explicit arguments.
- After any callback returns, the state word at offset 12 is written as zero.
- The same zero write also occurs when the original state word or callback address was zero.

The record address is captured before the callback. The subsequent zero write targets that captured record, even if the callback changes the object’s table address or count.

Consequently, a callback must leave that captured record writable until the post-callback write completes for execution to remain valid. The function contains no protection against a callback freeing or invalidating that storage.

The discriminator is not rechecked after a callback. A callback that changes it to nonzero still receives the post-callback state-word clearing.

### 4. Processing and compaction from index two

Both the source index and the destination index begin at two. Records zero and one are not candidates for removal in this phase, although they may have participated in the earlier callback sweep.

For each source index below the current signed count, the function captures the source record address using the current table address and examines its discriminator.

If the discriminator is zero:

1. The record’s first 32-bit word is passed to F-94cb53877c15.
2. After that call returns, the first word of the captured source record is written as zero.
3. The destination index does not advance.

This dependency call occurs even when the first word is zero. No null-value suppression is present.

If the discriminator is nonzero:

- When the destination index is less than the source index, D-01 is invoked with the current destination record address, the captured source record address, and a length value of 24.
- When the indices are equal, no D-01 call is made.
- The destination index advances by one in either case.

Within normal, non-overflowing traversal, retained records are encountered in increasing source-index order. If D-01 has the inferred copy behavior and intervening calls do not disrupt the table, this produces stable compaction.

The count is reread on later loop checks. Dependencies can change the remaining traversal. The table address is likewise not held constant across the entire phase.

### 5. Tail operation and count publication

After compaction traversal ends, D-02 is invoked with:

- An address equal to the current table address plus 24 times the final destination index.
- A length expression equal to 24 times the difference between the current object count and that destination index.

The function then writes the destination index to the object count field.

This ordering is significant: D-02 runs before the revised count is published. The count assignment occurs after D-02 returns, even if D-02 changed the count itself.

Under ordinary conditions with an initial count of at least two and no disruptive mutation, the published count is two plus the number of retained records encountered from index two onward. The requested suffix length is then nonnegative.

The function does not verify that condition. Counts below two, dependency-driven count changes, or arithmetic overflow can produce a negative or otherwise invalid length expression. The external parameter interpretation of such a value is not established.

### 6. Embedded-storage transition

If the final destination index is greater than two, the function returns after publishing the count.

Otherwise, it reads the current table address:

- If it equals the object address plus 332, the function returns without a storage-transfer call.
- If it differs, D-01 is invoked with the embedded address as destination, the current table address as source, and length 48.
- F-94cb53877c15 is then invoked with the table address read from the object after D-01 returns.
- After that dependency returns, the object’s table-address field is set to the embedded address.

The address passed to F-94cb53877c15 is not necessarily the same address supplied as D-01’s source: D-01 may modify the object’s table-address field.

Likewise, the final table-address assignment overwrites any change that F-94cb53877c15 makes to that field before returning.

The count is not republished after these calls. Its final value can therefore differ from the compaction result if one of these dependencies changes it.

## Side effects

Directly established writes are:

- Clearing bit value 16 in the object flag word on the full-maintenance path.
- Zeroing the state word of each reached zero-discriminator record during the callback sweep.
- Zeroing the first word of each reached zero-discriminator record at index two or higher during compaction.
- Publishing the compaction destination index as the object count.
- Publishing the embedded address as the table address when the storage-transition path completes.

Further memory effects depend on D-01, D-02, the named dependencies, and invoked callbacks.

No explicit allocation is visible. No ownership transfer or release semantics can be established solely from passing a value to F-94cb53877c15. If that dependency releases storage, callers must satisfy its ownership requirements both for removed-record values and for the table address.

There is no explicit error result, rollback, exception handler, or recovery path. Invalid memory, invalid callback addresses, dependency failure, and nonreturning callbacks are not handled locally.

Reentrancy and concurrent mutation are not established as supported. Repeated reads allow mutation to affect control flow, but do not provide synchronization or memory-safety guarantees.

## Dependencies

| Opaque ID | Established invocation contract | Unknown behavior |
|---|---|---|
| F-b3e71981d63f | One object-address argument; called only when the starting index is zero. | Initialization meaning, memory effects, ownership effects, failure behavior. |
| F-45a4578120f0 | No explicit arguments; called for a nonzero preliminary trigger. | Trigger handling, global access, object mutation, failure behavior. |
| F-a1cc018ab664 | One object-address argument; called after the flag write. | Maintenance meaning and resulting object state. |
| F-94cb53877c15 | One 32-bit argument; receives removed-record first words and, conditionally, the current table address. | Whether it releases memory, permits zero, changes globals, or changes the object. |
| D-01 | Three explicit arguments: destination address, source address, length value; observed lengths are 24 and 48. | Copy semantics, overlap behavior, return value, side effects. |
| D-02 | Two explicit arguments: suffix address and computed length. | Whether it clears memory, length interpretation, return value, side effects. |

D-01 and D-02 are document-local opaque labels for the two external dependencies without supplied public mappings. Their descriptions specify observed arguments, not verified implementations.

Callback targets are record-provided dependencies rather than fixed function identities. They receive no explicit arguments; their use of global state or implicit execution context is unknown.

## Acceptance examples

These examples are conditional behavioral checks derived from static evidence, not executed tests.

1. **Positive index inside the table.**  
   Given count five and starting index three, the function reads record three’s preliminary trigger. It calls F-45a4578120f0 exactly once if that trigger is nonzero, then returns. It does not itself clear the object flag or start compaction.

2. **Positive index at the bound.**  
   Given count five and starting index five, the preliminary loop is skipped. Full maintenance proceeds. This differs from an in-range positive index.

3. **Zero index with an empty preliminary scan.**  
   Given starting index zero, F-b3e71981d63f is called even if the resulting count makes the preliminary loop empty. Full maintenance then follows, subject to the storage and arithmetic preconditions.

4. **Callback eligibility.**  
   A reached zero-discriminator record with nonzero state and a nonzero callback address causes one callback invocation followed by a zero write to that record’s state word. With state zero, the callback-address field is not read.

5. **Stable retention under ordinary dependency behavior.**  
   Given count five, records two and four retained, record three removable, and dependencies that leave traversal state unchanged, F-94cb53877c15 receives record three’s first word, which is then zeroed. D-01 is requested to transfer record four to slot three. D-02 receives a suffix address at slot four and length 24. The published count is four. Actual transferred and cleared bytes depend on D-01 and D-02.

6. **Only reserved records remain.**  
   Given count two and an external table address, with dependencies leaving object fields unchanged, D-02 receives the address of slot two and length zero. The count is written as two. D-01 then receives the embedded destination, external source, and length 48; F-94cb53877c15 receives the external table address; the table field is finally set to the embedded address.

7. **Already embedded.**  
   Under the same count-two conditions with the table address already equal to the embedded address, the final 48-byte transfer request and table-address dependency call do not occur.

8. **Callback changes the table address.**  
   If a callback redirects the object table but leaves its original record writable, the state-clearing write still targets that original record. Later iterations may use the redirected table.

## Uncertainties

The following remain **documented_only**:

- The domain meaning of flags, discriminators, trigger words, callback state, and reserved records.
- The actual contracts of every dependency and callback.
- Whether D-01 copies bytes or D-02 clears bytes.
- Ownership and release requirements for record values and table storage.
- Valid count and index ranges beyond what the visible accesses require.
- Signed-overflow behavior, address-wrap behavior, external length signedness, alignment requirements, and byte order.
- Supported reentrancy, concurrency, and failure propagation.
- Whether any wider object invariant guarantees a count of at least two.

The inferred cleanup and compaction purpose does not resolve these unknowns.

## Evidence

This contract is based solely on the supplied static function representation and dependency mapping. Established claims follow visible branch conditions, signed comparisons, 32-bit field accesses, 24-byte record spacing, explicit call arguments, and write ordering.

Dependency internals and runtime outcomes were not supplied. No runtime tests have been performed. No claim of successful copying, clearing, deallocation, callback safety, or valid malformed-input execution is made.
