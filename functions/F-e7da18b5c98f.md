<!-- Trusted provenance: F-e7da18b5c98f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7da18b5c98f

## Purpose

F-e7da18b5c98f performs a conditionally dispatched operation on a caller-supplied object and returns a 32-bit status. On one path, a zero result from the dispatched operation causes a 32-bit quantity to be added to a field in the original object.

The established behavior comprises preparation, two ordered checks, selection between a stored-result path and a byte-gated dispatch path, and a final dependency call before every visible normal return.

The function directly selects two statuses:

- **2** when the first check returns nonzero.
- **21** when the second check returns nonzero and the subsequently read gate byte is nonzero.

Other results come from the object’s stored-status field or the dispatched operation. The domain meanings of these statuses and fields are not established. Calling the quantity a byte count, record count, or completed-work count would be inferred intent.

## Interface

The function has four inputs and a 32-bit integer result.

| Input | Representation at this boundary | Established use |
|---|---|---|
| Object | 32-bit address-bearing value | Passed to dependencies and used as the base of fixed-offset memory accesses. |
| Operation argument | Opaque 32-bit value | Forwarded unchanged to the dispatched operation. |
| Quantity address | Address of a 32-bit integer | Forwarded to the dispatched operation; read directly by this function only after that operation returns zero. |
| Preparation value | Opaque 32-bit value | Used to initialize a local 32-bit slot whose address is passed to preparation. The slot’s current value is later passed to the final dependency. |

The supplied representation supports the following partial object layout. Offsets are in bytes.

| Offset | Width | Explicit access by this function |
|---|---:|---|
| 20 | 8 bits | Read as a zero/nonzero gate after a nonzero second-check result. |
| 24 | 32 bits | Read and passed to the dispatched operation. |
| 28 | 32 bits | Read as the selected return status after a zero second-check result. |
| 32 | 32 bits | Read and overwritten after a zero dispatched-operation result. |

These accesses do not establish the complete object type, allocation size, alignment requirements, or field meanings. Every reached access must be valid under the execution environment. This function performs no explicit validity check before those accesses.

The quantity address must support a readable 32-bit integer when the dispatched operation returns zero. The dispatched operation may independently require a writable or otherwise constrained location. Its requirements are **documented_only** because its implementation is not supplied.

No visible restriction requires the quantity address to be distinct from the object. Aliasing can therefore affect the accounting result, subject to the execution environment’s memory-access rules and the unknown dependency contracts.

## Behavior

### Preparation and exposure of local storage

The function initializes a local 32-bit slot from the preparation value, then calls **F-3794f888f18b** with:

1. The address of that slot.
2. The object value.

The preparation dependency’s return value is not used.

Passing the slot’s address makes the slot accessible to that dependency. The supplied evidence does not establish that preparation leaves it unchanged, keeps its address private, or refrains from retaining or exposing that address during the invocation.

Consequently, the value supplied to the final dependency is **the slot’s current value when the final call’s argument is evaluated**. It is not necessarily the original preparation value or even the value present immediately after preparation returns. An intervening dependency or other action with access to the exposed slot may change it.

This function contains no later explicit assignment to the slot. That absence does not imply preservation by dependencies.

### First check

After preparation returns, the function calls **F-867a1fcaaaab** with the object value.

If the result is nonzero, the function:

1. Evaluates the current slot value as the argument to **F-fd58bea73663**.
2. Calls that dependency.
3. Returns **2**, if the dependency returns normally.

The first check’s particular nonzero result is discarded. Positive and negative nonzero results select the same path.

This path does not reach the second check, the explicit object-field reads, dispatch acquisition, or the accounting update.

If the first check returns zero, execution proceeds to the second check.

### Second check and stored status

The function calls **F-56e2874b0016** with the object value.

If the result is zero, the function reads the 32-bit field at object offset 28 and selects that value as its return status. It then calls **F-fd58bea73663** with the local slot’s current value at argument evaluation and returns the selected status if that call returns normally.

The status read occurs after the second check returns. A dependency’s earlier modification of the field therefore affects the selected status.

On this path, this function itself does not:

- Read the gate byte at offset 20.
- Acquire a dispatch object.
- Read the argument field at offset 24.
- Read through the quantity address.
- Read or write the accumulator at offset 32.

Dependencies may nevertheless perform such accesses. These exclusions concern only explicit accesses in this function.

The selected status is retained across the final call rather than reread from the object afterward. A final dependency’s ordinary mutation of the offset-28 field does not change that selection.

### Byte gate after a nonzero second check

If the second check returns nonzero, the function reads the byte at object offset 20.

If that byte is nonzero, the function calls **F-fd58bea73663** with the local slot’s current value at argument evaluation, then returns **21** if the call returns normally.

Every nonzero byte value selects this path. The byte need not equal one; interpreting its high bit as a sign does not change the zero/nonzero decision.

If the byte is zero, execution proceeds to dispatch acquisition.

### Dispatch acquisition and invocation

The function calls **F-a6226d4de001** with the object value and the integer **1**.

Its result supplies an address used as the dispatch object. The function reads the first 32-bit word at that address, uses that word as a dispatch-table address, and selects a callable entry at byte offset **52** within that table.

The dispatched operation receives these logical arguments:

1. The acquired dispatch-object address.
2. The 32-bit value read from the original object at offset 24.
3. The original operation argument.
4. The original quantity address.

The offset-24 read occurs after acquisition returns. An acquisition-side modification of that field can therefore change the argument delivered to the operation.

The supplied expression does not establish the relative evaluation order of the dispatch-table accesses and the offset-24 argument read. This contract does not impose an order between those reads.

No explicit null, range, or callable-entry validation occurs. Normal execution requires the acquired object, dispatch table, and selected entry to support the indicated reads and call. The function defines no local recovery for an invalid dispatch address.

### Accounting after a zero operation result

The dispatched operation’s 32-bit result becomes the selected return status.

If that result is zero, the function reads:

- The current 32-bit integer at object offset 32.
- The current 32-bit integer at the quantity address.

It adds those values and stores the result in the 32-bit field at object offset 32.

Both operand reads occur after the operation returns and before the accumulator store. Their relative order is not established by the supplied expression. The update uses post-operation state, not values captured before dispatch.

The quantity is not checked for positivity. A negative quantity can reduce the accumulator when the mathematical result is representable. A zero quantity still follows the update path and produces an explicit accumulator store.

The supplied representation uses 32-bit integer arithmetic but does not establish a portable signed-overflow rule. A defined wraparound result must not be inferred merely from the store width. Acceptance examples below keep sums within the signed 32-bit range.

If the operation returns nonzero, this function does not read through the quantity address or explicitly read or write the accumulator on account of that result. Mutations already performed by the operation are not rolled back.

### Final call and return

After either dispatched-operation outcome, the function calls **F-fd58bea73663** with the current value read from the exposed local slot when its argument is evaluated. If the call returns normally, the function returns the selected operation status.

Every visible normal-return path includes exactly one call to F-fd58bea73663. Its return value is unused.

The argument supplied to that call is a value evaluated before entry to the dependency. A change to the slot during the final dependency does not retroactively change the argument already supplied.

The function does not establish that every invocation returns normally. Invalid accesses, dependency nonreturn, and failures during indirect invocation have no visible local recovery path.

## Side effects

The explicit writes performed by this function are:

- Initialization of its local 32-bit preparation slot.
- A 32-bit write at object offset 32 following a zero dispatched-operation result.

The function does not directly write through the quantity address. It passes that address to the dispatched operation, which may write through it. There is no established guarantee that the operation initializes the quantity before returning zero; this function consumes whatever value is present afterward.

Dependency effects are not bounded by the visible argument uses. In particular:

- Preparation may modify the exposed slot, the object, or other state.
- Preparation may make the slot address available to later actions.
- Either check may modify object state or an accessible preparation slot.
- Acquisition may modify state before the offset-24 field is read.
- The dispatched operation may modify the quantity, accumulator, other object fields, or accessible shared state.
- The final dependency may modify state before returning.

No dependency is assumed to preserve globals, preserve the exposed slot, or limit its effects to the object.

The function contains no explicit allocation, ownership-transfer operation, reference-count operation, or separate release of the acquired dispatch object. The preparation/final-call pairing may suggest resource management, but that is inferred intent, not an established ownership contract. The absence of a separate release does not establish a leak.

If the quantity address aliases the accumulator field, the successful update reads the shared post-operation value twice and adds it to itself, provided that value remains unchanged between the reads and the accesses are valid. Other aliasing arrangements depend on their actual memory layout and dependency effects.

## Dependencies

| Opaque ID | Established boundary behavior |
|---|---|
| **F-3794f888f18b** | Called first, with the address of initialized local 32-bit storage followed by the object value. Its return value is unused. Exposure of the slot prevents assuming that later actions preserve it. |
| **F-867a1fcaaaab** | Called after preparation with the object value. Zero permits the second check; any nonzero result selects status 2. |
| **F-56e2874b0016** | Called only after a zero first-check result. Zero selects the offset-28 stored status; nonzero selects the byte-gated path. |
| **F-a6226d4de001** | Called only after a nonzero second-check result and a zero gate byte. Receives the object value and integer 1; supplies the dispatch-object address. |
| **F-fd58bea73663** | Called once before each visible normal return. Receives the preparation slot’s current value when that argument is evaluated. Its return value is unused. |

The dynamically selected operation has no supplied opaque function ID. Its dispatch location, argument structure, result width, and influence on subsequent accounting are established; its internal behavior is not.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless otherwise stated, dependencies return normally, memory accesses are valid, and no unmentioned action changes the values relevant to the example.

1. **First-check rejection.** Preparation completes and F-867a1fcaaaab returns −1. The function calls F-fd58bea73663 with the slot value current at argument evaluation, then returns 2. The second check and dispatch acquisition are not called.

2. **Stored status after mutation.** The first check returns zero. The second check writes 37 to offset 28 and returns zero. The selected result is 37. If the final dependency later changes that object field, the function still returns the previously selected 37.

3. **Noncanonical gate value.** The checks return zero and nonzero respectively, and the subsequently read gate byte is 128. The function calls the final dependency and returns 21 without acquiring a dispatch object.

4. **Acquisition changes a dispatched argument.** Dispatch is reached. Acquisition changes the offset-24 field to a different 32-bit value before returning a valid dispatch object. The dispatched operation receives the changed field value.

5. **Successful accounting.** The operation returns zero, the accumulator is then 100, and the quantity is then 12. The function stores 112 at offset 32 before calling the final dependency and returns zero afterward. The final dependency may subsequently change the stored accumulator.

6. **Negative quantity.** With a zero operation result, a post-operation accumulator of 100 and quantity of −12 produce an accumulator store of 88. No local negative-quantity rejection occurs.

7. **Operation failure after mutation.** The operation changes the accumulator to 500 and returns 9. This function performs no accounting update, calls the final dependency, and returns 9. The accumulator’s ultimate value still depends on later dependency effects.

8. **Aliased quantity and accumulator.** The quantity address identifies the accumulator field. After a zero-returning operation, the shared field contains 7 and remains unchanged until the accounting store. The function stores 14.

9. **Preparation value preserved until final argument evaluation.** Preparation replaces the initialized slot with 45. If no intervening action changes that slot before the final argument is evaluated, F-fd58bea73663 receives 45. Preparation’s write alone does not guarantee this outcome.

10. **Exposed slot modified later.** Preparation leaves 45 in the slot but exposes its address to a later check. That check changes the slot to 62, and no subsequent action changes it before final argument evaluation. F-fd58bea73663 receives 62, regardless of which normally returning branch follows.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the statuses, object fields, quantity, and operation argument.
- Complete object and dispatch-table layouts, alignment requirements, and memory-validity rules.
- Dependency-side validation, mutations, ownership, retention, and failure behavior.
- Whether preparation retains or exposes the local slot’s address, and which later actions can access it.
- Whether the dispatched operation initializes the quantity on success.
- Whether preparation and the final dependency acquire or release any resource.
- Signed-overflow behavior and address-arithmetic behavior outside valid accesses.
- Behavior after invalid pointers, invalid dispatch entries, faults, or dependency nonreturn.
- Concurrency, synchronization, and memory-order guarantees.

No explicit local retry, rollback, quantity-range validation, or exception translation is visible. This absence does not establish corresponding behavior inside dependencies.

## Evidence

This contract rests on static examination of the supplied function body and dependency-ID mapping. Established observations include the branch conditions, call sequence, argument forwarding, fixed-offset accesses, dispatch structure, accounting condition, and selected return values.

The preparation-slot argument is evaluated at the final call site. Because its address was previously passed to preparation, the evidence does not justify assuming that its post-preparation value remains unchanged across intervening calls.

Resource-management interpretations and domain-specific field names remain inferred intent rather than established semantics. No dependency implementation or complete type definition was supplied.

No runtime tests have been performed. Acceptance examples describe conditional behavior supported by the static evidence, not observed execution.
