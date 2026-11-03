<!-- Trusted provenance: F-afe4a0f8c883; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afe4a0f8c883

## Purpose

F-afe4a0f8c883 dispatches between two behaviors using the result of F-da61a61b6a91:

- When that result equals 2, it selects a one-byte value through an ordered sequence of dependency calls, writes that value to the supplied destination, and then calls a final dependency.
- For every other result, it passes the destination to F-e1f76e681c34 instead.

Every path that reaches the function’s ordinary return returns a 32-bit zero. This return value does not distinguish the selected byte, the delegated path, or dependency-level failures.

The interpretation of the selected values as application states, statuses, or categories is inferred intent, not established behavior. Their established meaning here is limited to the decision rules below.

## Interface

The supplied representation exposes four arguments:

| Argument | Represented form | Established use |
|---|---|---|
| Destination | Pointer to a one-byte object | Receives a direct byte write in the branch selected by a result of 2; otherwise is passed to F-e1f76e681c34. |
| Second argument | Opaque 32-bit value | Not referenced by this function’s visible operations. |
| Third argument | Opaque 32-bit value | Not referenced by this function’s visible operations. |
| Initial state | Represented 32-bit integer | Copied into a local state object before the dispatch query. Dependencies subsequently receive the address of that local object in the selection branch. |

The return is a 32-bit value with all bits zero whenever ordinary return is reached.

The integer-width description follows the supplied representation. A separate ABI declaration, calling convention, and original source-level types are not available.

### Destination requirements

In the branch selected by a query result of 2, the destination must designate writable storage for at least one byte at the time of the direct write. There is no visible null check, bounds check, or destination validation before that write.

In the other branch, destination requirements are those of F-e1f76e681c34 and are unknown here. The wrapper forwards the pointer without checking it.

The destination need not have any particular initial byte value for the visible selection logic: that logic does not read the destination. Dependencies may have additional requirements that are not established by this function.

### State storage and ownership

The function creates a local integer state initialized from the fourth argument. The selection dependency receives the address of this same object on every invocation within the call.

This is temporary local storage, not an explicitly allocated object or an ownership-transfer mechanism. Its validity is bounded by the containing invocation. Whether a dependency retains the address, reads it, or writes through it is not established by the caller alone. Retaining it would not extend the storage’s lifetime.

No visible operation allocates, frees, or transfers ownership of the destination. Unknown dependencies must not be assumed to preserve or release any caller-owned resource.

## Behavior

### Dispatch and initial ordering

The function first copies the initial-state argument into its local integer state. It then calls F-da61a61b6a91 with no explicit arguments.

The returned integer is compared with exactly 2. This is an equality test, not a truth-value test:

- Exactly 2 selects the byte-selection branch.
- Every other integer, including zero and negative values, selects the delegated branch.

No second query is made. Later changes to external state do not cause this function to reconsider the branch.

### Byte-selection branch

For a query result of 2, the observable sequence is:

1. Call F-1e38d5035e46 with no explicit arguments.
2. Call F-8af40cc8e379 with selector `0x5F` and the address of the local integer state.
3. Read the local state after that call returns and compare it with exactly 1.
4. Make any further selection calls required by the rules below.
5. Write the selected value as one byte at the destination.
6. Call F-cd9bbb6d9b7d with no explicit arguments.
7. Return the 32-bit value zero, if control returns normally from the preceding operations.

The selection rules are:

| State observations after the indicated calls | Selected byte | Selection calls made |
|---|---:|---|
| State equals 1 after selector `0x5F` | 1 | `0x5F` only |
| State does not equal 1 after `0x5F`, then equals 0 after `0x5E` | 4 | `0x5F`, then `0x5E` |
| State does not equal 1 after `0x5F`, is nonzero after `0x5E`, then equals 0 after `0x60` | 6 | `0x5F`, then `0x5E`, then `0x60` |
| State does not equal 1 after `0x5F`, is nonzero after `0x5E`, then is nonzero after `0x60` | 7 | `0x5F`, then `0x5E`, then `0x60` |

The same local state object is reused throughout. There is no visible reset to the fourth argument between selection calls. Consequently, a later call receives the storage left by the preceding call, rather than a fresh copy of the input.

The tests have different exactness:

- The first test accepts only the integer value 1.
- The second and third tests distinguish zero from every nonzero value.
- Negative values are therefore treated as “not 1” by the first test and “nonzero” by the later tests.

The visible code does not use a return value from F-8af40cc8e379. Its decision depends on the local state read after the call. If the dependency leaves that state unchanged, the existing state value controls the next decision. The caller does not visibly detect an omitted update.

Exactly one direct destination write occurs on a normally completed selection path. Earlier selection decisions do not write provisional values to the destination. The write is one byte wide, with value 1, 4, 6, or 7; no adjacent bytes are directly written by this function.

The destination write precedes F-cd9bbb6d9b7d. Therefore, the selected byte is established at that point in the observable sequence. The final dependency is not assumed to preserve it: if that dependency can reach the destination through external state or an alias, the byte present at function return may differ.

### Delegated branch

For every query result other than 2, the function calls F-e1f76e681c34 with the destination pointer.

On this path, the wrapper itself:

- Makes no call to F-1e38d5035e46.
- Makes no call to F-8af40cc8e379.
- Makes no direct destination read or write.
- Makes no call to F-cd9bbb6d9b7d.
- Returns zero if the delegated call returns normally.

The delegated dependency’s return value, if it has one, does not affect the wrapper’s ordinary return value. Its destination effects remain unspecified here.

### Arithmetic and error behavior

The visible selection logic performs no arithmetic on the input state and no pointer arithmetic. Its integer operations are equality comparisons with 2, 1, and 0. The selected constants all fit in one byte, so storing them does not require a narrowing-overflow interpretation.

There is no visible error return, retry, exception handling, or recovery path. Dependency return values are not used as error indicators, except that the query result selects the branch.

The zero return is conditional on ordinary completion. It does not establish what happens if a dependency does not return, transfers control elsewhere, or a memory access fails. In particular, there is no visible mechanism guaranteeing the final dependency call after a failed destination write.

## Side effects

Established direct effects are limited to local-state initialization and, in the selection branch, one destination-byte write.

Dependency calls can introduce additional effects not visible in this function. No claim is made that they preserve globals, caller storage, resources, or the destination.

The ordering constraints are significant:

- Local-state initialization precedes the query.
- The query precedes either branch’s dependency calls.
- In the selection branch, F-1e38d5035e46 precedes every selection call.
- Selection calls occur in the order `0x5F`, optionally `0x5E`, optionally `0x60`.
- Each state comparison follows its corresponding selection call.
- The destination write follows all selection calls actually taken.
- F-cd9bbb6d9b7d follows the destination write.

There is no visible synchronization, rollback, or atomic transaction spanning these effects. Thread safety and reentrancy depend on facts about the dependencies and surrounding environment that are unavailable here.

## Dependencies

| Opaque dependency ID | Explicit arguments | Caller-visible role |
|---|---|---|
| F-da61a61b6a91 | None | Produces the integer dispatch result; only equality with 2 matters. |
| F-1e38d5035e46 | None | Called once before selection calls, only when the dispatch result is 2. Its purpose is unknown. |
| F-8af40cc8e379 | Selector and address of the local integer state | Called one to three times. The caller examines the state after each call; no dependency return value controls selection. |
| F-cd9bbb6d9b7d | None | Called once after the direct destination write in the selection branch. Its purpose and effects are unknown. |
| F-e1f76e681c34 | Destination pointer | Handles every dispatch result other than 2. Its output behavior is unknown. |

The placement of F-1e38d5035e46 and F-cd9bbb6d9b7d could suggest setup and teardown, but no paired-resource, locking, or cleanup semantics are established. In particular, their positions do not prove that they acquire and release the same resource.

## Acceptance examples

These are conditional contract examples, not executed tests. Each assumes valid required storage and ordinary return from dependencies. State observations described below are conditions supplied by the dependency behavior, not asserted implementations of that dependency.

1. **Immediate selection of 1.**  
   The query returns 2. After selector `0x5F`, the state equals 1. The function writes byte 1 and then calls F-cd9bbb6d9b7d. Neither `0x5E` nor `0x60` is requested.

2. **Selection of 4 after a nonmatching first state.**  
   The query returns 2. The state is −1 after `0x5F` and zero after `0x5E`. The function writes byte 4. Selector `0x60` is not requested.

3. **Selection of 6.**  
   The query returns 2. The states after `0x5F`, `0x5E`, and `0x60` are respectively 0, 1, and 0. The function writes byte 6. A value of 1 at the second decision is nonzero and therefore requires the third selection call.

4. **Selection of 7 using a negative final state.**  
   The query returns 2. The states after the three selectors are respectively 2, −3, and −1. The function writes byte 7. Neither negative value is treated as zero.

5. **Selection calls that preserve the state.**  
   If F-8af40cc8e379 leaves the local state unchanged on every invocation, initial state 1 selects byte 1, initial state 0 selects byte 4, and any other initial state selects byte 7. Under this condition, byte 6 cannot be selected.

6. **Delegation without a wrapper write.**  
   The query returns −1. The destination is passed to F-e1f76e681c34. No selection calls or direct wrapper destination write occur. If the dependency returns normally, the wrapper returns zero regardless of that dependency’s return value.

7. **Post-write mutation by the final dependency.**  
   Selection chooses byte 4. The wrapper writes 4 before calling F-cd9bbb6d9b7d. If that dependency independently changes the destination, a different final byte does not contradict the established ordering contract.

8. **Unused argument variation.**  
   Changing only the second or third argument does not change any visible operand, comparison, or explicitly forwarded argument in this function. This does not assert preservation of undocumented machine state across dependencies.

## Uncertainties

The following are **documented_only**:

- The application meaning of the query result, selectors, and selected byte values.
- Exact ABI details beyond the supplied representation’s widths and forms.
- Whether F-8af40cc8e379 reads, overwrites, or retains the state pointer.
- Global-state, resource, allocation, and synchronization effects of every dependency.
- Destination requirements and effects of F-e1f76e681c34.
- Whether F-cd9bbb6d9b7d preserves the destination byte.
- Dependency failure signaling, nonlocal control flow, and behavior under invalid memory access.
- Thread safety and reentrancy.

## Evidence

Established behavior comes from the supplied static function representation: argument references, constant comparisons, conditional call structure, state-address reuse, one-byte assignment, call ordering, and constant ordinary return.

The possible interpretation as classification and the possible setup/teardown roles are inferred intent only. No dependency bodies or independent ABI declarations were provided. No runtime tests have been performed.
