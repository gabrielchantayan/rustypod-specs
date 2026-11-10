<!-- Trusted provenance: F-c7dac3ee6298; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c7dac3ee6298

## Purpose

F-c7dac3ee6298 conditionally reuses a stored 32-bit result associated with six input key components. When an enabled entry matches all six components, it returns that result without invoking dependencies. Otherwise, it obtains an auxiliary object, invokes operations selected by the inputs, updates the entry, captures its result, invokes a final dependency, and returns the captured value.

The comparisons, operations, and updates are established static behavior. Interpreting this arrangement as a single-entry cache is inferred intent. The domain meanings of the operations and result are unknown.

Four key components are stored as bytes but compared against full-width incoming values. Consequently, an update does not guarantee that a later identical request will match. Conversely, distinct full-width requests can produce identical stored key bytes.

## Interface

### Inputs

The supplied representation exposes seven inputs:

| Input | Represented width and observable role |
|---|---|
| State address | A 32-bit address-bearing value identifying the entry’s storage. |
| First coordinate | Signed 32-bit value used in equality comparison, an indirect call, and storage without narrowing. |
| Second coordinate | Signed 32-bit value used in equality comparison, an indirect call, and storage without narrowing. |
| Pointer-valued key | Pointer in the supplied 32-bit model; used for equality, a null/non-null branch, and low-byte storage. |
| Result selector | Unsigned 32-bit value selecting the result-producing operation. |
| Preparation selector | Unsigned 32-bit value selecting an earlier optional operation. |
| Additional value | Unsigned 32-bit value passed to an indirect operation and included in the stored key. |

“Coordinate,” “preparation,” and “result” describe roles, not established domain types. The original declarations and calling convention are **documented_only** unknowns.

The pointer-valued key is not directly dereferenced by this function. Its nonzero status controls one operation on the separately obtained auxiliary object.

### Return value

The function returns a 32-bit quantity whose signedness and semantic type are not established.

Its source is:

- The stored result on an enabled, fully matching entry.
- The return value of a selected result-producing operation, written to the result field and subsequently read back.
- The result field’s contents after earlier dependencies when no result-producing operation is selected.

On every normally completing nonmatching path, the return value is captured before F-54033250428c is called. Changes made by that final dependency do not cause a subsequent result-field read.

### State layout

Offsets below are byte offsets from the state address.

| Offset | Access width | Role |
|---|---:|---|
| 0xB1 | 1 byte | Enabled marker; zero disables lookup, any nonzero value enables it. |
| 0xB4 | 4 bytes | Stored first coordinate. |
| 0xB8 | 4 bytes | Stored second coordinate. |
| 0xBC | 1 byte | Low eight bits of the pointer-valued key. |
| 0xBD | 1 byte | Low eight bits of the result selector. |
| 0xBE | 1 byte | Low eight bits of the preparation selector. |
| 0xBF | 1 byte | Low eight bits of the additional value. |
| 0xC0 | 4 bytes | Stored result. |

No direct access to the intervening bytes is established. Byte order for the four-byte fields is not independently specified by the supplied representation.

### Preconditions and arithmetic

The state address must permit every access reached on the selected path. A matching path directly requires readable storage only. A nonmatching path requires writable marker and key fields, a readable result field, and a writable result field if a result-producing operation is selected.

The auxiliary object must remain suitable for each reached indirect dispatch. The body does not check for a null object, invalid dispatch-table address, or invalid target.

Address offsets denote effective memory locations; the representation does not establish portable behavior for invalid addresses or overflowing address calculations. There is no coordinate arithmetic. Selector decisions use the complete unsigned 32-bit inputs. The four narrowed key writes retain only the low eight bits, equivalent to reduction modulo 256 in the represented model.

## Behavior

### Ordered lookup

The first direct state access reads the enabled marker.

If it is zero, the function skips all stored-key comparisons and proceeds to the nonmatching path.

If it is nonzero, the function compares the following fields in order, stopping at the first mismatch:

1. Stored first coordinate against the incoming first coordinate.
2. Stored second coordinate against the incoming second coordinate.
3. Stored pointer-key byte, zero-extended and interpreted as a pointer-valued quantity, against the full incoming pointer-valued key.
4. Stored result-selector byte, zero-extended, against the full unsigned result selector.
5. Stored preparation-selector byte, zero-extended, against the full unsigned preparation selector.
6. Stored additional-value byte, zero-extended, against the full unsigned additional value.

If all comparisons succeed, the function reads and returns the four-byte result field. It performs no direct state writes and invokes no dependencies.

Incoming values are not narrowed for comparison. A represented pointer value above 255 cannot match the stored pointer-key byte. Likewise, any of the three unsigned scalar key components above 255 cannot match its corresponding stored byte. A null pointer-valued key can match a zero byte. These rules concern equality, not whether the pointer denotes accessible storage.

### Auxiliary-object acquisition

After a disabled entry or any mismatch, the function calls F-be45cd43f92d with the single explicit argument zero. Its return supplies the auxiliary object.

An initial local assignment from the pointer-valued key is overwritten by this returned object before any dependency receives that local slot. The key therefore does not supply the object used for dispatch.

If the incoming pointer-valued key is nonzero, the function next invokes the auxiliary object’s operation at dispatch-table byte offset 0x38. A null key skips that operation.

This invocation has no displayed explicit arguments. That establishes neither a zero-argument binary interface nor delivery of the pointer-valued key to the operation.

### Optional preparation operation

The full preparation selector chooses the following dispatch-table offset:

| Preparation selector | Offset |
|---:|---:|
| 1 | 0xEC |
| 2 | 0xD8 |
| 3 | 0xDC |
| 4 | 0xE0 |
| 5 | 0xE4 |
| 6 | 0xF8 |
| 7 | 0xF0 |
| 8 | 0xDC |
| 0 or any other value | No preparation operation |

Selectors 3 and 8 select the same dispatch position. Any return value from this stage is not directly used by F-c7dac3ee6298. No explicit arguments are displayed for the selected call.

Unsupported preparation selectors are not rejected; they simply omit this stage.

### Two unconditional operations

Every nonmatching execution that reaches this point invokes, in order:

1. The operation at offset 0x5C, with the auxiliary object, first coordinate, and second coordinate as explicit arguments.
2. The operation at offset 0x100, with the auxiliary object and the full unsigned 32-bit additional value as explicit arguments.

The additional value is not narrowed before being passed.

Each dispatch obtains a table address from the auxiliary object and a target at the specified byte offset. Targets are resolved when their calls are reached, rather than all being captured before the sequence. Earlier dependencies may alter the object or its dispatch table; preservation is not established.

### Result-producing operation

The full result selector controls the next stage:

| Result selector | Dispatch sequence |
|---:|---|
| 1 | 0x108 |
| 2 | 0x128 |
| 3 | 0x12C |
| 4 | 0x10C |
| 5 | 0x110 |
| 6 | 0x114 |
| 7 | 0x118 |
| 8 | 0x11C |
| 9 | 0x1C4 |
| 10 | 0x64, then 0x110 |
| 0 or any other value | No result-producing operation |

For selectors 1 through 9, the selected operation’s 32-bit return value is written to the result field.

For selector 10, the operation at 0x64 runs first. Its return is not directly used. After it returns, the target at 0x110 is resolved and invoked; that second operation supplies the value written to the result field.

These calls have no displayed explicit arguments. Their complete calling conventions remain unknown.

For zero and all selectors outside 1 through 10, the function makes no direct result-field write at this stage. It does not reject the request, synthesize an error, or initialize a replacement result. The result field can nevertheless have changed through dependency side effects.

### Entry update and completion

After the result-selection stage, the function performs these direct writes in order:

1. Set the enabled marker to one.
2. Store the first coordinate.
3. Store the second coordinate.
4. Store the pointer-valued key’s low eight bits.
5. Store the result selector’s low eight bits.
6. Store the preparation selector’s low eight bits.
7. Store the additional value’s low eight bits.

It then reads the result field, preserving that 32-bit value for return. Next it calls F-54033250428c with the address of the local slot containing the auxiliary object. If that dependency returns normally, the function returns the captured value without rereading the result field.

The marker is written before the complete new key. The body establishes no atomic publication, locking, or protection against reentrant or concurrent observation.

## Side effects

### Direct state effects

The matching path only performs the ordered lookup reads and the final result read.

On a nonmatching path, a selected result-producing operation is followed by a direct result-field write. The function then writes the enabled marker and all six stored key components, including when no result-producing operation was selected.

Before the pre-update dependency calls finish, this function does not directly clear the marker or overwrite the stored entry. **This does not establish preservation of the entry’s existing contents:** any of those dependencies may modify it through aliases. In particular, an unsupported result selector leaves the result field directly unwritten by this function, not necessarily unchanged overall.

The final dependency runs after the direct entry update and result capture. It may make further changes through aliases; those changes do not alter the already captured return value.

### Ownership and dependency effects

F-be45cd43f92d returns the auxiliary object. F-54033250428c later receives the address of the slot holding it.

This suggests acquisition and finalization, but does not establish allocation, ownership transfer, reference counting, or release. F-54033250428c must not be assumed to free the object, and F-be45cd43f92d must not be assumed to return fresh or exclusively owned storage.

No dependency is assumed to preserve global state, the caller’s state entry, the auxiliary object, or its dispatch table. The effects of receiving and potentially changing the auxiliary-object slot are unknown.

### Errors and interrupted execution

There is no explicit error return, validation branch, rollback, or recovery mechanism in the supplied body.

A dependency that does not return normally prevents the remaining sequence from being established. Failure after the entry update does not trigger any visible rollback. Invalid-memory behavior, exception handling, and nonlocal control transfers are **documented_only** unknowns.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-be45cd43f92d | Called once on a nonmatching path with explicit argument zero; its return supplies the auxiliary object. |
| F-54033250428c | Called after direct entry update and result capture, with the address of the auxiliary-object slot. |

Other dependencies are indirect operations identified only by dispatch-table byte offsets in this contract. Their opaque function IDs are unavailable; offsets identify dispatch positions, not stable function identities.

In the supplied 32-bit model, dispatch uses the table address read from the auxiliary object’s first word. The selected offset is added to that table address to locate the operation target. No evidence establishes that two resolutions at the same offset must yield the same target after intervening dependency calls.

For calls without displayed explicit arguments, implicit receiver handling and argument recovery remain **documented_only** unknowns.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example says otherwise, dependencies return normally, provide valid dispatch state, and do not modify the caller’s entry through aliases.

### Enabled complete match

Given a nonzero marker, equal coordinates, and stored bytes equal to the complete incoming pointer-valued key and three unsigned scalar inputs:

- The stored result is returned.
- Neither named dependency is called.
- No indirect operation or direct state write occurs.

A marker of two enables lookup just as a marker of one does.

### Early coordinate mismatch

Given a nonzero marker and a differing first coordinate:

- The stored second coordinate and later stored key fields are not read during lookup.
- The function proceeds to auxiliary-object acquisition and the nonmatching sequence.

Later operations may access memory independently; the lookup’s short circuit does not constrain dependency behavior.

### Disabled entry with a supported selector

Given a zero marker and result selector 2, suppose the operation at 0x128 returns 0x12345678:

- Stored key fields are not compared.
- The selected result is written to offset 0xC0.
- The marker and key are updated.
- The result is captured before the final dependency.
- The returned value is 0x12345678.

### Unsupported selector and dependency mutation

Given a nonmatching request with result selector zero and an initial result of 0xAABBCCDD:

- No result-producing operation is selected.
- The marker and key are nevertheless updated.
- Under the preservation condition above, the returned value is 0xAABBCCDD.

If an earlier dependency instead changes the result field to 0x11223344 and it remains so until capture, the function returns 0x11223344. The unsupported selector does not preserve the initial result against aliased writes.

### Full-width value cannot match its narrowed storage

Given additional value 256 on a nonmatching path:

- The operation at 0x100 receives 256.
- The stored additional-value byte becomes zero.
- A later otherwise-identical request cannot match that byte, because zero differs from 256.

The same asymmetry applies to the other narrowed key components.

### Different requests sharing stored bytes

A nonmatching request with result selector 257 stores result-selector byte one but selects no result-producing operation.

A subsequent request with result selector one can match the entry if every other comparison succeeds. Such a hit returns the stored result without invoking the operation ordinarily selected by one.

### Selector ten and target changes

For result selector ten, the operation at 0x64 precedes the operation at 0x110. Only the second return is directly stored.

If the first operation changes the dispatch-table entry for 0x110, the subsequent target resolution uses the resulting table state, provided it remains valid.

### Final dependency changes the result

If F-54033250428c changes the stored result after capture and returns normally, the function still returns the earlier captured value. The returned result and the entry’s post-call result may differ.

## Uncertainties

### Inferred intent

The enabled marker, six-component comparison, stored result, and early return support a single-entry-cache interpretation. They do not establish that the key is semantically complete or that reuse is correct for the underlying operations.

The narrowed fields may be intended for small enumerations or values. No range enforcement proves that intent.

### Documented_only unknowns

The evidence does not establish:

- Domain meanings of the inputs, auxiliary object, or result.
- Original declarations, full calling conventions, or implicit arguments.
- Allocation, borrowing, sharing, ownership, or release rules.
- Whether dependencies access the entry through aliases or modify shared storage.
- Thread safety, reentrancy, or permitted concurrent access.
- Dependency-specific errors or failure guarantees.
- Required initialization of the result field.
- Portable behavior outside the supplied address and pointer model.
- Whether an unsupported selector’s retained result is meaningful for the newly stored key.

These unknowns are not resolved by assuming conventional object, cache, or cleanup semantics.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping.

Established evidence covers access widths and offsets, short-circuit lookup order, full-width selector decisions, low-byte storage, dispatch mappings, displayed explicit arguments, direct write order, result capture before the final call, and paths without a direct result-field write.

The representation establishes the absence of direct entry writes before the pre-update dependencies complete. It does not establish that dependencies leave the entry unchanged.

The cache interpretation and acquisition/finalization interpretation are inferences. Callee semantics, ownership, aliasing effects, and unrecovered interface details remain unresolved.

No runtime tests have been performed.
