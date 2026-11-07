<!-- Trusted provenance: F-f8845acbf3e0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8845acbf3e0

## Purpose

F-f8845acbf3e0 conditionally processes an indexed range associated with an indirect collection. It resolves another object, obtains two signed counts, selects their minimum, and populates temporary storage with descriptor references before processing collection values.

For each selected position, it retrieves a 32-bit value into a reusable slot, reads a descriptor reference from temporary storage, optionally invokes that descriptor’s callback, and passes the resulting slot contents to a write-like dependency.

Populating temporary storage does **not** establish an immutable snapshot. Processing reads each entry after the closing phase call and that position’s retrieval call. Those calls, earlier callbacks, or other intervening dependencies may modify the storage through an alias. Using the originally captured references is therefore conditional on preservation of the relevant entries until they are read.

The return is a 64-bit bit pattern with separately constructed 32-bit components. The low component indicates normal completion or an explicit failure path. The high component starts as the second input and can be replaced by descriptor data when a callback is invoked.

Collection transformation, allocation, cleanup, and phase coordination are inferred roles. The dependencies’ underlying semantics are not established.

## Interface

### Inputs

| Input | Representation | Visible use |
|---|---|---|
| First input | Opaque 32-bit word | Unused |
| Second input | Unsigned 32-bit word | Initial high return component; argument to the opening phase call, every callback, and every write-like call |
| Third input | Pointer to a 32-bit collection-reference cell | Directly read for the initial check and second count query; passed as a pointer to retrieval operations and callbacks |
| Fourth input | Opaque 32-bit word | Initializes the reusable 32-bit slot after the initial nonzero check |

The first input is not forwarded to any visible dependency.

The third input must address a readable 32-bit cell even for the immediate-return path. The routine checks the cell’s contents; it does not first validate the pointer itself.

For every processed position, retrieval replaces the reusable slot before callback invocation or the write-like call. Thus the fourth input has no direct dataflow into those per-position arguments. This does not establish that unknown dependencies cannot observe memory through other aliases.

### Return representation

| Component | Established construction |
|---|---|
| Bits 31–0 | 1 for an initially zero cell or normal completion; 0 for resolution failure or allocation failure |
| Bits 63–32 | Initially the second input; replaced by descriptor offset-0 data before each invoked callback; set to zero on allocation failure |

If allocation-failure reporting returns normally, the result is exactly zero.

Resolution failure returns the second input in the high component and zero in the low component. Consequently, failure need not produce an all-zero result. The full result is not an ordinary Boolean.

The public meaning and intended signed interpretation of the return components are **documented_only**.

### Referenced layouts

A successful resolution result is treated as an object address. The routine reads a 32-bit field at byte offset 4 from that object.

A nonzero descriptor reference is interpreted as follows:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Replacement high return component and callback argument five |
| 4 | 32 bits | Callback argument six |
| 16 | 32 bits | Callback address; zero disables invocation |

No direct descriptor reads at offsets 8 or 12 are shown.

Temporary descriptor storage has consecutive 4-byte entries. Each entry holds a 32-bit descriptor reference or zero. The supplied representation uses 32-bit descriptor references and callback addresses.

## Behavior

### Initial check and resolution

The first input-memory read is the 32-bit cell addressed by the third input.

If the cell contains zero, the routine immediately returns high component equal to the second input and low component equal to 1. It invokes no dependency and requests no temporary storage.

Otherwise, it initializes the reusable slot from the fourth input and calls F-2abff646f58d without explicit arguments.

If resolution returns zero, the routine returns high component equal to the second input and low component zero. No phase call, count query, allocation request, reporting call, processing operation, or cleanup handoff occurs.

A nonzero resolution result enables the remaining behavior.

### Opening phase and count selection

The routine next performs these operations in order:

1. Calls F-890414d1fd99 with six visible arguments: 5, 2, 0, 0, the original second input, and the third input pointer.
2. Reads the resolved object’s 32-bit field at byte offset 4 and passes that value to F-10b760faca36.
3. Rereads the collection-reference cell and passes its current 32-bit contents to F-10b760faca36.
4. Selects the smaller result using signed 32-bit comparison.

The initial collection-reference value is not reused for the second count query. Earlier dependencies may affect the later read. Likewise, the resolved-object field is read after the opening phase call, not before it.

A zero or negative selected count is accepted without explicit diagnostic. It suppresses allocation and per-position processing.

### Temporary allocation and population

For a positive selected count, F-43da6b23bd44 receives three arguments:

- The selected count shifted left by two in the visible 32-bit size expression.
- Zero.
- Zero.

The intended mathematical size is four bytes per entry. No overflow check is visible. A positive signed count alone does not establish that the size expression or subsequent address calculations represent a valid full-size allocation.

Safe execution requires valid size arithmetic, valid entry-address arithmetic, and storage covering every accessed entry. Exact behavior outside those requirements is **documented_only**.

If allocation returns nonzero, the routine visits positions in ascending order from zero through the selected count minus one. At each position it:

- Rereads the resolved object’s field at byte offset 4.
- Calls F-6bbddce1b930 with that field value and the position.
- Writes the returned 32-bit reference into the corresponding temporary entry.

The resolved-object field may differ between calls. Only references are written into temporary storage; descriptor contents and collection values are not copied.

Every population iteration precedes the processing phase. This establishes an order of writes, not persistence of their values. A later population dependency could modify an earlier entry through an alias. Subsequent dependencies could also modify entries before processing reads them.

### Closing phase

After successful population, allocation failure, or a nonpositive selected count, the routine calls F-890414d1fd99 with four visible arguments: 6, 2, 0, 0.

This closing call occurs before the allocation-failure decision and before any per-position retrieval.

The opening and closing calls bracket count selection and temporary-storage population. Their visible argument counts differ. Locking, synchronization, ownership transfer, and preservation guarantees are not established.

In particular, the closing call is not shown receiving the temporary-storage address, but that does not prove the address is inaccessible through another alias.

### Allocation failure

A positive selected count with a zero allocation result takes the explicit allocation-failure path:

- The closing phase call occurs first.
- The pending high return component is set to zero.
- F-4fc89ec41a6d receives 15, 106, and 65.
- If reporting returns normally, the routine returns exactly zero.

There is no per-position retrieval, callback, write-like call, or temporary-storage cleanup handoff on this path.

Whether the reporting dependency returns, terminates execution, or transfers control elsewhere is **documented_only**.

### Per-position processing

For a positive selected count with nonzero allocated storage, positions are processed in ascending order. The selected count is not queried again.

At each position, the observable order is:

1. Call F-1aed8c556fc7 with the third input pointer and current position.
2. Store its 32-bit result into the reusable slot.
3. Read the current contents of the corresponding temporary descriptor entry.
4. If that reference is nonzero, read the descriptor’s callback address at byte offset 16.
5. If the callback address is also nonzero, load descriptor offset-0 data into the pending high return component and invoke the callback.
6. Call F-f42b2d921e94 with the original second input, current position, and current 32-bit slot contents.

Step 3 uses the entry’s value **when read**, not a separately retained copy of its population result. The closing phase call, that position’s retrieval call, earlier callbacks, and other intervening dependencies may have changed it through an alias.

Once the entry is read for a position, that loaded reference selects the descriptor for that position’s field reads. The routine does not reread the temporary entry between callback eligibility checking and callback invocation.

The callback receives six arguments:

| Position | Argument |
|---|---|
| 1 | Original second input |
| 2 | Third input pointer, not the collection-reference cell’s contents |
| 3 | Address of the reusable 32-bit slot |
| 4 | Current position |
| 5 | Descriptor word loaded from byte offset 0 |
| 6 | Descriptor word read from byte offset 4 |

The offset-0 load and assignment to the pending high component complete before evaluation of the offset-4 argument read. Both precede callback entry. No additional argument-expression evaluation order is asserted.

A zero descriptor reference suppresses all descriptor-field reads. A nonzero reference with a zero callback address suppresses the offset-0 and offset-4 reads and callback invocation. Both cases still perform retrieval and the write-like call.

Callback return values are ignored. A callback’s slot modification is consumed by the following write-like call. If no intervening operation changes the slot after retrieval stores its result, the write-like call receives that retrieval result.

Replacing the pending high return component does not replace the original second input. Later callbacks and write-like calls continue receiving the original second input.

### Completion

After normal processing, a nonzero temporary-storage address is passed once to F-5990a26eaaa3. Its result is ignored.

Normal completion returns low component 1. Its high component is the offset-0 word loaded for the last callback actually invoked, or the original second input if no callback was invoked.

A later skipped callback does not reset that component. Changing a descriptor’s offset-0 word after it has been loaded does not itself retroactively change the saved component.

These completion rules assume normal control returns and continued validity of required execution state and memory.

## Side effects

### Direct reads and writes

The routine directly writes:

- The reusable 32-bit slot.
- Each temporary descriptor entry during population.

It directly reads:

- The collection-reference cell.
- The resolved object’s field at byte offset 4.
- Each temporary descriptor entry during processing.
- Eligible descriptors’ callback-address and data fields.

No direct assignment to the collection-reference cell, resolved-object fields, or descriptor fields is shown.

### Aliasing and visibility

Neither valid memory nor normal dependency returns establish exclusive ownership or preservation of temporary storage.

Population initially writes reference values into entries. Processing later uses the contents found at each read. Guarantees that these are the originally populated references require an additional preservation condition covering all intervening operations.

Descriptor contents are also live rather than copied. A dependency can change callback eligibility or callback data before the corresponding field reads. Retrieval precedes both the temporary-entry read and descriptor-field reads, so retrieval-side mutations can affect either.

The write-like dependency’s apparent role does not establish which collection it updates, whether it succeeds, or whether it changes other shared state. Its return value is not inspected.

### Ownership and lifetime

The nonzero allocation result is handed to F-5990a26eaaa3 on normal completion. This proves a cleanup handoff, not actual release or exclusive ownership.

No cleanup handoff for the resolved object or descriptors is visible. Their ownership is **documented_only**.

Callbacks receive the address of one reusable slot. Later retrievals overwrite it. The routine grants no established right to retain or access that address after its lifetime.

No cleanup path for exceptions or nonlocal exits is shown. Dependencies must leave subsequently accessed memory usable for the normal execution described here.

## Dependencies

| Opaque ID | Observed call-site contract |
|---|---|
| F-2abff646f58d | No explicit arguments; zero selects resolution failure; nonzero is used as an object address |
| F-890414d1fd99 | Opening arguments: 5, 2, 0, 0, second input, third input pointer; closing arguments: 6, 2, 0, 0 |
| F-10b760faca36 | Called twice with 32-bit values; results compared as signed 32-bit counts |
| F-43da6b23bd44 | Receives the 32-bit size expression and two zeros; zero selects allocation failure |
| F-6bbddce1b930 | Receives current resolved-object field and position; returns a 32-bit reference for storage |
| F-1aed8c556fc7 | Receives third input pointer and position; returns the 32-bit value stored in the reusable slot |
| F-f42b2d921e94 | Receives original second input, position, and current slot contents |
| F-5990a26eaaa3 | Receives the nonzero temporary-storage address after normal processing |
| F-4fc89ec41a6d | Receives 15, 106, and 65 after allocation failure |

Descriptor-supplied callbacks are dynamic dependencies without supplied independent contracts. No dependency is assumed to preserve globals, preserve aliased storage, or acquire or release ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Required memory and arithmetic are valid, and dependencies return normally unless stated otherwise.

1. **Initially zero cell.** With second input 9 and collection-reference cell zero, the result has high component 9 and low component 1. No dependency runs.

2. **Resolution failure.** An initially nonzero cell followed by zero from F-2abff646f58d yields the second input above a zero low component. There is no phase call or failure-reporting call.

3. **Nonpositive minimum.** Count results 7 and 0 produce both phase calls but no allocation, retrieval, callback, write-like call, or cleanup handoff. The result has the original second input above low component 1. A negative signed minimum follows the same processing-skip path.

4. **Allocation failure.** Counts 3 and 5 produce a size argument of 12. If allocation returns zero, the closing phase call precedes reporting with 15, 106, and 65. Returning from reporting yields exactly zero.

5. **Callback transformation.** If the entry read selects an eligible descriptor with offset-0 word 23, retrieval returns 10, and the callback changes the slot to 17, the write-like call receives 17. The pending high component becomes 23. The callback return value is ignored.

6. **Closing-call replacement.** Suppose population writes descriptor reference A. If the closing phase call changes that entry through an alias to reference B, and retrieval preserves it, processing uses B. A is not selected merely because it was originally written.

7. **Retrieval-side replacement.** Suppose an entry contains A immediately before retrieval. If retrieval changes it to zero through an alias, processing reads zero and performs no descriptor-field reads or callback. It still passes the slot value to the write-like call.

8. **Earlier callback changes a later entry.** If position zero’s callback replaces position one’s entry with reference B, and intervening operations preserve that replacement, position one uses B. Separately, changing B’s callback address before its field read can enable or suppress invocation.

9. **Conditional preservation.** If every intervening operation preserves each populated entry until its processing read, processing uses the originally populated references. This is an explicit condition, not a guarantee of the routine.

10. **Last invoked callback controls the high component.** If one callback loads 23 and later positions skip invocation, normal completion retains high component 23. All selected positions still perform retrieval and a write-like call.

## Uncertainties

The following remain **documented_only**:

- Semantic meaning of the inputs, object fields, descriptor data, and return components.
- Synchronization or protocol guarantees of the phase calls.
- Nonnegative-count guarantees and maximum safe count.
- Exact overflow behavior outside valid size and address arithmetic.
- Allocation alignment, exclusivity, alias exposure, and preservation across calls.
- Whether F-5990a26eaaa3 releases memory or takes ownership.
- Ownership and lifetime management of resolved objects and descriptors.
- Whether retrieval and write-like operations address the same collection.
- Dependencies’ global effects, reentrancy, concurrency, and error behavior.
- Callback contracts and pointer-retention permissions.
- Recovery or cleanup after invalidation, exceptions, or nonlocal exits.

The source provides no guarantee that temporary descriptor entries retain their population values. That property requires independently established preservation by intervening operations.

## Evidence

**Established static evidence:** The supplied representation establishes input usage, 32-bit fields and temporary entries, signed count selection, allocation-size construction, ordered population and processing phases, post-retrieval entry reads, callback arguments and eligibility checks, offset-0-before-offset-4 data-read ordering, cleanup handoff, and 64-bit return construction.

**Inferred intent:** Resolution, counting, allocation, descriptor lookup, transformation, phase coordination, and cleanup describe apparent roles rather than independently established dependency semantics.

**Unknown preservation:** Temporary-entry writes and later reads are visible. Exclusive ownership, absence of aliases, and preservation across intervening calls are not established.

**Verification limit:** No runtime tests have been performed. This contract derives from the supplied static representation; prior prose, comments, and reviewer feedback are not independent behavioral evidence.
