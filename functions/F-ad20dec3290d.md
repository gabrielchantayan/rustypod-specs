<!-- Trusted provenance: F-ad20dec3290d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad20dec3290d

## Purpose

F-ad20dec3290d selects a 32-bit value according to a mode, two object-associated conditions, and, on some paths, a dynamically dispatched query. It then clears one byte in the supplied object, conditionally writes a 16-bit state value through another dependency, and dispatches a final object operation with the selected value.

The selected value initially equals zero. Some paths leave it unchanged; others obtain it from two successive dependencies. A zero result from the dynamic query also forces the conditional state update, regardless of the caller’s original request.

The domain meaning of the modes, object fields, four global values, selected value, and state value is not established. In particular, this contract does not identify the selected value as an allocation, resource handle, status code, or owned object.

## Interface

### Inputs

The function accepts two inputs:

- **Object address:** an address interpreted as an object with a dispatch-table pointer at offset 0 and accessible storage at offsets 40 and 41.
- **Update request:** a 32-bit integer. Zero means that the conditional state write is not initially requested; every nonzero value requests it. One branch overrides this input with the value 1.

The supplied representation uses 32-bit integer and address operations. The layouts below describe that representation; portability to a different pointer width is not established.

### Object layout used by this function

| Offset | Access | Meaning established here |
|---|---|---|
| 0 | Read as a dispatch-table pointer | Locates the final indirect operation. |
| 40–43 | Read as a 32-bit integer, then narrowed to a character | Supplies **condition A**, the zero/nonzero result of the narrowed value. |
| 41 | Read as a character | Supplies **condition B**, tested for zero/nonzero. |
| 40 | Written as one byte | Cleared before the optional state write and final dispatch. |

Condition A is obtained by narrowing the 32-bit word beginning at offset 40. It is not expressed in the supplied evidence as a direct byte read at offset 40. On a conventional little-endian target with eight-bit characters and low-byte narrowing, these identify the same byte. That representation equivalence is conditional, not a separately proven fact.

For zero/nonzero decisions, signed versus unsigned character interpretation does not change the result once the narrowed bit pattern is established.

### Other layouts

A dependency-provided query object is interpreted as having a dispatch-table pointer at offset 0. The function entry at byte offset 132 within that table is invoked.

The final operation uses the entry at byte offset 200 within the supplied object’s dispatch table.

A separate dependency returns an address at which the function writes the 16-bit value 3 at offset 128.

### Output

There is no explicit return value.

The final dispatched operation receives:

1. The original object address.
2. The selected 32-bit value.

Its return value, if any, is not used by the visible function.

### Preconditions

For an execution to reach normal completion:

- Every object read and write actually taken must reference valid, suitably accessible storage.
- Dispatch-table pointers and the selected entries must be valid for their respective indirect operations.
- The query-object dependency must return a usable address whenever the query path is taken.
- The state-address dependency must return an address permitting a two-byte write at offset 128 whenever the state update is taken.
- Dependencies must return control normally for subsequent described actions to occur.

There are no visible checks for null addresses, invalid tables, unavailable storage, or dependency failures.

## Behavior

### 1. Initial dependency sequence

The selected value is initialized to the 32-bit value zero.

The function then invokes:

1. F-7e4f952e879e, with no explicit arguments.
2. F-301ccddb48b5, with no explicit arguments.

The first dependency’s result is not used. The second supplies the 32-bit mode used for branch selection.

The object conditions are inspected after these calls. Consequently, changes those dependencies make to accessible object state can affect the subsequent selection.

### 2. Selection paths

Four global 32-bit values supply argument pairs on transformation paths. They are called **G-A**, **G-B**, **G-C**, and **G-D** here solely to distinguish their uses.

A transformation using one of these globals consists of:

- Calling F-ef0964bd9b8a with no explicit arguments.
- Passing its returned 32-bit value to F-1958950ee503 as the first argument.
- Supplying the selected global minus one as the second argument, the selected global as the third argument, and zero as the fourth argument.
- Replacing the selected value with the 32-bit result of F-1958950ee503.

The globals’ domain, units, and valid ranges are unknown. The argument pair must not be assumed to describe a range merely because its two expressions differ by one.

#### Mode equal to 2

Condition A is tested first.

- If condition A is zero, the function selects the transformation using G-A. It does not query condition B on this path.
- If condition A is nonzero, condition B is tested.
  - Nonzero condition B selects the dynamic query described below.
  - Zero condition B reaches the common eligibility gate.

#### Mode equal to 1

The function proceeds directly to the dynamic query.

It does not inspect either object condition before that query.

#### Every other mode

Condition A is tested.

- If condition A is zero, execution reaches the common eligibility gate.
- If condition A is nonzero, condition B is tested.
  - Nonzero condition B selects the dynamic query.
  - Zero condition B reaches the common eligibility gate.

No normalization limits the mode to 0, 1, or 2. Negative values and other positive values follow this “every other mode” branch.

### 3. Common eligibility gate

The gate uses the original mode result, not a fresh dependency call.

- If the mode is zero, the function selects the transformation using G-D without further condition reads at this gate.
- If the mode is nonzero, condition A is read again.
  - If this read yields zero, transformation is skipped.
  - If it yields nonzero, condition B is read again:
    - Nonzero condition B skips transformation.
    - Zero condition B selects the transformation using G-D.

Skipping transformation leaves the selected value at zero.

These repeated reads are part of the supplied control structure. Replacing them with cached condition values would require an additional assumption about storage stability and the execution model.

### 4. Dynamic query

On this path, the function:

1. Calls F-cdd33263ee89 with no explicit arguments.
2. Reads the returned object’s dispatch-table pointer.
3. Invokes the entry at byte offset 132.
4. Tests that operation’s 32-bit result for zero.

The visible indirect invocation supplies no explicit arguments. This contract does not infer an object receiver or additional arguments from an unspecified calling convention.

The result selects one of two actions:

- **Zero:** perform the transformation using G-B, then set the effective update request to 1.
- **Nonzero:** perform the transformation using G-C, without changing the effective update request.

The override occurs after the G-B transformation returns. If that transformation does not return, the override and all later actions are not reached.

### 5. Stable-state selection table

If condition reads are stable throughout selection, the preceding control flow reduces to:

| Mode | Condition A | Condition B | Selection |
|---|---|---|---|
| 1 | Not read | Not read | Dynamic query |
| 2 | Zero | Not read | G-A transformation |
| 2 | Nonzero | Zero | G-D transformation |
| 2 | Nonzero | Nonzero | Dynamic query |
| 0 | Zero | Not read | G-D transformation |
| 0 | Nonzero | Zero | G-D transformation |
| 0 | Nonzero | Nonzero | Dynamic query |
| Any value other than 0, 1, or 2 | Zero | Not read | No transformation; selected value remains zero |
| Any value other than 0, 1, or 2 | Nonzero | Zero | G-D transformation |
| Any value other than 0, 1, or 2 | Nonzero | Nonzero | Dynamic query |

This table is conditional on stable reads. The explicit gate behavior governs otherwise.

### 6. Global reads and arithmetic

F-ef0964bd9b8a is called before the selected global is used. Its effects therefore precede the relevant global reads; the function does not visibly snapshot these values before that call.

For the G-A, G-C, and G-D transformations, the supplied representation separately computes the global-minus-one argument and then reads the global for the third argument. It does not establish an atomic snapshot.

For G-B, both global expressions occur as arguments to one dependency invocation. Their relative evaluation order is not established by the supplied representation.

The decrement is expressed using 32-bit integer arithmetic. At the minimum signed 32-bit value, subtracting one exceeds the signed range. The exact executable behavior at that boundary is **documented_only**: the source representation alone does not establish whether the target wraps, traps, or has another applicable arithmetic rule. No saturation or bounds check is visible.

### 7. Common completion sequence

Every path that reaches completion performs these actions in order:

1. Write one zero byte at offset 40 of the original object.
2. Test the effective update request.
3. If that request is nonzero:
   - Call F-fdd8589eca2d with no explicit arguments.
   - Write the 16-bit value 3 at offset 128 of its returned address.
4. Read the original object’s dispatch-table pointer and invoke its entry at byte offset 200, supplying the original object address and selected value.

The byte clear is unconditional on all normally reaching paths. It does not clear the entire 32-bit word beginning at offset 40, and it does not explicitly clear the byte at offset 41.

The final dispatch-table lookup occurs after the optional dependency call and state write. Thus, any changes that dependency makes to the original object’s dispatch pointer can affect the final dispatch. No preservation of that pointer is assumed.

## Side effects

The function directly performs at most two memory writes before final dispatch:

- One byte of zero at object offset 40.
- Conditionally, two bytes encoding the value 3 at a dependency-provided address plus 128.

The second write occurs for every initially nonzero update request and also after a zero dynamic-query result, even when the initial request was zero.

The two write destinations are not proven disjoint. If they overlap, their stated order determines the direct write sequence. Later dependency or dispatched-operation effects may overwrite either result.

All invoked dependencies and indirect operations may have additional effects. No purity, global preservation, allocation, deallocation, reference counting, or ownership transfer is established.

Passing the selected value to the final operation does not itself prove that ownership is transferred. Likewise, obtaining that value does not prove that this function owns it or must release it.

No rollback is visible. A failure after the byte clear may leave that clear observable without final dispatch. A failure after the optional state write may leave both direct writes observable.

## Dependencies

| Opaque ID or operation | Role and visible contract |
|---|---|
| F-7e4f952e879e | First call; no explicit arguments; result unused. |
| F-301ccddb48b5 | Second call; returns the 32-bit mode. |
| F-ef0964bd9b8a | Called once on any transformation path; supplies the first transformation argument. |
| F-1958950ee503 | Receives four arguments: preceding result, selected global minus one, selected global, zero. Its 32-bit result becomes the selected value. |
| F-cdd33263ee89 | Supplies the object used for the dynamic query. Called only on query paths. |
| F-fdd8589eca2d | Supplies the address for the conditional 16-bit state write. |
| Query dispatch operation | Anonymous dependency at table offset 132; its zero/nonzero result selects G-B or G-C. |
| Final dispatch operation | Anonymous dependency at table offset 200; receives the original object and selected value. |

The anonymous operations have no supplied opaque function IDs. Their identities and internal behavior are **documented_only** rather than inferred.

## Acceptance examples

These are conditional static acceptance examples, not results from executed tests. Unless stated otherwise, condition reads remain stable, arithmetic is representable, addresses are valid, and dependencies return normally.

1. **Mode 2, condition A zero, update request zero.**  
   Selection uses G-A without reading condition B. After the transformation, offset 40 is cleared. F-fdd8589eca2d is not called. Final dispatch receives the transformation result.

2. **Mode 1, query result zero, update request zero.**  
   The object conditions do not govern query entry. Selection uses G-B. The effective update request becomes 1, so the byte clear is followed by the state-address call and the 16-bit write of 3, then final dispatch.

3. **Mode 1, query result nonzero, update request zero.**  
   Selection uses G-C. The byte clear occurs, but no state-address call or direct 16-bit state write occurs.

4. **Mode 0, condition A zero.**  
   Selection uses G-D. Condition B is not needed for selection. This differs from an unrecognized nonzero mode with condition A zero.

5. **Mode 7, condition A zero, update request negative.**  
   No transformation occurs; final dispatch receives zero. The negative update request is nonzero, so the conditional state write still occurs.

6. **Mode 2, condition A nonzero, condition B zero.**  
   The common gate rereads condition A and, if it remains nonzero, condition B. With stable values, selection uses G-D rather than querying.

7. **Any dynamic-query path with a negative query result.**  
   The result is nonzero, so selection uses G-C. The query does not force the update request to 1.

8. **F-ef0964bd9b8a changes the selected global.**  
   Transformation arguments use the global reads after that call, not a value presumed to exist at function entry.

9. **The state-address dependency changes the original object’s dispatch pointer.**  
   Assuming the resulting pointer remains valid, final dispatch uses the subsequently read pointer. A cached pre-dependency dispatch target is not equivalent to the visible ordering.

## Uncertainties

- **documented_only:** semantic names for the modes, conditions, globals, selected value, and state value 3.
- **documented_only:** exact target endianness, character-narrowing rules, and signed-overflow behavior.
- **documented_only:** concurrency, memory-mapped or volatile storage, synchronization, and whether separate reads can observe intervening changes.
- **documented_only:** dependency side effects, ownership rules, failure behavior, and implicit calling-convention arguments.
- **documented_only:** whether the final indirect transfer is an ordinary call or a tail-style transfer. The supplied evidence marks that transfer as incompletely recovered. Its target lookup and displayed arguments are visible, but a stronger stack or return-flow guarantee is not established.
- **Inferred intent only:** the function appears to choose a value and notify or update an object. This interpretation adds no domain-specific obligations.

## Evidence

This contract derives solely from the supplied static function representation and dependency-ID mapping.

Established evidence includes the branch comparisons, field offsets and access widths, dependency ordering, transformation arguments, unconditional byte clear, conditional 16-bit write, and final indirect-dispatch arguments.

The stable-state table is derived from that control flow under an explicit stability assumption. Domain interpretations and unresolved execution details are separated from established behavior above.

No runtime tests, executable inspection, or dependency implementation inspection have been performed.
