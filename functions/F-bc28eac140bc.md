<!-- Trusted provenance: F-bc28eac140bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc28eac140bc

## Purpose

F-bc28eac140bc dispatches a numeric selector into one of several ordered sequences of dependency calls. Some sequences also store or inspect a single byte in shared state.

The established behavior is the dispatch, call ordering, argument selection, byte access, and final status value. The available evidence does not establish the application-level meaning of these sequences. In particular, the numeric arguments must not be treated as named modes, events, or resource states without additional evidence.

The function has two explicit normal-return values:

- **0**, after the sequence for any selector from 1 through 7, except selector 3.
- **9**, for selectors 0 and 3 and all selectors outside the explicitly handled range.

A return of 0 establishes completion of the selected sequence only if every invoked dependency returns normally. It is not independently established as an application-level success indication.

## Interface

### Input

The function accepts one integer selector. The recovered representation uses a 32-bit integer model. The recognized values are 0 through 7.

Negative values and values greater than 7 take the unrecognized-selector path and return 9. There is no range normalization, masking, or conversion into a smaller selector before dispatch.

### Output

The function returns a 32-bit value. Every explicit normal return is either 0 or 9.

Dependency results are generally discarded. Two results are used on selector 1:

- A 32-bit value returned by F-22b3f347e7b9 is retained and later passed to F-884b5e61e19b.
- An integer returned by F-bea78103a5fd is stored, after narrowing, into one byte of shared state and is also compared at its original integer width against 2.

### Shared inputs and layout

For this contract, the following neutral labels identify shared storage without asserting its application meaning:

| Label | Established use |
|---|---|
| **Base slot** | Read at function entry to obtain a base address. |
| **Argument slot A** | Read to supply an argument to F-3ecba43a4401 or F-2290ce995e83, depending on the selector. |
| **Argument slot B** | Read to supply an argument to F-7de4ba5051b5 or F-884b5e61e19b, depending on the selector. |
| **State byte** | The single byte at byte offset 1 from the base address captured at entry. |

The recovered model represents addresses and the relevant scalar slots using 32-bit quantities. A complete platform ABI, including pointer representation and integer-to-address rules, is not independently supplied.

Only the state byte’s offset and one-byte width are established. No enclosing structure size, adjacent-field layout, allocation origin, or lifetime policy is established.

### Preconditions

For all selectors, the base slot must be readable because it is read before dispatch, even when the selected path does not dereference the resulting address.

Additional requirements depend on the selected path:

- Selector 1 requires a writable state byte at the time of its store.
- Selectors 4 and 5 require a readable state byte at the time of their final conditional check.
- Slots A and B must be readable when their selected paths access them.
- Dependency-specific argument and environmental requirements remain unknown.

The function contains no explicit pointer validation, lifetime checks, or local recovery from invalid memory access.

## Behavior

### Common entry behavior

The function first reads the base slot and retains its value for the remainder of the invocation. Dispatch occurs after that read.

This distinction is observable when dependencies modify shared storage:

- Later changes to the base slot do not redirect this invocation’s state-byte access.
- Later changes to slots A or B can affect subsequent argument reads, because those slots are read where needed rather than captured alongside the base address.
- No dependency is assumed to preserve the memory reached through the retained base address.

### Selectors 0 and 3, and unrecognized selectors

These paths perform no dependency calls and no state-byte access. They return 9 after the common entry read.

This includes all negative selectors and all selectors greater than 7.

### Selector 1

The following actions occur in order:

1. Call F-22b3f347e7b9 with no explicit arguments and retain its 32-bit result.
2. Read slot A and call F-3ecba43a4401 with that value.
3. Call F-bea78103a5fd with no explicit arguments.
4. Store the narrowed result of F-bea78103a5fd into the state byte.
5. Compare the original, unnarrowed result against 2. If it is not equal to 2, call F-c91e142739b4 with arguments 3 and 0.
6. Call F-515c23f9254b with argument 3.
7. Read slot B and call F-7de4ba5051b5 with that value.
8. Call F-884b5e61e19b with the result retained from step 1.
9. Call F-dd8ffe218bef with argument 2.
10. Call F-d10642db277a with argument 1.
11. Return 0.

The state-byte write occurs before the optional call to F-c91e142739b4. Under the recovered byte-narrowing model, the stored value is the low eight bits of the returned integer. The condition controlling the optional call uses the full returned integer, not the stored byte.

Consequently, an integer whose low byte equals 2 but whose full value is not 2 still triggers F-c91e142739b4.

The retained result from step 1 is not refreshed from any shared slot. It remains the argument for step 8 regardless of changes to slots A, B, or the base slot by intervening dependencies.

### Selector 2

The following actions occur in order:

1. Call F-22b3f347e7b9 with no explicit arguments.
2. Read slot A and call F-3ecba43a4401 with that value.
3. Call F-515c23f9254b with argument 3.
4. Read slot B and call F-7de4ba5051b5 with that value.
5. Call F-dd8ffe218bef with argument 2.
6. Return 0.

The result of F-22b3f347e7b9 is not subsequently consumed by this function on this path. Its call must nevertheless occur.

Unlike selector 1, this path does not:

- Call F-bea78103a5fd.
- Write the state byte.
- Call F-c91e142739b4.
- Pass the retained result to F-884b5e61e19b.
- Call F-d10642db277a.

### Selector 4

The following actions occur in order:

1. Call F-dd8ffe218bef with argument 3.
2. Read slot B and call F-884b5e61e19b with that value.
3. Call F-d10642db277a with argument 4.
4. Call F-515c23f9254b with argument 4.
5. Read the state byte through the base address retained at entry.
6. If the byte equals 1, call F-c91e142739b4 with arguments 4 and 0.
7. Return 0.

The state byte is read after all preceding calls, not at entry. Those dependencies may therefore affect whether the final conditional call occurs.

### Selector 5

The following actions occur in order:

1. Call F-dd8ffe218bef with argument 3.
2. Read slot B and call F-884b5e61e19b with that value.
3. Call F-d10642db277a with argument 4.
4. Call F-515c23f9254b with argument 1.
5. Call F-515c23f9254b with argument 4.
6. Read slot A and call F-2290ce995e83 with that value.
7. Call F-59198292037b with no explicit arguments.
8. Call F-5568acc28855 with no explicit arguments.
9. Read the state byte through the base address retained at entry.
10. If the byte equals 1, call F-c91e142739b4 with arguments 4 and 0.
11. Return 0.

The two consecutive calls to F-515c23f9254b are distinct required actions, with argument 1 first and argument 4 second.

Slot A is read only after both calls. The final state-byte read occurs after F-5568acc28855 returns.

### Selector 6

The following actions occur in order:

1. Call F-26834e1d79f1 with no explicit arguments.
2. Call F-884b5e61e19b.
3. Return 0.

The second call is shown without an explicit argument, although other paths call the same dependency with one argument. Its effective argument contract on this path is **documented_only**. The evidence does not justify supplying zero, slot B, a previous return value, or any other invented argument.

The order and occurrence of both calls are established; the complete calling convention of the second call is not.

### Selector 7

The following actions occur in order:

1. Read slot A and call F-3ecba43a4401 with that value.
2. Call F-dd8ffe218bef with argument 2.
3. Call F-515c23f9254b with argument 3.
4. Call F-d10642db277a with argument 1.
5. Return 0.

This path does not read or write the state byte. Its call order differs from selector 1: F-dd8ffe218bef precedes F-515c23f9254b here.

## Side effects

### Direct memory effects

The only explicit write performed by this function is the selector-1 store to the state byte.

Direct shared reads comprise:

- The base-slot read on every invocation.
- Selected reads of slots A and B immediately before their corresponding calls.
- The late state-byte read on selectors 4 and 5.

No other direct memory write is established. This does not limit writes performed by dependencies.

### Dependency effects and ordering

Every listed call is observable and must retain its position relative to the other listed calls and direct memory accesses. Discarding a result does not make its producing call optional.

Dependencies may mutate shared slots, modify the state byte, invalidate referenced memory, or perform external effects. Their implementations are unavailable, so none of those possibilities may be excluded by this contract.

In particular:

- A dependency that changes slot B before a later slot-B read can change the argument supplied by this function.
- A dependency that changes the base slot does not change the already retained base address.
- A dependency that changes the state byte before the selector-4 or selector-5 check can change whether F-c91e142739b4 is invoked.
- Selector 1 does not guarantee its stored byte remains unchanged until return, because later dependencies may overwrite it.

### Ownership and errors

No explicit allocation, release, ownership transfer, or reference-count operation is established within this function. Passing a slot value or retained result to a dependency does not establish whether that dependency borrows, retains, consumes, or releases it.

No dependency result is interpreted as an error status. F-bea78103a5fd’s result controls a branch, but the evidence does not identify that branch as error handling.

There is no visible rollback or cleanup mechanism for interrupted sequences. Behavior following a fault, non-returning dependency, or nonlocal control transfer is not established by the normal-return paths.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-22b3f347e7b9 | Called on selectors 1 and 2; its 32-bit result is retained, but forwarded only on selector 1. |
| F-3ecba43a4401 | Receives the then-current slot-A value on selectors 1, 2, and 7. |
| F-bea78103a5fd | Called on selector 1; result supplies the state byte and full-width comparison against 2. |
| F-c91e142739b4 | Receives 3 and 0 conditionally on selector 1, or 4 and 0 conditionally on selectors 4 and 5. |
| F-515c23f9254b | Receives literal arguments 1, 3, or 4 according to the selected sequence. |
| F-7de4ba5051b5 | Receives the then-current slot-B value on selectors 1 and 2. |
| F-884b5e61e19b | Receives the retained result on selector 1, slot B on selectors 4 and 5, and has an unresolved argument interface on selector 6. |
| F-dd8ffe218bef | Receives 2 on selectors 1, 2, and 7; receives 3 on selectors 4 and 5. |
| F-d10642db277a | Receives 1 on selectors 1 and 7; receives 4 on selectors 4 and 5. |
| F-2290ce995e83 | Receives the then-current slot-A value on selector 5. |
| F-59198292037b | Called without explicit arguments on selector 5. |
| F-5568acc28855 | Called without explicit arguments on selector 5. |
| F-26834e1d79f1 | Called without explicit arguments on selector 6. |

No dependency’s semantic role, global-state preservation, or memory-ownership behavior is established solely by these interactions.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests. Normal completion assumes valid required memory and normally returning dependencies.

1. **Unrecognized selector:** With selector −1 or 8, the function reads the base slot, invokes no dependencies, does not access the state byte, and returns 9. Selectors 0 and 3 behave the same way.

2. **Selector 1, exact comparison value:** If F-bea78103a5fd returns 2, the function writes byte value 2 and skips F-c91e142739b4. It continues through the remaining selector-1 sequence and returns 0.

3. **Selector 1, matching low byte only:** If F-bea78103a5fd returns 258, the function writes byte value 2 but calls F-c91e142739b4 with 3 and 0, because 258 is not equal to 2.

4. **Selector 1, retained value:** If F-22b3f347e7b9 returns a particular 32-bit value, F-884b5e61e19b receives that value after F-7de4ba5051b5 returns. A change to slot B does not replace the retained value.

5. **Selector 2, no byte access:** The function invokes F-22b3f347e7b9 but does not forward its result. It performs no direct state-byte access and ends after F-dd8ffe218bef receives 2.

6. **Selector 4, late condition:** If the state byte equals 1 when the late read occurs, F-c91e142739b4 receives 4 and 0. If the byte instead equals 0, 2, or 255, that call is omitted. All these cases return 0 after normal completion.

7. **Selector 5, intervening mutation:** If F-5568acc28855 changes the state byte to 1 before returning, the final conditional call occurs, provided that mutation affects the byte reached through the retained base address.

8. **Changed base slot:** If a dependency replaces the base-slot value during selector 4, the final read still uses byte offset 1 from the original base address. This does not establish that the original address remains valid.

9. **Selector 6, unresolved interface:** Acceptance requires the two established calls in order and a subsequent return of 0. A particular effective argument to F-884b5e61e19b cannot be required from the supplied evidence.

10. **Selector 7, ordering:** F-dd8ffe218bef receives 2 before F-515c23f9254b receives 3. Reversing those calls would not satisfy this contract.

## Uncertainties

- **documented_only — Selector-6 call interface:** The effective arguments and calling convention for F-884b5e61e19b are unresolved.
- **documented_only — Dependency semantics:** Application meanings, failure modes, ownership effects, and global preservation are unavailable.
- **documented_only — Platform details:** The recovered model supports 32-bit scalars and byte access, but a complete ABI and address-space definition are not supplied.
- **documented_only — Object lifetime:** The allocation extent and lifetime behind the retained base address are unknown.
- **documented_only — Concurrency and reentrancy:** No synchronization or atomicity guarantee is established. Ordered accesses within this invocation do not imply a coherent concurrent snapshot.
- **Inferred intent:** The selector likely chooses among coordinated state transitions. That interpretation is not required for any established behavior and supplies no additional guarantees.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence covers selector dispatch, explicit calls and their ordering, literal arguments, shared-value read locations, the one-byte store, the full-width comparison against 2, and explicit 32-bit return values.

Byte truncation and 32-bit scalar descriptions follow the recovered representation; complete target-platform semantics remain qualified as above. No application-level names or ownership rules have been inferred from dependency identities.

No runtime tests have been performed. The acceptance examples specify expected observations under stated conditions; they are not execution results.
