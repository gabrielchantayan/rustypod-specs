<!-- Trusted provenance: F-12374fed1dfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-12374fed1dfd

## Purpose

F-12374fed1dfd conditionally replaces a context’s array of 32-bit entries with a newly obtained array that excludes entries equal to a supplied target.

The established behavior is more specific than “remove one entry”:

- The function first searches for any matching entry.
- If none is found, it returns 0 without requesting replacement storage.
- If a match is found, it requests storage sized for one fewer entry.
- It then copies every nonmatching entry, preserving their relative order.
- It installs the replacement array, decreases the stored count by exactly one, invokes two dependencies in sequence, and returns 1.

This creates an important distinction when the target occurs more than once: all matching entries are omitted, but the stored count is reduced by only one. The function neither rejects duplicates nor adjusts the count to the number actually copied.

The inferred intent is removal from a collection whose target entries are unique. Uniqueness is not checked and is not established by the available evidence.

## Interface

### Inputs

The function receives two values represented as 32-bit integers in the supplied static evidence:

1. **Context address.** Used as the base address of a structure containing an array address and an entry count.
2. **Target value.** Compared for exact 32-bit equality against array entries. No interpretation as a string, object, index, or owned resource is established.

The relevant context layout is:

| Offset from context base | Width | Observed use |
|---|---:|---|
| `0xF64` | 4 bytes | Address of the entry array |
| `0xF68` | 4 bytes | Number of entries; interpreted as unsigned for loop bounds and allocation-size arithmetic |

Each array entry occupies four bytes. Entry positions are addressed at four-byte intervals from the array address.

The evidence uses a 32-bit address representation. It does not establish a portable interface for wider pointers, a calling convention, byte order, or alignment rules.

### Output

The return value is a 32-bit scalar:

- **0:** the search reached the current count without finding a match, or the replacement-storage dependency returned zero.
- **1:** the replacement and count update occurred, followed by normal return from both subsequent dependency calls.

A return of 1 does not prove that the resulting array is fully initialized, that the final count equals the number of copied entries, or that dependencies left the context unchanged.

No separate error code distinguishes “target absent” from “replacement storage unavailable.”

### Preconditions for a defined, useful operation

The caller must provide accessible context storage at both relevant offsets. The function performs no context-address validation.

Whenever an entry is read, the corresponding four-byte location in the saved array must be readable. Whenever an entry is copied, the corresponding replacement location must be writable.

For the ordinary intended result, additional conditions are needed:

- The count and array contents remain stable during the operation, including across the storage dependency.
- The target occurs exactly once.
- The requested storage size is representable without arithmetic wrap.
- The returned replacement region has sufficient writable capacity.
- Replacement writes do not corrupt unread source entries or the context.
- The dependencies support the supplied addresses and complete normally.

These are conditions for the ordinary interpretation, not checks implemented by this function.

## Behavior

### Initial state capture and search

The function first reads and saves the array address from context offset `0xF64`. This read occurs even when the count is zero.

It then searches from entry position zero upward. Before each candidate entry read, it reads the count at offset `0xF68` and compares that count with the current unsigned position.

If the position is greater than or equal to the count, the function returns 0 immediately. That exit performs no replacement-storage request, context update, or subsequent dependency call.

Otherwise, the function reads the four-byte entry at the current position from the saved array address:

- An entry equal to the target ends the search.
- A nonmatching entry advances the position by one and continues the search.

The count is reread for each search-bound check. The array address is not reread. Consequently, this is not a search over an internally captured, immutable array-and-count pair.

Under stable, ordinary inputs, an empty array returns 0 without reading an array entry. A target absent from a nonempty array causes entries to be read in increasing position order until the bound is reached.

### Replacement-storage request

After finding a match, the function rereads the count and passes the following byte quantity to F-f195b74c9758:

**four times the unsigned 32-bit count minus one, with subtraction performed before multiplication.**

The subtraction and multiplication operate in the evidenced 32-bit unsigned arithmetic domain. They must not be interpreted as arbitrary-precision arithmetic or as an implicitly checked allocation size.

For a stable count of one, the requested size is zero.

If F-f195b74c9758 returns zero, the function returns 0. On this path, F-12374fed1dfd itself has not replaced the context’s array address, decremented its count, or invoked either subsequent dependency.

That statement does not imply rollback of effects performed inside F-f195b74c9758. Its behavior beyond the observed argument and return-value use is unknown.

If the dependency returns a nonzero value, that value is treated as the address of writable replacement storage. There is no further validation of its size, alignment, ownership, or relationship to the saved source array.

### Filtering into replacement storage

The copy phase starts at the beginning of the saved source array, not at the position where the search found a match. Source and destination positions both begin at zero.

Before every source-entry read, the function rereads the context count and checks whether the unsigned source position is below it. For every source entry admitted by that bound:

- The entry is read from the saved source array.
- If it equals the target, no replacement entry is written for it.
- Otherwise, it is written at the next replacement position, and that destination position advances.

Thus, with stable inputs and noninterfering storage, the replacement’s written prefix contains all nonmatching entries in their original relative order.

The function does not:

- Stop after excluding the first match.
- Use the original search position to limit filtering.
- Initialize unused replacement slots.
- Store the number of entries actually copied.
- Recheck that any target remains present after the storage dependency returns.

The saved source address remains in use even if a dependency changes the array-address field in the context. In contrast, changes to the count can affect the copy bound because that field is repeatedly reread.

### Commit and subsequent calls

Once the copy loop terminates, the function performs these operations in order:

1. Writes the replacement address to context offset `0xF64`.
2. Reads the count at offset `0xF68` and writes back that value decreased by one.
3. Calls F-1137792fba7d with the originally saved array address.
4. Calls F-034558ef5993 with no explicit arguments.
5. Returns 1.

The stored count is not derived from the destination position. It is the count present at the time of the final count update, decreased by one.

The pointer write precedes the count update. Both precede the call receiving the old array address. There is no rollback path shown after either context write.

No dependency return value is used after the replacement-storage request. If the final two dependencies return normally, the function returns 1 regardless of their return values. If either does not return normally, the supplied evidence establishes no completed return or recovery behavior.

## Side effects

### Direct reads and writes

The function directly reads:

- The context’s array address once, before searching.
- The context’s count repeatedly during searching.
- The count again to calculate the storage request.
- The count repeatedly during copying.
- The count again when decreasing it.
- Entries from the saved array during searching and again during copying.

On the successful path, it directly writes:

- One replacement entry for each nonmatching entry encountered during copying.
- The replacement address into the context.
- The decremented count into the context.

Matching entries are not copied. The function does not directly modify entries in the saved source array. Whether replacement writes could affect that array through overlap is not established.

### Ownership and lifetime

The call to F-1137792fba7d receives the saved source-array address only after the replacement address and decremented count have been stored.

Its position and argument are consistent with relinquishing old storage, but release semantics are inferred, not established. The available evidence does not prove that it frees memory, accepts a null address, or preserves the replacement.

Likewise, F-f195b74c9758 is used as a storage provider, but its allocation, initialization, aliasing, and ownership guarantees are unknown.

No per-entry cleanup occurs directly in this function. Even if entries represent resource addresses elsewhere, this function only compares and copies their four-byte values.

### External state and observation

All three dependencies may have additional effects not described here. In particular, no guarantee is available that they preserve context fields, global state, source contents, or memory ownership.

The two context writes are separate observable operations. This function supplies no synchronization or atomic transaction around them. Its behavior must not be represented as an atomic collection replacement.

## Dependencies

| Opaque ID | Observed interaction | Established use | Not established |
|---|---|---|---|
| F-f195b74c9758 | Called once after a match, with the calculated 32-bit byte quantity | Zero causes return 0; nonzero is used as replacement address | Allocation policy, zero-size behavior, initialization, capacity guarantees, aliasing, side effects |
| F-1137792fba7d | Called with the saved original array address after both context writes | Return value is unused | Whether it releases storage, transfers ownership, or mutates context/global state |
| F-034558ef5993 | Called without explicit arguments after F-1137792fba7d returns | Return value is unused | Purpose, implicit inputs, state changes, failure behavior |

Calling F-034558ef5993 a notification, refresh, or persistence operation would be an interpretation, not an established contract.

## Acceptance examples

These examples describe conditional expectations from the static evidence. They are not runtime test results. Unless otherwise stated, they assume stable counts and entries, distinct adequate replacement storage, and dependencies that return normally without changing relevant state.

### Empty collection

Given a count of zero:

- The array-address field is still read.
- No array entry is read.
- No dependency is called.
- The function returns 0.
- The function makes no direct context writes.

### Target absent

Given entries `[11, 22, 33]`, count 3, and target 44:

- The search reads the three entries in order.
- No replacement storage is requested.
- The function returns 0 without directly changing the context.

### Exactly one target

Given entries `[11, 22, 33]`, count 3, and target 22:

- The requested replacement size is 8 bytes.
- The replacement receives `[11, 33]`.
- The replacement address is stored before the count becomes 2.
- F-1137792fba7d receives the old array address.
- F-034558ef5993 is called next.
- The function returns 1.

A target at either end is excluded by the same filtering rule; retained entries keep their relative order.

### Repeated target

Given entries `[11, 22, 22, 33]`, count 4, and target 22:

- The requested replacement size is 12 bytes.
- Only `[11, 33]` is written by this function.
- The stored count becomes 3, not 2.
- The third replacement slot is not initialized by this function.

Acceptance must preserve this distinction. A result with count 2 would implement a different behavior.

### All entries match

Given count 3 and three entries equal to the target:

- The requested size is 8 bytes.
- No replacement entries are written.
- The stored count becomes 2.
- The successful path still invokes both subsequent dependencies and returns 1.

### Single matching entry

Given count 1 and a matching sole entry, the storage request is zero bytes.

- If F-f195b74c9758 returns zero, the function returns 0 without its own context updates or subsequent calls.
- If it returns a nonzero usable address, no entry is copied, that address is installed, the count becomes zero, and the successful call sequence proceeds.

A zero-byte request does not independently imply success or failure.

### Storage request fails

Given a found target and a zero return from F-f195b74c9758:

- No copy or commit occurs.
- Neither later dependency is called.
- The function returns 0.
- Effects internal to the storage dependency remain outside this guarantee.

### Arithmetic wrap boundary

For a stable count of `0x40000001`, the 32-bit request calculation produces zero bytes. The subsequent copy phase still consults the context count rather than a size-derived capacity.

This establishes the absence of a size-overflow guard. It is not a safe execution example: valid backing storage and behavior for any resulting writes are not established.

## Uncertainties

The following are **documented_only**:

- The complete context type and meanings of unrelated fields.
- Whether callers enforce uniqueness of entries.
- Whether array entries are numeric values, handles, or addresses.
- Exact calling convention, byte order, and platform alignment requirements.
- Storage-provider capacity, initialization, zero-size, and aliasing guarantees.
- Ownership and release semantics of every dependency.
- Dependency effects on context fields, globals, or source contents.
- Behavior on invalid addresses, inadequate storage, overlapping regions, or abnormal dependency termination.
- Synchronization requirements for concurrent access.
- Language-level guarantees for extreme signed destination-index arithmetic and address calculations.

Unsigned request arithmetic and search-bound comparisons are visible in the supplied evidence. That does not establish safe traversal of arbitrarily large arrays or defined behavior after invalid memory access.

## Evidence

**Established static evidence:** the field offsets and four-byte accesses; saved source address; repeated count reads; first-match search; 32-bit replacement-size calculation; filtering of every matching value; pointer-before-count commit order; fixed decrement of one; dependency arguments and call order; and return values 0 and 1.

**Inferred intent:** removal of a unique target from a collection, obtaining replacement storage, and subsequently relinquishing old storage. These interpretations explain the ordinary path but do not override the duplicate-target or zero-size behaviors.

**Unknown:** dependency internals, broader ownership rules, caller invariants, and effects outside the displayed function.

No runtime tests have been performed. No dynamic observation supports stronger claims than the static behavior described above.
