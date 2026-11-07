<!-- Trusted provenance: F-038cdeb3ff46; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-038cdeb3ff46

## Purpose

F-038cdeb3ff46 conditionally operates on a caller-supplied object using a 32-bit selector captured from that object. A shared sentinel can bypass the operation. Otherwise, a dependency-produced classification byte selects one of two paths.

The paths differ in return-value treatment and update ordering:

- A sentinel match returns `1` without dependency calls or direct object writes.
- Classification byte `3` selects an operation whose result is normalized to `0` or `1`. A zero result also causes a four-byte object-field replacement after the completion dependency has run.
- Every other classification byte selects an operation whose 32-bit result is returned unchanged. This path may perform virtual dispatch, replace the same four-byte field, and set two one-byte flags before the completion dependency runs.

**Inferred intent:** the initial and completion calls may delimit a protected operation, and the replacement field may hold cached state. Neither interpretation establishes synchronization, resource ownership, or application-level success semantics.

## Interface

### Arguments and return value

| Item | Established behavior |
|---|---|
| First argument | A mutable object address, represented in the observed 32-bit address model. |
| Second argument | A four-byte value not inspected or forwarded by this function. |
| Third argument | An opaque four-byte value forwarded to the selected primary operation. |
| Return | A 32-bit integer: `1` for a sentinel match, normalized zero/nonzero on classification `3`, or the unchanged general-operation result otherwise. |

The function uses four-byte integer and address representations, two-byte signed gate-field reads, and one-byte classification and flag accesses.

### Directly accessed object layout

Offsets are in bytes.

| Offset | Width | Access |
|---|---:|---|
| `0x000` | 4 bytes | Dispatch-table address, read only when virtual dispatch is reached. |
| `0x1BC` | 1 byte | Written with `1` when the general-operation result is zero. |
| `0x1BD` | 1 byte | Written with `1` immediately after the preceding flag write. |
| `0x1C4` | 4 bytes | Destination of a replacement value on either retrieval path. |
| `0x1CC` | 4 bytes | Selector read at entry and retained for subsequent selector arguments. |
| Dynamically obtained offset | 2 bytes | Signed gate field compared with `-1`. |

The gate offset is read from a global only after the general operation returns a result other than `2`. It is not captured at entry and is not established as a fixed object-layout constant. Earlier dependencies can change which address is inspected.

The virtual call uses the four-byte callable entry at dispatch-table byte offset `0x2C`.

### Shared state read directly

Three distinct shared-state roles are visible:

1. A referenced four-byte sentinel value, read for the entry comparison.
2. A shared argument global, read separately for the initial dependency and for the branch-specific completion dependency.
3. A gate-offset global, read only when the general result is not `2`.

The two shared-argument reads do not reuse a retained value. Intervening dependencies may change that global, so the initial and completion calls need not receive equal arguments.

### Preconditions

Storage and callable targets must be valid for every access actually reached:

- The object selector and referenced sentinel must be readable at entry.
- Each reached shared-argument or gate-offset global read must be valid.
- Reaching the gate check requires a readable two-byte field at the address formed from the object address and the then-current global offset.
- Virtual dispatch additionally requires a readable dispatch-table address and a callable entry at offset `0x2C`.
- A replacement requires four writable bytes at object offset `0x1C4`.
- A zero general result requires writable bytes at `0x1BC` and `0x1BD`.
- Successful virtual retrieval requires the extraction dependency to establish an address permitting a four-byte read at address-plus-`8`.

No explicit null, alignment, bounds, address-overflow, or dispatch-target checks are visible. Invalid storage or outputs have no defined recovery result in this function.

## Behavior

### Entry and sentinel bypass

The function first reads the four-byte selector at object offset `0x1CC`. It then reads the referenced four-byte sentinel and compares the two values for equality.

If equal, it returns `1`. It does not invoke dependencies, read the shared-argument global, read the gate-offset global, or directly write the object.

If unequal, processing continues using the captured selector. Dependencies may subsequently alter the selector field, but later selector arguments still use the captured value.

### Initial calls and classification

For a non-sentinel selector, the function performs these actions in order:

1. Reads the current shared-argument global and passes that value to F-unknown-001.
2. Calls F-8affc51b0705 without explicit arguments and retains its four-byte result as an opaque context.
3. Calls F-27fdb2323185 with the context, the address of a four-byte classification area, and the captured selector.
4. Reads the classification area’s first byte and compares it with `3`.

The classification area is not explicitly initialized before the call. Its remaining three bytes are not directly consumed here. No return status from F-27fdb2323185 is checked.

Meaningful, dependency-determined branch selection therefore requires that dependency to establish the first byte. The function supplies no fallback classification when it does not.

### Classification byte equal to `3`

The function calls F-d4b11324e3f9 with these arguments, in order:

1. Retained context.
2. Third argument supplied to F-038cdeb3ff46.
3. Captured selector.

After that operation returns, the function reads the shared-argument global again and passes its current value to F-8fbd1566896d. This read occurs after the operation; it does not reuse the argument supplied to F-unknown-001.

The completion call precedes the following result-dependent processing.

**When the operation result is zero:**

- A four-byte temporary output is initialized to zero.
- F-16b5db5e981f receives the retained context, the output address, and the captured selector.
- The output’s resulting four bytes are written to object offset `0x1C4`.
- The function returns `0`.

No status from F-16b5db5e981f is checked. If it returns normally without modifying the output, the replacement value is zero.

**When the operation result is nonzero:**

- F-16b5db5e981f is not called.
- No direct replacement or flag write occurs.
- The function returns `1`, regardless of the original nonzero value’s magnitude or sign.

The general operation, gate check, and virtual sequence are absent from this path.

### Classification byte other than `3`

The function calls F-666fbd047c82 with:

1. Object address.
2. Captured selector.
3. Third argument.

Its 32-bit result is retained for the eventual return. Subsequent virtual results do not replace it.

#### General result exactly `2`

The function skips the entire optional-work region. In particular, it does not:

- Read the gate-offset global.
- Form or read the gate address.
- Initialize the 16-byte temporary.
- Read the dispatch table or invoke its target.
- Extract or directly replace the four-byte field.
- Directly write either flag.

It then reads the shared-argument global, passes that current value to F-8fbd1566896d, and returns `2`.

#### General result other than `2`: late gate-address calculation

Only after F-666fbd047c82 returns a value other than `2` does the function read the gate-offset global.

It combines that then-current offset with the object address using the observed 32-bit address-plus-offset arithmetic, and reads a signed two-byte value at the resulting address. No widened address calculation or overflow check is shown. The evidence does not establish safe behavior for a sum outside the valid address range.

This ordering is significant:

- The offset is not captured with the selector.
- F-unknown-001, classification dependencies, or F-666fbd047c82 may change it before use.
- Its unavailable value is distinct from a guarantee that the offset is fixed.
- The resulting two-byte access cannot be proven disjoint from the listed fields or confined to a particular object extent.

If the signed field equals `-1`, corresponding to the two-byte pattern `0xFFFF`, the virtual sequence is skipped. Every other two-byte value enables that sequence.

#### Optional virtual sequence

When the gate is not `-1`, the function:

1. Passes a 16-byte temporary area to F-2b447e614ee2.
2. Reads the object’s dispatch-table address after that call.
3. Obtains and invokes the entry at table offset `0x2C`.
4. Supplies the object address, captured selector, temporary-area address, and a zero fourth argument.
5. Tests the virtual operation’s 32-bit result for zero.

Thus a dependency that changes the object’s dispatch-table address before the table read can affect the invoked target.

If the virtual result is nonzero, extraction and the associated replacement write are skipped.

If the virtual result is zero:

- F-e2669f32f0eb receives the 16-byte temporary’s address and the address of a four-byte output slot.
- The slot is not explicitly initialized before that call.
- Its resulting contents are interpreted as a 32-bit address.
- Four bytes are read at that address plus `8`.
- Those four bytes are written to object offset `0x1C4`.

No extraction status, null check, or bounds check is examined. The address-plus-`8` calculation is also in the observed 32-bit address model; valid readable storage is required rather than established by the function.

#### Flags and completion

After the virtual sequence, or immediately after a gate value of `-1`, the retained general result determines flag writes:

- Result zero: write byte value `1` at `0x1BC`, then write byte value `1` at `0x1BD`.
- Any nonzero result: perform neither direct flag write.

The function then reads the shared-argument global again, passes its current value to F-8fbd1566896d, and returns the original general result unchanged.

Consequently, a nonzero general result other than `2` can still produce a replacement-field write. A zero general result sets both flags even when virtual work is skipped or its result is nonzero.

## Side effects

### Direct object writes and ordering

Direct persistent object writes are limited to the four-byte replacement and the two one-byte flags.

Their ordering differs by branch:

- Classification `3`: completion runs before output initialization, retrieval, and replacement.
- Other classifications: optional retrieval and replacement occur before flag writes; completion runs afterward.

The two flags are separate writes, ordered `0x1BC` before `0x1BD`. No atomicity or cross-thread visibility guarantee follows from this order.

A completion dependency on the general path may itself alter fields after this function’s direct writes. On classification `3`, the later direct replacement may overwrite a value that completion wrote at `0x1C4`. No dependency is assumed to leave the object unchanged.

### Shared-state observations

The shared argument is observed twice on every normally returning non-sentinel path: once before F-unknown-001 and once immediately for completion. Changes between these reads can change the completion argument.

The gate offset is observed once on a reached general-result-not-`2` path, after the general operation. Later changes to that global do not trigger another gate read in this invocation.

These are access-order guarantees, not synchronization guarantees.

### Temporary storage and ownership

The function passes addresses of temporary areas with these sizes:

- Four bytes for classification.
- Four bytes for classification-`3` output.
- Sixteen bytes for the virtual-operation argument.
- Four bytes for the extracted address.

Only the classification-`3` output is explicitly initialized by this function before its output-producing call. The 16-byte area is passed to F-2b447e614ee2 before virtual use, but that dependency’s initialization guarantees are unavailable.

No explicit allocation, deallocation, destruction, ownership transfer, or permission to retain these addresses is established. The extracted address is dereferenced but not explicitly released here.

### Errors and abnormal exits

There is no uniform error convention:

- Classification-`3` operation results are normalized.
- General-operation results are preserved.
- A nonzero virtual result suppresses extraction without changing the return.
- Several output-producing calls have no checked status.

Every normally returning non-sentinel path invokes F-8fbd1566896d once. Cleanup following exceptions, faults, or nonlocal exits is not established.

## Dependencies

F-unknown-001 and F-unknown-002 are document-local opaque IDs for dependencies without supplied IDs.

| ID | Observed interaction |
|---|---|
| F-unknown-001 | Receives the first read of the shared-argument global, before context retrieval. |
| F-8affc51b0705 | Produces the opaque four-byte context. |
| F-27fdb2323185 | Receives context, classification-area address, and captured selector. |
| F-d4b11324e3f9 | Classification-`3` operation; zero versus nonzero controls the normalized return and retrieval. |
| F-16b5db5e981f | Receives context, a zero-initialized four-byte output, and selector after completion. |
| F-666fbd047c82 | General operation; its result is retained and controls later work. |
| F-2b447e614ee2 | Receives the 16-byte temporary before dispatch-table lookup. |
| F-unknown-002 | Virtual target at table offset `0x2C`; receives object, selector, temporary address, and zero. |
| F-e2669f32f0eb | Supplies an address through a four-byte output slot after a zero virtual result. |
| F-8fbd1566896d | Receives a fresh read of the shared-argument global at the branch-specific completion point. |

These descriptions are call-site contracts, not complete dependency specifications. In particular, dependencies are not presumed to preserve globals, object fields, or resource state.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Dependencies are assumed to return normally and provide valid storage where required. “No write” means no direct write by this function.

1. **Sentinel bypass.** If the entry selector equals the referenced sentinel, return `1` without dependencies, shared-argument reads, gate-offset reads, or direct object writes.

2. **Classification-`3` replacement.** If classification is `3`, the operation returns zero, and retrieval supplies `0x12345678`, completion precedes retrieval; `0x1C4` receives that value; return zero. Neither flag is directly written.

3. **Untouched initialized output.** Under the same zero-result branch, if retrieval leaves its output untouched, the replacement is zero.

4. **Normalized nonzero result.** On classification `3`, operation results `2`, `7`, or any negative 32-bit value all produce return `1` and no retrieval.

5. **Fresh shared argument.** If the shared argument initially contains value A, F-unknown-001 receives A. If an intervening dependency changes it to B before completion’s read, F-8fbd1566896d receives B. This applies on either classification path.

6. **General result `2`.** Return `2` after completion without reading either the gate-offset global or the gate field, even if a dependency changed the offset.

7. **Late offset change.** Suppose the offset initially designates one two-byte field, but F-666fbd047c82 changes it to designate another and returns zero. The second field controls the gate. If it contains `0xFFFF`, virtual work is skipped regardless of the first field’s value; both flags are then written.

8. **General zero with excluded gate.** A zero general result and gate `-1` cause flag writes in offset order, followed by completion and return zero, without replacement retrieval.

9. **Nonzero general result with replacement.** A general result of `7`, an enabled gate, zero virtual result, and valid extracted value V cause `0x1C4` to receive V. Neither flag is directly written; return `7`.

10. **Virtual nonzero does not override zero.** A zero general result with a nonzero virtual result skips extraction but still sets both flags and returns zero.

11. **Selector mutation.** If a dependency changes the object’s selector after entry, subsequent selector arguments retain the originally captured value.

## Uncertainties

The following remain **documented_only**:

- Sentinel value and business meaning.
- Classification meanings and application interpretation of result codes.
- Gate-offset value, signed interpretation, mutability constraints, and possible overlap with other fields.
- Valid address ranges and behavior when 32-bit address calculations overflow or produce invalid storage.
- Synchronization or resource-management semantics of the initial and completion calls.
- Context representation and lifetime.
- Temporary representation, initialization guarantees, retention, and destruction obligations.
- Whether output dependencies reliably establish their required outputs.
- Dependency mutation of shared globals, object fields, or other memory.
- Exceptional exits, concurrency guarantees, and hardware-level atomicity.

No fallback behavior is inferred for these unknowns.

## Evidence

This contract is based solely on static inspection of the supplied function body and dependency-ID mapping.

Established evidence includes access widths and offsets, captured selector use, separate shared-argument reads, the late global-offset read, 32-bit address calculations, branch conditions, call ordering, output initialization, virtual-table slot, direct-write order, and return treatment.

Protected-operation and cached-state interpretations are inferred intent only. Dependency internals and resource guarantees remain unknown.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the static control flow, not observed executions.
