<!-- Trusted provenance: F-b9eeaece20cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9eeaece20cb

## Purpose

F-b9eeaece20cb acquires an object, conditionally performs a preparatory operation, dispatches one of nine operations, and passes a local object-reference slot to a final dependency before returning.

Two results are distinct:

- The function’s own 32-bit return value indicates whether the selector matched a supported operation: zero for selectors 1 through 9, one for every other selector, provided execution reaches the return.
- For a supported selector, the selected operation’s 32-bit result is written to caller-supplied output storage.

An unsupported selector still causes object acquisition and the final dependency call. It does not cause an operation dispatch or a direct write to the caller’s output storage.

These are properties of the supplied static representation. No runtime tests have been performed. The semantic purposes of the acquired object, preparation, operation results, and final dependency are not established.

## Interface

The interface has four arguments and a 32-bit return value.

| Argument | Representation | Established use |
|---|---|---|
| Context value | 32-bit opaque value | Forwarded unchanged to preparation and to each directly named operation dependency. Not passed to the indirect operation selected by selector 5. |
| Selector | 32-bit value | Compared against the exact values 1 through 9. All other bit patterns take the unsupported-selector path. |
| Output address | Address of a 32-bit storage location | Receives the selected operation’s result for supported selectors, after that operation returns and before the final dependency call. |
| Fourth argument | Pointer-shaped value in the supplied representation | Initially copied into a local reference slot, then overwritten by the acquisition result before any shown use of that slot’s contents. It does not select the object used for dispatch. |

The supplied representation uses 32-bit scalar and integer fields. Pointer-shaped values are shown, but their complete ABI representation and calling convention are not independently established. In particular, the table access at byte offset 264 should not be converted into a portable table-index requirement.

### Output storage

For selectors 1 through 9, normal completion requires the output address to identify writable storage for a 32-bit value when the result store occurs.

For unsupported selectors, the function itself does not dereference the output address. That does not establish a general guarantee that arbitrary output-address values are safe: dependencies may have side effects or access independently reachable memory.

The output value is an opaque 32-bit result. The wrapper does not interpret it as success, failure, a count, a pointer, or any other semantic type.

### Acquired object

Acquisition uses F-be45cd43f92d with the scalar argument zero.

A non-null result becomes the object supplied to preparation, the selected operation, and the final local-slot call. A null result transfers control to F-a817d8d58c10, which is represented as non-returning. No normal return value or output result is established for that path.

### Fourth argument

No dereference, validation, or direct release of the fourth argument is shown. Its initial placement in the local slot is superseded by the acquisition result.

Consequently, a null or otherwise unusable fourth argument does not itself trigger a shown check. This is a statement about this function’s direct use, not a guarantee about arbitrary external aliasing or dependency behavior.

## Behavior

### 1. Acquire before examining selector behavior

The function establishes a default return status of one and calls F-be45cd43f92d with zero.

The returned reference replaces the contents of a local object-reference slot. The function checks that returned reference for null.

If it is null:

- F-a817d8d58c10 is called.
- The supplied representation marks this call as non-returning.
- No preparation, selected operation, output store, or final local-slot call is shown on this path.

This ordering applies even to unsupported selectors. A selector that will eventually be rejected does not bypass acquisition or the null-result check.

### 2. Classify the selector

The supported selector set is exactly 1 through 9.

Selector zero and all other 32-bit values are unsupported. There is no range normalization, truncation to a smaller selector width, masking, or default operation shown.

For an unsupported selector, execution skips the preparation check and operation dispatch, then proceeds to the final dependency call with the local object-reference slot.

### 3. Read the preparation flag for supported selectors

Each supported-selector path reads a one-byte field at byte offset 4 from a base obtained through a global reference.

The byte is tested for zero:

- Zero: preparation is skipped.
- Any nonzero byte: F-c5890456f563 is called with the context value and acquired object.

The test is based on the byte’s zero/nonzero value, not on one specific nonzero encoding. Signed interpretation of that byte does not alter the branch condition.

The read occurs after acquisition has returned and after the acquired reference has passed the null check. Thus, any relevant state changes made by acquisition precede the flag read.

There is one shown preparation decision per supported invocation. The flag is not rechecked after preparation or after the selected operation.

Validity of the global base and readability of its byte at offset 4 are preconditions for ordinary execution along a supported-selector path. No null check or other validation of that base is shown.

### 4. Dispatch exactly one selected operation

After any required preparation returns, the selector determines the operation:

| Selector | Operation | Arguments |
|---|---|---|
| 1 | F-340934523342 | Context value, acquired object |
| 2 | F-14a1dde009b8 | Context value, acquired object |
| 3 | F-884980348650 | Context value, acquired object |
| 4 | F-6696b8f6fa58 | Context value, acquired object |
| 5 | Indirect operation from the object’s table at byte offset 264 | Acquired object only |
| 6 | F-9ed3442834fe | Context value, acquired object |
| 7 | F-56997e3074cf | Context value, acquired object |
| 8 | F-8f320b5f10c1 | Context value, acquired object |
| 9 | F-4f3138b13520 | Context value, acquired object |

For the directly named operations, the context value and acquired object reference are forwarded without shown transformation.

Selector 5 has a distinct memory-access requirement:

1. Read the table reference from the beginning of the acquired object.
2. Obtain the callable entry at byte offset 264 from that table.
3. Invoke the entry with the acquired object as its sole shown argument.

Those reads occur after preparation, if preparation is enabled. An implementation that reads and caches the table or callable entry before preparation would not preserve the shown ordering when preparation changes the object’s table-related state.

The indirect callable has no supplied opaque identity. Its parameter semantics, additional ABI requirements, and side effects are unknown.

### 5. Store the selected result

When the selected operation returns, its 32-bit result is retained for the output store.

The function changes its own return status to zero and writes the operation result into the caller-provided 32-bit output location.

There is no conditional interpretation of that result. In particular:

- A zero operation result is written as zero and does not cause the wrapper to report an unsupported selector.
- A nonzero operation result is written unchanged and does not cause the wrapper to change its return status to one.
- All 32 bits of the represented result are retained.

No output-address validation is shown. An invalid destination may therefore prevent execution from reaching the final dependency call.

### 6. Pass the local reference slot to the final dependency

F-54033250428c receives the address of the local object-reference slot.

This call occurs:

- After the selected-result store for supported selectors.
- Without a selected-result store for unsupported selectors.
- After acquisition and its successful null check in both cases.

The slot contains the acquired reference unless some intervening behavior has modified it through an independently available alias. Passing its address allows F-54033250428c to inspect or modify the slot; its actual behavior is not supplied.

After that dependency returns, the function returns its retained status: zero for a supported selector, one for an unsupported selector.

## Side effects

### Directly established effects

The function directly causes or performs:

- One acquisition call with argument zero.
- A null-result check on the acquired reference.
- For a supported selector, a read of the preparation-control byte.
- At most one preparation call.
- Exactly one selected operation call on a supported path that reaches dispatch.
- Exactly one 32-bit output store on a supported path whose selected operation returns and whose output store completes.
- One final call with the address of the local object-reference slot on paths that reach that stage.

There is no direct output store on an unsupported-selector path.

### Dependency effects and ordering

Dependencies may affect memory, global state, object state, allocation state, or control flow. Their implementations are not supplied, so preservation of those states cannot be assumed.

The important ordering constraints are:

- Acquisition precedes the preparation-control read.
- Preparation precedes the operation, including the indirect table reads for selector 5.
- The selected operation precedes the output store.
- The output store precedes the final dependency call.

The output store is therefore not necessarily the final write to that location during the invocation. A dependency with an independent alias could modify the same storage afterward.

Likewise, “no direct output store” on an unsupported selector is narrower than “the output location is unchanged.” The latter requires additional assumptions about dependency side effects and aliases.

### Ownership

The acquisition/final-call pairing suggests a managed-reference or cleanup protocol, but that purpose is inferred rather than established.

The available evidence does not establish that:

- Acquisition allocates or transfers ownership.
- Preparation preserves object lifetime.
- An operation preserves object lifetime.
- F-54033250428c releases, destroys, decrements, or clears anything.
- The caller owns or must release the returned operation value.

No explicit ownership transfer through the output store is shown. Its value remains opaque.

### Arithmetic and layout

The shown dispatch contains no size calculation, allocation-size arithmetic, or increment/decrement operation.

The layout-sensitive accesses are a byte at offset 4 in global-referenced state and an indirect callable at byte offset 264 in an object-associated table. The latter is a byte displacement, not an established entry count.

## Dependencies

| Opaque identity | Role in this function | Established result use |
|---|---|---|
| F-be45cd43f92d | Acquires a reference using argument zero | Null-checked, retained, and supplied to later calls |
| F-a817d8d58c10 | Handles null acquisition | Represented as non-returning |
| F-c5890456f563 | Optional preparation for supported selectors | No returned value is consumed |
| F-340934523342 | Selector 1 operation | 32-bit result stored to output |
| F-14a1dde009b8 | Selector 2 operation | 32-bit result stored to output |
| F-884980348650 | Selector 3 operation | 32-bit result stored to output |
| F-6696b8f6fa58 | Selector 4 operation | 32-bit result stored to output |
| F-9ed3442834fe | Selector 6 operation | 32-bit result stored to output |
| F-56997e3074cf | Selector 7 operation | 32-bit result stored to output |
| F-8f320b5f10c1 | Selector 8 operation | 32-bit result stored to output |
| F-4f3138b13520 | Selector 9 operation | 32-bit result stored to output |
| F-54033250428c | Receives the address of the local reference slot before normal return | No returned value is consumed |

The selector 5 callable is identified only by its table location. No supplied dependency identity can be assigned to it.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless a criterion states otherwise, assume acquisition returns a usable non-null object, required memory accesses are valid, and invoked dependencies return normally.

1. **Supported direct operation, preparation disabled.**  
   With selector 2 and a zero preparation byte, F-14a1dde009b8 receives the original context value and acquired object. F-c5890456f563 is not called. The operation’s 32-bit result is stored to the output location before F-54033250428c is called. The wrapper returns zero.

2. **Noncanonical true preparation byte.**  
   With selector 8 and preparation byte 128, preparation occurs before F-8f320b5f10c1. The byte need not equal one to enable preparation.

3. **Preparation changes indirect dispatch state.**  
   With selector 5 and preparation enabled, suppose preparation replaces the object’s table reference with another valid table. The indirect call is resolved from the object after preparation, using byte offset 264 in the then-current table. The context value is not a shown argument to this call.

4. **Operation result does not control wrapper status.**  
   With selector 4, an operation result whose 32 bits are all set is stored unchanged. The wrapper still returns zero after the final dependency returns.

5. **Unsupported zero selector.**  
   With selector zero, acquisition still occurs. The preparation byte is not read by this function, no selected operation is called, and no direct output store occurs. F-54033250428c receives the local reference-slot address, and the wrapper returns one.

6. **Other unsupported bit patterns.**  
   Selector 10 and the all-ones 32-bit selector take the same unsupported-selector path as zero. Neither is coerced into a supported selector.

7. **Null acquisition overrides selector handling.**  
   If acquisition returns null, F-a817d8d58c10 is called even when the selector is unsupported. No normal return or final dependency call is established.

8. **Fourth argument does not supply the dispatch object.**  
   If the fourth argument differs from the non-null acquisition result, the acquired reference—not the fourth argument—is used for preparation and dispatch.

9. **Post-store dependency effects remain observable.**  
   If F-54033250428c independently modifies storage aliased by the output address, its modification occurs after the wrapper’s result store. Acceptance must not require the stored operation result to remain there on return without a dependency-side noninterference assumption.

## Uncertainties

The following are **documented_only**:

- The domain meaning of the context, selector values, object, preparation flag, and operation results.
- Full pointer widths, calling convention, alignment requirements, and byte order beyond the represented accesses.
- The global state layout beyond its byte at offset 4.
- The object and table layouts beyond the reads required for selector 5.
- Whether any dependency allocates, releases, retains, destroys, or otherwise changes ownership.
- Dependency behavior on failure, exceptions, nonlocal exits, or invalid input.
- What happens if the dependency represented as non-returning unexpectedly returns.
- Concurrency guarantees, synchronization, reentrancy, and whether external mutation can race with the shown accesses.
- Whether caller-provided storage aliases the object, global state, or memory independently accessible to dependencies.

No additional error code, retry, rollback, exception cleanup, or output initialization is established.

## Evidence

**Established static evidence:** The supplied representation shows acquisition with zero, a null-result branch to a non-returning dependency, nine supported dispatch cases, a conditional byte-controlled preparation call, the selector 5 indirect call, a 32-bit result store, and a common final local-slot call. It also shows that the fourth argument’s initial slot value is overwritten before dispatch.

**Inferred intent:** The repeated preparation step appears to establish per-operation state. The final slot-address call appears consistent with reference cleanup. Neither interpretation is required to describe the established call sequence.

**Unknowns:** Dependency internals, ownership semantics, object validity rules, and runtime behavior are not supplied. This contract does not treat descriptive source labels or comments as independent proof.

**Verification status:** Static analysis of the supplied material only. No runtime tests have been performed.
