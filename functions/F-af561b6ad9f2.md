<!-- Trusted provenance: F-af561b6ad9f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af561b6ad9f2

## Purpose

F-af561b6ad9f2 performs a budget-limited sequence of dependency calls, using a caller-supplied character cursor to determine whether each iteration may begin. It accumulates signed integer contributions from F-ff169115f82a and returns the accumulator plus one.

The established behavior is not a general string-processing contract. The function checks a character for zero, exposes its local cursor to F-d3922c7fe0ec, and obtains a contribution from F-ff169115f82a. Neither forward cursor movement nor any relationship between the inspected character and the contribution is established.

The budget limits entered iterations and associated dependency-call pairs. It does not establish a limit on bytes consumed, cursor displacement, dependency-internal work, or elapsed time.

## Interface

### Inputs

**Initial cursor**

A pointer to character storage, possibly null. The pointer is supplied by value.

A null initial cursor bypasses iteration. A nonnull cursor must designate readable storage whenever execution reaches a character check. Merely being nonnull does not establish readability.

**Iteration budget**

A signed integer. A strictly positive remaining budget permits a character check and potentially an iteration. Zero and negative budgets prevent iteration without reading through the cursor.

### Output

The return value has the source’s signed integer type.

For an execution that completes normally with defined arithmetic, the result is one plus the contributions accumulated during its completed iterations. An execution with no iterations returns exactly one.

There is no separate status output, cursor output, or explicit error return. Contribution values are not checked for error sentinels: any normally returned signed integer is used in the accumulation.

### Widths and representation

The available evidence establishes character-sized storage and signed integer inputs, contributions, accumulation, and return. It does not establish:

- Integer width or exact representable range.
- Pointer width or representation.
- The number of bits in a character-sized object.
- Plain-character signedness.
- Target byte order.
- Target-specific handling of invalid memory access or signed overflow.

A concrete 32-bit contract cannot be established from this function alone. In particular, 32-bit wrapping arithmetic must not be assumed.

The character comparison distinguishes zero from nonzero. Whether plain characters are signed does not change that distinction.

### Storage, layout, and ownership

The function creates a local cursor initialized from the initial cursor. F-d3922c7fe0ec receives the address of this local cursor object, allowing it to replace the cursor used by later checks.

That argument is not the address of a pointer variable owned by the caller. Replacing the local cursor therefore does not, by itself, replace the caller’s pointer variable.

No buffer length, encoding, terminator guarantee, external record layout, or supported layout of neighboring local storage is supplied. Receiving the cursor’s address does not authorize treating adjacent storage as additional arguments.

The wrapper does not explicitly allocate, release, or transfer ownership of the character storage. Dependency ownership and lifetime effects remain unknown.

## Behavior

### Entry and initial guard

The accumulator begins at zero. The local cursor begins with the supplied pointer value, and the remaining budget begins with the supplied budget.

The function checks whether the initial cursor is null before considering the loop:

- A null initial cursor causes an immediate no-iteration result of one.
- Neither dependency is called on that path.
- No character is read on that path, regardless of the budget.

This is an initial-pointer guard, not a validity check repeated before every dereference.

### Iteration admission

For a nonnull initial cursor, each loop condition first checks whether the remaining budget is strictly positive.

If the budget is zero or negative, the condition stops without reading through the cursor. This includes the condition reached after the last permitted iteration.

If the budget is positive, the function reads the character designated by the current local cursor:

- A zero character terminates the loop without entering another iteration.
- A nonzero character admits an iteration.

Consequently, a positive budget requires a valid readable cursor at every reached character check. The function supplies no recovery path for a null, dangling, or otherwise unreadable cursor at that point.

The read that admits an iteration precedes both dependency calls. The function does not explicitly pass the character value to either dependency.

### Ordering within an iteration

Each entered iteration has this order:

1. Call F-d3922c7fe0ec with the address of the local cursor.
2. If that call returns normally, call F-ff169115f82a with no explicit arguments.
3. If the second call returns normally, add its signed integer result to the accumulator.
4. Decrease the remaining budget by one.
5. Evaluate the next loop condition, checking the budget before any character read.

No return value from F-d3922c7fe0ec is used.

There is no cursor read, null check, or zero-character check between the two dependency calls. If the first dependency replaces the cursor with null or a pointer to a zero character, that replacement does not itself prevent the second call.

If a dependency fails to return normally, the source does not establish completion of the remaining iteration steps or a normal function result.

### Cursor and character state across calls

The function contains no independent cursor-advance operation. Later checks use the cursor value present in the local cursor object when those checks occur.

F-d3922c7fe0ec can directly replace that object through its argument. F-ff169115f82a receives no explicit pointer argument, but that does **not** establish that it preserves the cursor. The first dependency could retain or publish the local cursor’s address, making the object accessible to the second dependency through shared state.

Similarly, neither dependency is established to preserve the character storage. Even if the cursor value remains unchanged, the character at that address may change before the next check.

Therefore, subsequent behavior depends on state at the next condition, not solely on state immediately after the first dependency:

- A readable zero character at the next positive-budget check ends iteration.
- A readable nonzero character at that check permits another iteration.
- A null or unreadable cursor at that check leads to an invalid access, without a specified recovery result.
- If the remaining budget is no longer positive, the function does not perform that character check.

The same principle applies when the first dependency temporarily sets the cursor to null. The second dependency is still called; it may leave that value unchanged or replace it through an available alias. A later null dereference is established only if the cursor remains null when the next positive-budget character check occurs.

### Bounds and termination

Assuming normal dependency returns, valid memory access, and defined arithmetic, the number of entered iterations cannot exceed a positive initial budget. Cursor progress is unnecessary for that bound: a cursor that repeatedly designates a nonzero character still encounters budget exhaustion.

A zero-character check can terminate the loop earlier. The function does not require that such a character ever be reached.

The iteration bound is not a guarantee of normal termination. A dependency may not return, may terminate execution, or may cause another failure. Those behaviors are outside the supplied dependency evidence.

### Accumulation and final increment

For completed iterations with contributions \(r_1,\ldots,r_n\), the normal result is

\[
1+\sum_{k=1}^{n}r_k.
\]

This describes the result only when each executed signed addition is defined and representable. The actual sequence of intermediate accumulator updates matters. A mathematical final sum that fits does not repair an earlier overflow.

The final addition of one is a separate operation with its own representability requirement. For example, a maximum-valued accumulator cannot be incremented representably under ordinary signed-integer semantics.

The budget decrement has a different boundary: it executes only after a strictly positive budget admitted the iteration. Decreasing that positive signed value by one remains representable. No analogous range check protects the accumulator.

## Side effects

### Direct effects

The function directly:

- Reads a character at each reached positive-budget character check.
- Initializes and updates its local state.
- Exposes the address of its local cursor to F-d3922c7fe0ec.
- Calls the dependencies in the specified order.
- Uses each normally returned contribution in a signed addition.

It does not directly write through the character cursor, allocate memory, release memory, or update a caller-owned cursor variable.

These statements describe the wrapper’s own operations, not the transitive effects of its dependencies.

### Dependency-mediated effects

Neither dependency is established to preserve:

- Global or shared state.
- The local cursor.
- Characters in the input storage.
- Storage ownership or lifetime.
- Allocation state or other resources.

The absence of explicit allocation or release in this function does not establish that the complete call is allocation-free or that the initial storage remains live after dependency calls.

The local cursor object exists for this invocation. Its address is usable for access consistent with that lifetime, including possible access by another dependency through a retained alias while the invocation remains active. The source neither proves that such retention occurs nor provides a valid post-return lifetime for the object.

## Dependencies

### F-d3922c7fe0ec

**Established invocation:** receives the address of the function’s local character cursor.

**Established timing:** first dependency called in each entered iteration, after the positive-budget and nonzero-character checks.

**Established result handling:** no returned value is used.

**Unknown behavior:** cursor replacement, cursor progress, character-storage changes, alias retention or publication, global effects, allocation or release, ownership changes, errors, and termination.

There is no basis for specifying that this dependency advances one character, consumes one encoded unit, preserves the cursor, or leaves readable storage behind.

### F-ff169115f82a

**Established invocation:** called with no explicit arguments; supplies a signed integer contribution.

**Established timing:** called after F-d3922c7fe0ec returns normally, before accumulation and budget decrement.

**Established result handling:** its return value is added without range validation or error-sentinel interpretation.

**Unknown behavior:** contribution meaning and range, shared-state access, cursor changes through aliases, character-storage changes, memory ownership effects, errors, and termination.

The call order is established. A semantic data dependency between the calls is not. Absence of explicit arguments is not evidence of independence from the first call or preservation of its effects.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Unless stated otherwise, dependency calls return normally, required character reads are valid, and all executed arithmetic is representable.

1. **Null initial cursor**  
   With a null initial cursor and a budget of seven, the result is one. There are no character reads and no dependency calls. The same result holds for any representable budget.

2. **Nonpositive budget**  
   With a supplied nonnull cursor and a budget of zero or minus three, the result is one. The function does not dereference the cursor or call either dependency. This does not assert that arbitrary invalid pointer representations are valid inputs.

3. **Initial zero character**  
   With a readable initial cursor designating a zero character and a positive budget, the function reads zero, calls neither dependency, and returns one.

4. **One permitted iteration**  
   With a readable nonzero initial character, a budget of one, and a contribution of four, the result is five. F-d3922c7fe0ec precedes F-ff169115f82a. After that pair, budget exhaustion prevents a further character read, irrespective of the final cursor value. Effects or failures inside the dependencies remain separate conditions.

5. **Early stop after both dependencies**  
   With a budget of three and a readable nonzero initial character, suppose the first dependency replaces the cursor and the second returns six. If, **after both dependencies return normally, the current cursor designates a readable zero character when the next condition checks it**, exactly one dependency-call pair occurs and the result is seven. The first dependency’s replacement alone is insufficient: the second dependency may change the cursor or the character.

6. **No cursor progress**  
   With a budget of two, suppose each reached character check sees the same readable nonzero character and the contributions are three and minus two. Two call pairs occur and the result is two. This example requires the stated cursor and character conditions across both calls; no preservation guarantee is inferred from their interfaces.

7. **Temporary or persistent null replacement**  
   With a budget of two and a readable nonzero initial character, suppose the first dependency sets the local cursor to null and returns normally. The second dependency is still called without an intervening cursor check. If it returns normally and the cursor is **still null when the next positive-budget character check occurs**, that check attempts a null dereference. If the second dependency replaces the cursor through an alias, the next check instead uses the replacement; the temporary null value alone does not determine the outcome.

8. **Final-increment overflow**  
   If execution ends the loop with the accumulator equal to the maximum representable signed integer, adding one is not representable. The source establishes no wrapping result, saturation result, or error code.

9. **Intermediate overflow despite a fitting mathematical result**  
   If one contribution makes an accumulator addition overflow, later negative contributions do not restore a defined execution merely because the complete mathematical sum would fit. Each executed addition must satisfy the arithmetic precondition independently.

## Uncertainties

The following remain **documented_only**:

- Concrete character, integer, and pointer widths and the target ABI.
- Target-specific behavior beyond the ordinary signed operations shown.
- Input encoding, buffer extent, and termination conventions.
- Dependency-internal memory access and cursor evolution.
- Whether the first dependency retains or publishes the local cursor’s address.
- Whether the second dependency accesses that address or changes the character storage.
- Contribution meaning, range, and possible error conventions.
- Dependency effects on globals, resources, ownership, and storage lifetime.
- Exceptions, nonlocal exits, and nonreturning behavior.
- Any intended semantic meaning for the final increment.

No explicit error channel is present. Invalid reads, arithmetic overflow, or dependency failures are not converted by this function into a specified error result.

## Evidence

**Established static evidence:** the supplied body establishes the initial null guard, budget-first short-circuit condition, zero-character test, local cursor indirection, ordered dependency calls, signed accumulation, budget decrement, and final addition of one.

**Inferred intent:** bounded cursor processing or measurement is plausible. No parsing, encoding, counting, sizing, or ownership interpretation is adopted.

**Evidence limits:** dependency implementations and target ABI information were not supplied. The possible sharing of the cursor’s address explains why preservation cannot be assumed; it is not evidence that sharing actually occurs. Subsequent-check claims are conditioned on the cursor and character state at the check itself.

No runtime tests have been performed. All acceptance examples are conditional static deductions, not observed executions.
