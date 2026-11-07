<!-- Trusted provenance: F-0152cdc5db1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0152cdc5db1c

## Purpose

F-0152cdc5db1c stores a requested mode byte in caller-supplied state, selects dependency operations using the original mode and additional state, and ordinarily invokes a final indirect operation.

The established behavior is a control-flow and memory-access contract. The application meanings of the mode, flag, selector, global value, and dependency results are unknown. Descriptive terms below identify structural roles, not confirmed domain semantics.

One branch returns immediately after a selector mismatch. That branch retains the initial mode-byte write and global read but invokes no dependencies, including the ordinary completion operation.

## Interface

### Inputs and result

| Input | Representation established by the supplied source | Use |
|---|---|---|
| State address | Address carried in a 32-bit input | Base for the state accesses described below |
| Mode | Signed 32-bit integer | Supplies a stored byte and independently controls the first branch |
| Selector argument | Unsigned 32-bit integer | Compared against an unsigned state byte on one branch |

The function has no specified return value. Results from dependencies select subsequent actions; none is exposed as an explicit result of this function.

The mode has two distinct uses:

- The write to state retains its low eight bits.
- The branch tests the entire original 32-bit argument for zero.

Thus, a nonzero mode whose low byte is zero still selects the nonzero-mode branch. The stored byte is not read back to make that decision.

### State layout

Offsets are byte offsets from the supplied state address.

| Offset | Access width | Established role |
|---|---:|---|
| `0x20` | 32 bits | Object pointer used for the set-flag indirect operation |
| `0x44` | 32 bits | Value passed to F-f591b4c58273 or F-c36a3e9987c8 |
| `0x76` | 8 bits, unsigned | Selector compared with the full selector argument |
| `0x8A` | 8 bits | Flag tested for zero versus nonzero |
| `0xA0` | 8 bits | Destination of the mode-byte write |

A global 32-bit value is read after the mode-byte write and before any dependency invocation. Its saved value is called the **captured global value**.

On selected clear-flag paths, the address returned by F-fdd8589eca2d is used for a subsequent 32-bit read at offset `0x20`. Whether that address aliases the state object, an object returned by another dependency, or other storage is unknown. The contract imposes no non-aliasing requirement.

The two accesses described at offset `0x20` have different roles: one obtains an object pointer from state; the other obtains a comparison value from the address returned by F-fdd8589eca2d. Those roles do not establish that their storage is distinct.

### Indirect operation layout

The supplied representation uses 32-bit object and table pointers. Each selected indirect operation reads an object’s table pointer and then obtains a callable target at a byte offset within that table.

| Object source | Table byte offset | Role |
|---|---:|---|
| F-3c71be02d8c5 | `0xDC` | First clear-flag gate |
| F-6ba41446243e | `0x38` | Second clear-flag gate |
| State field at `0x20` | `0xC0` | Value used for the unsigned threshold decision |
| F-3c71be02d8c5, called at completion | `0xD4` | Ordinary completion operation |

No explicit arguments are displayed for these indirect operations. Their complete calling conventions, including any implicit receiver, are not established.

### Preconditions

For execution to follow the described path, the accessed storage and callable targets must remain valid when used:

- The byte at state offset `0xA0` must be writable at entry.
- Every state field selected by the branch must be readable at its actual access point.
- Dependency-returned addresses must support the reads performed through them.
- Object tables and selected targets must be valid for the indirect invocations.
- Address-plus-offset calculations must designate the intended accessible storage.

The function supplies no visible validation for these conditions. The representation does not establish portable behavior for invalid addresses, invalid targets, or address arithmetic outside valid storage.

## Behavior

### Entry sequence

Every invocation begins with these observable actions, in order:

1. Write the low eight bits of the mode to state offset `0xA0`.
2. Read the global 32-bit value and retain it for subsequent comparisons or notification arguments.
3. Select a branch using the full-width mode argument.

The global read occurs even when its result will not be used. The mode-byte write precedes that read; no independent non-aliasing guarantee between storage locations is established.

Later dependency calls do not cause an explicit refresh of the captured global value.

### Nonzero mode

If the original mode is nonzero:

1. Read the current 32-bit value at state offset `0x44`.
2. Invoke F-f591b4c58273 with that value and integer `1`, in that order.
3. Proceed to common completion.

No result from F-f591b4c58273 controls a branch here.

On this path, F-0152cdc5db1c does not itself read state offsets `0x8A`, `0x76`, or `0x20`. This says nothing about memory accessed by the invoked dependencies.

### Zero mode: clear flag

If the mode is zero, read the byte at state offset `0x8A`. A zero byte selects the clear-flag path.

First, invoke F-3c71be02d8c5, read the returned object’s table pointer, and invoke its target at table offset `0xDC`.

- A zero result leads directly to common completion.
- A nonzero result permits the next gate.

For that next gate, invoke F-6ba41446243e and then the target at table offset `0x38` on its returned object. The zero or nonzero result determines how the subsequent comparison is used.

#### Second gate returns zero

Invoke F-fdd8589eca2d. Read the 32-bit value at offset `0x20` from its returned address and compare that value with the captured global value.

- If they differ, proceed directly to common completion. Do not invoke F-c36a3e9987c8.
- If they are equal, read the current 32-bit state field at `0x44` and invoke F-c36a3e9987c8 with that value.

After F-c36a3e9987c8:

- A nonzero result leads directly to common completion.
- A zero result causes F-d260cfbdb3c8 to be invoked before common completion.

F-bb48a4c96702 is not invoked on this subpath, regardless of the comparison or status result.

The comparison read is through the address actually returned by F-fdd8589eca2d. If that address is the state address, the comparison reads the same storage identified as state offset `0x20`; it is not redirected to a separate object.

#### Second gate returns nonzero

Invoke F-fdd8589eca2d and read the 32-bit value at offset `0x20` from its returned address.

- If that value equals the captured global value, proceed directly to common completion.
- If it differs, invoke F-bb48a4c96702 with the captured global value, integer `1`, and integer `0`, in that order. Then proceed to common completion.

Neither F-c36a3e9987c8 nor F-d260cfbdb3c8 is invoked on this subpath.

### Zero mode: nonzero flag

A nonzero byte at state offset `0x8A` selects the set-flag path.

Read the unsigned byte at state offset `0x76`. Zero-extend it to 32 bits and compare it with the entire unsigned selector argument.

If the values differ, return immediately. This is the only explicit path that bypasses common completion. The entry write and global read have already occurred, but no dependency has been invoked.

Because the selector argument is not truncated, values from `256` through `0xFFFFFFFF` cannot match any stored selector byte.

If the values match:

1. Read the object pointer from state offset `0x20`.
2. Read its table pointer and invoke the target at table offset `0xC0`.
3. Interpret the result as an unsigned 32-bit value.
4. Select exactly one threshold dependency:
   - Below `1000`: invoke F-94e449488afb with the state address.
   - At least `1000`: invoke F-52a8d93d5e77 with the state address and integer `1`.

The threshold is strict and unsigned. A result of `999` takes the first alternative; `1000` and `0xFFFFFFFF` take the second. There is no subtraction or other threshold arithmetic shown.

After the selected dependency returns, read the current 32-bit value at state offset `0x44` and invoke F-c36a3e9987c8 with it. This read is not an entry-time snapshot. A change made by the selected dependency affects the value passed onward, provided the storage remains valid.

Then:

- A zero result from F-c36a3e9987c8 causes F-d260cfbdb3c8 to be invoked.
- A nonzero result causes F-bb48a4c96702 to be invoked with the captured global value, integer `1`, and integer `0`.

Both alternatives then reach common completion.

### Common completion

Every path other than the explicit selector-mismatch return reaches common completion if all preceding operations return normally.

Completion consists of a fresh invocation of F-3c71be02d8c5, followed by resolution and invocation of the returned object’s target at table offset `0xD4`.

On a clear-flag path, the earlier object used for the `0xDC` operation is not visibly reused. The two calls to F-3c71be02d8c5 may return equal or different addresses. Neither identity nor distinctness is guaranteed.

The final indirect transfer is represented as an invocation, but its exact transfer mechanics are unresolved in the supplied recovery. Selecting and transferring control to the `0xD4` target is established; a complete machine-level return convention is not.

## Side effects

### Direct effects

The only explicit write performed by F-0152cdc5db1c is the byte write at state offset `0xA0`. It occurs on every path and is not undone by the early return.

Other directly established observations are:

- The global read occurs after the entry write and before all dependency calls.
- State fields are read at their branch-specific points, not collected at entry.
- Indirect dispatch reads the relevant object pointer, table pointer, and target.
- Comparison through F-fdd8589eca2d uses the returned address without a visible null check or identity check.

The function shows no explicit allocation, deallocation, retain, release, or ownership transfer.

### Effects through dependencies and aliasing

Dependency effects are unknown. They may modify globals, state, returned objects, table pointers, ownership, or external resources. Lack of an explicit state argument does not prove that a dependency cannot access state through another reference.

No dependency is assumed to preserve the global. Nevertheless, comparisons and F-bb48a4c96702 arguments use the captured value rather than an explicit later global read.

No dependency is assumed to preserve state fields. In particular, reads of state offset `0x44` and the comparison field returned through F-fdd8589eca2d occur after earlier calls and can observe their effects.

Returned objects are not assumed distinct from state or from one another. Aliasing can make a dependency’s write visible through a later access under another role. It does not change which address expression the function uses.

If a dependency releases storage needed later, does not return, or transfers control elsewhere, normal continuation is not assured. No recovery or lifetime repair is visible.

## Dependencies

| Opaque function ID | Established interface and use |
|---|---|
| F-f591b4c58273 | Receives the 32-bit state value at `0x44` and `1` on the nonzero-mode path; result not used for branching |
| F-3c71be02d8c5 | Returns an object used for the clear-flag `0xDC` operation and is called afresh for completion’s `0xD4` operation |
| F-6ba41446243e | Returns an object used for the clear-flag `0x38` operation |
| F-fdd8589eca2d | Returns the address used for a 32-bit comparison read at offset `0x20`; aliasing and ownership are unknown |
| F-c36a3e9987c8 | Receives the current 32-bit state value at `0x44`; its zero/nonzero result controls subsequent calls |
| F-94e449488afb | Receives the state address when the unsigned threshold result is below `1000` |
| F-52a8d93d5e77 | Receives the state address and `1` when the threshold result is at least `1000` |
| F-d260cfbdb3c8 | Invoked on the specified zero-result paths; no explicit arguments are displayed |
| F-bb48a4c96702 | Receives the captured global value, `1`, and `0` on the specified notification paths |

The indirect targets have no supplied opaque function IDs. Their provider and table offsets identify the invocation sites without inventing target identities or semantics.

## Acceptance examples

These are conditional acceptance examples, not runtime observations. They assume valid accesses and normal return from every operation needed to reach the stated outcome.

1. **Full-width mode controls branching.**  
   Mode `256` writes zero at state offset `0xA0` but selects the nonzero-mode path. F-f591b4c58273 receives the current field at `0x44` and `1`, followed by common completion.

2. **Selector comparison preserves the argument width.**  
   With mode zero, a nonzero flag, stored selector `1`, and selector argument `257`, the function returns after the entry write and global read. It invokes no dependencies.

3. **First gate short-circuits.**  
   With mode zero and a clear flag, a zero result from the `0xDC` operation leads to completion without invoking F-6ba41446243e or F-fdd8589eca2d.

4. **Clear-flag equality permits the status operation.**  
   Given a nonzero `0xDC` result, zero `0x38` result, and comparison value equal to the captured global, F-c36a3e9987c8 receives the current field at `0x44`. Its zero result causes F-d260cfbdb3c8; its nonzero result does not. Neither alternative invokes F-bb48a4c96702.

5. **Mismatch depends on the second gate.**  
   A comparison mismatch with a zero `0x38` result leads directly to completion. The same mismatch with a nonzero `0x38` result invokes F-bb48a4c96702 with the captured value, `1`, and `0` before completion.

6. **Threshold boundaries are unsigned.**  
   On a matching set-flag path, `999` selects F-94e449488afb. Values `1000` and `0xFFFFFFFF` select F-52a8d93d5e77. The latter is not treated as a negative value.

7. **A later state read observes a dependency’s change.**  
   If the selected threshold dependency changes state offset `0x44` and leaves that storage valid, F-c36a3e9987c8 receives the changed value.

8. **Global mutation does not refresh the capture.**  
   If a dependency changes the global after entry and a later branch invokes F-bb48a4c96702, its first argument remains the value captured at entry.

9. **The comparison address may alias state.**  
   Suppose the clear-flag gates return nonzero and zero respectively, and F-fdd8589eca2d returns the state address itself. The subsequent comparison uses the 32-bit contents of state offset `0x20`. Equality with the captured global permits F-c36a3e9987c8; inequality bypasses it. No requirement for a distinct returned object applies.

## Uncertainties

The following remain **documented_only**:

- Application meanings of all fields, constants, and dependency results.
- Complete state and object layouts, allocation extents, alignment requirements, and byte order.
- Independent verification of the represented 32-bit ABI.
- Complete indirect-call signatures and implicit arguments.
- Whether returned addresses alias state, globals, one another, or other storage.
- Ownership, borrowing, retention, release, and lifetime rules.
- Dependency mutations, resource effects, and failure behavior.
- Concurrency guarantees, atomicity, and synchronization.
- Exact mechanics and return behavior of the final indirect transfer.

There is no established error-result, diagnostic, exception-handling, or rollback contract. Zero and nonzero dependency results are branch conditions, not proven success or failure indicators.

## Evidence

This contract is based solely on static examination of the supplied source representation. The prior draft and reviewer feedback are not independent behavioral evidence.

Established evidence comprises access widths and offsets, explicit argument values, zero/nonzero branches, short-circuit order, unsigned selector and threshold comparisons, and the order of visible reads, writes, and invocations.

The returned address from F-fdd8589eca2d is used directly for the offset-`0x20` comparison read. No visible identity test, allocation, or other operation establishes its distinctness from state or other storage. Aliasing therefore remains unresolved rather than prohibited.

An interpretation that the function coordinates a mode transition is inferred intent only. Dependency semantics and ownership cannot be derived from their invocation positions.

No runtime tests have been performed.
