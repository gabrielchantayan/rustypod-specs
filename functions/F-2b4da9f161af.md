<!-- Trusted provenance: F-2b4da9f161af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b4da9f161af

## Purpose

F-2b4da9f161af performs an ordered sequence of state changes and notifications:

1. Obtain a state-storage pointer and clear its first byte.
2. Invoke a dependency with a global argument and a zero argument.
3. When the input is zero, obtain an object, invoke one of its dynamically selected operations with zero, and set the fourth byte of the original state storage to one.
4. Examine up to two entries in a globally referenced table and, if either is nonzero, notify the receiver associated with the first nonzero entry.

This is the established behavior of the supplied static representation. Descriptions such as “reset,” “enable,” “activate,” or “select” would assign meanings not established by that representation. The operations may serve such purposes, but their domain meaning is documented_only.

The function does not return a result or report whether a table entry was found.

## Interface

### Input

The function accepts one integer input, represented as a 32-bit signed integer in the supplied material.

Only equality with zero affects its own control flow:

- Zero enables the conditional object call and subsequent fourth-byte write.
- Every nonzero value skips both operations.

The input is not forwarded to a dependency. Positive and negative nonzero values have the same local behavior.

### Output

The function has no declared return value. Its observable results are memory writes, calls, and any effects produced by those calls.

There is no explicit status code, error object, or success indicator.

### Referenced storage and layouts

The following descriptive names identify roles, not private source identifiers.

| Role | Required layout or interpretation |
|---|---|
| State storage | A pointer returned by F-c26f463fe480. Byte offset 0 is written on every normally progressing invocation. Byte offset 3 is written only on the zero-input path, after the conditional call returns. |
| Global dependency argument | A global value evaluated for the call to F-13ef3c7b2c3d. Its type and domain meaning are not established. |
| Table base | A globally supplied address used to read at most two consecutive 32-bit entries, at byte offsets 0 and 4. |
| Conditional receiver | The pointer returned by F-cd96e43d3119. Its first 32-bit word supplies a dispatch-table address. |
| Selected table entry | A nonzero 32-bit table value used as an address. The word at byte offset 16 supplies another receiver pointer. |
| Selected receiver | The pointer obtained from the selected entry. Its first 32-bit word supplies a dispatch-table address. |

The representation uses four-byte integer accesses, four-byte table spacing, and pointer-bearing words consistent with a 32-bit address model. These widths are material to this contract. A wider host implementation must not silently substitute native-width table entries or change byte offsets.

The supplied material does not establish byte order, alignment guarantees, or a portable external serialization format.

### Preconditions for normal completion

No explicit validation is visible. Normal completion therefore requires valid storage and valid call targets wherever execution reaches a dereference or indirect call.

In particular:

- F-c26f463fe480 must return a pointer writable at byte offset 0.
- If the input is zero and the conditional call returns normally, the originally returned state pointer must remain writable at byte offset 3.
- The conditional receiver and its dispatch-table entry must be valid when the zero-input path reaches them.
- The table must support the entry reads actually performed.
- A selected nonzero entry must expose a valid receiver at byte offset 16, and that receiver must expose a valid dispatch-table target.

These are access requirements, not evidence that the function checks or enforces them.

## Behavior

### Unconditional prefix

The first operation is a call to F-c26f463fe480 with no explicit arguments. The returned pointer is retained for later use.

After that call returns, the function writes the byte value zero at offset 0 of the returned storage. This is a one-byte write, not a four-byte integer clear. No neighboring-byte write is established.

The function then evaluates the global dependency argument and calls F-13ef3c7b2c3d with two explicit arguments:

- The evaluated global value.
- Integer zero.

This call follows the first-byte write. Consequently, the dependency can observe that write if its behavior and memory relationships allow it to do so. The global argument is evaluated after F-c26f463fe480 has returned and after the byte write; it must not be treated as an entry-time snapshot.

No return value from F-13ef3c7b2c3d is used.

### Additional behavior for zero input

If the input equals zero, the function calls F-cd96e43d3119 with no explicit arguments.

It then uses the returned pointer as a receiver:

- Read the dispatch-table address from the receiver’s first 32-bit word.
- Obtain the callable target at byte offset 52 within that dispatch table.
- Invoke that target with the receiver pointer followed by integer zero.

The target’s identity and complete behavior are not supplied. Its return value, if any, is not used.

Only after this indirect call returns does the function write byte value one at offset 3 of the state pointer originally returned by F-c26f463fe480.

The function does not call F-c26f463fe480 again before this write. If an intervening dependency changes which storage that accessor would now return, the fourth-byte write nevertheless uses the retained original pointer. Conversely, the function provides no protection if an intervening call invalidates that pointer.

For nonzero input, the accessor call, indirect call, and offset-3 write are all skipped. In particular, skipping the offset-3 write does not imply that this byte is cleared or otherwise preserved by dependencies.

### Two-entry search

After the unconditional prefix and any zero-input operations, the function searches the table.

The search is bounded to two 32-bit entries:

1. Read the entry at table byte offset 0.
2. If that entry is nonzero, select it without reading the entry at byte offset 4.
3. If the first entry is zero, read the entry at byte offset 4.
4. Select the second entry if it is nonzero. If it is also zero, finish without a table-derived call.

Thus the selection rule is “first nonzero among two entries,” not “process every nonzero entry.”

There is no length field, sentinel scan beyond the second entry, validation of a nonzero entry, or fallback after a selected entry fails.

The table lookup occurs after all earlier calls have returned. Values present before those calls are not necessarily the values read by this function. The static representation does not show a snapshot of the table taken at entry.

The search uses only the first two indices. Their byte displacements are 0 and 4; there is no input-dependent indexing or reachable index arithmetic overflow in this search.

### Call through a selected entry

If a table entry is selected, its nonzero 32-bit value is treated as an address.

The function then:

- Reads a receiver pointer from byte offset 16 of the selected entry.
- Reads that receiver’s dispatch-table address from its first 32-bit word.
- Obtains a callable target at byte offset 20 within that dispatch table.
- Invokes the target with the receiver pointer followed by integer one.

The selected table-entry address is not itself the receiver passed to this call. The receiver comes from its offset-16 field.

The function returns after this call completes. It does not inspect the other table entry, use the call’s return value, or perform an explicit subsequent memory write.

If both table entries are zero, it returns without this indirect call.

## Side effects

### Direct writes

Exactly two direct write sites are visible:

| Condition | Destination | Width | Value |
|---|---|---:|---:|
| Every invocation that reaches the write | Original state pointer, offset 0 | 1 byte | 0 |
| Zero input, after the conditional indirect call returns | Same retained pointer, offset 3 | 1 byte | 1 |

This does not establish the final values of those bytes. Later dependencies may modify them or aliased storage.

The function itself does not directly modify the table, its selected entry, or either receiver in the supplied representation. Callees may do so.

### Call ordering

For zero input, the observable call order is:

1. F-c26f463fe480.
2. F-13ef3c7b2c3d.
3. F-cd96e43d3119.
4. The conditional receiver’s offset-52 dispatch target.
5. The selected receiver’s offset-20 dispatch target, if a nonzero table entry exists.

For nonzero input, the third and fourth calls are omitted.

The offset-0 byte write occurs between the first and second calls. The conditional offset-3 byte write occurs after the fourth call and before the table search.

### Ownership and lifetime

No explicit allocation, deallocation, reference-count operation, or ownership transfer is visible in this function.

Returning or passing a pointer does not establish whether it is owned, borrowed, retained, or released. Dependency ownership behavior is documented_only.

The retained state pointer creates a lifetime requirement across intervening calls on the zero-input path. The supplied material does not prove those calls preserve its allocation.

### Failure behavior

There are no explicit null checks, error branches, retries, or recovery actions.

A zero result from F-c26f463fe480 is still used as a write destination. A zero conditional receiver is still dereferenced. A nonzero table entry with an invalid offset-16 receiver is not skipped in favor of another entry.

The exact result of an invalid memory access or invalid indirect call is outside the supplied evidence. No exception, trap, process termination, or recoverable error policy is established.

If a dependency does not return normally, later operations are not established to occur. Earlier writes and effects have no visible rollback.

## Dependencies

| Opaque function ID or target | Arguments supplied | Result used | Established role |
|---|---|---|---|
| F-c26f463fe480 | None explicitly | State-storage pointer | Provides the address used for direct byte writes. |
| F-13ef3c7b2c3d | Global argument, integer 0 | No | Runs after the first-byte write. |
| F-cd96e43d3119 | None explicitly | Conditional receiver pointer | Called only for zero input. |
| Unidentified offset-52 dispatch target | Conditional receiver, integer 0 | No | Runs before the conditional fourth-byte write. |
| Unidentified offset-20 dispatch target | Receiver from selected entry’s offset-16 field, integer 1 | No | Runs once if the two-entry search finds a nonzero entry. |

No opaque IDs are supplied for the two dynamically resolved targets. Their behavior must not be inferred from their offsets or argument values.

None of these dependencies is established to preserve globals, table contents, state bytes, receiver lifetimes, or unrelated memory.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes valid reached accesses and normally returning callees unless stated otherwise.

### Nonzero input; both entries zero

For an input such as −1, the function calls F-c26f463fe480, writes zero to state offset 0, and calls F-13ef3c7b2c3d. It skips the conditional receiver path, reads both table entries, and makes no table-derived call.

There is no direct offset-3 write. Its final value is not specified.

### Zero input; both entries zero

The function performs the unconditional prefix, obtains the conditional receiver, and calls its offset-52 target with that receiver and zero.

Only after that call returns does it write one to state offset 3. It then reads both zero table entries and returns without the offset-20 call.

An implementation that writes offset 3 before the offset-52 call does not satisfy the ordering contract.

### Both entries nonzero

The first entry is selected. The second entry is not read by the search.

Exactly one table-derived dispatch occurs, using the receiver stored at offset 16 of the first entry and integer one. Calling both entries would violate the contract.

### First entry zero; second entry nonzero

Both entries are read, in that order. The offset-20 target is resolved through the receiver belonging to the second entry.

### An earlier dependency changes the table

If F-13ef3c7b2c3d changes the table before returning, the subsequent search uses the table state available when the reads occur, not a value captured before that call.

This example assumes the resulting table address and reached entries remain valid.

### Conditional call changes state bytes

If the offset-52 target writes state offset 0, the function does not subsequently clear that byte again.

If that target writes state offset 3, the function subsequently overwrites that byte with one, provided the retained pointer remains valid and the call returns normally.

### Invalid first selected entry

If the first table entry is nonzero but its receiver chain is invalid, the function contains no fallback to the second entry. Any specific runtime failure outcome remains documented_only.

## Uncertainties

- **Inferred intent:** The ordering resembles coordinated state management and receiver notification. No particular subsystem or business meaning is established.
- **documented_only:** The global argument’s type, meaning, and permitted values.
- **documented_only:** Dependency side effects, ownership contracts, reentrancy, and failure conventions.
- **documented_only:** Identities and full interfaces of dynamically selected targets beyond the explicit arguments used here.
- **documented_only:** Thread safety, synchronization, atomicity, and behavior under concurrent mutation.
- **documented_only:** Additional ABI details and whether any return-register contents are observable to callers despite the declared no-result interface.
- **documented_only:** Behavior outside the supplied reachable representation. The supplied material reports omitted unreachable regions, but does not establish additional behavior from them.

## Evidence

This contract is based solely on the supplied static function representation and the supplied opaque dependency mapping.

The established evidence covers input branching, access widths and offsets, direct write values, explicit call arguments, call ordering, two-entry search bounds, first-nonzero precedence, and both indirect-dispatch chains.

No dependency implementation, caller behavior, runtime trace, or independent ABI specification was supplied. No runtime tests have been performed. Acceptance examples describe consequences of the visible operations under stated conditions; they are not claims of observed execution.
