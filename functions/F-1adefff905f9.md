<!-- Trusted provenance: F-1adefff905f9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1adefff905f9

## Purpose

F-1adefff905f9 performs a handle-producing operation on a resource identified by an opaque 32-bit input. Its visible behavior includes resolving a containing resource, looking up or conditionally creating an entry, enforcing selected access restrictions, optionally performing a reset-like operation, and initializing a handle record.

The resource model resembles opening a named file, with optional creation and truncation. Those names describe **inferred intent**, not established dependency semantics. The contract below uses “lookup,” “creation,” and “reset” only as labels for the observable call sequences.

Success returns the nonnegative handle obtained from a dependency and requests status value zero. Failure returns signed −1. Most failures also request a specific nonzero status; the earliest failure does not explicitly set one.

## Interface

### Inputs

| Input | Width | Established use |
|---|---:|---|
| Resource identifier | 32 bits | Passed unchanged to F-91c65c9d4f51 and F-658f097ae4e8. Its representation and validity rules are unknown. |
| Options | Unsigned 32 bits | Controls access classification, creation, exclusion, reset, and state-bit updates. Its low 16 bits are recorded on success. |
| Auxiliary input | 32 bits | No visible behavioral use and no visible forwarding to a dependency. |

The options are interpreted as follows:

| Mask | Observable meaning |
|---|---|
| `0x00000003` | If both bits are zero, selects the zero-access class. Any nonzero combination selects the nonzero-access class. |
| `0x00000004` | Sets bit 2 in a successful entry’s 32-bit state field. |
| `0x00000100` | Enables the creation attempt when lookup returns null. |
| `0x00000200` | Requests the reset sequence for an eligible existing entry. |
| `0x00000400` | Together with `0x100`, rejects an otherwise eligible existing entry. |
| `0x00000800` | Sets bit 1 in the state field, but only for the nonzero-access class. |

Other option bits do not affect visible decisions. Bits 0–15, including otherwise unexamined bits, are stored in the handle record; bits 16–31 are not stored there.

### Output

The return is a signed 32-bit value:

- Success: the nonnegative result previously obtained from F-c63d695765b6.
- Failure: −1.

The function does not return an independent resource pointer. The successful handle is associated with a record populated through F-667717e3fb3e.

### Memory layouts used

The visible implementation uses 32-bit pointer-like values and byte offsets.

**Handle record**

| Offset | Width | Visible access |
|---:|---:|---|
| `+0` | 32 bits | Initially written as zero; later written with the selected entry address. |
| `+4` | 16 bits | Written with the low 16 option bits on success. |
| `+8` | 32 bits | Written as zero on success. |
| `+32` | 32 bits | Written with the result of F-56c9e29af49d on success. |

The write at `+4` is only 16 bits. It does not establish initialization of bytes `+6` and `+7`.

**Selected entry**

- A 32-bit word at `+0` supplies a backing-object value to several dependencies.
- A 32-bit word at `+4` supplies the address of an associated state structure.

**Associated state structure**

| Offset | Width | Visible access |
|---:|---:|---|
| `+11` | 8 bits | Bit 0 may prevent nonzero-access use of an existing entry. |
| `+28` | 32 bits | Saved, cleared, and conditionally restored during reset. |
| `+36` | 32 bits | Compared with exactly 1 during successful finalization. |
| `+40` | 32 bits | Cleared conditionally, then modified by option-dependent bitwise OR operations. |

A backing object also supplies a 16-bit value at offset `+120` during successful reset processing.

Required record sizes, pointer validity, alignment, and dependency-produced buffer contents are preconditions of normal execution. No visible checks establish them.

## Behavior

### 1. Establish the initial context and allocate a handle

The function first calls F-91c65c9d4f51 with the resource identifier.

- A negative result immediately returns −1.
- This path does not explicitly call F-038b46411c0c.
- No explicit cleanup call is made for that negative result.

For a nonnegative result, the function calls F-c63d695765b6.

- A negative result causes F-038b46411c0c to be called with `0x18`, followed by return −1.
- This allocation-failure path does **not** call F-591e5dd605b9 for the previously acquired context. Whether that context requires such a call is unknown.

For a nonnegative handle, F-667717e3fb3e is called with that handle and zero. The returned address is treated as a writable handle record, without a null check. The record’s first 32-bit word is cleared.

F-6443787f9616 is then called with the initial context. Its result, if any, is ignored.

### 2. Resolve and validate the containing resource

F-658f097ae4e8 receives three distinct temporary storage areas and the original resource identifier. Their visible capacities are:

- 260 bytes for the first output;
- 256 bytes for the second output;
- 4 bytes for the third output.

These areas are not visibly initialized before the call.

If F-658f097ae4e8 returns zero, processing fails with status `2`.

Otherwise:

1. F-1cb585febd84 receives the second output.
2. Its result and that same output are passed to F-ddcbbbbc4a8e.
3. A zero result from F-ddcbbbbc4a8e causes failure with status `2`.
4. F-8dfcd5b8f35c receives the first output.
5. A zero result causes failure with status `2`.
6. F-e135feff4093 receives that nonzero containing-resource value.
7. A zero result causes failure with status `2`.
8. F-3dacd4ff9d73 receives the containing-resource value.
9. A nonzero result causes failure with status `2`.

These tests short-circuit in the listed order. In particular, the final predicate is not called if the preceding predicate returns zero.

The containing-resource value, once nonzero, is retained for later cleanup even if a subsequent check fails.

### 3. Look up an entry

F-d05b3803629b is called with six arguments:

1. zero;
2. the containing-resource value;
3. the second temporary output;
4. the third temporary output;
5. zero;
6. the 32-bit all-ones value, `0xffffffff`.

#### Null lookup result

If lookup returns null:

- Without option `0x100`, fail with status `2`.
- With `0x100` and the zero-access class, fail with status `0x0d`.
- With `0x100` and the nonzero-access class, call F-53f6a8475775 with the containing-resource value and the second and third temporary outputs.
  - A null result fails with status `0x1c`.
  - A nonnull result is stored in the handle record and proceeds directly to successful finalization.

The newly produced entry bypasses the existing-entry predicates, the existing-entry exclusion check, the byte-level access check, and the reset sequence. Consequently, option `0x200` does not cause this function to reset a newly produced entry.

#### Nonnull lookup result

The entry address is written into the handle record **before** eligibility checks.

F-e135feff4093 is called with the entry. Only if it returns zero is F-3dacd4ff9d73 called with the entry. To proceed, both results must be zero. Otherwise, failure status is `0x0d`.

For an eligible existing entry:

1. If both `0x100` and `0x400` are set, fail with status `0x11`.
2. Otherwise, for the nonzero-access class, read the byte at associated-state offset `+11`.
3. If that byte has bit 0 set, fail with status `0x0d`.
4. The zero-access class does not perform this byte read.

The exclusion test is specifically the conjunction of `0x100` and `0x400`. Setting `0x400` alone does not trigger status `0x11`.

### 4. Optionally reset an existing entry

An eligible existing entry proceeds directly to finalization if `0x200` is clear.

If `0x200` is set, the following sequence occurs, including for the zero-access class:

1. Call F-40e4fc419124 with the entry’s backing-object and associated-state values; retain its 32-bit result.
2. Read and retain the 32-bit field at associated-state offset `+28`.
3. Call F-27e49f66c358 with the backing-object value, associated-state value, and zero.
4. Write zero to associated-state offset `+28`.
5. Call F-cb9b1812bac7 with the entry address, 1, and 1.

If F-cb9b1812bac7 returns nonzero:

- Call F-01a422616ce4 with the backing-object value and the result retained from F-40e4fc419124.
- Read the backing object’s 16-bit value at `+120` and pass it to F-052651f0e13e.
- Proceed to successful finalization.

If F-cb9b1812bac7 returns zero:

- Call F-27e49f66c358 with the backing-object value, associated-state value, and the retained result from F-40e4fc419124.
- Restore the saved 32-bit value to associated-state offset `+28`.
- Fail with status `0x0d`.

This is only a visible restoration sequence, not evidence of full rollback. Dependency side effects are unknown. On successful reset, there is no explicit restoration of the saved `+28` value; intervening dependencies may also modify it.

### 5. Finalize a successful entry

Finalization performs the following visible operations in order:

1. Read the selected entry’s associated-state address.
2. Read its 32-bit field at `+36`.
3. If that value is exactly 1, write zero to the state field at `+40`.
4. For the nonzero-access class, OR bit 0 into the field at `+40`.
5. For the nonzero-access class with option `0x800`, additionally OR bit 1 into that field.
6. If option `4` is set, OR bit 2 into that field.
7. Write the low 16 option bits to handle-record offset `+4`.
8. Write zero to handle-record offset `+8`.
9. Call F-56c9e29af49d using:
   - the selected entry’s first 32-bit word; and
   - the 32-bit word at offset `+4` of the entry currently referenced by the handle record.
10. Store its result, without checking for failure, at handle-record offset `+32`.
11. Call F-e657f684a3c8 with the handle-record address.
12. Call F-66819ba2201a with the containing-resource value.
13. Call F-591e5dd605b9 with the initial context.
14. Call F-038b46411c0c with zero.
15. Return the allocated handle.

The second argument to F-56c9e29af49d is obtained through the handle record, not simply assumed to equal the earlier associated-state value. Intervening dependencies are not established to preserve either record.

When the `+36` field is not 1, existing state bits are preserved except for the explicitly ORed bits. When it is 1, the explicit clear occurs before those OR operations.

### 6. Common failure cleanup

After successful handle allocation, ordinary failure paths perform:

1. F-b3f16c6113a2 with the allocated handle.
2. F-66819ba2201a with the containing-resource value, but only if that value is nonzero.
3. F-591e5dd605b9 with the initial context.
4. F-038b46411c0c with the selected failure status.
5. Return −1.

No explicit clearing of the handle record precedes this cleanup. In particular, an existing entry may already have been stored there.

## Side effects

Established direct writes include handle-record initialization, entry association, option storage, result storage, state-bit changes, and the reset-related writes described above.

Dependency calls may have additional effects. Their ownership, allocation, synchronization, persistence, and status behavior are not established here.

The visible lifecycle has these boundaries:

- The allocated handle is returned only after finalization.
- On ordinary post-allocation failure, it is passed to F-b3f16c6113a2.
- A nonzero containing-resource value is passed to F-66819ba2201a on both success and ordinary failure.
- The initial context receives F-6443787f9616 before resolution and F-591e5dd605b9 during later cleanup, except on handle-allocation failure.
- There is no separate, explicit entry-disposal call.

It is unknown whether these dependencies acquire, release, retain, or merely inspect their arguments. The temporary storage is function-local; whether a callee retains pointers into it is unknown.

## Dependencies

Opaque IDs identify dependencies without asserting their internal contracts.

| ID | Observed role |
|---|---|
| F-91c65c9d4f51 | Produces the initial signed context result. |
| F-c63d695765b6 | Produces the signed handle result. |
| F-667717e3fb3e | Maps the handle and zero to the writable record. |
| F-6443787f9616 | Receives the initial context before resolution. |
| F-658f097ae4e8 | Receives the identifier and three output areas. |
| F-1cb585febd84 | Computes a value from the second output. |
| F-ddcbbbbc4a8e | Validates or classifies that output using the computed value. |
| F-8dfcd5b8f35c | Produces the containing-resource value. |
| F-e135feff4093, F-3dacd4ff9d73 | Supply predicates, with different required results for container and entry. |
| F-d05b3803629b | Performs the entry lookup. |
| F-53f6a8475775 | Attempts to produce an entry after missing lookup. |
| F-40e4fc419124 | Supplies the retained reset-related value. |
| F-27e49f66c358 | Applies zero or that retained value during reset handling. |
| F-cb9b1812bac7 | Determines reset success through a zero/nonzero result. |
| F-01a422616ce4, F-052651f0e13e | Receive successful-reset follow-up calls. |
| F-56c9e29af49d | Supplies the value stored at handle-record offset `+32`. |
| F-e657f684a3c8 | Receives the completed handle record. |
| F-b3f16c6113a2 | Receives the handle on ordinary failure. |
| F-66819ba2201a | Receives the nonzero containing-resource value during cleanup. |
| F-591e5dd605b9 | Receives the initial context during later cleanup. |
| F-038b46411c0c | Receives the selected final status. |

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid memory and dependency behavior sufficient to reach each stated branch.

1. **Initial-context failure:** If F-91c65c9d4f51 returns a negative value, return −1 without an explicit status-setting call.

2. **Handle-allocation failure:** If the initial context is nonnegative and F-c63d695765b6 returns negative, request status `0x18` and return −1. Do not expect a visible F-591e5dd605b9 call on this path.

3. **Missing entry without creation:** After successful container validation, null lookup with options zero fails with status `2` and ordinary failure cleanup.

4. **Missing entry with zero-access creation:** Null lookup with options `0x100` fails with `0x0d`; F-53f6a8475775 is not called.

5. **Creation bypasses reset:** Null lookup with options `0x301` attempts creation. A nonnull creation result proceeds to finalization without F-cb9b1812bac7, despite `0x200` being set.

6. **Existing-entry exclusion precedes access-byte rejection:** For an eligible existing entry, options `0x501` fail with `0x11` before the access-byte read. Options `0x401` do not trigger this exclusion check.

7. **Zero-access reset remains possible:** For an eligible existing entry, options `0x200` enter the reset sequence without reading the access-restriction byte.

8. **Reset failure restoration:** If F-cb9b1812bac7 returns zero, the retained reset-related value is passed back to F-27e49f66c358, the saved `+28` field is explicitly restored, and final status is `0x0d`.

9. **State-bit replacement versus preservation:** Assuming no intervening mutation, options `0x805` produce state value `7` when the `+36` field equals 1. If that field is not 1 and the initial state is `0x80`, the resulting state is `0x87`.

10. **Option-width boundary:** On an otherwise successful path, options `0x80000001` store `0x0001` in the 16-bit handle field. The high bit does not select an additional visible branch.

## Uncertainties

The following are **documented_only**:

- Semantic names for status values, resource kinds, and option bits.
- Whether the initial-context calls implement locking or another lifecycle.
- Allocation and release semantics of handles, entries, and containing resources.
- Whether dependencies mutate shared memory, global state, or records already obtained.
- Whether the reset sequence changes size, allocation, contents, or another property.
- Encoding and termination requirements for temporary outputs.
- Behavior for invalid pointers, undersized records, buffer overflow, dependency exceptions, or abnormal termination.
- Concurrency guarantees and atomicity of the visible reads and writes.
- Evaluation order among separate argument expressions within a single dependency call.
- Endianness and broader ABI details beyond the visible access widths.

No automatic rollback, global preservation, or memory ownership transfer is assumed.

## Evidence

**Established static evidence:** The supplied function body supports the branch conditions, widths, offsets, direct writes, dependency arguments, cleanup order, return values, and explicit status-setting calls described above.

The exclusion condition uses a 32-bit complement masked by `0x500`; it is satisfied precisely when both selected option bits are present. Option recording explicitly narrows to 16 bits. State updates use 32-bit bitwise operations, and the access restriction uses an 8-bit read. No size arithmetic or overflow-sensitive length computation is visible in this function.

**Inferred intent:** Opening a file-like resource, creating a missing entry, truncating or resetting existing state, and acquiring or releasing synchronization are plausible interpretations only.

**Runtime evidence:** None. No runtime tests have been performed.
