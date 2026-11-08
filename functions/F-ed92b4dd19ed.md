<!-- Trusted provenance: F-ed92b4dd19ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed92b4dd19ed

## Purpose

F-ed92b4dd19ed requests and initializes an 80-byte object, then attempts to register its address in a globally selected collection of 32-bit pointer entries. It takes no explicit arguments and returns a signed 32-bit integer.

On normally completed execution with valid, stable memory:

- An initially zero collection reference produces −1 before either dependency is called.
- An object address already present in the selected collection produces −1, after object initialization.
- Otherwise, the function optionally calls a collection dependency and searches for the first zero-valued entry.
- Any found entry, **including entry zero**, receives the object pointer. Before storing that pointer, the function increments the collection’s 32-bit field at byte offset 8.
- A successful insertion returns its zero-based index. Failure to find a zero-valued entry returns −1.

The function searches within a signed bound; it does not append outside that range. Initialization is not undone on registration failure.

Object creation and registration are inferred intent. The memory accesses, dependency calls, and branch outcomes described below are the established behavior.

## Interface

### Invocation and result

The function has no explicit parameters. Its inputs are global state, reachable memory, and dependency behavior.

| Signed 32-bit result | Meaning on normally returning execution |
|---|---|
| −1 | The initial collection reference was zero, a duplicate object pointer was found, or no zero-valued entry was found. A late zero-pointer check also leads here if execution reaches it. |
| 0 | Entry zero was selected; the field at offset 8 was incremented and the obtained object pointer was stored in entry zero. |
| Positive index | That entry was selected; the field at offset 8 was incremented and the obtained object pointer was stored at the returned index. |

A fault or a dependency that does not return normally has no established function result.

### Implicit inputs and representation

The function uses:

- A global pointer identifying a slot that holds a collection reference.
- Two global 32-bit initializer values, called **initializer A** and **initializer B** here.
- F-8157a180d901, which receives a storage-size argument.
- F-2723bbdc94dc, which receives a collection address.
- The selected collection and its entry array.

The supplied representation uses 32-bit pointers and four-byte array entries. Offsets in this contract are byte offsets.

The selected collection has at least these accessed fields:

| Offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Address of the entry array |
| 4 | 32 bits | Signed scan bound |
| 8 | 32 bits | Compared with the scan bound; incremented on insertion |

No semantic meaning such as occupancy or capacity is established for either numeric field.

### Preconditions for normal execution

The function does not validate these conditions:

- The global slot pointer identifies a readable slot.
- F-8157a180d901 returns storage supporting every listed initialization write.
- The collection reference reread after initialization identifies a readable collection.
- Every examined entry is readable.
- On insertion at **any nonnegative index**, including zero, the field at offset 8 and the selected entry are writable.
- Address calculations and signed arithmetic are valid for the execution environment.
- The collection remains usable after F-2723bbdc94dc returns.

The initial reference being nonzero establishes none of the later memory-validity conditions.

## Behavior

### 1. Initial reference gate

The function captures the address of the globally selected collection-reference slot and checks the reference used by the initial gate.

If that reference is zero, it returns −1. Neither dependency is called, and no object initialization is performed.

Otherwise, it calls F-8157a180d901 with the integer argument 80.

The initial nonzero reference is not retained as the eventual collection address. After object initialization, the function rereads the reference from the captured slot address.

Consequently, a dependency that changes the captured slot’s contents can change the collection subsequently processed. Changing only the global pointer that originally selected the slot does not redirect that later reread: the previously captured slot address remains in use.

This distinction does not establish synchronization guarantees for concurrent changes during the initial global accesses.

### 2. Object initialization

After F-8157a180d901 returns, initializer B is read and retained as a 32-bit value. Initializer A is then read for the first object write.

The following writes occur in the listed order relative to the returned address:

| Order | Offset | Width | Value |
|---:|---:|---:|---|
| 1 | 0 | 32 bits | Initializer A |
| 2 | 4 | 32 bits | 0 |
| 3 | 8 | 32 bits | 0 |
| 4 | 12 | 8 bits | 0 |
| 5 | 16 | 32 bits | Previously read initializer B |
| 6 | 20 | 32 bits | 1 |
| 7 | 24 | 32 bits | 16 |
| 8 | 36 | 32 bits | 0 |
| 9 | 28 | 32 bits | 32767 |
| 10 | 32 | 32 bits | 1000 |
| 11 | 40 | 32 bits | 0 |
| 12 | 44 | 32 bits | 0 |
| 13 | 48 | 32 bits | 32767 |
| 14 | 52 | 32 bits | 0 |
| 15 | 56 | 32 bits | 1 |
| 16 | 60 | 8 bits | 0 |
| 17 | 61 | 8 bits | 0 |
| 18 | 62 | 8 bits | 0 |
| 19 | 64 | 32 bits | 0 |
| 20 | 68 | 32 bits | 32767 |
| 21 | 72 | 32 bits | 0 |
| 22 | 76 | 8 bits | 0 |

The explicit writes leave gaps at bytes 13–15, byte 63, and bytes 77–79. This function does not establish their values. It does not establish that the entire 80-byte region was previously cleared.

The nonzero-pointer check follows all these writes. It is therefore not a protective allocation-failure check: a zero return from F-8157a180d901 causes writes beginning at address zero to be attempted first.

If execution nevertheless completes those writes and reaches the check with a zero pointer, the result is −1 and collection processing is skipped. Whether that path can execute normally depends on the environment.

### 3. Collection selection and duplicate search

For a nonzero object pointer, the function reads the collection reference from the captured slot and uses that collection address for the remaining processing.

The duplicate search examines entries in increasing index order, starting at zero. Each entry is four bytes wide. It compares each examined entry directly with the obtained object pointer.

A match returns −1 immediately. The collection dependency is not called, and no insertion update follows.

Initialization has already occurred. If the returned object address aliases an object already represented in the collection, that storage has already received the initialization writes before duplicate detection.

This first scan rereads the signed field at offset 4 for its loop condition. Entry accesses obtain the array base from the collection rather than relying on an immutable snapshot of the entire structure. Changes to those fields can therefore affect the accesses, subject to the execution environment’s rules.

A zero or negative stable bound causes no entries to be examined.

### 4. Optional collection dependency

After a duplicate-free search, the function compares the collection’s fields at offsets 8 and 4.

If they are equal, F-2723bbdc94dc is called with the selected collection address. No return value is consumed.

Equality alone controls this call. Equal zero values and equal negative values also satisfy the condition. The function does not require a conventional nonnegative collection state.

After the call, the function continues using the same collection-structure address. It does not reread the collection-reference slot. Mutations to that structure can affect subsequent field reads; replacement of the slot’s reference alone does not select a different structure for the remaining operations.

No preservation, allocation, or ownership behavior is assumed for this dependency.

### 5. Available-entry search

After the optional call, the function reads the field at offset 4 once into a signed 32-bit bound for the second search.

It examines entries in increasing order, beginning at index zero, stopping at the first zero-valued entry strictly below that captured bound. Array-entry addressing uses a four-byte stride, with the array base read from the selected collection during entry accesses.

Unlike the duplicate search, this search does not reread the bound for each iteration.

If the captured bound is zero or negative, no entries are examined and the search result is −1. If the bound is positive but every examined entry is nonzero, the search result is also −1.

Backing storage beyond the captured bound is not searched, regardless of the field at offset 8.

### 6. Insertion and return

A successful search produces an index greater than or equal to zero and strictly less than the captured bound.

For such an index, including zero, the function performs these observable operations in order:

1. Read the 32-bit field at offset 8.
2. Add one.
3. Write the result back to offset 8.
4. Read the array base from offset 0.
5. Store the obtained object pointer in the selected four-byte entry.
6. Return the selected index.

The final insertion condition compares a nonnegative search result with the captured bound. It does **not** retain an earlier index-zero distinction. Since a normally found index is strictly below that bound, index zero satisfies the insertion condition just as a positive index does.

If the search result is −1, these insertion operations are skipped and −1 is returned.

The field update precedes the entry write. If the later access faults, the increment may already be observable. There is no rollback.

## Side effects

### Object storage and reachability

The function performs the ordered initialization writes before duplicate detection or available-entry search. It contains no explicit deallocation operation.

Initialization remains observable even when registration fails because a duplicate exists or no zero-valued entry is available.

On successful insertion, including insertion at index zero, the collection receives the object pointer. The function itself returns only an integer index.

On failure after storage acquisition, no separate pointer output is provided. Storage may remain reachable through dependency-maintained state or preexisting references, but that is not established here. A memory leak cannot be concluded without the dependency’s ownership contract.

### Collection mutations

On every successful insertion, direct collection mutation consists of:

- Incrementing the field at offset 8.
- Writing the object pointer into the selected entry.

The function does not directly increment the field at offset 4.

No such insertion mutation occurs after duplicate detection or an unsuccessful available-entry search. Either dependency may have additional effects not established by this function.

Aliasing can make the initialization writes affect collection state, initializer storage, or other reachable memory. No non-aliasing guarantee is supplied.

### Error and synchronization behavior

There is no explicit locking, rollback, release, error callback, or exception-handling mechanism.

The result −1 combines multiple conditions and does not imply that the function had no side effects. No distinct out-of-memory result is established.

The accesses are not presented as atomic. Concurrent mutation is not assigned a stronger contract than the visible access structure and the execution environment permit.

## Dependencies

### F-8157a180d901

- Called once if the initial reference gate is nonzero.
- Receives the integer value 80.
- Returns a value treated as an object-storage pointer.
- Is followed by initializer reads and object writes.
- Is not followed by a protective null check before those writes.

Its role as an allocator is inferred from the size argument and use of the result. Its ownership policy, failure convention, effects on globals, and effects on existing memory are unknown.

### F-2723bbdc94dc

- Called at most once.
- Called only after the duplicate search completes without a match.
- Called when the fields at offsets 8 and 4 compare equal.
- Receives the selected collection address.
- Has no consumed return value.
- Is followed by fresh collection-field reads.

Its possible resizing role is inferred, not established. It may mutate globals or collection storage; preservation of the selected structure, array, entries, and ownership relationships is not guaranteed by the supplied evidence.

## Acceptance examples

These are conditional acceptance cases, not executed tests. Unless stated otherwise, assume valid writable storage, normal dependency returns, stable memory, no relevant aliasing, and arithmetic within valid ranges.

### Initially zero reference

Given a readable initial slot containing zero:

- Return −1.
- Call neither dependency.
- Perform no object initialization.

### Duplicate object pointer

Given a nonzero initial reference and an obtained object address already present within the duplicate scan’s range:

- Complete object initialization before detecting the duplicate.
- Return −1.
- Do not call F-2723bbdc94dc.
- Perform no subsequent insertion increment or entry write.

### Insertion at index zero

Given a bound of 2, a field at offset 8 equal to 5, and entries consisting of zero followed by an unrelated nonzero pointer:

- Find no duplicate.
- Do not call F-2723bbdc94dc, because 5 differs from 2.
- Select index zero.
- Change the field at offset 8 from 5 to 6.
- Store the obtained object pointer in entry zero, after the field write.
- Return 0.

Zero is a successful insertion result, not a special no-write result.

### Insertion at a positive index

Given a bound of 3, a field at offset 8 equal to 7, and entries consisting of an unrelated nonzero pointer, zero, and another unrelated nonzero pointer:

- Select index 1.
- Change the field at offset 8 from 7 to 8.
- Store the obtained pointer in entry 1.
- Return 1.

### No zero-valued entry

Given a bound of 2, a field at offset 8 equal to 5, and two nonzero entries neither equal to the obtained pointer:

- Return −1 after initialization.
- Perform no insertion increment or entry write.
- Perform no explicit release of the obtained storage.

### Equal fields followed by dependency mutation

Given no duplicate and equal fields at offsets 4 and 8:

- Call F-2723bbdc94dc.
- Use the same collection-structure address afterward.
- Search according to the bound and array state read after the call.

If the dependency leaves bound 1, field 8 equal to 4, and entry zero empty, insertion changes field 8 to 5, stores the pointer at entry zero, and returns 0.

If it instead leaves no empty entry below the bound, return −1 even if unused backing storage exists beyond that bound.

### Nonpositive bound

Given no duplicate and unequal fields, with a captured second-search bound of zero or a negative value:

- Examine no entries in that search.
- Return −1 without insertion.

If the fields are equal, the dependency call occurs first and its effects determine the subsequently captured bound.

### Changed collection-reference slot

If F-8157a180d901 changes the captured slot to reference another valid collection:

- Complete initialization before rereading that slot.
- Process the replacement collection.

If it changes only the global pointer selecting the slot, the later reread still uses the previously captured slot address.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the object fields and collection fields.
- Ownership acquisition, transfer, and release rules.
- Dependency failure conventions and preservation guarantees.
- Whether relevant memory regions may alias.
- Caller expectations and surrounding collection invariants.
- Byte order, alignment requirements, and invalid-address behavior.
- Runtime consequences of signed arithmetic or address-calculation overflow.
- Concurrency behavior beyond the absence of explicit synchronization.

The field increment uses 32-bit arithmetic represented as signed integer arithmetic. Incrementing the maximum signed value raises an overflow question; neither saturation nor guaranteed wraparound is established. Four-byte entry addressing also requires valid multiplication and address addition.

Index-zero insertion is not unresolved in the supplied control flow: a successfully found entry zero takes the same increment-and-store path as any other found index.

## Evidence

**Established static evidence:** the supplied function representation supports the initial gate, captured-slot reread, dependency arguments and call conditions, initialization widths and order, distinct scan-bound handling, duplicate rejection, and increment-before-store insertion at every successfully found index, including zero.

**Inferred intent:** allocation, object creation, registration, and collection growth are plausible interpretations. They do not establish ownership or replace the observed conditions.

**Unknown:** dependency implementations, caller contracts, surrounding invariants, and target-specific execution behavior were not supplied.

No runtime tests have been performed.
