<!-- Trusted provenance: F-dfa22dba8c35; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dfa22dba8c35

## Purpose

F-dfa22dba8c35 conditionally updates a record’s byte-sized state and invokes operations on an object obtained through two dependencies.

The function accepts an unsigned 32-bit request. After calling both dependencies, it compares the full request with a state byte read through a previously captured record pointer. If an object is available and the values differ, it writes four auxiliary bytes, invokes one or two indirect operations, and finally stores the request’s low eight bits as the state.

The return value is a zero-extended byte, but its source depends on the object-acquisition result:

- **Non-null object:** return the initial state observation, obtained before either dependency call through a separate reference to the global record pointer.
- **Null object:** return a later state observation through the captured record pointer.

The pointer capture and the initial observation use two distinct, ordered source-level references to the global pointer. They are not represented as a single pointer read followed by a read through that captured pointer. Treating the two references as identifying the same record requires the global pointer to remain unchanged between them.

**Inferred intent:** the request appears to control an enabled/disabled condition because zero and nonzero requests select different notifications. That interpretation does not restrict the accepted request to zero or one. The full 32-bit value affects change detection, while only its low eight bits are stored.

## Interface

### Input

One unsigned 32-bit **request**, ranging from 0 through 4,294,967,295.

Its three uses are distinct:

| Use | Established interpretation |
|---|---|
| Change detection | Compare the full 32-bit request with the unsigned value of an 8-bit state |
| Common indirect-operation argument | False for zero; true for every nonzero request |
| Final state write | Store the low eight bits, equivalent to reduction modulo 256 |

Every request greater than 255 differs from every possible state byte. Such a request can therefore cause an update even when its low byte already matches the stored state.

### Output

An unsigned 32-bit result containing a zero-extended 8-bit observation. Every ordinary return is between 0 and 255 inclusive.

The result is not a success flag, an update count, or necessarily the final state. With a non-null object, it can describe a different record from the one subsequently compared or updated if the two initial global-pointer references resolve differently.

### Global pointer and record locations

The supplied representation establishes this distinction:

1. A first reference to the global pointer captures the pointer used for subsequent comparison, conditional writes, and the null-object return read.
2. A second reference to the global pointer supplies the address from which the initial state byte is read.

The initial byte read is not expressed through the captured pointer. When the global pointer remains unchanged between these references, both identify the same record. Otherwise, their identity is not established.

This is a source-level ordering statement. It does not establish two physical machine loads, a defined concurrent-mutation mechanism, or synchronization guarantees.

The directly accessed record locations are:

| Byte offset | Access |
|---|---|
| 0 | Initial observation through the second global-pointer reference; later comparison or null-object return read through the captured pointer; conditional final write through the captured pointer |
| 8 | Conditional byte write: zero for a nonzero request, one for a zero request |
| 9 | Conditional byte write of zero |
| 10 | Conditional byte write of zero |
| 11 | Conditional byte write of zero |

The initial observation need not address the same storage as the later accesses unless pointer identity is established.

The auxiliary updates are four separate byte writes in ascending offset order. The evidence does not establish a single 32-bit field or a single atomic write. No direct access to offsets 1 through 7 is shown.

### Object and dispatch representation

F-6d4a3ea23b92 returns a pointer-like result tested against null. For a non-null object, indirect operations are selected through its leading table reference at table byte offsets 0x10 and 0x20.

The supplied representation uses a 32-bit integer read for the leading table reference. This establishes the represented dispatch calculation, not a portable object ABI, complete table layout, allocation size, or ownership model.

### Preconditions for ordinary completion

The function does not validate the following requirements:

- The pointer obtained by the second initial global reference must permit the initial state-byte read, even if object acquisition later returns null.
- The captured pointer must permit the later state-byte read.
- On a changing, non-null-object path, captured-record offsets 0 and 8 through 11 must be writable.
- Required record storage must remain valid across the intervening calls.
- A non-null object must support every reached table-reference read, table-entry access, and indirect invocation.
- All calls must return normally for the function to reach its ordinary return.

Pointer validity, lifetime extension, reference counting, ownership transfer, and synchronization are not established.

## Behavior

### Initial observations and dependency calls

The observable source-level sequence begins with:

1. Capture the record pointer from the global pointer location.
2. Reference that global pointer location again and read the state byte through the pointer supplied by this second reference. Retain this byte as the **initial observation**.
3. Call F-4ffd7bc0e22a without explicit arguments.
4. Call F-6d4a3ea23b92 with the preceding call’s four-byte result and the value one.

Both dependencies run on every execution that reaches them, including executions whose state eventually compares equal to the request.

After these calls, direct record accesses use the captured pointer, not another reference to the global pointer. A dependency’s replacement of the global pointer does not redirect those later accesses. It also does not guarantee that the captured record remains alive.

No dependency is assumed to preserve the record, global pointer, object storage, or any other relevant memory.

### Null-object outcome

If F-6d4a3ea23b92 returns null, the function:

- Performs no direct auxiliary or state writes.
- Invokes no indirect object operation.
- Reads state offset 0 through the captured pointer.
- Returns that newly read byte, zero-extended to 32 bits.

The initial observation is not returned on this branch, although its read has already occurred.

The later byte can differ from the initial observation because dependencies changed the captured record, because the initial observation addressed a different record, or both.

Null does not produce a distinct numeric error code. Its return range overlaps completely with the non-null-object return range.

### Non-null object and equal state

For a non-null object, the function reads state offset 0 through the captured pointer and compares its unsigned value with the entire 32-bit request.

If equal:

- There are no direct record writes.
- Neither indirect operation runs.
- The function returns the retained initial observation.

The comparison byte and returned byte therefore have different observation times and potentially different record identities.

Dependency mutations can change whether an update occurs. An initially matching state can become different before comparison; an initially different state can become equal. Equality is decided using the post-dependency read, not the initial observation.

### Non-null object and nonzero changing request

If the comparison differs and the request is nonzero, the function performs these effects in order:

1. Writes zero to captured-record offset 8.
2. Writes zero to offset 9.
3. Writes zero to offset 10.
4. Writes zero to offset 11.
5. Invokes the operation selected at dispatch-table offset 0x10, passing the object as receiver and a true logical argument.
6. After that operation returns, writes the request’s low eight bits to captured-record offset 0.
7. Returns the initial observation.

Requests such as 256 take this nonzero path despite storing zero.

### Non-null object and zero changing request

If the comparison differs and the request is zero, the function performs these effects in order:

1. Writes one to captured-record offset 8.
2. Writes zero to offset 9.
3. Writes zero to offset 10.
4. Writes zero to offset 11.
5. Invokes the operation selected at dispatch-table offset 0x20, passing the object as receiver and zero as the other explicit argument.
6. After that operation returns, invokes the operation selected at dispatch-table offset 0x10, passing the object as receiver and a false logical argument.
7. After that operation returns, writes zero to captured-record offset 0.
8. Returns the initial observation.

The common offset-0x10 operation receives the logical zero/nonzero predicate. Its exact ABI argument width is not established by the supplied representation.

### Visibility across indirect operations

All four auxiliary writes precede the first indirect operation. The final state write follows every indirect operation on the selected path.

Consequently, the first indirect operation is entered after the auxiliary update but before this function’s final state write. It must not be assumed to observe the entry state: dependencies or other permitted mutations may already have changed it.

On the zero-request path, selection of the offset-0x10 operation occurs after the offset-0x20 operation returns. The later dispatch uses a new read of the object’s leading table reference. The earlier operation is not assumed to preserve that reference or the selected entries.

There is no further state comparison after callbacks. If a callback changes captured-record offset 0 and execution returns normally, the final state write overwrites that change. Callback changes to offsets 8 through 11 are not repaired afterward.

These observations describe ordering within the supplied representation, not atomicity or inter-thread visibility guarantees.

## Side effects

The function’s direct writes are limited to the changing, non-null-object path:

- Four ordered byte writes at captured-record offsets 8, 9, 10, and 11.
- One later byte write at captured-record offset 0.

No direct writes occur on the null-object or equal-state paths.

Indirect effects are not bounded by the supplied evidence. Both named dependencies can affect shared state even when no direct update follows. Indirect operations may affect the record, object, dispatch table, globals, or other memory.

No allocation, release, retain, or ownership-transfer operation is directly shown. This absence does not establish that the returned object is borrowed, owned, retained, or leaked. It also does not establish that dependencies leave record or object lifetimes intact.

No explicit locking, rollback, recovery handler, or transactional boundary is shown. If an indirect operation fails to return normally, auxiliary writes may already have occurred while the final state write remains unreached.

There is no explicit error status for an invalid pointer, invalid dispatch entry, or nonlocal exit. Behavior under those conditions remains unresolved.

## Dependencies

| Dependency | Established use | Unresolved contract |
|---|---|---|
| F-4ffd7bc0e22a | Called once, without explicit arguments, before object acquisition; returns a four-byte value passed onward | Result meaning, validity, side effects, and ownership |
| F-6d4a3ea23b92 | Called once with the preceding result and one; supplies the null-tested object reference | Meaning of one, ownership, lifetime, and guarantees associated with a non-null result |
| Unresolved indirect target at table offset 0x20 | Called only for a changing zero request; receives the object and zero; return value is unused | Target identity, full calling convention, effects, and failure behavior |
| Unresolved indirect target at table offset 0x10 | Called for every changing request with a non-null object; receives the object and the request’s nonzero predicate; return value is unused | Target identity, argument width, effects, and failure behavior |

Only F-4ffd7bc0e22a and F-6d4a3ea23b92 have supplied opaque dependency IDs. The indirect targets are identified by their dispatch positions, not by invented implementation identities.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, assume valid storage and dispatch targets, normal returns, an unchanged global pointer between the two initial references, a non-null object, and no dependency or callback mutation of relevant state.

1. **Equal request.** Initial state is 1 and request is 1. Both dependencies run. The captured-record comparison is equal. No direct writes or indirect operations occur. The result is 1.

2. **Nonzero byte update.** Initial state is 0 and request is 7. Offsets 8 through 11 receive zero in order. The offset-0x10 operation receives true. State offset 0 becomes 7 only after that operation returns. The result is 0.

3. **Zero update.** Initial state is 7 and request is 0. Auxiliary bytes become 1, 0, 0, and 0. The offset-0x20 operation precedes the offset-0x10 operation; the latter receives false. The final state becomes zero after both return. The result is 7.

4. **Request 256.** Initial state is 0. The full-width comparison differs, so the nonzero update path runs and notifies true. The final stored byte is nevertheless zero. Repeating the request under the same conditions repeats the update path.

5. **Matching low byte.** Initial state is 1 and request is 257. The comparison differs despite matching low bytes. The nonzero update path runs, finally storing 1 and returning 1.

6. **Maximum request.** Initial state is 255 and request is 4,294,967,295. The comparison differs. The nonzero update path runs, and the final byte is 255. The matching final byte does not make the full request equal.

7. **Null after mutation.** The initial observation is 2. A dependency changes the captured record’s state to 9, then acquisition returns null. The function returns 9 without direct writes or indirect operations.

8. **Equality established by a dependency.** The initial observation is 2, request is 9, and a dependency changes captured-record state to 9. The non-null path detects equality and returns 2 without an update.

9. **Callback mutation overwritten.** Initial state is 3 and request is zero. An indirect operation writes 5 to captured-record state. If both operations return normally, the function subsequently stores zero and returns 3.

10. **Global replacement during a dependency.** After both initial references have identified one record, a dependency replaces the global pointer while preserving the original record. Later direct reads and writes still address the captured original record.

11. **Distinct initial-reference records, conditionally.** If an execution environment permits the first global reference to capture one valid record and the second to identify another valid record, the initial observation comes from the latter. Subsequent direct accesses use the former. A non-null-object return reports the latter’s initial byte; a null-object return reports the former’s later byte. This example specifies the consequence of distinct reference results, not a supported concurrent-mutation mechanism.

## Uncertainties

The following remain **documented_only**:

- Business meanings of the state, auxiliary bytes, and dependency arguments.
- Whether callers are intended to supply only Boolean requests.
- Any guarantee that the two initial global references identify the same record.
- Physical load count or optimization of those source-level references.
- Defined behavior under concurrent pointer replacement or record mutation.
- Indirect target identities and complete calling conventions.
- Ownership, lifetime, allocation, retention, and release responsibilities.
- Dependency and callback preservation of globals, storage, or dispatch state.
- Reentrancy, thread safety, atomicity, and memory-order guarantees.
- Fault, exception, cancellation, and other nonlocal-exit behavior.

The enabled/disabled interpretation is inferred intent only. Full-width comparison, low-byte storage, ordered writes, dispatch order, and branch-dependent return observations are established behavior.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping.

Established static evidence includes:

- Unsigned 32-bit input and return representations.
- A global-pointer capture followed by a separate global-pointer reference for the initial byte observation.
- Both initial references occurring before the two dependency calls.
- Use of the captured pointer for subsequent direct record accesses.
- A four-byte intermediate dependency result.
- Separate null-object return and non-null-object comparison paths.
- Comparison of an unsigned byte value against the full request.
- Four ordered auxiliary-byte writes before indirect operations.
- A final low-byte state write after those operations.
- Return of the retained initial observation on ordinary non-null-object paths and a later captured-record observation on the null-object path.

No runtime tests have been performed. The source-level pointer references do not prove two physical machine loads or defined concurrent-mutation behavior. Acceptance examples are conditional static deductions, not runtime verification results.
