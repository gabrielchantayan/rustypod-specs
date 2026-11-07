<!-- Trusted provenance: F-0cb2f22d6ad6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0cb2f22d6ad6

## Purpose

F-0cb2f22d6ad6 performs a stateful operation on a caller-supplied object. It marks the object during an initial dependency call, optionally refreshes a 32-bit field, and then either returns an early status or proceeds through a dependency-selected object and a final virtual dispatch.

**Established behavior:** the function has two normal return forms: the exact 32-bit value `1` on an early-exit path, and the 32-bit result saved from the initial dependency call on the completed path. A third path invokes a dependency identified as non-returning.

**Inferred intent:** the temporary marker may indicate an operation in progress, and the optional field update may refresh an associated identifier or state value. Neither interpretation is necessary to the contract. The meanings of the selected objects, numeric constants, and final dispatch arguments remain undocumented.

## Interface

### Inputs

The function accepts:

- **Object:** an address used as a mutable object with a virtual-dispatch table and fields at fixed byte offsets.
- **Option:** a 32-bit unsigned value. Zero and nonzero control one branch, but the original value is also forwarded unchanged later. It is not normalized to a Boolean.

The visible representation uses 32-bit words and addresses. Offsets below are byte offsets from the object address.

| Offset | Width | Visible use |
|---|---:|---|
| `0x00` | 32 bits | Virtual-table address used for the final dispatch. |
| `0x20` | 8 bits | First early-exit condition. Only one byte is inspected. |
| `0x41` | 8 bits | Second early-exit condition, inspected only if the first byte is zero. |
| `0xC0` | 32 bits | Receives a dependency result when the option is nonzero. |
| `0xE8` | 32 bits | Read as an argument to the final virtual dispatch. |
| `0xF0` | 8 bits | Written to `1` before the initial dependency and to `0` after it returns. |

The source expresses the `0x20` condition through a word access followed by conversion to a character. Its effective predicate is the low eight bits of that word, not whether the entire 32-bit word is nonzero. Exact machine-load width and endianness-dependent byte correspondence are not independently established.

### Output

The return value is 32 bits:

- `1` if either early-exit condition is nonzero;
- otherwise, the exact saved result of F-ff499a944413, provided the completed path returns normally.

No result from the final virtual dispatch replaces that saved result.

### Preconditions

For a normal completed invocation, the object and all subsequently used dependency-supplied objects must remain valid for their visible accesses and dispatches.

In particular:

- The object must support the writes at `0xF0` and, when applicable, `0xC0`.
- The relevant condition fields must be readable.
- The completed path requires a readable field at `0xE8`, a valid virtual-table address at `0x00`, and a callable entry at virtual-table byte offset `0x13C`.
- Dependency-supplied dispatch receivers must have valid tables and applicable callable entries.

The function does not validate the initial object address. It also does not check every dependency-supplied receiver for null before dereferencing it.

No ownership transfer, allocation guarantee, release obligation, or preservation of shared state by an unknown dependency can be inferred from this body.

## Behavior

### Initial operation and temporary marker

The first explicit object mutation is an 8-bit write of `1` at offset `0xF0`.

The function then calls F-ff499a944413 with the object and saves its 32-bit result. After that call returns normally, it writes an 8-bit `0` at offset `0xF0`.

This sequence does not preserve the marker’s previous value. For example, an entry value of `7` is replaced by `1`, and then by `0`.

The clearing write is not shown as exception-safe or otherwise protected cleanup. If the dependency fails to return normally, this body provides no guarantee that the marker is cleared. The dependency’s own writes are unknown.

### Optional field refresh

If the 32-bit option is nonzero, the function:

1. Calls F-76b1c50b12f7 with no visible arguments.
2. Calls F-22be49cfb47c with no visible arguments.
3. Writes the latter result as a 32-bit value at object offset `0xC0`.

The first call’s result is not consumed by this function.

If the option is zero, both calls and this direct field write are skipped.

This branch occurs before the early-exit conditions are evaluated. Consequently, a nonzero option can cause the refresh even when the function subsequently returns `1`.

### Early-exit conditions

The function evaluates the eight-bit condition at offset `0x20`.

- If it is nonzero, the function returns `1`.
- If it is zero, the function evaluates the byte at offset `0x41`.
- If that second byte is nonzero, the function returns `1`.

The second condition is not evaluated on the first condition’s nonzero path.

Both conditions are evaluated after the initial dependency and, when enabled, the optional refresh calls. Their values must therefore be understood as the values observed at that point, not necessarily their entry values.

On either early return, the function does not perform the later object-selection sequence or the final virtual dispatch. The initially saved result is discarded.

### Object selection

If both conditions are zero, the function calls F-d99467d4d2b3 with an explicitly visible zero-valued argument. It tests the result against zero.

If that result is zero, it invokes F-a817d8d58c10, which is identified in the supplied static evidence as non-returning.

If the result is nonzero, the function:

1. Calls F-938c950e5046 and obtains a dispatch receiver.
2. Calls F-d99467d4d2b3 again.
3. Dispatches through the receiver’s virtual-table entry at byte offset `0xE8`, passing the receiver and the second F-d99467d4d2b3 result.
4. Treats the dispatch result as a selected object address.

The second F-d99467d4d2b3 call has no explicit arguments in the supplied representation. Its true signature and relationship to the first call are unresolved. The contract does not assume that it returns the same value twice or that it is a pure query.

The receiver returned by F-938c950e5046 is dereferenced without a visible null check.

If the selection dispatch returns null, the function invokes F-a817d8d58c10. No ordinary return value is established for this path.

### Conditional secondary operation

For a nonnull selected object, the function invokes its virtual-table entry at byte offset `0x1C`, passing that object. The result is treated as a 32-bit value and compared with zero.

If the result is zero, the function skips the secondary operation and proceeds to completion.

If the result is nonzero, the function calls F-682adfd5a561 with:

- that result;
- the constant `0x6800`.

The return value is treated as another object address.

If it is null, the function:

1. Calls F-8157a180d901 with `0x1C`;
2. Calls F-2606ad6d534d with no visible arguments;
3. Dispatches through the selected object’s virtual-table entry at byte offset `0x2C`, passing the selected object and the result of F-2606ad6d534d.

If it is nonnull, the function dispatches through that returned object’s virtual-table entry at byte offset `0x58`. No explicit call arguments are shown for this dispatch. An implicit receiver convention must not be invented.

Results from these dispatches are not used by this function. Assuming the calls return normally, both branches proceed to completion. In particular, the null result from F-682adfd5a561 does not itself cause an immediate return or the visible non-returning path.

### Completion

After the secondary operation, or directly after its zero-result bypass, the function:

1. Calls F-fdd8589eca2d.
2. Calls F-8f9bd59e3ec5 with that result and the original object.
3. Reads the original object’s dispatch table and field at `0xE8` for a final dispatch through virtual-table byte offset `0x13C`.
4. Returns the saved result from F-ff499a944413.

The final dispatch receives five visible arguments:

- the original object;
- its 32-bit field at `0xE8`;
- the original 32-bit option;
- zero;
- the original option XOR `1`.

The field and dispatch target used at completion are not captured before the intervening calls. Dependencies may have changed them.

The XOR operation is a 32-bit bitwise operation, not logical negation. It toggles only the least-significant bit:

- `0` becomes `1`;
- `1` becomes `0`;
- `2` becomes `3`;
- `0xFFFFFFFF` becomes `0xFFFFFFFE`.

No addition, length computation, or allocation-size arithmetic is established by the visible body. The constants `0x6800` and `0x1C` are forwarded without a demonstrated unit or semantic interpretation.

## Side effects

### Direct writes

The function directly performs:

- an 8-bit write of `1` at object offset `0xF0`;
- an 8-bit write of `0` at the same offset after the initial dependency returns;
- a conditional 32-bit write at offset `0xC0`.

No other direct object-field assignment is shown.

### Indirect effects and ordering

All dependencies and virtual dispatches may have additional effects not described by this body. Their effects must not be treated as absent merely because their return values are ignored.

Established ordering includes:

- Marker set before F-ff499a944413; marker cleared after its normal return.
- Optional refresh before either early-exit decision.
- First condition before the conditional second condition.
- Successful selection before the selected-object operations.
- F-8157a180d901 before F-2606ad6d534d on the null secondary-result path.
- F-fdd8589eca2d before F-8f9bd59e3ec5.
- F-8f9bd59e3ec5 before the final dispatch.
- Final dispatch before return of the saved result.

The source does not establish a left-to-right machine evaluation order among argument preparation and target reads within a single call expression.

### Ownership and failure

There is no explicit allocation or release operation whose semantics can be established locally. Dependency-returned addresses are used as receivers, but that use does not establish whether they are borrowed, retained, newly owned, or released by any subsequent call.

There is no local rollback of prior writes or dependency effects on later failure. No retry or recovery behavior is shown.

## Dependencies

Opaque IDs identify direct dependencies. Virtual calls are identified only by their receiver role and table byte offset because no opaque IDs for their targets are supplied.

| Dependency | Visible role |
|---|---|
| F-ff499a944413 | Initial object operation; supplies the completed-path return value. |
| F-76b1c50b12f7 | First call in the optional refresh sequence. |
| F-22be49cfb47c | Supplies the value written at object offset `0xC0`. |
| F-d99467d4d2b3 | First supplies a zero/nonzero gate; called again to supply a selection argument. |
| F-938c950e5046 | Supplies the receiver for selection dispatch at `0xE8`. |
| F-682adfd5a561 | Receives a nonzero selected-object result and `0x6800`; returns an address tested for null. |
| F-8157a180d901 | Called with `0x1C` on that null-result branch. |
| F-2606ad6d534d | Supplies an argument for selected-object dispatch at `0x2C`. |
| F-fdd8589eca2d | Supplies the first argument to the completion helper. |
| F-8f9bd59e3ec5 | Completion helper receiving a dependency result and the original object. |
| F-a817d8d58c10 | Non-returning destination when the selection gate fails or selection returns null. |

Unknown virtual targets are the selection receiver’s `0xE8` entry, selected object’s `0x1C` and `0x2C` entries, secondary object’s `0x58` entry, and original object’s `0x13C` entry.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume valid memory and normal returns from all calls except the explicitly non-returning dependency.

1. **First condition is nonzero.**  
   With option zero and the first condition observed as nonzero, the function sets the marker, calls F-ff499a944413, clears the marker, and returns `1`. It does not evaluate the second condition or enter selection.

2. **Refresh precedes early return.**  
   With option `2`, the function performs both refresh calls and writes the second result at `0xC0` before evaluating the conditions. A subsequent nonzero second condition still causes return `1`.

3. **Completed path preserves the initial result.**  
   If the initial dependency returns `0xA5A50000`, both conditions are zero, selection succeeds, and all remaining calls return, the function returns `0xA5A50000`, regardless of the final dispatch’s return value.

4. **Zero selected-object result bypasses the secondary operation.**  
   If dispatch at selected-object offset `0x1C` returns zero, F-682adfd5a561 is not called. Completion still occurs.

5. **Null secondary result takes the alternate branch.**  
   If that dispatch returns nonzero and F-682adfd5a561 returns null, F-8157a180d901 receives `0x1C`, followed by F-2606ad6d534d and selected-object dispatch at `0x2C`. Completion follows if those calls return.

6. **Non-Boolean option remains non-Boolean.**  
   With option `2`, a completed invocation passes `2`, zero, and `3` as the final dispatch’s last three arguments. It does not convert `2` to `1`.

7. **Selection gate fails.**  
   If the first F-d99467d4d2b3 result is zero, F-a817d8d58c10 is invoked. Selection dispatch and completion are not reached through the visible control flow.

8. **Marker is not restored.**  
   If the marker initially contains `7`, normal return from the initial dependency is followed by a direct write of zero, not restoration of `7`.

## Uncertainties

The following remain **documented_only**:

- Application-level meaning of the operation, fields, constants, and virtual methods.
- Exact dependency signatures, especially the differing visible argument lists for F-d99467d4d2b3.
- Calling convention and argument interpretation of the dispatch at `0x58`.
- Dependency ownership, lifetime, synchronization, reentrancy, and global-state effects.
- Whether the non-returning dependency terminates, throws, transfers control, or behaves otherwise.
- Behavior after invalid pointers, invalid virtual tables, or memory-access failure.
- Exact machine-load implementation of the narrowed first condition.
- Exceptional cleanup and externally imposed concurrency guarantees.

The explicit marker clear does not prove that the marker remains zero afterward: later dependencies may modify it. Likewise, the optional assignment does not prove the final value of the field at `0xC0`.

## Evidence

This contract is based solely on static inspection of the supplied restricted source representation. Its established claims concern visible access widths, offsets, call order, branches, arguments, and return-value propagation.

Names suggesting higher-level purpose are not treated as semantic evidence. Unknown callee behavior is not assumed to preserve state or manage ownership.

No implementation access, runtime execution, or runtime tests have been performed. The acceptance examples describe conditional obligations derived from static evidence, not observed executions.
