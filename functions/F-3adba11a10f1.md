<!-- Trusted provenance: F-3adba11a10f1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3adba11a10f1

## Purpose

F-3adba11a10f1 conditionally places a signed integer value into the first zero-valued slot of a caller-supplied integer collection.

The established behavior has three important properties:

- An existing occurrence of the requested value causes an immediate return of −1, before any dependency call or direct write.
- If two header fields are equal, F-53e02005f830 is called before the search for an available slot.
- Successful placement increments the header field at byte offset 8. It does **not** directly increment the scan-length field at byte offset 4.

The routine is therefore not established to be a conventional append operation or a conventional growable-container insertion. In particular, it returns −1 when the searched range contains no zero, even if storage beyond that range might exist.

A plausible intent is uniqueness-preserving insertion into a zero-marked slot collection. That interpretation is inferred, not a substitute for the field-level contract below.

## Interface

### Inputs

The routine receives:

1. **A writable header address.** The supplied representation uses three consecutive integer-sized fields:

   | Byte offset | Contract name | Observed use |
   |---|---|---|
   | 0 | Storage address | Supplies the base address for slot reads and the eventual slot write. |
   | 4 | Scan length | Controls the initial duplicate search and, after any dependency call, supplies the bound for the zero-slot search. |
   | 8 | Auxiliary counter | Compared with the scan length before the dependency call; incremented on successful placement. |

2. **A requested integer value.** This value is compared against existing slots and, on successful placement, stored in a slot.

The supplied representation describes signed integer values with a four-byte slot stride. Its address-bearing header field is also represented as an integer. This contract uses a **32-bit reconstructed layout**: three four-byte header fields and four-byte slot values. It does not establish compatibility with a native 64-bit-pointer layout.

The auxiliary counter may be intended to represent capacity or another container quantity. Its semantic name is not established by the observed operations.

### Output

The result is a signed integer:

- **−1:** Either the requested value was found during the initial search, or no zero-valued slot was found during the later search.
- **A nonnegative index:** The index of the first zero-valued slot found during the later search. The routine then increments the auxiliary counter and stores the requested value at that index.

The result alone cannot distinguish duplicate rejection from absence of an available slot.

### Preconditions

For defined, meaningful execution:

- The header must permit every read and write reached by the selected execution path.
- The storage address must identify readable four-byte slots for every index actually inspected.
- A successful placement requires writable storage at the selected slot.
- Slot-address calculations, index arithmetic, and the auxiliary-counter increment must remain within the execution environment’s valid arithmetic and address domain.
- If F-53e02005f830 is called, it must return normally for the remainder of this contract to apply. Its resulting header and storage state must support the accesses that follow.

The routine performs no explicit null-pointer, allocation-size, alignment, or structural-consistency validation.

## Behavior

### 1. Search for an existing value

The routine begins at slot index zero and searches in increasing index order.

Before each iteration, it reads the current scan length from the header. When the current index is less than that signed length, it obtains the storage address and reads the four-byte slot at that index.

If the slot equals the requested value, the routine immediately returns −1. On this path:

- F-53e02005f830 is not called.
- The routine performs no direct header write.
- The routine performs no direct slot write.

The initial search does not snapshot the scan length once for the entire search. Its bound is read repeatedly. The storage address is likewise obtained for individual slot accesses rather than retained as a permanent entry-time snapshot.

For an ordinary, nonoverlapping collection with no external mutation, this phase simply searches indices zero through scan length minus one.

If the scan length is zero or negative, this phase reads no slots.

### 2. Conditionally invoke the dependency

After the initial search finishes without a match, the routine compares the auxiliary counter with the scan length.

If they are equal, it calls **F-53e02005f830**, passing the same header address.

If they differ, it does not call that dependency. In particular, an auxiliary counter smaller than the scan length does not independently trigger the call.

There is no established ordering between the two header reads used solely to evaluate this equality. Both occur before any resulting dependency call.

The dependency’s return value, if any, is not used. There is no explicit status check.

The routine does not perform another duplicate search after the dependency returns. Consequently, a dependency that introduces the requested value into storage does not cause the initial rejection rule to be applied again.

### 3. Find the first zero-valued slot

After the optional dependency call, the routine reads the scan length and retains that value as the bound for the second search.

It then searches from index zero upward, reading each addressed slot until:

- A slot equals zero, in which case that index is selected; or
- The retained bound is exhausted, in which case the selected result is −1.

Unlike the first search, this search uses a retained length. It still obtains the storage address for the individual slot accesses.

This search does not inspect a slot at the retained scan length. It does not search unused storage beyond that range, regardless of the auxiliary counter.

A retained scan length of zero or a negative value results in no slot reads and a result of −1.

### 4. Update state when a slot was found

The supplied control conditions require a nonnegative selected index, no greater than the retained scan length, and different from that length.

For an index produced by the second search, these conditions reduce to the following established behavior:

- A found zero slot is updated.
- The no-slot result, −1, is not updated.

On success, the routine performs these direct state changes in this order:

1. Read the current auxiliary counter.
2. Add one.
3. Write the incremented value back to the auxiliary-counter field.
4. Read the storage address from the header.
5. Write the requested value into the selected slot.

It then returns the selected index.

There is no direct write to the scan-length field and no direct write to the storage-address field.

The order matters: the counter changes before the slot write, and the storage address for that write is read after the counter change. No rollback is present if the later access fails.

### Zero as a requested value

Zero participates in ordinary equality comparisons during the initial duplicate search.

Therefore, when the requested value is zero and the initial searched range contains a zero, the routine returns −1 immediately. It does not treat that zero as an insertion opportunity.

For stable storage without a dependency-induced change, requesting zero cannot successfully fill an existing zero slot: the initial search would already have rejected it.

Success with a requested value of zero remains possible conditionally—for example, if the dependency creates a zero slot after the initial search has completed. Such success increments the auxiliary counter and writes zero back to the selected slot.

## Side effects

### Direct effects

The routine directly writes only on the successful-slot path:

- One increment of the auxiliary counter.
- One four-byte slot write.

The scan length is unchanged by the routine’s own explicit writes.

A return of −1 is not a general promise of unchanged state. It is unchanged by direct writes, but the optional dependency may already have changed the header, storage, or other state.

### Ownership and lifetime

No direct allocation or deallocation appears in the supplied body. The header and storage are accessed through caller-supplied state.

The routine does not establish that it acquires ownership of either region. It also does not establish that the dependency preserves storage ownership or lifetime. The dependency might replace or otherwise alter storage; its actual contract is unavailable.

The requested value is stored as an integer bit pattern, not dereferenced as an object address. No ownership operation on a resource represented by that value is established.

### Aliasing and concurrent mutation

The supplied body does not reject overlap between the slot region and the header. Such overlap can make a slot write also change a header field. The “no direct scan-length write” statement describes the explicit field accesses, not a guarantee against changes through aliased storage.

No synchronization, atomicity, or concurrent-access protocol is established. The read order described above does not make concurrent mutation safe.

### Errors

There are no explicit error codes other than the overloaded −1 result. Invalid memory, invalid addressing, dependency failure, and arithmetic overflow have no established recovery behavior in this routine.

## Dependencies

### F-53e02005f830

**Invocation condition:** The initial duplicate search found no match, and the auxiliary counter equals the scan length at the subsequent equality check.

**Argument:** The original header address.

**Result handling:** No returned status or value is used.

**Subsequent behavior:** The scan length is read again after the call, and later slot accesses use storage addresses obtained from the header. Thus, changes to those fields can affect the second search and final write.

**Unknown contract:** The supplied evidence does not establish whether this dependency:

- Allocates, releases, relocates, or initializes storage.
- Changes either counter.
- Preserves existing values or uniqueness.
- Touches globals or other externally visible state.
- Reports failure indirectly.
- Always returns normally.

Its apparent placement at an equality boundary suggests a storage-management role, but that is inferred intent only.

## Acceptance examples

The following examples assume valid four-byte storage, representable arithmetic, no concurrent mutation, and no header/storage overlap unless stated otherwise.

### Duplicate rejection precedes the equality-triggered call

Given:

- Slots: [12, 0, 27]
- Scan length: 3
- Auxiliary counter: 3
- Requested value: 12

Expected behavior:

- Return −1 on the first slot.
- Do not call F-53e02005f830, despite equality of the two fields.
- Leave the header and slots unchanged by direct writes.

### First zero is filled; scan length is not incremented

Given:

- Slots: [12, 0, 27, 0]
- Scan length: 4
- Auxiliary counter: 6
- Requested value: 19

Expected behavior:

- No dependency call.
- Return 1.
- Slots become [12, 19, 27, 0].
- Auxiliary counter becomes 7.
- Scan length remains 4.

The later zero is not selected.

### No zero means no append

Given:

- Searched slots: [12, 27]
- Scan length: 2
- Auxiliary counter: 8
- Requested value: 19

Expected behavior:

- Return −1.
- No dependency call.
- No direct writes.

Even if additional allocated storage exists, it is not searched or used by this invocation.

### Dependency changes the later search range

Given an initial state with no requested-value match and equal header counters, suppose F-53e02005f830 returns with:

- Slots: [12, 27, 0]
- Scan length: 3
- Auxiliary counter: 5
- Requested value still 19

Expected continuation:

- Search the post-call range of three slots.
- Return 2.
- Store 19 at index 2.
- Increment the post-call auxiliary counter from 5 to 6.

This example is conditional on the dependency producing that state; it does not assert that the dependency actually does so.

### Zero is rejected when already present

Given:

- Slots: [12, 0]
- Scan length: 2
- Auxiliary counter: 4
- Requested value: 0

Expected behavior:

- Return −1 during the initial search.
- Do not increment the auxiliary counter.
- Do not rewrite the zero slot.

### No post-dependency uniqueness check

Suppose the initial search finds no requested value, the equality condition triggers the dependency, and that dependency returns with:

- Slots: [19, 0]
- Scan length: 2
- Auxiliary counter: 4
- Requested value: 19

Expected continuation:

- Select index 1 because it contains zero.
- Increment the auxiliary counter to 5.
- Store 19 at index 1.
- Return 1, leaving two occurrences of 19.

Uniqueness across dependency-induced changes is not guaranteed.

### Empty range without a dependency call

Given scan length 0 and auxiliary counter 2:

- Neither search reads any slot.
- The dependency is not called.
- The result is −1.
- There are no direct writes.

If both fields were zero, the dependency would instead be called before the second search.

## Uncertainties

The following items are **documented_only**, not resolved by assumptions:

- The precise ABI and machine-level signed-overflow behavior. Four-byte slots and the reconstructed 32-bit layout are supported by the supplied representation; wrapping behavior at arithmetic boundaries is not established.
- The semantic meaning and valid range of the auxiliary counter.
- Whether valid callers require nonnegative lengths or nonoverlapping header and storage regions.
- The complete behavior, failure modes, and ownership effects of F-53e02005f830.
- Whether zero is formally reserved as an empty-slot marker by callers.
- Whether external synchronization is required or supplied.
- Fault behavior for invalid addresses and behavior when address calculations exceed their valid domain.

In particular, incrementing a maximum signed 32-bit auxiliary counter is outside this contract’s defined arithmetic examples. It must not be presumed to produce a particular wrapped value.

## Evidence

**Established static evidence:** The supplied body contains an ordered duplicate search, an equality-guarded dependency call, a separate first-zero search, a conditional auxiliary-counter increment, and a subsequent slot write. The scan-length field has no explicit assignment. Both duplicate rejection and zero-search exhaustion return −1.

**Inferred intent:** Zero likely marks availability, and the equality-triggered dependency may manage storage. Neither inference changes the observed field updates or search bounds.

**Unavailable evidence:** No dependency implementation, caller contract, allocation metadata, ABI specification, or runtime trace was supplied.

No runtime tests have been performed. The acceptance examples specify consequences of the supplied static behavior under their stated preconditions; they are not reports of executed tests.
