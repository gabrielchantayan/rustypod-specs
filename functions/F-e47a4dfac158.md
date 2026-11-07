<!-- Trusted provenance: F-e47a4dfac158; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e47a4dfac158

## Purpose

F-e47a4dfac158 initializes an output word, evaluates conditions for a whole-structure operation, and otherwise performs an iterative operation until a comparison returns exactly 1.

**Established static behavior:** the function reads and writes 32-bit fields, invokes four dependencies, and returns a 64-bit value assembled from two 32-bit components. The preliminary conditions accept any nonzero comparison result; the fallback loop terminates only on an exact result of 1.

**Inferred intent:** the structure resembles a header-based container. The whole-structure branch resembles clearing that container and restoring header links; the fallback resembles advancing through and removing a range. These interpretations are not required by this contract and do not establish container semantics or ownership.

**Unknown:** the dependencies’ implementations, ownership rules, shared-state effects, and failure behavior. In particular, passing addresses of temporary working values does not establish that a dependency stops accessing them when its call returns.

## Interface

### Inputs

| Input | Representation | Direct role |
|---|---|---|
| Output slot | Address of a writable 32-bit word | Receives an initial header reference and branch-specific later writes. |
| Structure reference | 32-bit address-like value | Identifies storage with directly accessed fields at byte offsets 16 and 20. |
| Cursor input | Address of readable 32-bit storage | Passed to comparisons and an advance-like dependency; its first word supplies a retained object reference during iteration. |
| Boundary input | Opaque 32-bit value | Passed as an argument to comparison calls. Its interpretation is dependency-defined. |

The cursor input’s minimum readable extent describes only the function’s direct accesses. Dependencies may require additional storage or stronger validity conditions.

### Directly accessed layout

All offsets below are byte offsets; all listed fields are 32 bits wide.

| Location | Direct use |
|---|---|
| Structure + 16 | Header reference, read repeatedly. |
| Structure + 20 | Tested for nonzero when both preliminary comparisons pass; explicitly cleared on the whole-structure branch. |
| Header + 4 | Supplied to F-87ea3af356b2; subsequently cleared on that branch. |
| Header + 8 | Read during initialization; subsequently assigned a header reference on that branch. |
| Header + 12 | Assigned a header reference on that branch. |

This is a partial access layout, not a complete type definition. It does not establish that the field at structure offset 20 is a count or that the header fields are links.

### Working values and result

Two 32-bit working values contribute to the return:

- The **working object reference** starts with the output-slot address.
- The **working state reference** starts with the structure reference.

The fallback branch returns the working state reference in the high 32 bits and the working object reference in the low 32 bits, using their values when the return is constructed after the terminating comparison.

The whole-structure branch instead returns:

- High 32 bits: the header reference captured immediately before its final output write.
- Low 32 bits: the working object reference, which this branch leaves initialized to the output-slot address.

The halves are concatenated bit patterns. No signed extension, success-code interpretation, or count interpretation is established. Interpretation as an externally visible pair depends on an unavailable calling convention.

### Preconditions

Every direct access requires valid storage at the address actually used:

- The output slot must permit the initial 32-bit write.
- The structure must permit reading its header-reference field.
- The header selected after the initial output write must permit reading offset 8.
- Each fallback iteration requires a readable first cursor word.
- The whole-structure branch requires its indicated field reads and writes to be valid.

No explicit null, bounds, alignment, overflow, or allocation checks appear. Dependency-specific preconditions are additional and unknown.

Aliasing is not rejected. The order of stores and subsequent reads can make it observable. Validity must hold for the addresses produced by those later reads, not merely for the addresses present on entry.

## Behavior

### Initialization and initial output

The first direct operation reads the structure’s header reference and writes it to the output slot.

After that write, the function reads the structure’s header reference again and reads the selected header’s field at offset 8. That field is copied into temporary 32-bit storage for the first comparison.

The two header-reference reads are distinct. For example, if the output slot aliases the structure’s header-reference field, the initial write occurs before the second read. No immutable header snapshot is maintained.

The working object and state references are initialized as described above. A separate temporary reference holder is initialized with the cursor-input address. This holder is not the fallback loop’s cursor.

### First preliminary comparison

F-6183894e2484 receives:

1. The cursor input.
2. The address of the temporary containing the header-offset-8 value.

The result is interpreted as a 32-bit value:

- Zero skips the remaining whole-structure checks and reaches the fallback loop.
- Any nonzero value permits the second preliminary comparison.

The temporary is passed by address and is therefore exposed to dependency access. The function does not subsequently read its contents directly. This absence of a later direct read is not a guarantee that the dependency neither retains the address nor uses it through shared state.

### Second preliminary comparison

After a nonzero first result, the function rereads the structure’s header reference and stores it in the separate temporary reference holder. F-6183894e2484 then receives:

1. The boundary input.
2. The address of that holder.

A zero result reaches the fallback loop without a direct read of structure offset 20 for this decision.

For a nonzero result, the function reads structure offset 20:

- Zero reaches the fallback loop.
- Any nonzero 32-bit pattern selects the whole-structure branch.

Neither preliminary comparison requires an exact result of 1. Negative signed results, as well as positive results other than 1, satisfy these nonzero checks.

The holder’s contents are not subsequently consumed directly by the enclosing function. Dependency retention and indirect effects remain unknown.

### Whole-structure branch

This branch requires both preliminary comparisons to return nonzero and the subsequent read of structure offset 20 to be nonzero.

The function reads the current header reference and its field at offset 4, then calls F-87ea3af356b2 with:

1. The original structure reference.
2. The header-offset-4 value.

Its return value is ignored.

If that call returns normally, the function performs these direct stores in order:

1. Assigns a header reference read from the structure to offset 8 of the header selected through the structure.
2. Clears offset 4 of the header selected through the structure.
3. Assigns a header reference read from the structure to offset 12 of the header selected through the structure.
4. Clears structure offset 20.

These operations use fresh header-reference reads. F-87ea3af356b2 is not established to preserve the original header reference or any original header storage.

For each assignment involving both a destination header and a header-reference value, the representation establishes the store but does not establish an ordering between the separate address-side and value-side reads within that assignment. With stable storage, they identify the same current header. A stronger ordering under concurrent changes is not established.

After those stores, the function captures the structure’s current header reference, writes that captured value to the output slot, and constructs the branch’s return. The captured value supplies the high half even if the output write aliases and changes another field. The low half is the output-slot address.

This branch performs no fallback-loop comparison or iterative call.

### Fallback loop

Every path not taking the whole-structure branch reaches a comparison through F-6183894e2484 with:

1. The cursor input.
2. The boundary input.

The loop terminates only if the result equals 1. A result of zero, a negative result, or a positive result other than 1 enters another iteration.

Each iteration has this direct order:

1. Read and retain the first 32-bit word at the cursor input.
2. Call F-b65c251036f5 with the cursor input.
3. Assign the retained, pre-call word to the working object reference.
4. Call F-abde1e4c3702 with the address of the working state reference, the original structure reference, and the address of the working object reference.
5. Read the working state reference and write that value to the output slot.
6. Perform the next loop-condition comparison.

The value retained in step 1 survives cursor changes made by F-b65c251036f5. Step 3 supplies that retained value to F-abde1e4c3702 rather than rereading the advanced cursor.

Both working values are exposed by address to F-abde1e4c3702. Its changes to the working state determine the subsequent direct output write. Its changes to either working value can affect the eventual return.

The enclosing function does not reinitialize the working state between iterations. It does overwrite the working object reference with a newly retained cursor word before each later F-abde1e4c3702 call. These facts describe direct assignments only; intervening dependencies may also change exposed storage.

### Termination and retained-address effects

When a loop comparison returns exactly 1, there is no further direct output write. The function constructs its return from the working values then present.

A comparison result of 1 does **not** establish that those values still equal the values left by the preceding F-abde1e4c3702 call. That dependency received their addresses and may retain them. A later F-6183894e2484 call may access those addresses through dependency-shared state and modify either value before returning.

Consequently:

- The output write preceding the comparison uses the working state read for that write.
- The return uses the working values after the terminating comparison’s effects.
- Those observations need not agree.
- The comparison may also have independent effects on output or other caller-reachable storage; preservation is not established.

If the first fallback comparison returns 1, no iteration occurs. With no dependency modification of the relevant working storage, the return has the structure reference in its high half and the output-slot address in its low half. The function’s only direct output write in that execution is the initialization write.

## Side effects

Direct caller-visible writes are:

- The initial output write.
- One output write after each completed iterative operation.
- On the whole-structure branch, three header-field writes, one structure-field clear, and a final output write.

The function also initializes and updates temporary storage, some of which is exposed by address to dependencies.

There is no explicit allocation or deallocation primitive in the supplied representation. This does not establish that the complete operation allocates or releases nothing: dependencies may do either.

No ownership transfer is independently established. Passing a reference or an address does not establish acquisition, destruction, release, or exclusive access. Nor does return from a dependency establish that it has relinquished access to an exposed address.

The representation provides no lifetime extension for temporary storage after this function returns. Whether dependencies retain such addresses, and the consequences of accessing them later, remain unresolved.

There is no explicit error-result path. Comparison results control branches rather than reporting a documented error category. Other dependency return values are ignored. Invalid accesses, dependency faults, exceptional exits, and nonreturning calls are not characterized.

Fallback termination requires an eventual exact comparison result of 1. The advance-like call alone does not prove progress or termination.

## Dependencies

| Opaque ID | Observed interaction | Unresolved behavior |
|---|---|---|
| F-6183894e2484 | Returns a 32-bit comparison result; accepts cursor/temporary-address, boundary/holder-address, and cursor/boundary arrangements. | Comparison meaning, argument interpretation, mutation, shared-state access, retained-address access, and result range. |
| F-87ea3af356b2 | Receives the structure reference and current header-offset-4 value on the whole-structure branch. | Traversal, clearing, destruction, allocation, release, header replacement, and other memory effects. |
| F-b65c251036f5 | Receives the cursor input after its first word has been retained. | Whether it advances the cursor, its validity requirements, and effects on other storage. |
| F-abde1e4c3702 | Receives addresses of both working references and the original structure reference. | Meaning of updates, ownership effects, retention of supplied addresses, and coordination with later dependency calls. |

No dependency is assumed to preserve globals or caller-reachable memory merely because that memory is absent from its explicit arguments.

## Acceptance examples

These are conditional static-contract examples, not observed executions. Each assumes valid storage and only the dependency effects explicitly stated.

### Immediate fallback termination

Suppose the initial header reference is 0x00001000, the structure reference is 0x00002000, and the output-slot address is 0x00003000. The first preliminary comparison returns zero, and the first fallback comparison returns 1. Dependencies preserve the relevant storage.

Expected observations:

- The output receives 0x00001000.
- Neither iterative dependency is called.
- The return high half is 0x00002000.
- The return low half is 0x00003000.

The output value need not equal the return’s high half.

### Non-Boolean comparison results

A fallback comparison returning 2 enters an iteration. The 32-bit pattern 0xffffffff, interpreted as −1 by the displayed signed comparison type, also enters an iteration.

Either nonzero pattern passes a preliminary comparison’s condition. Treating the fallback condition as ordinary nonzero truth would violate the contract.

### Stable-header whole-structure operation

Suppose both preliminary comparisons return nonzero, structure offset 20 contains 3, and the header reference remains 0x00001000 throughout the branch.

After F-87ea3af356b2 returns, the direct stores produce, in order:

1. Header offset 8 receives 0x00001000.
2. Header offset 4 receives zero.
3. Header offset 12 receives 0x00001000.
4. Structure offset 20 receives zero.
5. The output receives 0x00001000.

The return high half is 0x00001000; the low half is the output-slot address. No claim is made about objects affected inside the dependency.

### Zero whole-structure field

Suppose both preliminary comparisons return nonzero, but structure offset 20 is zero.

F-87ea3af356b2 and the explicit header-reset sequence are skipped. A fallback comparison still occurs. The zero field alone does not cause a return.

### Retained cursor object with preserved working values

Suppose an iteration reads object reference A, after which F-b65c251036f5 changes the cursor’s first word to B.

F-abde1e4c3702 receives A through the working-object address. Suppose it leaves working state S and working object reference R. The following direct output write receives S.

If the next comparison returns exactly 1 **and preserves both working values**, the return high half is S and the low half is R.

### Terminating comparison changes retained working storage

Use the same iteration, but suppose F-abde1e4c3702 retains both working-value addresses. The subsequent comparison accesses them through shared dependency state, changes the working state to T and the working object reference to U, and returns 1.

Expected observations:

- The preceding direct output write received S.
- No additional direct output write occurs on loop exit.
- The return high half is T and the low half is U.

If the comparison does not independently alter the output slot, that slot remains S. An exact comparison result of 1 does not imply preservation of either working value.

## Uncertainties

The following remain **documented_only**:

- Logical structure and element types.
- Meanings of the header fields and structure offset 20.
- Dependency comparison semantics and ownership rules.
- Whether dependencies retain temporary addresses or communicate through shared state.
- Permitted aliasing, concurrency, reentrancy, alignment, and address-space constraints.
- Behavior after invalid memory access, dependency failure, or nontermination.
- External interpretation of the 64-bit composite result.
- Ordering or atomicity guarantees beyond the established direct sequence.

All displayed field accesses are 32 bits, and the return concatenates two 32-bit halves. No count increment or decrement is visible.

The fixed byte-offset calculations have no explicit checked-overflow handling. This contract does not infer wrapping, trapping, or fault guarantees for invalid address arithmetic; its access descriptions require valid resulting addresses.

## Evidence

This contract derives solely from the supplied static function representation and opaque dependency mapping.

Established evidence includes direct access widths and offsets, initialization order, short-circuited branch conditions, dependency argument arrangements, explicit store order, retention of the pre-advance cursor word, exposure of working-value addresses, and construction of the two-part return.

The possibility of later mutation through retained addresses is unresolved dependency behavior, not evidence that such mutation occurs. Return examples therefore either require preservation explicitly or describe the values after the terminating comparison.

No dependency implementations, independent type definitions, ABI documentation, or runtime observations were available. No runtime tests have been performed.
