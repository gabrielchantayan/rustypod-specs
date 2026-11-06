<!-- Trusted provenance: F-b816af9a7d2b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b816af9a7d2b

## Purpose

F-b816af9a7d2b coordinates an indexed lookup, a state-dependent operation, and one of two request paths. The path is selected by whether a status returned by F-2c4fc5b4a966 is zero.

On the zero-status path, it requests an operation identified by `0x14`, conditionally increments a per-index byte counter, and passes a successful result to F-3105e44fc8f9.

On the nonzero-status path, it requests an operation identified by `0x19`, supplying a writable four-byte temporary initialized to 7. A successful result is passed to F-256e8a77aa59.

The represented function does not return normally when the initial lookup or either selected request produces zero. Those cases invoke F-a817d8d58c10, which is represented as non-returning.

These are behavioral descriptions of the supplied static representation. The meanings of the requests, records, status codes, and returned objects are not established.

## Interface

### Inputs

The function accepts four machine-word arguments:

| Position | Contract-level name | Represented width | Visible use |
|---|---|---:|---|
| First | Context word | 32 bits | Passed unchanged to F-2c4fc5b4a966 and, on one path, F-3105e44fc8f9. |
| Second | Record index | Signed 32 bits | Passed unchanged to lookup and state-operation dependencies; used to address two global tables. |
| Third | Unused word | 32 bits | No visible use in this function. |
| Fourth | Initial temporary word | 32 bits | Initially copied into a local four-byte temporary. That temporary is overwritten with 7 before its address is supplied to a dependency. |

The context word is not visibly dereferenced by this function. Its interpretation belongs to its consumers.

The unused word has no visible influence on branching, dependency arguments, or the returned bit pattern. The initial temporary word likewise has no visible influence on those outcomes: the only path that exposes the temporary first replaces its contents with 7.

### Normal result

The result is represented as a 64-bit value assembled from two 32-bit halves:

| Selected path | Upper 32 bits | Lower 32 bits |
|---|---|---|
| Zero status | Zero | Result of F-3105e44fc8f9 |
| Nonzero status | Address of this invocation’s local four-byte temporary | Previously obtained nonzero status from F-2c4fc5b4a966 |

This specifies bit placement, not a memory serialization or a high-level aggregate type. No signed interpretation of the complete result is established.

The nonzero-status result includes an address into the returning invocation’s local storage. The function does not establish that this address remains usable after return. Consumers must not treat the returned upper half as a valid post-return dereferenceable pointer on the strength of this function alone.

### Referenced layouts

Two global table bases are referenced:

- **Record table:** records have a stride of 36 bytes. A one-byte state field is read at record offset 4. A four-byte field is read at record offset 16 on the zero-status success path.
- **Counter table:** entries have a stride of 4 bytes. A one-byte flag is read at entry offset 0. If that flag equals 1, the byte at entry offset 1 is incremented.

The remaining bytes in these records are not accessed directly by this function.

### Preconditions

For a defined normal execution, each address actually accessed must be valid for the corresponding operation:

- The selected record’s state byte must be readable after a successful lookup.
- On zero-status executions, the selected counter entry’s flag must be readable.
- If that flag equals 1, its adjacent counter byte must be readable and writable.
- On zero-status executions with a nonzero request result, the retained record’s four-byte field must be readable.
- Every invoked dependency’s own preconditions must hold.

No index-range check, table-size check, pointer validation, or arithmetic-overflow check is visible here.

## Behavior

### Context acquisition and lookup

The function first copies the fourth argument into its local four-byte temporary.

It then invokes a context-producing dependency with no visible arguments. That dependency has no assigned opaque ID in the supplied map. Four bytes are added to its returned machine-word value, and the adjusted value is passed as the first argument to F-b37968c9f644. The record index is the second argument.

If F-b37968c9f644 returns zero, the function invokes F-a817d8d58c10. No record-state read, state operation, request, or counter access is visibly performed on this path.

A nonzero lookup result is retained as the first argument for the later request.

### Record selection and preliminary state operation

After a successful lookup, the function computes the selected record address from the record-table base and the index, using a 36-byte stride. That computed address is retained for the later offset-16 read.

It reads the state byte at offset 4.

A preliminary invocation of F-2c4fc5b4a966 occurs exactly when the state byte is outside the set:

- 0
- 3
- 4
- 5

This characterization holds whether the represented byte load is interpreted as signed or unsigned. Although the supplied expression uses an unsigned comparison after subtracting 3, the excluded byte values are precisely those listed above.

The preliminary invocation visibly supplies the context word and record index. Its result is discarded.

The function then always invokes F-2c4fc5b4a966 again, this time visibly supplying three arguments: the context word, record index, and the word value 3.

Only the result of this second invocation selects the request path. The earlier state byte is not reread.

### Zero-status path

If the second F-2c4fc5b4a966 invocation returns zero, the function invokes F-cd7e1905512f with six arguments, in this order:

1. The nonzero lookup result.
2. Zero.
3. Zero.
4. `0x14`.
5. Zero.
6. Zero.

After that invocation returns, and before testing whether its result is zero, the function accesses the counter table.

It reads the selected entry’s flag byte at offset 0. If and only if that byte equals 1, it reads the byte at offset 1, adds one, and stores the result back into that byte. At the represented byte-store level, this is an eight-bit wrapping increment; a counter byte of 255 becomes 0.

The counter operation is not conditioned on request success. It can therefore occur even when the request result is zero and the function subsequently enters the non-returning failure path.

After the counter operation:

- If the request result is zero, the function invokes F-a817d8d58c10.
- If the request result is nonzero, the function reads the four-byte field at offset 16 from the previously computed record address.

The successful request result and that freshly read record field are passed to F-3105e44fc8f9 together with the original context word, in this order:

1. Context word.
2. Request result.
3. Four-byte record field.

The function then returns a 64-bit result whose upper half is zero and whose lower half is the result of F-3105e44fc8f9.

No nonzero check is applied to that final dependency result. A zero result from F-3105e44fc8f9 therefore produces a normal all-zero 64-bit return.

### Nonzero-status path

If the second F-2c4fc5b4a966 invocation returns nonzero, the function replaces the local four-byte temporary with the word value 7.

It invokes F-cd7e1905512f with six arguments:

1. The nonzero lookup result.
2. Zero.
3. Zero.
4. `0x19`.
5. Address of the local four-byte temporary.
6. One.

The pointer permits the dependency to access the temporary during the invocation. The supplied representation does not establish whether that dependency reads, modifies, or retains it.

If the request result is zero, the function invokes F-a817d8d58c10.

If the request result is nonzero, it invokes F-256e8a77aa59 with the request result and zero. No result from F-256e8a77aa59 is visibly used.

The function then returns:

- The temporary’s address in the upper 32 bits.
- The saved nonzero status from the second F-2c4fc5b4a966 invocation in the lower 32 bits.

The temporary’s contents are not returned as either half. Any modification of those contents by F-cd7e1905512f does not replace the saved status in the lower half.

This path performs no direct counter-table access and no direct read of the record’s offset-16 field.

## Side effects

### Direct effects

The visible direct writes are:

- Initialization of the local four-byte temporary from the fourth argument.
- Replacement of that temporary with 7 on the nonzero-status path.
- Conditional increment of one byte in the counter table on the zero-status path.

There is no direct write to the record table.

No rollback is visible. In particular, a counter increment performed after a zero request result remains a preceding side effect when F-a817d8d58c10 is invoked.

### Ordering and global state

Dependency effects occur in call order. The function must not be described as if dependencies preserve global tables, record contents, or other state.

The retained record address is computed after lookup and before either state-operation call. The offset-16 field is read later, after the zero-status request and counter handling. Consequently:

- Changes to the contents at the retained record address before the later read can affect the argument supplied to F-3105e44fc8f9.
- Replacing the global record-table base after the address was computed does not cause that later read to select a new record address.

The counter-table base is accessed after F-cd7e1905512f returns on the zero-status path. A dependency’s earlier change to that base can therefore affect the counter entry selected. The representation accesses the base again when the flag equals 1; no atomic snapshot or synchronization guarantee is established.

### Ownership and lifetime

No ownership transfer, allocation, or release contract is established for the lookup result or either request result.

Passing the nonzero request result to F-256e8a77aa59 does not, by itself, prove that this dependency frees or consumes it.

The local temporary belongs to the current invocation. Its address is exposed both as a request argument and as part of the nonzero-status return. Neither use establishes an extended lifetime.

## Dependencies

| Opaque ID | Visible role and use | Not established |
|---|---|---|
| F-b37968c9f644 | Lookup using the adjusted context-provider result and record index. Zero selects failure. | Ownership and meaning of a nonzero result. |
| F-2c4fc5b4a966 | Optional preliminary operation, followed by a required operation with third argument 3. The latter result selects the path. | State-transition meaning, global effects, and the complete argument contract of the preliminary invocation. |
| F-cd7e1905512f | Six-argument request using selector `0x14` or `0x19`. Zero selects failure. | Request semantics, result ownership, and pointer-retention behavior. |
| F-3105e44fc8f9 | Processes the successful zero-status request result and record metadata. Supplies the lower return half. | Result meaning and side effects. |
| F-256e8a77aa59 | Called with the successful nonzero-status request result and zero. | Whether it releases, transfers, records, or otherwise changes the result. |
| F-a817d8d58c10 | Common failure endpoint, represented as non-returning. | Whether it terminates, raises an exception, transfers control, or performs another action. |

The initial no-argument context provider is an additional dependency without an opaque ID in the supplied map. Its result is used only after adding 4. Its semantics remain documented_only.

The preliminary and required invocations of F-2c4fc5b4a966 have different visible argument counts. A missing third argument must not be interpreted as an implicit zero.

## Acceptance examples

These examples are conditional static expectations, not executed tests. They assume invoked dependencies return normally unless identified as non-returning, and that the accessed storage is valid.

1. **Lookup failure:** If F-b37968c9f644 returns zero, F-a817d8d58c10 is invoked. There is no intervening record-state read or state-operation call.

2. **State-operation count:** State bytes 0, 3, 4, and 5 produce only the required F-2c4fc5b4a966 invocation. State bytes 1, 2, 6, and 255 produce the preliminary invocation first, followed by the required invocation.

3. **Successful zero-status request:** If the required state operation returns zero and the `0x14` request returns nonzero, F-3105e44fc8f9 is called after counter handling. If it returns `0x89ABCDEF`, the complete result is `0x0000000089ABCDEF`.

4. **Counter update before failure:** If the required status is zero, the `0x14` request returns zero, and the counter flag is 1, the counter is incremented before F-a817d8d58c10 is invoked.

5. **Counter boundary:** With flag 1 and counter 255, the stored byte becomes 0. With flag 0 or 2, no counter write is performed.

6. **Nonzero-status result:** If the required status is 9 and the `0x19` request succeeds, its writable argument initially contains the four-byte value 7. F-256e8a77aa59 is invoked before return. The lower return half remains 9; the upper half contains the temporary’s address.

7. **Metadata mutation:** If a dependency changes the offset-16 field at the retained record address before its eventual read, the changed field value is passed to F-3105e44fc8f9.

8. **Input independence:** Holding dependency behavior and relevant storage fixed, changing only the third or fourth input word does not change the visible dependency arguments or returned bit pattern. The fourth word’s initial local copy is overwritten before exposure.

## Uncertainties

The following are documented_only:

- Valid index bounds, table lengths, and the intended handling of negative indices.
- Address-calculation behavior when multiplication or addition exceeds the represented range. No checked arithmetic is shown; portable signed-overflow behavior must not be inferred.
- The semantic identities of the state byte, counter flag, counter, and offset-16 field.
- The complete calling convention of the preliminary F-2c4fc5b4a966 invocation.
- Whether dependencies mutate globals, retain pointers, transfer ownership, or perform nonlocal control transfers.
- Whether the pointer-bearing upper return half is an intentional public interface or an artifact of the supplied representation.
- The exact failure mechanism of F-a817d8d58c10.
- Concurrency, synchronization, reentrancy, and signal-interaction guarantees.

Calling the two request paths different “modes” is inferred intent only. Their established distinction is their selector, arguments, subsequent calls, direct table accesses, and returned halves.

## Evidence

The evidence is the supplied static function representation and dependency-ID map. They establish the visible branches, argument values, access widths and offsets, call order, direct writes, and return-half construction described above.

Comments and type renderings do not independently establish runtime behavior, ownership, or application semantics. No implementation inspection beyond the supplied material and no runtime tests have been performed.
