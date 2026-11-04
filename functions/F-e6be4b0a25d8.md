<!-- Trusted provenance: F-e6be4b0a25d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6be4b0a25d8

## Purpose

F-e6be4b0a25d8 performs a preliminary operation, publishes a bounded signed quantity and a dependency-produced value, and conditionally invokes an indirect callback. It returns a 32-bit status: zero only when the callback’s return value equals the quantity observed after the callback; one on every other normally returning path.

The function appears intended to prepare and complete a transfer of at most 10,240 units. That interpretation is **inferred intent**, not an established contract: the units, the meaning of the dependency-produced value, and the callback’s operation are unknown.

The established behavior includes several important qualifications:

- The quantity has an upper bound but no explicit lower bound.
- The preliminary operation can affect the state subsequently read.
- The no-argument dependency runs after the first quantity write and before the quantity supplied to the callback is read.
- The equality check uses the quantity read after the callback, not a saved copy of the callback argument.
- No explicit cleanup or rollback occurs on failure.

## Interface

### Arguments

The function accepts six positional arguments.

| Position | Contract-level name | Representation and use |
|---|---|---|
| 1 | Descriptor | Pointer to storage interpreted as 32-bit signed integer fields. The second field supplies a selector; the first field supplies a quantity after the preliminary operation succeeds. |
| 2 | Callback object | Pointer forwarded to the preliminary operation and later used as the callback receiver. Its first 32-bit field supplies the base address of an indirect-dispatch structure. |
| 3 | Auxiliary value | Opaque 32-bit value forwarded unchanged to F-f68f8fcdbc7a. |
| 4 | Context value | Opaque 32-bit value forwarded unchanged to F-f68f8fcdbc7a and, for most selectors, also used as that dependency’s fifth argument. |
| 5 | Value output | Pointer to writable 32-bit integer storage. Receives the result of F-fab3799b3190 on paths that reach that dependency. |
| 6 | Quantity output | Pointer to writable 32-bit signed integer storage. Receives the bounded quantity and may subsequently receive zero. |

These are role names, not claims about source-level types or domain meaning.

The supplied representation uses four-byte integer fields and four-byte opaque values. Its dispatch expression also represents object and callback-table addresses through integer-sized storage. This contract describes that represented 32-bit layout; it does not establish portability to a different pointer width or ABI.

### Selector and object layouts

The descriptor’s first field is at byte offset 0. Its second field is at byte offset 4. The second field is read and converted to a character-sized value before selection.

On the represented eight-bit character interpretation, selector matching depends on the low eight bits, not equality of the entire second field. Values whose narrowed result is 2, 3, or 4 receive special handling. Signed versus unsigned interpretation of other character values does not change the branches because all other values follow the same default path.

For callback dispatch:

- The callback object’s first field supplies a dispatch-table base.
- The callback target is obtained from byte offset 16 within that table.
- The callback receives four arguments: the callback object, a nonzero dependency result, the current quantity output, and the literal value 2.

The meanings of the table’s other entries are unknown.

### Return value

The result is always one of:

- **0:** the indirect callback returned normally and its signed integer result equaled the quantity output read after that return.
- **1:** the preliminary operation returned nonzero, the no-argument dependency returned zero, or the callback result differed from the post-callback quantity.

There are no distinct return codes for those failure cases. Faults, nonlocal exits, and nonreturning dependencies are not converted to status 1 by any visible handling.

### Preconditions

Requirements are path-dependent:

- The descriptor’s second field must be readable at entry.
- All requirements imposed by F-f68f8fcdbc7a must hold when it is called.
- After that dependency returns zero, the descriptor’s first field must be readable and the quantity output must be writable.
- When F-fab3799b3190 returns, the value output must be writable.
- If its result is nonzero, the callback object, dispatch-table entry, and callback target must be valid for the indicated reads and call.
- Storage read through the quantity output must remain readable at each later use.

No null checks, capacity checks, dispatch-target checks, or explicit alias checks are visible.

## Behavior

### Selector choice and preliminary operation

The function first reads the descriptor’s second field and narrows it to a character-sized selector. It chooses a fifth argument for F-f68f8fcdbc7a as follows:

| Narrowed selector | Selected fifth argument |
|---|---|
| 2 | First fixed global value |
| 3 or 4 | Second fixed global value |
| Any other value | Context value, the fourth argument |

The fixed globals’ contents and meanings are not supplied. They are distinct storage references, not necessarily distinct values.

The represented operation initially reads the first fixed global value before applying the selection conditions. For selectors other than 2, that initial value is replaced by the context value; for selectors 3 and 4, it is then replaced by the second fixed global value. Thus the represented initial read and the ultimately selected argument are separate facts.

F-f68f8fcdbc7a is called with:

1. Descriptor.
2. Callback object.
3. Auxiliary value.
4. Context value.
5. Selected value.

Its result is tested only for equality to zero. If it is nonzero, F-e6be4b0a25d8 returns 1 without directly writing either output and without calling F-fab3799b3190 or the indirect callback.

“Without directly writing” is significant: F-f68f8fcdbc7a has received pointers and may itself modify relevant storage. Its effects are not constrained by this function.

### Quantity publication

If F-f68f8fcdbc7a returns zero, the function reads the descriptor’s first signed 32-bit field **after that call**.

If the value is greater than 10,239, the quantity selected for publication is 10,240. Otherwise, it is the field value itself. Equivalently, this is a signed upper cap at 10,240:

- 10,239 remains 10,239.
- 10,240 remains 10,240.
- Larger positive values become 10,240.
- Zero remains zero.
- Negative values remain negative.

The quantity output is then written with that value.

There is no addition, multiplication, or unsigned conversion in this bounding step. In particular, a negative quantity is not rejected or normalized to zero.

### Dependency result publication

After the quantity-output write, the function calls F-fab3799b3190 with no explicit arguments.

Its signed 32-bit return value is written to the value output. The zero/nonzero branch is based on the returned value retained by the caller, not on a subsequent reread of the value output.

If the returned value is zero:

1. The value output receives zero.
2. The quantity output receives zero.
3. The function returns 1.

The indirect callback is not invoked on this path.

If the returned value is nonzero, the function proceeds to indirect dispatch. Negative results count as nonzero and are forwarded; no positive-only check is present.

### Callback and completion

The function obtains the indirect target through the callback object and invokes it with:

- The callback object as receiver.
- The nonzero value returned by F-fab3799b3190.
- The quantity output’s value as read for this call.
- The literal integer 2.

The callback is invoked even when the quantity is zero or negative, provided the preceding dependency returned nonzero.

The quantity argument is not necessarily the originally published bounded quantity. F-fab3799b3190 may have affected reachable or global storage, and the output pointers may alias. The function reads the quantity output again when preparing the callback.

After the callback returns, the function reads the quantity output again and compares that value with the callback’s return value:

- Equal: return 0.
- Unequal: return 1.

The callback argument’s quantity and the comparison’s quantity can therefore differ. A claim that success proves the callback processed exactly the original bounded quantity would be stronger than the evidence supports.

The expression used to dispatch the callback does not establish a portable ordering among the callback-target lookup and the separate argument reads. It does establish that they occur after publication of the dependency result and before execution of the callback, followed by the final quantity read after the callback returns.

## Side effects

### Direct writes and their order

For a preliminary nonzero result, the function performs no direct output writes.

For a preliminary zero result followed by a zero no-argument dependency result, the direct writes occur in this order:

1. Bounded quantity to the quantity output.
2. Zero to the value output.
3. Zero to the quantity output.

For a preliminary zero result followed by a nonzero no-argument dependency result, the direct writes occur in this order:

1. Bounded quantity to the quantity output.
2. Dependency result to the value output.
3. No further direct writes by this function.

Dependency and callback effects can occur between or after these writes.

### Aliasing

The implementation contains no visible alias exclusion. Where overlapping storage is otherwise valid under the applicable execution model, the stated access order governs the result.

In particular, if the two output pointers designate the same 32-bit location:

- The dependency-result write replaces the earlier quantity.
- On a nonzero result, absent another intervening mutation, that result becomes the quantity passed to the callback.
- The final comparison uses whatever remains in the shared location after the callback.
- On a zero result, the shared location ends at zero.

Output storage could also overlap fields used for later dispatch. Such overlap may change the callback target or invalidate a subsequent access. The function does not preserve an independent copy of all dispatch state before writing outputs.

### Ownership and failure state

No direct allocation, release, reference-count operation, or ownership annotation is visible. F-fab3799b3190 might return an address-like token, handle, resource identifier, or another integer; its identity and ownership semantics are unknown.

The function does not visibly release or undo that value after a callback mismatch. This establishes only the absence of direct cleanup here, not a leak: either dependency or the callback may have ownership effects that are not provided.

A return of 1 does not mean outputs are unchanged. It may occur before direct writes, after both outputs are zeroed, or after publication of nonzero state and callback side effects.

## Dependencies

| Opaque function ID or role | Invocation and established use | Unknown contract |
|---|---|---|
| F-f68f8fcdbc7a | Called once on every normally entered path, after selector selection. Receives five arguments. Zero permits continuation; any nonzero result produces status 1. | Validation rules, state mutation, global effects, ownership effects, and meaning of the return value. |
| F-fab3799b3190 | Called once only after F-f68f8fcdbc7a returns zero and the quantity output is written. Receives no explicit arguments. Its return is stored in the value output. | Whether it allocates or obtains a resource, whether it reads previously published state, and whether it changes globals or reachable storage. |
| Indirect callback, opaque identity unavailable | Loaded through the callback object’s dispatch table at byte offset 16. Called only for a nonzero F-fab3799b3190 result. Its return is compared with the post-call quantity output. | Meaning of operation value 2, unit of quantity, mutation permissions, error conventions, ownership transfer, and concrete function identity. |

The two fixed global values are additional data dependencies. Their values, initialization requirements, mutability, and semantic names are undocumented.

## Acceptance examples

These examples are conditional specifications, not executed tests. Unless stated otherwise, assume valid disjoint storage, returning dependencies, and no mutations beyond those explicitly described.

1. **Selector 2.**  
   A descriptor whose second field narrows to 2 causes F-f68f8fcdbc7a to receive the first fixed global value as its fifth argument. The context value remains its fourth argument.

2. **Selector 3 or 4.**  
   Either narrowed selector causes the second fixed global value to be used as the fifth argument.

3. **Default selector.**  
   A narrowed selector of 5 causes the context value to appear as both the fourth and fifth arguments. Under the represented eight-bit narrowing, a second-field value of 258 instead selects the selector-2 branch.

4. **Preliminary rejection.**  
   If F-f68f8fcdbc7a returns −1, the overall result is 1. Neither subsequent dependency is called. Output changes made by F-f68f8fcdbc7a itself remain possible.

5. **Post-call descriptor read.**  
   If F-f68f8fcdbc7a changes the first descriptor field from 25 to 30 and returns zero, the quantity first published is 30.

6. **Upper boundary.**  
   First-field values of 10,239, 10,240, and 10,241 produce initial quantity writes of 10,239, 10,240, and 10,240 respectively.

7. **Negative quantity.**  
   A first-field value of −5 is published as −5. If F-fab3799b3190 returns 7, the callback is still invoked with quantity −5. If it returns −5 without changing the output, overall status is 0.

8. **Zero dependency result.**  
   With an initial quantity of 80, a zero result from F-fab3799b3190 produces quantity-output writes of 80 and then zero, with a zero value-output write between them. Overall status is 1; no callback occurs.

9. **Zero quantity with a usable result.**  
   A quantity of zero does not skip dispatch. If F-fab3799b3190 returns 7 and the callback returns zero while leaving the quantity at zero, overall status is 0.

10. **Post-callback comparison.**  
    If the callback receives quantity 80, changes the quantity output to 12, and returns 12, overall status is 0. If it instead returns 80 while leaving that output at 12, overall status is 1.

11. **Shared output location.**  
    If both outputs designate the same valid location, the initial quantity is 80, and F-fab3799b3190 returns 7, the shared location becomes 7 before dispatch. With no further mutation, the callback receives quantity 7, and a callback return of 7 produces status 0.

## Uncertainties

**documented_only:**

- The domain meaning and units of the descriptor fields.
- The exact target ABI and whether the character narrowing is precisely eight-bit truncation outside the represented environment.
- The two fixed global values and whether their reads have special memory-access semantics.
- Both named dependencies’ internal behavior and all callback implementations.
- Whether any caller contract prohibits negative quantities, aliasing, or mutation of quantity storage.
- The ownership and lifetime of the nonzero dependency result.
- Whether operation value 2 denotes reading, writing, or another action.
- Error reporting beyond the binary return status.
- Concurrency, reentrancy, atomicity, and synchronization requirements.

None of these unknowns is resolved by assuming that callees preserve globals, preserve output values, or acquire or release memory.

## Evidence

**Established static evidence:** The supplied body shows selector-based argument selection, a preliminary zero check, a signed upper-cap comparison, ordered output stores, a no-argument dependency call, indirect dispatch through a table entry at byte offset 16, and a post-callback equality check. It contains no explicit lower-bound check, rollback, cleanup, or differentiated error status.

**Inferred intent:** The bounded quantity and callback equality check resemble a transfer-completion operation. This interpretation is not needed to apply the behavioral contract.

**Verification limit:** No runtime tests have been performed. The acceptance examples describe consequences of the supplied static evidence under their stated conditions; they are not observed executions.
