<!-- Trusted provenance: F-6fa39a8df0d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6fa39a8df0d4

## Purpose

F-6fa39a8df0d4 attempts to obtain one character from a stream-like object. On success, it advances the object’s cursor and returns the character converted to a 32-bit integer. On a locally detected read failure, it writes a separate status value, calls a diagnostic dependency, and returns zero if that dependency returns normally.

Two access modes are visible:

- **Direct-memory mode:** read one backing byte when the unsigned cursor is below the unsigned extent.
- **Callback mode:** request one unit through an indirect callback and accept only a callback result exactly equal to `1`.

A zero return does not distinguish a successful zero byte from a local read failure. The status output provides a local distinction, subject to aliasing and dependency side effects.

The stream interpretation is inferred intent. The accesses, comparisons, calls, and writes described below are established static evidence. No runtime tests have been performed.

## Interface

### Inputs and result

| Input | Width and established use |
|---|---|
| Object reference | A reference represented in the supplied 32-bit layout. Provides the backing address, extent, cursor, and callback address. |
| Status-output reference | Identifies storage receiving full 32-bit writes: initially zero, then `0x00000055` on the local read-failure path. |
| Auxiliary argument | A 32-bit value with no visible effect on the non-null-object path. On the null-object path, its working value is replaced by a value read from global storage for the third argument to F-540af0155702. |

The returned value is a 32-bit integer. Success converts the temporary read value through a plain character before conversion to the integer result type. Failure returns zero.

The supplied representation does not establish plain-character signedness. Consequently, a successful byte with its high bit set does not have one unconditional numerical result: signed-character and unsigned-character interpretations differ.

### Object layout

The function accesses the following fields in a 32-bit word layout:

| Byte offset | Width | Established role |
|---:|---:|---|
| 0 | 32 bits | Backing address used in direct-memory mode. |
| 4 | 32 bits | Extent, interpreted as unsigned for the direct-memory bounds comparison. |
| 8 | 32 bits | Cursor, interpreted as unsigned for that comparison, passed to the callback, and updated on success. |
| 20 | 32 bits | Callback address; zero selects direct-memory mode. |

Bytes at offsets 12–19 are not visibly accessed. Their meaning is unknown.

The indirect callback receives four arguments:

1. The object reference.
2. The object’s cursor value read for that invocation.
3. The address of a temporary 32-bit read-result cell.
4. The integer value `1`.

The temporary cell is initialized to zero before either read mode is attempted.

### Null-object diagnostic inputs

When the object reference is null, the function prepares three arguments for F-540af0155702:

| Argument position | Provenance |
|---:|---|
| First | A fixed address identifying assertion-related diagnostic text. |
| Second | A 32-bit value loaded from one global data location. |
| Third | A 32-bit value loaded from another global data location. |

The third argument’s global value is read **before** the second argument’s global value. Both reads precede initialization of the temporary read-result cell and the diagnostic call.

These two values are not established as constants. Their contents, types beyond the visible 32-bit loads, and stability across calls are unknown. In particular, the second diagnostic argument is not the caller’s original status-output reference.

### Storage and lifetime preconditions

Ordinary execution requires readable object fields and writable status-output storage. Direct-memory success additionally requires the addressed backing byte to be readable. Callback mode requires a callable callback address and satisfaction of any additional callback requirements.

The function does not validate:

- The status-output reference.
- The object’s allocation size.
- The backing address or its allocation extent.
- The callback address.
- Agreement between the stored extent and accessible backing storage.

Null-object handling additionally reads two global 32-bit data locations before invoking its diagnostic dependency.

No allocation, release, or ownership transfer is visible in this function. The callback receives access to temporary function-local storage; continued validity after this function returns is not established. Dependency retention, allocation, release, and ownership behavior remain unknown.

Independent object, status, and backing storage make their roles independent. Separation is not enforced, and aliasing can change subsequent behavior.

## Behavior

### Entry and null-object handling

For a non-null object, the function initializes the temporary 32-bit read-result cell to zero and proceeds to status initialization. The two null-diagnostic global reads do not occur on this path.

For a null object, the established sequence is:

1. Select the fixed assertion-related diagnostic-text address.
2. Read a 32-bit value from global storage for the third diagnostic argument.
3. Read a 32-bit value from a different global location for the second diagnostic argument.
4. Initialize the temporary read-result cell to zero.
5. Call F-540af0155702 with the text address and the two loaded values.

The order of the two global reads matters. The source does not establish an atomic snapshot of both locations, immutable contents, or synchronization with other writers. The call receives the values obtained by those reads, not a guarantee about the globals’ contents at call time.

No status-output write precedes this diagnostic call.

There is no visible early return after it. If F-540af0155702 returns normally, the function writes zero through the caller’s original status-output reference and then attempts to read the callback field through the still-null object reference.

Null-object handling is therefore not established as a safe, returning error path. It must not be specified as returning zero with status `0x55`. Termination, exceptional control flow, or the outcome of continued null-based access depends on the dependency and execution environment.

### Status initialization and mode selection

After temporary initialization and any returning null-object diagnostic call, the function writes a full 32-bit zero to the caller’s status-output location.

This write precedes:

- Reading the callback-selection field.
- The direct-memory bounds comparison.
- Any backing-byte read.
- Any callback invocation.

Mode selection uses the callback field as observed after this status write. If the status output aliases that field, clearing the status can change which mode is selected.

### Direct-memory mode

A zero callback address selects direct-memory access.

The extent and cursor are compared as unsigned 32-bit values:

- **Cursor below extent:** read one unsigned byte from the backing address plus cursor and place its value in the temporary 32-bit cell.
- **Cursor equal to or above extent:** enter the local read-failure path without reading a backing byte.

All 32-bit field patterns participate in this unsigned comparison. A cursor with its high bit set is treated as a large unsigned value, not rejected by a separate negative-value test.

The comparison establishes only a relationship between stored numeric values. It does not establish valid backing storage, safe address formation, or correspondence between extent and allocation size.

The representation does not establish an order between the individual extent and cursor reads within the comparison. It also supplies no synchronization guarantee against concurrent mutation.

### Callback mode

A nonzero callback address selects indirect access.

The callback receives the object reference, the cursor read for the invocation, the temporary cell’s address, and a requested quantity of one. The function performs no local cursor-versus-extent check before this call.

After the callback returns:

- A result exactly equal to `1` selects success.
- Every other result selects local read failure.

Zero, negative results, and positive results greater than one are all rejected.

The function does not check which bytes of the temporary cell the callback changed. If the callback returns `1` without modifying the cell, the initialized zero is used as the successful character value. If it writes a value but returns anything other than `1`, the function takes the failure path and does not return that value as a character.

Callback changes to object fields or other reachable state are not rolled back. In particular, successful cursor advancement uses the cursor read after the callback returns, not a saved pre-call cursor.

### Successful completion

On either success path, the function:

1. Reads the object’s current cursor.
2. Stores that cursor incremented by one.
3. Returns the temporary value after character conversion and conversion to the 32-bit integer result.

For ordinary, non-overflowing cursor values, the stored cursor is the current value plus one.

The increment is expressed through a signed 32-bit field, whereas the bounds comparison interprets the same bits as unsigned. Signed-overflow behavior is not independently established by the supplied representation. Guaranteed wrapping at the signed boundary is therefore not part of this contract.

The function performs no final status write on success. With independent storage and no external mutation, direct-memory success leaves status zero. Callback success does not guarantee a zero final status if the callback can reach and alter that storage. Aliasing can also make the cursor write alter the status output.

### Local read failure

Direct-memory bounds failure and a callback result other than `1` share this sequence:

1. Write `0x00000055` to the 32-bit status-output location.
2. Obtain the object’s cursor and extent for diagnostic arguments.
3. Call F-4457a9a44a68 with a fixed read-failure diagnostic-text address, followed by the cursor and extent values.
4. Return zero if that dependency returns normally.

Both diagnostic field reads follow the status-error write and precede the call. Their order relative to each other is not established.

The failure path performs no explicit cursor increment. An unchanged cursor is nevertheless not guaranteed: callback effects, status aliasing, or diagnostic dependency effects may alter it.

The final status is likewise not unconditionally `0x55`. That value is explicitly written before the diagnostic call, but the dependency may modify reachable storage. If the dependency does not return normally, the zero return is not reached.

## Side effects

The locally established externally relevant effects are:

- On null-object entry, two ordered 32-bit global reads and an assertion-related diagnostic call.
- A 32-bit status clear before mode selection, if execution reaches that point.
- One backing-byte read on direct-memory success.
- One indirect callback invocation in callback mode.
- A cursor write on success.
- A 32-bit status-error write and diagnostic call on local failure.

The function contains no visible writes to the two null-diagnostic global locations. That does not establish their immutability or preservation by dependencies.

Aliasing makes access order observable:

- Status aliasing the cursor causes the initial clear to replace the cursor before reading.
- Status aliasing the callback field can replace a previously nonzero callback address with zero and select direct-memory mode.
- Status aliasing the extent can make a request fail after clearing the extent.
- Status aliasing cursor or extent on failure can change the values subsequently passed to F-4457a9a44a68.
- Status overlapping backing storage can change the byte later read.
- Status aliasing the cursor can be changed again by the successful cursor increment.

These are consequences of the visible accesses, not evidence that overlapping storage is intended.

No global-state preservation, rollback, memory-ownership, or synchronization guarantee is established for either fixed dependency or the callback.

## Dependencies

### F-540af0155702

Called only on the null-object path.

Its first argument is a fixed assertion-related diagnostic-text address. Its second and third arguments are values loaded from two global 32-bit data locations. The third argument’s value is loaded first, followed by the second argument’s value; both reads occur before temporary-cell initialization and before the call.

Neither loaded value is the caller’s ordinary argument merely forwarded unchanged. Their contents and semantic roles beyond diagnostic use are unresolved.

Whether this dependency returns, terminates execution, raises an exception, produces external output, retains references, changes globals, or allocates or releases memory is unknown. Non-returning assertion behavior is inferred intent, not an established guarantee.

### F-4457a9a44a68

Called after status is set to `0x55` on either local failure route.

It receives a fixed read-failure diagnostic-text address and the then-read cursor and extent values. Its return value is unused.

Output destination, formatting, control-flow effects, memory effects, and global-state effects are unknown.

### Indirect read callback

Selected by the object’s callback field rather than a fixed dependency identity.

The locally enforced success criterion is an integer result exactly equal to `1`. Writable temporary storage is supplied, initially zero. The function does not independently enforce transfer size, extent rules, object preservation, ownership rules, or reference-retention restrictions.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless otherwise stated, storage is valid and independent, dependencies return normally without relevant mutation, there is no concurrent mutation, and arithmetic avoids unresolved overflow cases.

1. **Direct-memory success.** With callback zero, extent `3`, cursor `1`, and addressed byte `0x41`, status is cleared, cursor becomes `2`, and the result is `65`.

2. **Successful zero byte.** With cursor below extent and addressed byte zero, the result is zero, status remains zero, and the cursor advances. A zero result alone does not indicate failure.

3. **Cursor at extent.** With callback zero and cursor and extent both `3`, no backing byte is read. Status is written as `0x55`, F-4457a9a44a68 is called with cursor and extent both `3`, and the function returns zero. Under the stated conditions, cursor remains `3`.

4. **Unsigned comparison.** With callback zero, cursor bits `0xFFFFFFFF`, and extent `3`, bounds failure occurs without a backing-byte read. The cursor is not treated as a negative offset that passes the comparison.

5. **Callback beyond extent.** With a callback installed and cursor equal to extent, the callback is still invoked. If it writes `0x42` and returns `1`, the result is `66` and the post-callback cursor advances.

6. **Callback changes cursor.** If the callback receives cursor `7`, changes the object cursor to `20`, writes `0x43`, and returns `1`, this function stores cursor `21` and returns `67`.

7. **Rejected callback result.** If the callback writes `0x41` but returns `2`, status is written as `0x55`, the failure diagnostic is called, and the result is zero.

8. **Unwritten successful cell.** If the callback returns `1` without modifying the temporary cell, the result is zero and the current cursor advances.

9. **High-bit character.** A successful value whose character-sized portion is `0xFF` yields `-1` under signed-character conversion or `255` under unsigned-character conversion. Selecting one requires additional target evidence.

10. **Null diagnostic provenance and order.** Suppose the first global read yields `17` and the subsequent global read yields a 32-bit value represented here as `Q`. F-540af0155702 receives the fixed text address, then `Q`, then `17`. Both reads precede temporary initialization. No status clear occurs before the call. These values are sampled global contents, not fixed diagnostic constants.

11. **Returning null diagnostic.** If F-540af0155702 returns normally, the function next clears the original status output and attempts null-based object access. A clean error return is not an acceptance requirement.

## Uncertainties

The following remain **documented_only**:

- Plain-character signedness and high-bit character results.
- Signed cursor-increment overflow behavior.
- Address arithmetic, invalid-address handling, and memory-fault outcomes.
- Contents, intended types, mutability, initialization, and synchronization of the two null-diagnostic global locations.
- Semantic interpretation of the loaded diagnostic values beyond their argument positions.
- Dependency return, termination, output, retention, allocation, release, and global-state behavior.
- Callback requirements beyond the visible arguments and exact-success result.
- Thread safety, reentrancy, and concurrent mutation behavior.
- Meanings of object fields not accessed here.

The fixed diagnostic-text address is distinct from the two dynamically loaded diagnostic values. Nothing in the supplied function establishes that those globals contain fixed values or remain unchanged across invocations.

Interpreting the object as a stream and callback success as a transferred-unit count is consistent with the behavior, but adds no enforcement.

## Evidence

Established static evidence consists of the supplied function representation: object-field offsets; the ordered null-path global loads; temporary initialization after those loads; diagnostic argument provenance; status writes of zero and `0x55`; unsigned direct-mode comparison; one-byte direct load; four-argument callback invocation; exact comparison with `1`; success-only cursor increment; and final character conversion.

The supplied material does not include dependency implementations, callback implementations, authoritative target character semantics, global-value definitions, or runtime observations. No runtime tests have been performed.
