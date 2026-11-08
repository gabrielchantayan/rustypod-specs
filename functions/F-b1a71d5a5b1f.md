<!-- Trusted provenance: F-b1a71d5a5b1f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1a71d5a5b1f

## Purpose

F-b1a71d5a5b1f selects a 32-bit value from an indexed collection, conditionally installs it as the receiver’s current value, issues notifications, and optionally updates a secondary object.

Selection proceeds backward from the collection’s final index. The first examined candidate strictly less than the receiver’s current value under **unsigned 32-bit comparison** is selected. If no candidate qualifies, the final collection record is obtained again and selected. This is not a minimum search; collection ordering is not established.

After selection and notifications, the function derives a scalar using the receiver’s then-current value and a mode byte. A nonzero secondary-object reference enables conditional field-group writes, flag updates, and further calls.

These structural effects are established by the supplied representation. Domain meanings for the collection, scalar, tags, and flags remain unknown.

## Interface

**Input:** one receiver address.

**Output:** no explicit return value. Observable results consist of memory accesses, writes, and dependency calls.

The representation uses 32-bit integer words and 32-bit address-bearing fields. All offsets below are byte offsets. Dispatch offsets identify positions within dispatch tables, not object fields.

### Receiver fields

| Offset | Established use |
|---|---|
| `0x00` | Dispatch-table address. |
| `0x20` | Current 32-bit value; read and conditionally written. Selection comparisons interpret it as unsigned. |
| `0x2D` | Explicit one-byte character read, compared with `3`. |
| `0x30` | Address of the collection region passed to F-3bc04b368704. |
| `0x34` | Signed 32-bit collection count. |
| `0x88` | Object address used for the changed-value notification. |
| `0x8C` | Address of an embedded region passed to F-438680ffebfe. |
| `0x94` | Optional secondary-object address; zero skips that object’s update. |

Each collection lookup supplies an address whose 32-bit word at offset `0x04` is the candidate value. No other record layout is established.

### Secondary-object fields

| Offset | Established use |
|---|---|
| `0x00` | Dispatch-table address. |
| `0x34` | Optional associated-object address. |
| `0x48` | 32-bit flags; bit `0x20` is set. |
| `0x54`, `0x60` | First tag pair. |
| `0x58` | First baseline-copy destination. |
| `0x64` | First scalar destination. |
| `0x6C`, `0x78` | Second tag pair. |
| `0x70` | Second baseline-copy destination. |
| `0x7C` | Second scalar destination. |
| `0x80` | Baseline word, also passed by address to F-6d16652f2d03. |
| `0xA0` | **32-bit integer read, followed by conversion to a character value and comparison of that converted value with zero.** |

The access at secondary offset `0xA0` is not represented as a byte dereference. It differs from the explicit byte dereference at receiver offset `0x2D`. Describing the secondary condition as testing the byte at the lowest address would additionally require established character-conversion and endianness assumptions.

The address returned by the secondary dispatch at table offset `0x5C` must support a 32-bit read and write at offset `0x48`.

### Preconditions for ordinary completion

- All accessed receiver, record, secondary-object, and returned-object storage must remain valid through its last use.
- Each directly written field must be writable.
- Collection lookups must return addresses supporting the subsequent record-value accesses.
- Invoked dispatch entries must be callable with the actual machine-level interface.
- On a changed-value path, the receiver’s object at `0x88` must support the dispatch at table offset `0x40`.
- When the associated-object condition reaches the access at secondary offset `0xA0`, the complete 32-bit field must be readable.
- Dependencies must not invalidate storage that the function subsequently accesses.

These are caller and dependency obligations, not checks performed by the function. No bounds checks, allocation checks, explicit error result, or general pointer validation are established.

## Behavior

### Entry and count capture

F-29eca01f3b88 is called first. Its arguments and effects are not established.

The receiver’s signed 32-bit count at `0x34` is then captured. Subsequent changes to that field do not change the scan bound or fallback index for this invocation.

The scan position starts from the captured count and is decremented before its negative check. For zero and negative counts whose decrement is representable, the function returns without collection lookup or later notifications. The entry dependency has already run.

The minimum signed 32-bit count requires separate treatment because its initial decrement is not representable as a signed 32-bit result.

### Backward selection and fallback

For a positive captured count, lookup begins at index `count − 1` and moves downward.

Each lookup calls F-3bc04b368704 with the address of receiver offset `0x30` and the selected index. After that call, the returned record’s word at `0x04` is compared with the receiver’s current word at `0x20`.

Both comparison operands are interpreted as unsigned 32-bit values:

- A candidate strictly less than the current value terminates the scan.
- A candidate equal to or greater than the current value does not terminate it.

If all indices through zero are exhausted, F-3bc04b368704 is called again for index `count − 1`. The record returned by this additional call becomes the selected record.

The current value is read after each lookup rather than captured once before scanning. Lookup side effects can therefore change later comparisons. The fallback lookup can also return a different record or changed contents from the first lookup of the same index.

No ordering between the two operand reads within an individual comparison is established here.

### Conditional current-value update

The selected record’s value and the receiver’s current value are compared for equality.

If equal, the function neither writes the current field nor makes the two changed-value notification calls.

If unequal, the established sequence is:

1. Read the selected record’s value for assignment and write receiver offset `0x20`.
2. Call F-438680ffebfe with the address of receiver offset `0x8C` and a fixed data-object address.
3. Dispatch through table offset `0x40` of the object referenced at receiver offset `0x88`, passing that object and the receiver’s current value.

The current value supplied to the dispatch is obtained after F-438680ffebfe returns. It need not equal the word just installed.

Record values are accessed separately during scan comparison, final equality testing, and assignment when applicable. The function does not establish record immutability or a single captured value shared by those operations.

### Three receiver dispatches

Every execution continuing beyond selection invokes the receiver’s dispatch entry at table offset `0x58` three times, including when the selected and current values were equal.

For description, designate four fixed global words as **A**, **B**, **C**, and **D**. The visible call forms are:

1. Receiver, A, B, followed by additional argument-like values.
2. Receiver, C, D.
3. Receiver and a retained value of C.

The first call’s additional material includes a dispatch-target value and two values without established local initialization. A reliable complete signature cannot be reconstructed from this representation.

The retained C value is captured after the first call and before the second. The third call uses that retained value even if the second call changes C’s global storage.

The dispatch target is obtained through the receiver’s table for each represented invocation. Neither table stability nor preservation of receiver fields or globals across these calls is guaranteed.

### Scalar derivation

After the receiver dispatches, the function explicitly reads the receiver’s mode byte at `0x2D`.

For mode `3`, F-870c4fcf7295 receives:

- First argument: the represented 32-bit arithmetic result of subtracting `76000` from the then-current value and multiplying by `20`.
- Second argument: `1000`.

Its result is increased by `18` to form the scalar.

For every other mode value, the first argument instead subtracts `88000` and multiplies by `14`; the second argument remains `1000`. The helper result is increased by `20`.

The current word is read for this calculation after the receiver dispatches. It is not necessarily the value selected earlier.

The helper’s operation is unknown. Its second argument does not establish division, rounding, saturation, or signedness semantics. The arithmetic is represented at 32-bit width, without an established widened intermediate. Overflow in subtraction, multiplication, or the final addition is unresolved.

Scalar derivation occurs before the optional secondary-object reference is read. A zero secondary reference does not suppress F-870c4fcf7295 or its effects.

### Secondary-object field selection

After scalar derivation, receiver offset `0x94` is read. If zero, the function returns. Otherwise, that secondary-object address is retained.

A 32-bit tag matches when clearing bit `0x20` produces `0x80000002`. Exactly `0x80000002` and `0x80000022` satisfy this comparison.

The first field group has priority:

- Read the tag at `0x54`.
- Only if it matches, read the tag at `0x60`.
- If both match, write the scalar to `0x64`, then read the baseline at `0x80` and write it to `0x58`.

If the first group does not qualify, consider the second:

- Read the tag at `0x6C`.
- Only if it matches, read the tag at `0x78`.
- If both match, write the scalar to `0x7C`, then read the baseline at `0x80` and write it to `0x70`.

At most one group is written. If the first group qualifies, the second group’s tags are not examined. If neither qualifies, neither group receives these writes.

### Remaining secondary-object operations

Whether or not a tag pair qualifies, the function proceeds in this order:

1. Read the 32-bit flags at secondary offset `0x48`, set bit `0x20`, and write the result.
2. Call F-6d16652f2d03 with the retained secondary address and the address of its field at `0x80`.
3. Test the associated-object address at secondary offset `0x34` against zero.
4. Only if that address is nonzero, read the **32-bit integer field** at secondary offset `0xA0`, convert that value to a character value, and compare the converted result with zero.
5. If the converted result is zero, dispatch through the associated object’s table offset `0x128`.
6. Dispatch through the retained secondary object’s table offset `0x5C`, passing that secondary object, and obtain an address.
7. Read the 32-bit flags at returned-address offset `0x48`, set bit `0x40`, and write the result.

The associated-object call has no explicit arguments in the supplied representation; its complete interface is unknown. The returned address from the final dispatch is not checked for zero.

The `0xA0` condition tests the converted character value, not directly whether the entire integer is zero. Exact results for values outside the character type’s representable range require target conversion semantics. A lowest-address-byte interpretation additionally requires an endianness assumption.

All these accesses occur after F-6d16652f2d03, so that dependency’s changes can affect the associated-object condition and subsequent dispatches.

## Side effects

Direct writes established by the representation are:

- Receiver offset `0x20`, on unequal selection.
- One secondary field group, if its tags qualify.
- Secondary offset `0x48`, setting bit `0x20`.
- Returned-object offset `0x48`, setting bit `0x40`.

The flag operations preserve other bits from the word read for that operation. They are not established as atomic operations.

There is no direct write to the secondary field at `0xA0`; its relevant operation is a 32-bit read followed by character conversion.

Dependencies may modify other fields, globals, relationships, records, or memory lifetimes. No dependency is assumed to preserve globals or to allocate, retain, transfer, or release ownership in any particular way.

No explicit allocation or release is shown in this function. Retaining an address for later use does not establish ownership.

Effects occur incrementally. No rollback, transaction boundary, or recovery path is visible. An invalid access or abnormal dependency exit may occur after earlier writes and calls have taken effect.

## Dependencies

| Dependency | Established role |
|---|---|
| F-29eca01f3b88 | Unconditional entry call before count capture. |
| F-3bc04b368704 | Indexed lookup receiving the collection-region address and an index; supplies a record address. |
| F-438680ffebfe | Changed-value notification receiving an embedded-region address and a fixed data address. |
| F-870c4fcf7295 | Receives a computed 32-bit argument and `1000`; its result contributes to the scalar. |
| F-6d16652f2d03 | Receives the secondary address and its baseline-field address. |

Unresolved indirect dependencies are identified by dispatch location: receiver table `0x58`, notified-object table `0x40`, associated-object table `0x128`, and secondary-object table `0x5C`. No opaque function identities are supplied for those targets.

Complete calling conventions, failure behavior, preservation guarantees, and ownership effects are not established.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, assume valid memory, normally returning dependencies, stable record contents, no relevant dependency mutation, and representable arithmetic.

1. **Empty collection.** Count zero causes return after F-29eca01f3b88. No lookup, receiver notification, scalar helper, or secondary-object work follows.

2. **Strict predecessor.** With count three, indexed values `[100, 200, 300]`, and current value `250`, lookup indices are `2`, then `1`. Value `200` is installed before the changed-value notifications.

3. **Equality does not stop scanning.** With the same records and current value `300`, index `2` is skipped as equal; index `1` supplies `200`.

4. **Fallback repeats a lookup.** With those records and current value `100`, lookup indices are `2`, `1`, `0`, then `2` again. The final lookup supplies the selected record.

5. **Equal fallback still permits later work.** One record containing `100` and current value `100` produces two lookups of index `0`. No current-value write or changed-value notifications occur. The three receiver dispatches and scalar derivation still occur.

6. **Unsigned selection.** Current word `0xFFFFFFFF` exceeds candidate `0x00000001` for scan purposes. That candidate terminates the scan.

7. **Mode-three calculation.** Mode `3` and derivation-time current value `76000` give helper arguments zero and `1000`. A helper result of `7` produces scalar `25`.

8. **Other-mode calculation.** Mode `0` and current value `88000` give the same helper arguments. A result of `7` produces scalar `27`.

9. **Absent secondary object.** A zero reference skips secondary accesses but does not skip scalar derivation.

10. **First-group priority.** If both tag pairs qualify, only the first group receives scalar and baseline writes. The second pair is not read.

11. **No qualifying pair.** Group writes are skipped, but both subsequent flag-update stages and intervening calls remain reachable.

12. **Associated-object short circuit.** If the associated reference is zero after F-6d16652f2d03, the `0xA0` field is not read and the associated dispatch is skipped. The final secondary dispatch still occurs.

13. **Character conversion of representable values.** With a nonzero associated reference, integer value zero at `0xA0` converts to zero and permits the associated dispatch. Integer value one converts to nonzero and suppresses it.

14. **Conditional narrowing case.** If the target uses an eight-bit character conversion retaining the low eight value bits, integer `0x00000100` at `0xA0` converts to zero and permits the associated dispatch. This example requires that conversion assumption. It still describes a 32-bit read, not a byte load; identifying those bits with the lowest-address byte additionally requires little-endian layout.

## Uncertainties

- **documented_only — character conversion:** Character signedness and out-of-range integer-to-character conversion semantics are not independently established. Neither is endianness. The 32-bit read at secondary `0xA0` is established; a lowest-address-byte load is not.
- **documented_only — call interfaces:** Complete signatures for the entry call and several indirect calls are unavailable.
- **documented_only — arithmetic:** F-870c4fcf7295’s operation, overflow behavior, result interpretation, and rounding remain unresolved. No clamp is visible.
- **documented_only — minimum count:** Decrementing `−2147483648` is outside representable signed 32-bit arithmetic. The resulting execution is not established.
- **documented_only — mutation and concurrency:** No synchronization, atomicity, or dependency preservation guarantee is supplied.
- **documented_only — domain meaning:** Tag meanings, flag meanings, global-word meanings, and the fixed data object’s contents are unknown.
- **documented_only — lifetime and failure:** Ownership, dependency allocation or release, exceptions, and nonlocal exits remain unknown.

## Evidence

This contract uses static inspection of the supplied function representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence.

Established observations include field widths and offsets, explicit byte access at receiver `0x2D`, the 32-bit integer access followed by character conversion at secondary `0xA0`, comparison forms, branch structure, direct writes, and ordering across explicit calls.

Interpretations such as “notification” describe call placement rather than proven domain semantics. Unavailable dependency behavior and target-specific conversion rules remain explicitly unresolved.

No runtime tests, implementation access, or independent calling-convention verification have been performed.
