<!-- Trusted provenance: F-bd674b4bdd60; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd674b4bdd60

## Purpose

F-bd674b4bdd60 creates or prepares an object from three caller-supplied values, then selects an action according to a 32-bit field in a caller-supplied context.

On an ordinary return, the function reports one of three 32-bit values:

- **0**: the context field was nonzero, a processing dependency returned zero, and the function decremented the field.
- **12**: the context field was nonzero and that dependency returned nonzero.
- **33**: the context field was zero and the function invoked an indirect operation on the prepared object.

These are established control-flow outcomes, not established semantic error names. In particular, the evidence does not establish that 12 means allocation failure, that 33 means a particular operating-system error, or that zero means all requested work completed beyond the dependency’s reported result.

The function also invokes two dependencies around its context-dependent work. Their placement suggests entry and exit synchronization, but their actual behavior is unknown.

## Interface

### Inputs

The visible interface has four arguments:

| Position | Contract role | Visible representation |
|---|---|---|
| First | Address-like value identifying a mutable context | A 32-bit integer in the supplied representation |
| Second | Opaque value forwarded to object preparation | 32 bits |
| Third | Opaque value forwarded to object preparation | 32 bits |
| Fourth | Opaque value forwarded to object preparation | 32 bits |

The second, third, and fourth arguments receive no visible interpretation or validation in this function. Their order at the preparation boundary differs from their order at this interface: preparation receives the second argument, then the fourth, then the third, after a value obtained from another dependency.

The first argument is used both unchanged and with byte offsets. The directly relevant context locations are:

| Offset from context base | Use |
|---|---|
| `0x130` / 304 bytes | Address passed to each of the two bracketing dependencies |
| `0x23c` / 572 bytes | Address of a 32-bit field tested and, on one path, decremented |

No complete context layout is established. In particular, the region at offset 304 must not be described as a lock solely from the surrounding call pattern.

### Output

The ordinary output is a 32-bit value: 0, 12, or 33.

A null result from the preparation dependency causes a call to a dependency annotated as non-returning. No ordinary result is established for that path.

There are no explicit output-pointer arguments. Observable outputs can nevertheless include context mutation, dependency effects, and an indirect call through the prepared object.

### Preconditions for ordinary completion

The caller and dependencies must collectively supply addresses and state suitable for all accesses reached on the selected path:

- The context-derived addresses must be valid for their respective uses.
- The 32-bit field at offset 572 must be readable. On the decrement path it must also be writable.
- The prepared object must satisfy the requirements of the selected processing path.
- On the zero-field path, the object must support a table-pointer read from its beginning and an indirect callable entry at byte offset 4 within that table.
- Every dependency needed for ordinary completion must return normally, except for the explicitly non-returning failure dependency.

These requirements are not enforced by visible checks other than the prepared-object null check. The function does not visibly check the context for null, verify address ranges or alignment, or validate an indirect target.

The supplied representation is 32-bit-oriented. The actual calling convention, pointer ABI, and behavior of invalid address arithmetic remain documented_only.

## Behavior

### Preparation precedes context inspection

The function first calls **F-8157a180d901** with the value 12. The result is a single 32-bit value.

It then calls **F-36e8a1855d6b** with four arguments, in this order:

1. The result of F-8157a180d901.
2. This function’s second argument.
3. This function’s fourth argument.
4. This function’s third argument.

The result of F-36e8a1855d6b is treated as an object pointer.

There is no separate visible check of the result from F-8157a180d901. Even if that result is zero, it is forwarded to F-36e8a1855d6b. The subsequent null check concerns the preparation result, not the earlier result.

If preparation returns null, the function invokes **F-a817d8d58c10** without explicit arguments. The supplied static evidence marks this dependency as non-returning. The ordinary path therefore ends there: neither bracketing dependency nor the context-field access is reached.

The value 12 supplied to F-8157a180d901 may denote a storage size, but that interpretation is not established. The evidence does not independently establish that this dependency allocates memory or that the preparation dependency constructs an object in that memory.

### Entry operation and field selection

After obtaining a nonnull prepared object, the function selects 33 as its default result and invokes **F-unmapped-entry** with the address at context offset 304.

Only after this dependency returns does the function read the 32-bit context field at offset 572. Consequently, branch selection uses the field’s value after the entry dependency’s effects. It does not use a snapshot taken before that call.

The field is compared with zero. This is an equality test, not a positive-capacity test. Every nonzero bit pattern takes the processing path, including values that would be negative under a signed 32-bit interpretation.

### Zero-field path

If the field is zero, the function invokes **F-indirect-slot-4** through the prepared object:

- A table pointer is obtained from the beginning of the object.
- A callable entry is obtained at byte offset 4 within that table.
- The indirect target is invoked with the object pointer as its sole explicit argument.

No additional explicit argument, including a release flag, is established.

The indirect call’s return value does not determine this function’s result. If the call returns normally, the selected result remains 33.

This function performs no direct write to the field at offset 572 on this path. That does not imply that the field remains unchanged: the indirect target or another dependency may modify it. There is no second field test and no branch reconsideration after the indirect call.

The indirect operation resembles an object disposal operation, but disposal, destruction, deallocation, and ownership release are not established semantics.

### Nonzero-field path

If the field is nonzero, the function calls **F-2c68918ef928** with four arguments:

1. The original context value.
2. The prepared object pointer.
3. The value 1.
4. The value 0.

The returned 32-bit integer is tested only for equality with zero.

If the dependency returns zero:

- The selected result becomes 0.
- The function reads the context field at offset 572 again.
- It subtracts one from that newly read value and writes the result back to the same location.

The decrement is therefore based on the field’s value **after** F-2c68918ef928 returns. The initial branch-selection value is not reused. A dependency mutation of the field can change the decrement’s operand.

If the dependency returns any nonzero value:

- The selected result becomes 12.
- This function performs no direct decrement or other direct write to the context field.

The nonzero dependency result is collapsed to 12; it is not propagated.

There is no visible indirect object operation on either nonzero-field outcome. Whether F-2c68918ef928 retains, consumes, releases, or leaves ownership of the object unchanged is unknown.

### Exit operation and return

After either context-dependent path completes normally, the function invokes **F-unmapped-exit** with the address at context offset 304.

It then returns the selected 32-bit result.

There is no visible branch based on either bracketing dependency’s return value. The exit dependency is reached after the indirect call, after the successful processing decrement, or after a nonzero processing result. It is not reached on the preparation-null path.

No recovery path is established if a dependency fails to return normally, an indirect call is invalid, or a memory access faults. In particular, unconditional exception-safe or fault-safe execution of the exit dependency is not established.

### Arithmetic boundaries

The decrement operates on a 32-bit field. No visible saturation, lower-bound check, or post-call zero check protects it.

For values whose subtraction by one is representable as a signed 32-bit integer, the arithmetic effect is unambiguous. For example, if the post-processing field is zero, the function writes negative one, whose 32-bit representation is `0xffffffff`.

The supplied high-level representation does not resolve the signed-overflow semantics at the minimum signed value. A machine-level wrapping result must not be elevated to an established contract without additional evidence.

Context-offset calculations likewise have no visible overflow checks. Ordinary execution requires that they identify the intended accessible locations.

## Side effects

The function directly exposes the following ordered effects:

1. A call requesting or supplying the value 12 to F-8157a180d901.
2. A preparation call using that result and the reordered caller arguments.
3. On preparation failure, the non-returning dependency call.
4. Otherwise, the entry dependency call.
5. A read of the context field at offset 572.
6. Either an indirect object call or a processing dependency call.
7. On a zero processing result, a fresh field read followed by a decrementing write.
8. The exit dependency call.

The only explicit memory write established in this function is the decrementing write on the zero processing-result path. All dependency calls may have additional effects not described by this body.

No ownership guarantee follows merely from a call occurring. In particular:

- The lifetime of the first dependency’s result is unknown.
- Preparation’s ownership and failure-cleanup rules are unknown.
- The zero-field indirect operation is not proven to release storage.
- The processing dependency is not proven to take ownership.
- There is no visible local cleanup after a nonzero processing result.

The entry/exit pattern is insufficient to establish mutual exclusion, atomicity, or freedom from data races. Those properties depend on the unmapped dependencies and the surrounding execution environment.

## Dependencies

The mapped opaque IDs are supplied identifiers. The three descriptive opaque IDs below are local labels for dependencies without supplied IDs; they do not establish cross-document identity.

| Opaque ID | Established use | Semantics not established |
|---|---|---|
| **F-8157a180d901** | Called with 12; its 32-bit result is forwarded | Allocation, initialization, failure convention, ownership |
| **F-36e8a1855d6b** | Called with the preceding result and arguments two, four, three; returns a pointer-like value | Construction, storage ownership, cleanup |
| **F-a817d8d58c10** | Called when preparation returns null; annotated non-returning | Termination, exception, diagnostic, or other failure mechanism |
| **F-2c68918ef928** | Called on a nonzero field with context, object, 1, 0; result tested against zero | Processing meaning, field mutation, retention, release |
| **F-unmapped-entry** | Called with context plus 304 before field inspection | Synchronization, validation, memory effects |
| **F-unmapped-exit** | Called with context plus 304 before ordinary return | Synchronization, cleanup, memory effects |
| **F-indirect-slot-4** | Selected through the object’s table at byte offset 4; called with the object | Concrete target, destruction, release, return semantics |

No dependency is assumed to preserve global state or context fields.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Each assumes valid accesses and normal return from dependencies unless stated otherwise.

1. **Preparation returns null.**  
   F-a817d8d58c10 is invoked. No ordinary return value, entry call, field read, or exit call is established.

2. **The first dependency returns zero, but preparation returns a valid object.**  
   Zero is forwarded into preparation. The function does not independently reject it and proceeds with the nonnull prepared object.

3. **The field is zero after the entry call.**  
   The indirect operation is invoked with the object. F-2c68918ef928 is not called. After the exit call, the result is 33, regardless of the indirect operation’s return value.

4. **The entry dependency changes the field from zero to two.**  
   The processing path is selected because the field read occurs after entry. The zero-field indirect operation is not selected.

5. **Processing returns zero and leaves the field at two.**  
   The function writes one to the field, calls the exit dependency, and returns zero.

6. **Processing returns zero but changes the field from two to seven.**  
   The function writes six, not one. The decrement uses the post-call field.

7. **Processing returns zero but leaves the field at zero.**  
   The function writes negative one in the 32-bit field. It does not reject the post-call zero or switch to the zero-field path.

8. **Processing returns a nonzero value and changes the field.**  
   The function performs no direct decrement, calls the exit dependency, and returns 12. It does not undo the dependency’s mutation.

9. **The field contains negative one when inspected.**  
   The processing path is selected because the field is nonzero. If processing returns zero and leaves that value unchanged, the function writes negative two.

## Uncertainties

The following remain **documented_only**:

- The actual ABI, pointer width, alignment rules, and complete context layout.
- The semantic meaning and permissible domain of the field at offset 572.
- Whether the bracketing dependencies implement synchronization.
- Allocation, preparation, and object ownership contracts.
- The indirect target’s identity and whether it ends the object’s lifetime.
- The meaning of the constants 1 and 0 passed to processing.
- Error names corresponding to results 12 and 33.
- Signed-overflow behavior at the decrement boundary.
- Reentrancy, concurrent access, nonlocal exits, and dependency-induced invalidation of addresses.
- The mechanism underlying the annotated non-returning failure call.

Inferred intent is limited to a possible resource-preparation operation followed by a gated processing attempt. That interpretation is not required to reproduce the established observable behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes argument forwarding order, literal values, context offsets, the preparation-result null check, placement of dependency calls, zero/nonzero branch conditions, the fresh read before decrement, the indirect table-entry offset, and ordinary result selection.

The non-returning property is supplied as an annotation, not independently verified. Allocation, synchronization, destruction, ownership transfer, and application-level status meanings are not established.

No runtime tests, execution traces, dependency implementations, or independent ABI evidence were available or used.
