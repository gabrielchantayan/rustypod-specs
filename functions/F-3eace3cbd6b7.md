<!-- Trusted provenance: F-3eace3cbd6b7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3eace3cbd6b7

## Purpose

F-3eace3cbd6b7 searches a table using an unsigned numeric argument. Each eight-byte entry contains a pointer-sized field and a one-byte key. The function examines the first character reached through an entry’s pointer before comparing that entry’s key with the argument.

With a stable table base, stable entries, and stable character storage, it returns the pointer from the first matching entry encountered before a sentinel. A sentinel is an entry whose pointer addresses a zero character. Encountering that character causes an immediate zero return, without examining the entry’s key.

The external table base is evaluated separately for the character check, the key comparison, and the return-value load. The function does not visibly capture a table-wide or entry-wide snapshot. If the base or referenced storage changes during execution, those operations need not concern the same table, entry contents, or character storage.

The likely intent is a small-key lookup returning string-like storage. The established behavior is narrower: only one character is examined through each pointer used for a termination check, and no complete-string validation occurs.

## Interface

### Input and comparison width

The sole argument is an unsigned 32-bit value, ranging from zero through 4,294,967,295.

Each compared key is an unsigned eight-bit value. Its numeric value is compared against the full argument; the argument is not visibly truncated to eight bits.

Consequently:

- Arguments from zero through 255 can equal a key.
- Arguments greater than 255 cannot equal any key read by this function.
- An argument greater than 255 does not trigger an immediate return. Traversal continues until a zero character is observed or execution encounters conditions not covered by the visible normal paths.

Key zero has no special termination meaning.

### Result

The result is a 32-bit quantity obtained in one of two ways:

- An observed zero first character produces the numeric value zero.
- An equal key comparison leads to a separate 32-bit load from the pointer-field position selected by a fresh evaluation of the external table base.

Under stable-memory assumptions, the second result is the pointer used for that entry’s earlier character check. Without those assumptions, the result is not guaranteed to equal the previously checked pointer, address nonempty storage, or even be nonzero.

There is no separate status result. Under stable, valid-memory conditions, zero represents sentinel termination. Without stability, a separately loaded return field could also contain zero; the result alone does not identify which path produced it.

### External entry layout

The supplied representation uses an eight-byte entry stride:

| Offset within an entry | Width | Observed role |
|---|---:|---|
| 0–3 | 32 bits | Pointer field for the character check; raw result field on the matching path |
| 4 | 8 bits | Unsigned key |
| 5–7 | 24 bits | Not directly inspected |

The pointer accesses and entry layout reflect a 32-bit pointer representation. Byte order, alignment requirements, allocation boundaries, and meanings for the remaining three bytes are not established.

The external table-base storage is itself an input to each relevant address calculation. Its initialization, update protocol, and concrete value are not supplied.

### Preconditions for normal execution

Every source-level access actually reached must be valid independently:

- Each evaluation of the external table base must be permitted.
- The pointer-field location selected for a character check must be readable.
- The loaded pointer must permit a one-character read.
- After a nonzero character, the key-byte location selected using the next table-base evaluation must be readable.
- After an equal key comparison, the field selected using the final table-base evaluation must permit a 32-bit read.
- Index and address calculations must remain valid for the execution environment.

These requirements do not imply that the separate base evaluations produce equal values.

A null pointer is not a supported termination marker for the character-check access: the function attempts to dereference the loaded pointer. In contrast, the final result field is returned without being dereferenced again.

A reachable match or zero-character observation is needed for a visible normal return. No explicit table length limits traversal.

## Behavior

### Ordered source-level evaluations

Traversal begins at entry index zero. A nonmatching key advances the signed traversal index by one, corresponding to an eight-byte increase in the intended entry offset.

Within each visited index, the representation establishes this order:

1. The external table base is evaluated for the pointer-field access. The pointer is loaded, and the first character through it is read.
2. If that character is zero, the function returns zero. There is no subsequent key access or result-field load on this path.
3. If that character is nonzero, the external table base is evaluated again for the key-byte access at the current index. The unsigned key is compared with the full argument.
4. An unequal comparison advances traversal.
5. An equal comparison leads to another evaluation of the external table base. A 32-bit value is loaded from the pointer-field position at the current index and returned.

Thus an iteration ending in a mismatch contains two source-level base evaluations. An iteration ending in a match contains three. An iteration ending at the character check contains one.

These are counts of evaluations expressed in the supplied representation, not established hardware-load counts. Compiler transformations, memory-model constraints, and exact instruction ordering are not supplied.

### Stable-table lookup behavior

If the table base, relevant entry fields, and relevant character storage remain stable throughout the call, each iteration refers to one consistent entry.

Under those assumptions:

- Entries are considered in increasing index order.
- Each entry’s first character is checked before its key.
- A zero character terminates traversal regardless of that entry’s key.
- A matching ordinary entry returns its stored pointer.
- Among duplicate matching keys before the sentinel, the earliest entry wins.
- Entries after the matching entry or sentinel are not inspected.

A nonmatching key does not protect against an invalid pointer: the character read precedes the key read.

Only the first character matters to this function. A nonzero first character permits comparison even if later characters are inaccessible or no terminating zero character exists farther into the storage.

### Sentinel precedence and its limits

A zero character actually observed during the termination check always takes precedence over a key comparison at that index. The function immediately returns zero.

With stable base, entries, and character storage, a pointer to an empty character sequence cannot be returned through the matching path: its first character would terminate the search before comparison.

That guarantee does not extend to mutable storage. For example, the final field may differ from the field used for the character check, or the checked character may change before return. The function performs no final character check. The evidence therefore does not establish that every returned pointer addresses nonempty storage at return time.

### Separate evaluations and mutation

No visible synchronization binds the character observation, key observation, and result-field load together.

If changing state can be observed between these evaluations, a nonzero character check may use one table base, the key comparison another, and the result load another. Even with an unchanged base, entry fields or pointed-to characters could change independently.

Accordingly, without stability:

- An equal key comparison need not describe the entry whose character was checked.
- The returned field need not belong to the table used for either earlier decision.
- The returned value need not equal the pointer previously dereferenced.
- “First matching entry” need not describe a match in any single consistent table snapshot.

The representation establishes the separate evaluations, not a supported concurrency protocol. Which changes are observable, and whether unsynchronized modification has defined execution semantics, remain unresolved. No particular race outcome is promised.

### Arithmetic and traversal bounds

The traversal index is represented as a signed 32-bit integer. Entry addressing uses that index scaled by eight; the key access adds four bytes to the corresponding entry position.

There is no explicit limit on the index, no allocation-bound check, and no validation of the computed address.

The mathematical scaled offset reaches 2,147,483,648 at index 268,435,456, beyond the positive signed 32-bit range. Continued increments can also eventually exceed the index’s signed range. The supplied representation does not establish a portable overflow outcome or the exact effective-address behavior at these boundaries.

The function must not be specified as deliberately wrapping around to the beginning of a table. Address-space overflow, out-of-allocation traversal, and invalid reads are not visibly converted into a not-found result.

### Completion and errors

The two visible normal completion paths are the zero-character return and the field return following an equal key comparison.

There is no explicit error branch, diagnostic, retry, fallback lookup, or exception handling. Invalid storage, arithmetic overflow, and traversal without a reachable completion condition have no specified recovery behavior here.

## Side effects

No external writes are visible. The function changes only its internal traversal state and produces its result.

Its externally relevant reads include:

- Separate evaluations of the external table base for each reached access stage.
- A pointer-field read and one pointed-to character read for each termination check.
- A key-byte read only after a nonzero character observation.
- A separate 32-bit field read only after an equal key comparison.

No later index is reached after either return path. No key is read after a zero character is observed.

There is no visible allocation, deallocation, character copying, ownership transfer, reference-count adjustment, or lifetime extension. The successful matching path returns an existing field value rather than constructing a buffer.

The evidence grants no permission to modify or release returned storage. Its ownership, lifetime, accessibility after return, and validity as a complete string depend on external arrangements not supplied here.

No synchronization is visible. The source-level sequence does not establish atomic accesses, a consistent snapshot, or exact hardware read counts.

## Dependencies

No function calls are visible, so there are no callee dependencies requiring opaque function IDs.

The function depends directly on:

- External storage holding the table base.
- Eight-byte entry spacing and the field positions described above.
- Readable pointer, key, and character storage at every access actually reached.
- Valid index and address arithmetic.
- A reachable completion condition for finite normal execution.

The table producer, its update protocol, and storage owners are not identified. No unknown initializer or other unseen dependency is assumed to preserve the table base, keep entries stable, or own or release returned memory.

## Acceptance examples

These are conditional specifications derived from static evidence, not executed tests. Unless an example explicitly discusses changing state, assume a stable table base, stable entries and character storage, valid access permissions, and no arithmetic or address overflow.

### First-entry match

The first entry has key 17 and points to a nonzero first character. With argument 17, the function checks that character, reads the key, and returns the pointer field through the separate final access. No second entry is inspected.

### Match after a mismatch

The first entry has key 4; the second has key 9. Both point to nonzero first characters.

With argument 9, the function checks the first entry’s character before rejecting its key, then checks the second entry’s character and key and returns the second entry’s pointer.

### Sentinel before a later match

The first entry is nonempty and nonmatching. The second points to a zero character. A third entry has the requested key.

The result is zero at the second entry. Neither the second entry’s key nor any field of the third entry is needed.

### Sentinel with an equal key

The first entry points to a zero character and stores key 23. With argument 23, the function returns zero without reading that key. Under the stated stability assumptions, it does not return the pointer to the empty sequence.

### Duplicate keys

Two entries before the sentinel have key 42 and nonzero first characters. With argument 42, the first entry supplies the result. This first-match guarantee depends on the stable-state assumptions; it is not a snapshot guarantee under mutation.

### Full-width comparison

An entry has key 1 and a nonzero first character. Argument 257 does not match it. If traversal subsequently reaches a valid sentinel, the function returns zero there. Argument 257 is not treated as argument 1.

### Key zero

An entry with key zero and a nonzero first character matches argument zero. The same numeric key attached to a zero first character does not cause a match: the character check terminates first.

### Invalid character pointer

An entry’s key differs from the argument, but its character pointer is unreadable. The function attempts the character read before accessing that key. Neither a normal not-found result nor a particular fault is specified.

### No complete-string requirement

An entry has a readable, nonzero first character and a matching key. No second character read is required before returning its pointer. Safe use of that result as a complete string is not established.

### Different values at separate evaluations

Conditionally, suppose the execution model permits the first base evaluation to select a readable pointer with a nonzero first character, the second to select a readable key equal to the argument, and the third to select a readable field containing value R.

The source-level matching path returns R; it does not revalidate R as a pointer or check its first character. This describes the consequence of those stipulated observations, not proof that any particular concurrent schedule is permitted. If R is zero or points to an empty sequence, the earlier character check does not itself exclude that result.

## Uncertainties

The following remain **documented_only**:

- Concrete table contents, capacity, initialization, and allocation boundaries.
- Width and update semantics of the external base storage beyond its represented address use.
- Key meanings and intended interpretation of character storage.
- Ownership, mutability, and lifetime of returned storage.
- Byte order and machine-level alignment requirements.
- Exact arithmetic and effective-address behavior at overflow boundaries.
- Fault behavior for invalid reads.
- Permitted concurrent modifications and their language or platform semantics.
- Whether separate source-level base evaluations become separate machine loads.
- Atomicity, load reuse, and hardware ordering.

The inferred intent is a byte-keyed string-like lookup. Established static behavior consists of the ordered base evaluations and memory accesses, sentinel precedence at the character check, full-width key comparison, forward traversal, and separate result-field load.

## Evidence

The supplied static representation establishes:

- An unsigned 32-bit argument and a 32-bit result.
- A signed traversal index initialized to zero.
- Eight-byte entry spacing, a pointer field at offset zero, and an unsigned key byte at offset four.
- A fresh source-level table-base evaluation for the character-check pointer, subsequent key, and final result field.
- A pointed-to zero-character check before each key comparison.
- Immediate zero return on that observation.
- Index advancement after unequal comparison.
- A separate 32-bit return-field load after equal comparison.
- No visible calls, external writes, bounds checks, synchronization, or explicit error handling.

No runtime tests have been performed. The acceptance examples state conditional consequences of this static evidence; they do not establish deployed table contents, actual machine-load counts, or supported mutation behavior.
