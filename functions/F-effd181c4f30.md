<!-- Trusted provenance: F-effd181c4f30; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-effd181c4f30

## Purpose

F-effd181c4f30 obtains a 32-bit value and commits it to the first available entry in an externally supplied table. Availability is determined solely by an entry’s first 32-bit word: the all-ones value, interpreted as −1, marks an available entry.

If the initial scan finds no available entry, the function requests replacement storage for an increased entry count, delegates transfer and initialization of entries, publishes the replacement descriptor, and selects an entry using the descriptor values then present.

The commit consists of a direct first-word write followed by a delegated update of the second word. The function returns an address-derived index calculated after that delegated update.

This establishes a reusable-slot mechanism, not its application domain. The acquired value’s meaning, the second word’s meaning, resource ownership, and the purpose of the surrounding local context are not established.

## Interface

### Arguments

| Argument | Representation | Visible use |
|---|---|---|
| First input | Pointer to a readable 32-bit word | Its word is supplied to F-23df8d149813. |
| Second input | Pointer to a readable 32-bit word | Its word is supplied to F-4aefb2d3a7dc. |
| Third input | Opaque 32-bit value | Passed unchanged to F-47d8e93d0701. |

The first input word is read before the context-establishment call. The second input word is read after that call returns. Consequently, effects of F-23df8d149813 can change the value subsequently obtained through the second input.

The function performs no stores **through the first or second argument pointer**. This is not a promise that their underlying storage remains unchanged. No non-aliasing requirement is established: either input word could overlap an entry or descriptor field written later, and dependencies may also modify accessible storage.

Neither input pointer is checked for nullness or validity. Each must identify readable storage at the time of its dereference for ordinary execution.

### Result

The visible result is a signed 32-bit integer.

- An acquisition result of exactly −1 causes a return of −1 after context cleanup.
- Otherwise, the result is formed from the selected entry’s address minus the descriptor’s base, with the resulting signed difference shifted right by three bits.

For stable, valid, non-overflowing table geometry, this yields the selected entry’s zero-based index. That interpretation is conditional. The base used in the result calculation is read **after** F-47d8e93d0701 returns, not necessarily the base used when the entry was selected.

The computed result is retained across context cleanup. Cleanup-induced descriptor changes do not cause recalculation.

No distinct result is provided for replacement-storage failure, transfer failure, initialization failure, or payload-update failure.

### Descriptor and entry layout

F-529ccfe5c361 supplies an address interpreted as a descriptor containing at least two consecutive 32-bit fields:

| Descriptor byte offset | Role |
|---|---|
| 0 | Entry-storage base address |
| 4 | Entry count |

Entries have an eight-byte stride:

| Entry byte offset | Role |
|---|---|
| 0 | 32-bit occupancy/value word; exactly −1 means available |
| 4 | 32-bit payload field addressed by F-47d8e93d0701 |

The supplied representation uses 32-bit words, addresses, and address arithmetic. It does not establish byte order.

### External configuration storage

Growth uses four external 32-bit configuration storage locations, designated here as configuration locations A, B, C, and D in replacement-storage argument order.

These labels identify **storage locations**, not values captured for the whole operation:

- All four locations are read for F-4500dbc81938 after F-3846b98eb3b0 returns.
- Configuration location B is read again for F-56c3b4c706c6 after the transfer and initialization calls.
- The later value from B need not equal the earlier value supplied to F-4500dbc81938.

No ordering among the four configuration reads, or between argument reads belonging to the same call, is established by the supplied representation.

## Behavior

### Context and acquisition boundary

The operation begins by passing an eight-byte local context area, the first input word, and the integer 6 to F-23df8d149813.

After that call returns, the second input word is read and supplied to F-4aefb2d3a7dc together with the 32-bit all-ones value.

Exactly −1 is recognized as acquisition failure. On that path:

- F-7c87d6325a96 receives the original context-area address.
- The function returns −1.
- F-529ccfe5c361 is not called.
- No table scan, growth, or entry commit is performed directly by this function.

This does not constrain effects inside the context, acquisition, or cleanup dependencies. Other negative acquisition results continue through the table-update path.

### Available-entry selection

After a nonfailure acquisition, F-529ccfe5c361 supplies the descriptor address. The function obtains a starting base and an end address corresponding to that base plus eight times the count.

The scan proceeds from the starting address in eight-byte increments. It checks equality with the end address before examining an entry. For each examined entry, it reads only the first word and selects the first occurrence of −1.

Under valid, non-wrapping geometry, this is an ascending, first-available scan over the initial entry count. A zero count produces no entry read and proceeds directly to growth.

The scan endpoint is established before traversal. The scan does not repeatedly read the descriptor count. Its termination check is address equality rather than an independently maintained remaining-entry count. Overflow or invalid geometry therefore cannot be assigned ordinary bounded-scan behavior without additional environmental guarantees.

When an available entry is found, the function makes no growth calls and performs no direct descriptor writes on that path. It does not directly read the selected entry’s old second word before passing its address to F-47d8e93d0701.

### Replacement storage when no entry is available

Growth uses a fresh descriptor-count read to form an increased requested count. The requested count is retained as a 32-bit quantity and used with unsigned comparisons in the subsequent transfer-extent and initialization decisions. This read is distinct from the count used to establish the earlier scan endpoint.

F-3846b98eb3b0 then receives a four-byte local temporary area. Its 32-bit return value is retained as the source-like operand for initializing added positions.

Only after that call returns are configuration locations A through D read for the replacement-storage request. F-4500dbc81938 receives:

- The retained requested count.
- Two integer arguments, each equal to 8.
- The four configuration values read for this call.

Its return value is used as the replacement-storage base. There is no visible null, sentinel, or other failure check before that use.

After F-4500dbc81938 returns, the descriptor base and count are read again. The initial transfer source is that base. The transfer extent is the unsigned minimum of the current count and the retained requested count.

F-704efc28383b is called with destination/source-like operands for each transfer iteration. The destination begins at the replacement base; both source and destination positions advance by eight bytes after each call.

The transfer termination address is significant: it is evaluated using a **fresh descriptor-base read at each loop condition**, plus eight times the retained transfer extent. Thus, the loop is not unconditionally a transfer of one immutable captured range. Under a stable descriptor base, it makes exactly the selected number of calls in entry order. If a dependency changes that base, the stopping address changes while the advancing source position does not reset. Extra iterations, altered termination, or invalid accesses are not excluded.

After transfer, positions from the retained transfer extent up to, but excluding, the requested count are initialized through F-704efc28383b. Each call receives the appropriate replacement-entry address and the initialization value returned by F-3846b98eb3b0. With stable counts and ordinary one-entry growth, this initializes exactly one position. If an earlier dependency changed the descriptor count before the transfer extent was determined, the number can differ.

### Replacement publication and selection

After transfer and initialization, F-56c3b4c706c6 receives:

- The descriptor base read for this call.
- A newly read value from configuration location B.

The base can differ from the initial transfer source. The configuration value can differ from the value previously supplied as the second configuration argument to F-4500dbc81938. Neither is required to have been preserved by intervening dependencies.

On normal return from F-56c3b4c706c6, the function publishes the retained requested count to descriptor offset 4 and then the replacement base to descriptor offset 0. These are separate writes, count first; no atomic descriptor replacement is established.

F-07fef63861ea is then called with a separate four-byte local temporary area. No initialization of that area is visible before this call.

Only after F-07fef63861ea returns does the function select the entry at the descriptor’s then-current base plus eight times its then-current count, minus eight. Accordingly, the selected entry is the last replacement entry only if the relevant descriptor values remain consistent. Dependency-induced mutation can change that selection.

### Entry commit and completion

Both reuse and growth converge on the same commit behavior:

- The acquired 32-bit value is written directly to the selected entry’s first word.
- F-47d8e93d0701 receives the selected entry’s second-word address and the third input.
- After that call returns, the descriptor base is read for the signed address-difference calculation described under Result.
- F-7c87d6325a96 receives the original context-area address.
- The previously calculated result is returned.

The first-word write precedes the payload call. There is no visible rollback, compensating release, or explicit error branch after it. If the payload dependency does not return normally, the normal cleanup and return sequence is not established.

## Side effects

### Direct writes and aliasing

The normal nonfailure path directly overwrites the selected entry’s first word. Growth also directly writes the descriptor count and base, in that order.

The function passes writable-address-shaped operands to dependencies for context handling, entry transfer, temporary handling, and payload update. The exact writes performed by those dependencies are outside the supplied evidence.

Absence of a store through an input argument is not storage preservation. For example, if the first input points to a selected entry’s first word, that word is first read as an input and later overwritten as part of the commit. If an input overlaps a descriptor field, growth publication may overwrite it. No distinct-storage precondition is visible.

### Ownership and lifetime

The growth sequence suggests storage replacement, but this is inferred intent rather than an established ownership contract.

The evidence does not determine whether:

- F-4500dbc81938 allocates, reserves, or obtains storage by another mechanism.
- F-704efc28383b copies, moves, retains, destroys, or otherwise transforms entry resources.
- F-56c3b4c706c6 releases memory.
- F-47d8e93d0701 consumes the third input or disposes of an existing payload.
- Either temporary-area dependency creates or destroys an owned object.
- Context establishment or cleanup acquires or releases synchronization resources.

A behavioral implementation must preserve the visible delegation and ordering without inventing these ownership rules.

### Error and synchronization boundaries

Only acquisition result −1 has an explicit failure branch. There are no visible checks for invalid descriptors, invalid storage, allocation failure, arithmetic overflow, malformed initialization values, or dependency-reported update errors.

No synchronization or reentrancy guarantee is established. In particular, count-before-base publication is observable if the environment permits observation between those writes. The surrounding context does not by itself prove mutual exclusion.

## Dependencies

| Opaque function ID | Established interface role |
|---|---|
| F-23df8d149813 | Receives an eight-byte context-area address, the first input word, and 6. |
| F-4aefb2d3a7dc | Receives the second input word and all ones; exactly −1 selects explicit failure. |
| F-529ccfe5c361 | Supplies the descriptor address. |
| F-3846b98eb3b0 | Receives a four-byte temporary-area address and returns the initialization operand. |
| F-4500dbc81938 | Receives requested count, 8, 8, and values read from configuration locations A–D after F-3846b98eb3b0 returns. |
| F-704efc28383b | Receives destination/source-like operands for transferred or initialized entries. |
| F-56c3b4c706c6 | Receives the currently read descriptor base and a fresh read of configuration location B after transfer and initialization. |
| F-07fef63861ea | Receives a distinct four-byte temporary-area address after descriptor publication and before growth-path selection. |
| F-47d8e93d0701 | Receives the selected entry’s second-word address and the third input. |
| F-7c87d6325a96 | Receives the original context-area address on both explicit return paths. |

No listed dependency has an established contract to preserve configuration storage, descriptor fields, entry contents, or other globals.

## Acceptance examples

These are conditional, static acceptance criteria, not executed tests. Unless overridden, assume valid readable and writable storage, non-overflowing arithmetic, normally returning dependencies, and stable descriptor values.

### Exact acquisition failure

If F-4aefb2d3a7dc returns −1, context establishment precedes acquisition, context cleanup follows it, and the result is −1. No table-accessor call or direct table update occurs.

Dependency side effects remain permitted.

### First available entry

For four entries with first words 12, −1, 34, and −1, an acquired value of 77 selects entry 1.

Its first word becomes 77 before F-47d8e93d0701 receives its second-word address. Growth dependencies are not called. If the base remains unchanged through the result calculation, the return is 1.

### Stable full-table growth

For two occupied entries, an unchanged descriptor, and usable replacement storage:

- The requested count is 3.
- Two existing entries are delegated for transfer in ascending entry order.
- Entry 2 is delegated for initialization.
- F-56c3b4c706c6 precedes descriptor publication.
- Count 3 is written before the replacement base.
- F-07fef63861ea precedes final entry selection.
- Entry 2 receives the acquired value and payload call.
- The return is 2 if the base remains stable.

This does not assert byte-copy semantics or successful release of old storage.

### Configuration changes across dependencies

Suppose configuration location B contains one value when F-4500dbc81938’s arguments are read, and a transfer dependency changes that location before the later call.

F-4500dbc81938 receives the earlier value; F-56c3b4c706c6 receives the later value. Acceptance must not require equality.

Likewise, changes to any configuration location during F-3846b98eb3b0 are reflected in the subsequent replacement-storage arguments.

### Empty table

With count zero, no entry is scanned. Under stable ordinary growth, the requested count is 1, no existing entry is transferred, and one entry is initialized and committed.

The recorded old base is still supplied to F-56c3b4c706c6. No null-base exemption is visible.

### Aliased input storage

If the first input points to the first word of an available entry that is later selected, its initial value is read for context establishment. The commit later overwrites that same storage with the acquired value.

This satisfies the absence of stores through the input argument while disproving any unconditional promise that the input word remains unchanged.

### Other negative values and changing bases

An acquired value of −2 follows the nonfailure path and is written into the selected entry. A later scan does not treat −2 as available.

If F-47d8e93d0701 changes the descriptor base, the return calculation uses that changed base. It need not equal the original selected-entry index.

If F-704efc28383b changes the descriptor base during transfer, acceptance must reflect the changed stopping address rather than require the initially calculated number of transfer calls.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of all inputs, payloads, context data, and configuration values.
- Dependency ownership, destruction, allocation, error, and global-state preservation contracts.
- Valid maximum counts, alignment requirements, and environmental synchronization.
- The purpose and contents of both four-byte temporary areas, especially the separately passed area with no visible prior initialization.
- Behavior when dependencies mutate descriptor fields, except for the explicit read timings and resulting data dependencies described above.
- Behavior on faults, nonlocal exits, or dependencies that do not return normally.
- Overflow semantics beyond the visible 32-bit representation and unsigned comparisons.

In particular, count increment, multiplication by eight, address addition, and address subtraction require representable, valid geometry for the ordinary table interpretation. A count increment whose 32-bit result becomes zero is not rejected visibly, but safe growth, safe traversal, and a meaningful return are not established for that case. Exact overflow behavior must not be inferred solely from the displayed expression types.

The signed right shift has the ordinary index interpretation for nonnegative, eight-byte-aligned differences. Its precise lower-level meaning for negative or unrepresentable differences remains unresolved.

## Evidence

**Established static evidence:** The supplied body establishes argument-read timing across calls, the exact −1 acquisition branch, descriptor offsets, eight-byte entry stride, first-available selection, growth call ordering, configuration-read timing, the fresh later read of configuration location B, dynamic transfer-stop evaluation, count-before-base publication, commit ordering, and post-payload result calculation.

It also establishes the narrower input-access fact: the argument pointers are dereferenced for reads, while later direct stores target selected-entry or descriptor addresses. It does not establish that those storage regions are disjoint.

**Inferred intent:** The structure resembles insertion into a reusable-slot table with storage replacement when full and a context scoped around the operation. Allocation, release, ownership, and locking interpretations are not proven.

**Unknowns:** Dependency implementations, environmental invariants, configuration meanings, and exceptional behavior were not supplied. They remain explicit contract boundaries.

No runtime tests have been performed.
