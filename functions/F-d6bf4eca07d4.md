<!-- Trusted provenance: F-d6bf4eca07d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6bf4eca07d4

## Purpose

F-d6bf4eca07d4 conditionally passes a selected descriptor and invocation-local storage to F-9b9c1c7483f4, then dispatches that dependency’s returned value through an entry associated with the supplied object.

An existing latch can suppress all dependency calls. Otherwise, two ordered queries determine the subsequent branch. One branch calls two additional dependencies before descriptor processing. Another passes the `SwitchLayout_IncorrectCombo` descriptor to F-9b9c1c7483f4, dispatches the returned value, performs a subsequent local-storage operation, writes a one-byte latch, and calls F-979606fd565e with 1000.

The descriptor names suggest screen navigation and layout switching. Actual display, navigation, layout changes, and the effects of the indirect callback are unknown. Likewise, the call to F-3caf67059a0e with integer 1 does not establish a state change or even a request for one.

Every explicit normal return is the 32-bit value 1. This result does not distinguish suppressed processing from completed dispatch.

## Interface

### Arguments

The function accepts four arguments:

| Argument | Representation | Established use |
|---|---|---|
| Object reference | Pointer to the represented object | Supplies fields, the dispatch-table reference, and the destination of the conditional latch write. |
| Second argument | 32-bit opaque value | Not directly used. |
| Third argument | 32-bit opaque value | Copied into an invocation-local word after the initial latch gate. That word’s address is passed to F-9b9c1c7483f4 on descriptor-processing paths. |
| Fourth argument | 32-bit opaque value | Copied into a separate invocation-local word after the initial latch gate. That word’s address is passed to F-9b9c1c7483f4 and, after normal indirect callback return, to F-8a5aadac202f. |

The third and fourth arguments are not established as pointers, strings, resource identifiers, or variadic arguments. Passing the address of the third-argument copy does not establish a larger argument sequence or authorize any particular interpretation of adjacent storage.

The two copies are distinct local words. No externally usable layout or adjacency relationship between them is established.

### Object representation

The supplied representation uses four-byte integer words. The relevant byte offsets are:

| Offset | Access width | Established role |
|---|---:|---|
| `0x00` | 32 bits | Reference used to locate the indirect dispatch entry. |
| `0xB0` | 32 bits | Argument to F-e39523d6b348 on the selector-2 branch. |
| `0xB4` | 32 bits | Repeatedly loaded as an argument to several dependencies. |
| `0xBC` | 32-bit read; 8-bit write | Initial latch test and conditional latch assignment. |

The initial latch check reads the word at `0xBC`, narrows it to a character-sized value, and compares that value with zero. Under the represented eight-bit narrowing, only the low eight value bits determine this comparison. Whether a nonzero narrowed value is interpreted as positive or negative does not affect the zero test.

The later latch assignment writes the byte value 1 at address `object + 0xBC`. It does not explicitly assign the remaining three bytes of the word. Whether the addressed byte corresponds to the low eight value bits of the word depends on target byte order, which is not independently established.

Indirect dispatch uses a callable entry at byte offset `0x11C` in the table referenced by the object’s first word.

### Result and preconditions

Every explicit normal completion returns 1 as a 32-bit value.

The object must support every access reached along the selected path. In particular:

- The initial word at `0xBC` must be readable even if processing stops immediately.
- Reached dependencies must receive arguments satisfying their own, unavailable contracts.
- Descriptor-processing paths require a valid dispatch-table reference and callable entry.
- The incorrect-combination path additionally requires writable storage at `object + 0xBC`.

No null check, bounds check, dispatch-target validation, or dependency-result validation is visible. Continued access after dependency calls requires the relevant object storage to remain valid; this function does not establish that callees preserve its lifetime.

## Behavior

### Initial latch gate

The first object-dependent operation tests the character-sized narrowing of the word at `0xBC`.

If that value is nonzero, the function returns 1 immediately. On this path:

- No dependency is called.
- Neither argument copy is initialized by the subsequent copy operations.
- No object write is performed by this function.

If the narrowed value is zero, the function copies the third and fourth arguments into separate local words and proceeds.

There is no second explicit latch check later in the invocation.

### Ordered queries

After the local copies, the function performs two calls in this order:

1. Read the field at `0xB4` and call F-2ab73c80407c with that value. Retain the returned 32-bit integer as the earlier condition.
2. Read the field at `0xB4` again and call F-0c534ffec98e with that value. Use the returned 32-bit integer as the selector.

The repeated field reads are significant. F-2ab73c80407c may change object state, referenced state, or globals; preservation is not established. Consequently, the two dependencies need not receive the same field value.

The first query occurs even when the eventual branch does not use its result. The retained earlier condition is not recomputed after the selector query.

### Selector equal to 2

For selector 2, the function:

1. Reads the current field at `0xB0` and calls F-e39523d6b348 with that value.
2. After that call returns, reads the current field at `0xB4` and calls F-3caf67059a0e with that value and integer 1.
3. Selects a descriptor whose contents are unavailable.
4. Executes the shared descriptor-processing sequence.
5. Returns 1.

The earlier condition does not affect this branch.

The fresh `0xB4` read occurs after F-e39523d6b348 returns. A modification made by that dependency can therefore affect the argument supplied to F-3caf67059a0e.

The results of these two additional calls are not inspected. Their application-level meanings and effects are unavailable. In particular, F-3caf67059a0e is established only as a call with the freshly loaded field and integer 1, not as a setter or state-change request.

This branch contains no direct latch assignment and no call to F-979606fd565e. That absence does not imply that dependencies leave the latch or other state unchanged.

### Selector equal to 0

For selector 0, the retained earlier condition must equal exactly 1:

- If it is not 1, return 1 without descriptor processing or indirect dispatch.
- If it is 1, select `GotoScreen_VolumeLimit`, execute the shared descriptor-processing sequence, and return 1.

Other nonzero condition values do not qualify. For example, condition 2 does not behave like condition 1.

This branch does not directly call F-e39523d6b348, F-3caf67059a0e, or F-979606fd565e, and does not directly assign the latch.

### Selector other than 0 or 2

For every other selector value:

1. Read the field at `0xB4` again.
2. Call F-f0c29a6e58c5 with that value.
3. If the returned integer is nonzero, return 1 immediately.
4. If it is zero, select `SwitchLayout_IncorrectCombo` and execute the shared descriptor-processing sequence.
5. After F-8a5aadac202f returns, write the byte value 1 at `object + 0xBC`.
6. Call F-979606fd565e with the object reference and integer 1000.
7. Return 1.

The earlier condition is irrelevant to this branch. The nonzero predicate test accepts any nonzero result as grounds for suppressing further processing, not just 1.

The selector comparison excludes only 0 and 2. Values represented as negative integers therefore enter this branch as well.

The established descriptor behavior is passing `SwitchLayout_IncorrectCombo` to F-9b9c1c7483f4 and dispatching the returned value. Displaying an incorrect-combination message is not established.

The direct latch write occurs after construction-like processing, indirect dispatch, and F-8a5aadac202f, but before F-979606fd565e. The meaning and units of 1000 are unknown.

### Shared descriptor-processing sequence

Each descriptor-processing path performs these operations in order:

1. Call F-9b9c1c7483f4 with:
   - the address of the local word initialized from the fourth argument;
   - the selected descriptor;
   - the address of the local word initialized from the third argument.
2. Retain its 32-bit returned value.
3. Read the object’s dispatch-table reference and obtain the callable entry at table offset `0x11C`.
4. Invoke that entry with the object reference and the retained returned value.
5. After normal callback return, call F-8a5aadac202f with the same local-storage address used as the first argument to F-9b9c1c7483f4.

The callback receives the returned value from F-9b9c1c7483f4, not explicitly either local-storage address. Those values could nevertheless alias; the dependency implementation is unavailable.

The dispatch target is resolved after F-9b9c1c7483f4 returns. Changes to the object’s table reference or relevant table contents during that dependency can affect the invoked target.

No explicit validity or success test is applied to the descriptor-processing result or callback result. F-8a5aadac202f is reached only after normal return from the preceding operations.

## Side effects

### Direct storage effects

The only direct object assignment shown is the one-byte write of 1 at `0xBC` on the unsuppressed other-selector branch.

The third and fourth arguments are copied into local words. Their addresses are exposed to F-9b9c1c7483f4, and the fourth-argument copy’s address is subsequently exposed to F-8a5aadac202f. The dependencies’ contracts do not establish whether they read, overwrite, or retain these addresses.

No direct global assignment is shown. This does not establish preservation of globals across dependency calls.

### Ownership and lifetime

The pairing of F-9b9c1c7483f4 with a later F-8a5aadac202f call suggests a temporary preparation-and-cleanup protocol. That is inferred intent, not an ownership guarantee.

Established behavior is limited to the address passing and normal call order. It does not establish:

- Whether either dependency allocates or releases memory.
- Whether the returned value is a pointer, handle, identifier, or other scalar.
- Whether the callback acquires ownership.
- Whether the callback may retain the value.
- Whether F-8a5aadac202f invalidates the dispatched value.
- Whether retained pointers into the local copies would remain usable.

The local words have invocation-local storage duration; no longer-lived replacement storage is shown.

### Re-entry and state visibility

On the incorrect-combination path, the indirect callback runs before this invocation’s direct latch assignment. Re-entry during the callback is therefore not prevented by that assignment. A nested invocation can pass its initial gate if the relevant narrowed latch value is still zero.

A dependency could independently change the latch earlier, but no such behavior is established. Conversely, a latch change after the initial gate does not cause the current invocation to recheck or abort.

No locking, atomic update, or concurrency protocol is visible.

### Errors and abnormal completion

There is no explicit error return, rollback, or result-dependent recovery sequence.

- Normal returns carrying failure-like values from untested dependencies do not themselves redirect the visible control flow.
- Effects of earlier calls are not explicitly reversed if a later operation fails.
- F-8a5aadac202f is not visibly protected by an abnormal-exit cleanup mechanism.
- An abnormal exit before the latch assignment prevents reaching that assignment through the visible normal sequence.
- An abnormal exit from F-979606fd565e occurs after the direct latch write; no restoring write is shown.

The surrounding runtime’s fault, exception, and unwinding behavior remains unknown.

## Dependencies

| Dependency | Established use |
|---|---|
| F-2ab73c80407c | First query, using a load from `0xB4`; its retained result controls the selector-0 branch through equality to 1. |
| F-0c534ffec98e | Second query, using a fresh `0xB4` load; returns the selector. |
| F-e39523d6b348 | Called with the current `0xB0` field only for selector 2. |
| F-3caf67059a0e | Called after F-e39523d6b348, with a fresh `0xB4` load and integer 1. No state effect is established. |
| F-f0c29a6e58c5 | Called for selectors other than 0 and 2; any nonzero result suppresses descriptor processing. |
| F-9b9c1c7483f4 | Receives two local-storage addresses and a selected descriptor; returns the 32-bit value passed to indirect dispatch. |
| F-8a5aadac202f | Receives the first local-storage address after normal indirect callback return. |
| F-979606fd565e | Receives the object and 1000 after the direct latch assignment. |

There is also an indirect dependency obtained through dispatch-table offset `0x11C`. Its opaque function ID and implementation are unavailable.

No dependency is established as pure, globally preserving, allocating, releasing, displaying, or state-setting.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example specifies mutation, assume valid reached storage, normal dependency returns, and no dependency changes relevant to the example.

1. **Latch already nonzero.** If narrowing the word at `0xBC` produces a nonzero byte value, return 1 without invoking any dependency.

2. **Only upper value bits set.** If the latch word has nonzero upper bits but zero low eight value bits, the initial gate permits processing. Both initial queries occur.

3. **Selector 0, qualifying condition.** With a zero latch test, condition 1, and selector 0, pass `GotoScreen_VolumeLimit` to F-9b9c1c7483f4, dispatch its returned value, call F-8a5aadac202f, and return 1. No direct latch write occurs. Actual navigation is not an acceptance requirement established here.

4. **Selector 0, noncanonical condition.** With condition 2 and selector 0, return 1 after the two queries. No descriptor-processing call occurs.

5. **Selector 2.** Call F-e39523d6b348, then F-3caf67059a0e with a fresh `0xB4` value and 1. Process the unavailable descriptor, dispatch the returned value, and call F-8a5aadac202f. This expectation neither requires nor establishes a state change.

6. **Other selector suppressed.** With selector 7 and a nonzero F-f0c29a6e58c5 result, return 1 without descriptor processing, indirect dispatch, direct latch assignment, or F-979606fd565e.

7. **Other selector accepted.** With selector 7 and predicate result zero, process `SwitchLayout_IncorrectCombo`, dispatch the returned value, call F-8a5aadac202f, write the latch byte, then call F-979606fd565e with 1000. No visible display outcome is implied.

8. **Mutation between initial queries.** If F-2ab73c80407c changes `0xB4`, F-0c534ffec98e receives the newly loaded value rather than necessarily the first query’s argument.

9. **Mutation before the second selector-2 call.** If F-e39523d6b348 changes `0xB4`, F-3caf67059a0e receives the subsequent field value.

10. **Dispatch-target mutation.** If F-9b9c1c7483f4 replaces the object’s dispatch-table reference with another valid reference, the ensuing callback is resolved through the replacement.

11. **Re-entry before latch assignment.** If the incorrect-combination callback re-enters the function while the narrowed latch remains zero, that nested invocation is not suppressed by the outer invocation’s still-pending latch write.

## Uncertainties

The following remain **documented_only**:

- The unavailable descriptor’s contents and meaning.
- Application-level meanings of the query results and numeric selector values.
- Actual display, navigation, layout, or state effects of dependencies and indirect dispatch.
- Target byte order and whether the addressed latch byte is the byte corresponding to the initial test’s low value bits.
- Dependency effects on globals, objects, referenced storage, and memory ownership.
- The representation, ownership, and lifetime of the dispatched value.
- Whether either local-storage address is retained or used to access additional storage.
- The meaning and units of 1000.
- Concurrency, synchronization, and re-entry guarantees beyond the visible ordering.
- Runtime behavior on invalid accesses or abnormal exits.

There are no range calculations, counters, or variable-size computations in the supplied representation. Relevant numeric operations are fixed-width copies, narrowing, exact comparisons, fixed-offset address calculation, and a one-byte constant assignment. They do not establish saturation, checked address arithmetic, or overflow recovery.

## Evidence

**Established static evidence:** The supplied function representation establishes the initial narrowed latch test, local argument copies, ordered queries, repeated field loads, exact branch comparisons, descriptor selection, dependency arguments, indirect dispatch offset, subsequent local-storage call, one-byte latch assignment, final 1000-valued call, and constant normal return.

**Inferred intent:** Descriptor names suggest UI-related operations. The repeated local-storage address across F-9b9c1c7483f4 and F-8a5aadac202f suggests temporary resource handling. Neither inference establishes display behavior, ownership, release behavior, or a state-change operation.

**Verification status:** No runtime tests have been performed. The acceptance examples describe conditional consequences of the supplied static evidence, not observed executions.
