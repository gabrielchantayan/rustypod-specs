<!-- Trusted provenance: F-8b02bf1bee62; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8b02bf1bee62

## Purpose

F-8b02bf1bee62 manages a cached resource associated with a caller-supplied 32-bit selection value. When a non-null cached resource already matches that value, it returns immediately. Otherwise, it invokes a disposal-like operation on the previous resource, obtains a replacement, updates object state, notifies an optional associated object, and updates another property through virtual dispatch.

The replacement path also conditionally invokes a secondary processing sequence involving an object-embedded region, an optional related object, and a separately resolved recipient.

“Resource,” “selection,” “disposal,” and “notification” describe inferred roles, not established type or ownership contracts. The established behavior is the ordering of field accesses, calls, conditional branches, and return-word construction described below.

## Interface

### Inputs

The function has two visible inputs:

1. **Object address.** A pointer to writable, word-addressable object storage. The supplied representation uses 32-bit words and 32-bit pointer-bearing fields.
2. **Requested selection.** A 32-bit integer value. The function stores and compares its full 32-bit representation. No range check or normalization is visible.

The object must provide the following accessible locations on the paths that use them:

| Byte offset | Width | Observed role |
|---|---:|---|
| `0x00` | 32 bits | Pointer to the object’s virtual dispatch table |
| `0x34` | 32 bits | Optional related-object address, captured for secondary processing |
| `0x38` | 32 bits | First argument to F-a9ad53981123 |
| `0x44` | 32 bits | Cached selection value |
| `0x80` | Not established | Start of an embedded region passed by address |
| `0xA0` | 32-bit field, narrowed for the check | Low eight bits tested as a secondary-processing gate |
| `0xE8` | 32 bits | Optional associated-object address |
| `0x1E0` | 32 bits | Cached resource address or zero |

These roles do not establish the full object layout or the size of the embedded region.

### Result

The visible result is a 64-bit value assembled from two 32-bit words:

- **Matching-cache path:** the low word is the cached resource address; the high word is the cached selection.
- **Replacement path:** the low and high words are the final contents of the first two words of a local record. Initially, those words hold the object address and the address of its embedded region at `0x80`, respectively.

Thus, the result is not uniformly a resource pointer. On the replacement path, dependencies receive the local record by address and may alter the words subsequently returned.

Whether the source-level API intentionally exposes a 64-bit scalar, a pair of machine results, or another ABI-specific result convention is **documented_only**. Consumers must not assume a single pointer result from the supplied representation alone.

### Preconditions

There is no visible validation of the object address, dispatch tables, or dependency-provided addresses. Execution requires:

- Valid object storage for every field accessed on the selected path.
- Valid dispatch tables and callable entries wherever virtual calls occur.
- Valid resource and associated-object storage when their respective fields are nonzero.
- Valid global gate storage if evaluation reaches that gate.
- Continued validity of object and local-record arguments for the duration of calls using them.

No concurrency, reentrancy, exception, or callback-lifetime guarantee is established.

## Behavior

### 1. Inspect the existing cache

The function first reads the cached resource field at `0x1E0`.

If it is nonzero, the function compares the cached selection at `0x44` with the requested selection.

- If they match, it immediately returns the cached resource in the low result word and the cached selection in the high result word.
- No replacement lookup, associated-object notification, property update, or secondary processing occurs on this path.
- If they differ, it invokes the function found at byte offset `0x04` in the cached resource’s dispatch table.

The supplied call representation shows no explicit arguments for this resource operation. Its calling convention and any implicit receiver are unknown. It must not be specified as a confirmed destructor or release operation.

After that call returns, the function explicitly stores zero into the cached resource field. It does not explicitly clear the cached selection first.

A zero cached resource bypasses both the selection-match return and the resource operation. In particular, a matching cached selection does not produce the fast path when the cached resource is zero.

### 2. Obtain and install a replacement

The function calls F-a9ad53981123 with:

1. The current 32-bit field at object offset `0x38`.
2. The requested selection.

The `0x38` field is read after any old-resource operation. Therefore, callback changes to that field can affect the lookup.

After F-a9ad53981123 returns, the function performs these explicit stores in order:

1. Store the requested selection at `0x44`.
2. Store the lookup’s 32-bit result at `0x1E0`.

There is no visible rejection of a zero lookup result. A zero result is installed as an empty cached resource while the requested selection remains installed.

The function does not visibly preserve or restore the old resource if lookup fails to return a usable resource.

### 3. Notify the associated object, if present

After installing the replacement, the function reads and retains the associated-object address at `0xE8`.

If that captured address is nonzero, it makes three calls in this order:

1. Invoke the object’s dispatch-table entry at byte offset `0x190`, with the object address followed by the values zero and one.
2. Invoke the object’s dispatch-table entry at byte offset `0x198`, with the object address.
3. Invoke the captured associated object’s dispatch-table entry at byte offset `0xA4`, with the captured associated-object address and the object’s then-current cached resource value.

The second object dispatch-table entry is selected after the first callback returns. The associated-object address, however, is the earlier captured value, not a fresh read of `0xE8`.

Consequently:

- Changing `0xE8` during either object callback does not redirect the third call.
- Changing `0x1E0` during those callbacks can change the resource argument supplied to the associated object.
- Correctness requires that the captured associated object remain valid until its call completes.

No preservation of object fields or global state by these calls is established.

### 4. Update a property derived from the current resource

After the optional notification sequence, the function checks the cached resource field again.

- If the current field is zero, the property argument is the 32-bit value zero.
- Otherwise, the function calls F-004c5654fd0a with the current cached resource and a second machine-level value.

That second value is not a stable, independently specified input:

- Without associated-object notification, the visible representation carries forward the lookup result as this argument.
- With associated-object notification, it uses an additional register value left by the associated-object call.

The semantic meaning and validity requirements of that second argument are **documented_only**. It must not be replaced in the contract with the requested selection or an assumed constant.

The function then selects the object’s dispatch-table entry at byte offset `0x16C` and invokes it with the object address and the property argument. For a nonzero resource, the dispatch entry is selected after F-004c5654fd0a returns.

This property update occurs on the replacement path even when the installed resource is zero, unless an earlier call prevents normal continuation.

### 5. Prepare a local record and evaluate the secondary-processing gates

After the property update, the function initializes a local record with three 32-bit words:

| Relative offset | Initial content |
|---|---|
| `0x00` | Object address |
| `0x04` | Address of the object’s embedded region at `0x80` |
| `0x08` | Integer value one |

It then calls F-3fac671648b8 with no explicit arguments visible in the supplied representation.

Secondary processing occurs only if all three conditions hold, evaluated in this order:

1. F-3fac671648b8 returned nonzero.
2. Narrowing the field at object offset `0xA0` to eight bits produces zero.
3. The byte at a referenced global address is nonzero.

The global byte is not read when either earlier condition fails. The object flag check is not performed when the first condition fails.

The flag condition concerns the low eight bits of the field’s value, not equality of the entire 32-bit field to zero. A field value of `0x00000100`, for example, satisfies this gate.

### 6. Perform secondary processing

When all gates pass:

1. Capture the current 32-bit field at object offset `0x34`.
2. Call F-e57b05520b65 with the address of the local record and two identical addresses, both pointing to the embedded region at object offset `0x80`.
3. If the captured `0x34` value was nonzero, call F-ee8733a08210 with the local-record address and the captured value plus byte offset `0xD0`.
4. Call F-ffe0d92c57b7 with the object address.
5. If that call returns nonzero, use its result as the first argument to F-c9b5d17d8d1e, with the local-record address as the second argument.
6. After that call returns, call F-4d037ecf0f82 with the same captured recipient value and the same local-record address.

The optional related-object address is captured before F-e57b05520b65 executes. Changes to field `0x34` during that dependency do not change the address supplied to F-ee8733a08210.

Likewise, the recipient from F-ffe0d92c57b7 is reused for both subsequent calls. No second recipient lookup or nonzero check occurs between them.

Adding `0xD0` is a 32-bit address calculation in the supplied representation, with no visible overflow or bounds check. A valid execution requires the resulting address to satisfy the callee’s requirements; wraparound is not established as supported behavior.

### 7. Return

The replacement path returns the final first and second words of the local record as the low and high halves of the 64-bit result.

If no dependency modifies those words, the result contains the object address below the embedded-region address. If a dependency modifies either word, its final value is returned instead. No visible local-record cleanup follows the secondary calls.

## Side effects

The function’s explicit persistent writes are:

- Clearing `0x1E0` after the old-resource operation, when that operation is reached.
- Writing the requested selection to `0x44`.
- Writing the lookup result to `0x1E0`.

The matching-cache path contains none of these explicit writes.

Other side effects are delegated. Virtual operations and dependencies may modify object state, the embedded region, the local record, or global state. Their names and apparent roles do not establish preservation guarantees.

No direct allocation, memory copy, or deallocation is visible. The old-resource operation may have lifetime effects, but ownership transfer, reference counting, destruction, and release semantics are not established.

The local record has function-local lifetime. Dependencies receive its address, but there is no evidence establishing that they retain it safely, copy its contents, or refrain from retaining it.

There is no explicit error return, retry, rollback, or recovery path. Zero-valued dependency results control specific branches; they are not converted into a uniform failure result. Exceptional termination and invalid-memory behavior remain unspecified.

## Dependencies

| Dependency | Visible contract at this call site |
|---|---|
| F-a9ad53981123 | Receives the current `0x38` field and requested selection; its 32-bit result is installed as the cached resource. |
| F-004c5654fd0a | Called only for a nonzero current resource; receives that resource and the machine-level second value described above; returns the property argument. |
| F-3fac671648b8 | Called on every replacement path after the property update; a zero result disables secondary processing. |
| F-e57b05520b65 | Receives the local-record address and the embedded-region address twice. Aliasing of those latter arguments is intentional in the visible call. |
| F-ee8733a08210 | Called only for a nonzero captured related-object field; receives the local-record address and the related address advanced by `0xD0`. |
| F-ffe0d92c57b7 | Receives the object address; a nonzero result enables the final pair of calls. |
| F-c9b5d17d8d1e | Receives the resolved recipient and mutable local-record address. |
| F-4d037ecf0f82 | Receives the same recipient and local-record address after F-c9b5d17d8d1e returns. |

The five virtual dispatch sites are identified by their table offsets and argument behavior above. No opaque IDs were supplied for their targets.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

1. **Non-null matching cache.** Given a valid nonzero cached resource and a cached selection equal to the request, the function returns that resource as the low word and that selection as the high word. No dependency or virtual operation described above is called.

2. **Null cache with matching selection.** Given a zero cached resource and an already matching selection, F-a9ad53981123 is still called. The selection alone does not authorize reuse.

3. **Replacement lookup returns zero.** Given a mismatching nonzero resource whose virtual operation returns normally, the cache is cleared before lookup. If lookup returns zero, the requested selection and zero resource are installed. Assuming callbacks leave the resource zero, F-004c5654fd0a is skipped and the `0x16C` property operation receives zero.

4. **Associated-object field changes during notification.** Given a nonzero captured associated object, changing `0xE8` in the first object callback does not change the associated object used for the later `0xA4` call. This example requires the originally captured object to remain valid.

5. **Resource changes during notification.** If a callback changes `0x1E0`, the associated-object call receives the resource present when its argument is evaluated. The subsequent property branch uses the resource present after that call returns.

6. **Eight-bit gate.** With a nonzero F-3fac671648b8 result and nonzero global gate byte, a flag field of `0x00000100` permits secondary processing; `0x00000001` prevents it.

7. **Secondary processing disabled.** If F-3fac671648b8 returns zero, the later gates and secondary dependencies are skipped. With no local-record mutation, the replacement-path result contains the object address and its `0x80` embedded-region address.

8. **Local-record mutation.** If a secondary dependency changes either of the local record’s first two words and later calls preserve that change, the changed word appears in the final result. The original addresses are not unconditionally restored.

9. **No resolved recipient.** If the gates pass but F-ffe0d92c57b7 returns zero, earlier secondary calls still occur as applicable, but neither F-c9b5d17d8d1e nor F-4d037ecf0f82 is called.

## Uncertainties

The following are **documented_only**:

- Source-level types and the intended ABI meaning of the two-word result.
- The semantic identity and ownership of cached resources and associated objects.
- The calling convention and lifetime effects of the old-resource virtual operation.
- The meaning of F-004c5654fd0a’s second argument, especially after associated-object notification.
- The layout, capacity, encoding, and ownership of the embedded region.
- Whether the three-word local record is a public structure, temporary descriptor, or partial representation of a larger construct.
- Memory retention, allocation, release, global mutation, and exceptional behavior inside dependencies.
- Synchronization and reentrancy requirements.
- Address-overflow behavior beyond the visible unchecked 32-bit calculation.

These unknowns do not justify adding validation, cleanup, rollback, or preservation behavior absent from the evidence.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping. Established observations include field offsets, 32-bit word operations, virtual-table offsets, explicit store ordering, conditional call ordering, short-circuit gates, captured versus reread addresses, and two-word result construction.

Resource-management and notification terminology is inferred intent. Dependency internals, ownership contracts, ABI interpretation, and runtime validity are not established.

No runtime tests have been performed.
