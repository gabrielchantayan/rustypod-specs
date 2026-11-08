<!-- Trusted provenance: F-33e1c7c6ba3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-33e1c7c6ba3f

## Purpose

F-33e1c7c6ba3f coordinates a request involving a target object, a selector, an ambient context, and an external service. It first checks eligibility, constructs a temporary object, and enforces an unsigned numeric limit. Accepted requests are submitted to the temporary object. A service predicate then determines whether an additional service operation is invoked.

An accepted request sets a byte in an associated state object and returns zero. Rejection at either visible gate returns the 32-bit value `0xffffffd7`, equivalent to −41 when interpreted as a signed 32-bit integer.

These are structural descriptions. The application meanings of the request, selector, numeric contribution, temporary object, and state byte are not established by the supplied evidence.

## Interface

### Inputs

The function receives two 32-bit inputs:

| Input | Established use |
|---|---|
| Target | Used as an object address, passed to dependencies, and used for fixed-offset memory reads. |
| Selector | Compared with signed −1. Other values are forwarded without local range validation. |

The selector has two behavior classes:

- **Sentinel:** all 32 bits set, interpreted as −1.
- **Explicit:** every other 32-bit value, including other negative values.

The target must support the accesses reached by the selected execution path. There is no local null, bounds, lifetime, or object-type validation.

### Return value

The return value is 32 bits:

- `0`: the request passed both visible gates and reached the final state write and temporary-object cleanup.
- `0xffffffd7`: eligibility failed, or the unsigned limit comparison rejected the request.

Zero does not establish that the additional service operation occurred. That operation is conditional on a separate predicate. Nor does zero establish an application-level outcome reported by a dependency: dependency return values other than the eligibility result, numeric value, context comparison, and predicate are not checked.

### Referenced layouts

All offsets below are byte offsets. Pointer-bearing fields are accessed as 32-bit values in the supplied representation.

| Object | Offset | Access and role |
|---|---:|---|
| Captured state object | `0xf40` | A 32-bit value read for eligibility, then read again for temporary-object initialization. |
| Captured state object | `0xf00` | A 32-bit pointer read near completion. |
| Object referenced through state offset `0xf00` | `0xb92` | One byte written with value `1`. |
| Target | `0xef4` | An unsigned 32-bit contribution, read only in one sentinel subcase. |
| Target | `0xe3c` | One byte tested for zero versus nonzero. |
| Captured ambient context | `0x40` | A 32-bit pointer used in one sentinel subcase. |
| Object referenced through context offset `0x40` | `0x2e` | An unsigned 16-bit contribution, zero-extended to 32 bits. |
| Captured ambient context | `0x18c` | One byte whose least significant bit controls the explicit-selector mode. |
| Service object | `0` | A pointer to a dispatch table. |
| Service dispatch table | `0x1b8` | Predicate operation. |
| Service dispatch table | `0x128` | Conditional additional operation. |

The function also supplies dependencies with the address of a 32-byte local storage region. Its internal layout and semantic type are unknown.

## Behavior

### 1. Capture ambient objects and check eligibility

The function first calls F-267f68fe54d6 with no visible argument and retains its 32-bit result as the ambient context. It then calls F-8ff48652068d and retains its result as the state-object address.

It reads the 32-bit field at state offset `0xf40` and passes that value to F-3b037d560480.

If this dependency returns zero, the function immediately returns `0xffffffd7`. This path does not acquire the service, initialize the local temporary, invoke its cleanup dependency, or perform the final state-byte write.

The two ambient-object acquisition calls still precede this rejection and may have their own effects.

### 2. Acquire the service and initialize temporary storage

After a nonzero eligibility result, the function:

1. Calls F-87be4aa684d1 and retains the returned service-object address.
2. Reads state offset `0xf40` again.
3. Calls F-1c06d0a5910b with the local storage address, this newly read field value, and zero.
4. Calls F-d5511c118375 with the local storage address and retains its 32-bit result as the base quantity.

The second read of state offset `0xf40` is independent of the eligibility read. Intervening dependencies may change that field, so the checked value and initialized value need not match.

No return value from the initialization call is tested.

### 3. Apply the selector-specific limit

Both selector classes use a strict comparison against 1000, with **unsigned 32-bit addition and comparison**.

Consequently, the contract is not simply “the mathematical total must be at most 1000.” Addition wraps modulo \(2^{32}\), and a base quantity whose bit pattern represents a negative signed integer participates as an unsigned 32-bit value.

#### Sentinel selector

The function calls F-267f68fe54d6 again, this time with the target as a visible argument, and compares the result with the previously captured ambient context.

- If the results are equal, it calls F-30529d2a9890 with the target, then reads the target’s unsigned 32-bit field at `0xef4`. This read occurs after the call.
- If the results differ and the captured ambient context is nonzero, it reads the pointer at context offset `0x40`, then reads an unsigned 16-bit value at offset `0x2e` in the referenced object.
- If the results differ and the captured ambient context is zero, the contribution is zero.

The base quantity and selected contribution are added modulo \(2^{32}\). A resulting unsigned value greater than 1000 rejects the request.

Equality between the two context results selects the target-field path even if both results are zero.

#### Explicit selector

The function adds unsigned one to the base quantity modulo \(2^{32}\). A result greater than 1000 rejects the request.

This limit check precedes the explicit branch’s read of the captured context’s flag byte. A rejected explicit request therefore does not require that later read to succeed.

#### Limit rejection

On rejection from either selector class, the function calls F-c1466ea05103 with the local storage address and then returns `0xffffffd7`.

The final state byte is not written on this path. Effects already caused by acquisition, initialization, quantity calculation, context lookup, or F-30529d2a9890 are not locally reversed.

### 4. Submit the accepted request

For an accepted sentinel request:

1. Read target offset `0xe3c` and normalize it to zero or one.
2. Call F-be0967308708 with the target to obtain a 32-bit mode.
3. Call F-d3be6a307196 with, in order:
   - the local storage address;
   - the target;
   - selector −1;
   - the originally captured ambient context;
   - the mode;
   - the normalized flag.

The flag is captured before the mode-producing dependency runs.

For an accepted explicit request:

1. Read the byte at captured-context offset `0x18c`.
2. If its least significant bit is clear, call F-be0967308708 with the target and use its result as the mode.
3. Otherwise, use mode `2`, without calling F-be0967308708.
4. Read target offset `0xe3c` and normalize it to zero or one.
5. Call F-d3be6a307196 with the local storage address, target, original selector, captured context, mode, and normalized flag.

Thus, when the explicit branch obtains its mode from a dependency, the submission flag is read afterward. The sentinel branch reads that flag beforehand.

### 5. Evaluate the service predicate and conditionally act

After F-d3be6a307196 returns, the function reads the service’s dispatch-table pointer and invokes the operation at byte offset `0x1b8`, passing the service object.

A zero result skips the additional service operation. A nonzero result enables it.

For a sentinel selector with a nonzero predicate:

1. Read and normalize target offset `0xe3c` again.
2. Call F-be0967308708 again.
3. Read the service’s current dispatch-table pointer and the operation at offset `0x128`.
4. Invoke that operation with:
   - the service object;
   - selector −1;
   - the originally captured context;
   - the newly obtained mode;
   - the newly captured flag;
   - zero.

The second mode call is independent of the first and may return a different value.

For an explicit selector with a nonzero predicate:

1. Read and normalize target offset `0xe3c` again.
2. Read the service’s current dispatch-table pointer and operation at offset `0x128`.
3. Invoke that operation with the service object, original selector, captured context, previously selected mode, newly captured flag, and zero.

The explicit branch reuses its mode; it does not recompute it after the predicate.

The additional operation’s return value is ignored.

### 6. Mark state, clean up, and return

Regardless of whether the predicate enabled the additional operation, an accepted path:

1. Reads the pointer currently stored at offset `0xf00` in the originally captured state object.
2. Writes byte value `1` at offset `0xb92` in the referenced object.
3. Calls F-c1466ea05103 with the local storage address.
4. Returns zero.

The write precedes cleanup. Cleanup may have effects of its own; the function does not restore the byte afterward.

## Side effects

The only direct persistent-memory write visible in the function is the final one-byte store of `1`. It is absent on both rejection paths.

Dependencies may mutate the target, captured objects, referenced objects, service dispatch table, globals, or the local storage. Their effects are not bounded by this function’s visible reads and writes. In particular:

- The state and ambient-context addresses are captured once and reused.
- Fields within those captured objects are read at the stated times, not all snapshotted initially.
- The service dispatch table is read separately for each indirect call.
- The target flag is reread before the conditional service operation.
- The final destination pointer is obtained after the service calls.

No explicit allocation, ownership transfer, or direct release of the target, context, state, or service is visible. The local storage has automatic lifetime, but whether its dependencies acquire, retain, transfer, or release other resources is unknown.

The cleanup dependency is called once on each normally returning path after initialization. It is not called on the initial eligibility rejection. Abnormal control flow from dependencies or invalid memory accesses has no visible local recovery mechanism.

## Dependencies

| Opaque ID or slot | Established interaction |
|---|---|
| F-267f68fe54d6 | Supplies the initial ambient context; called again with the target only for the sentinel branch. |
| F-8ff48652068d | Supplies the captured state-object address. |
| F-3b037d560480 | Receives the first state-field value; zero rejects. |
| F-87be4aa684d1 | Supplies the service-object address after eligibility succeeds. |
| F-1c06d0a5910b | Receives local storage, the second state-field value, and zero. |
| F-d5511c118375 | Supplies the base quantity from local storage. |
| F-30529d2a9890 | Called before reading the target contribution when sentinel context comparison succeeds. |
| F-be0967308708 | Supplies a mode; invocation count and timing depend on selector, context flag, and predicate. |
| F-d3be6a307196 | Receives the accepted request and its selected parameters. |
| Service slot `0x1b8` | Returns the predicate controlling the additional operation. |
| Service slot `0x128` | Receives the conditional additional operation; return value ignored. |
| F-c1466ea05103 | Receives local storage on limit rejection or final completion. |

No independent opaque IDs are supplied for the two indirect operations.

## Acceptance examples

These are conditional contract examples, not runtime observations. They assume valid reachable memory and normally returning dependencies.

1. **Eligibility rejects.**  
   If F-3b037d560480 returns zero, the result is `0xffffffd7`. The service is not acquired, and no local temporary cleanup or final byte write occurs.

2. **Explicit-selector boundary.**  
   With base quantity 999, adding one produces 1000, so the request passes the numeric gate. With base quantity 1000, the result is 1001, so cleanup occurs and the function returns `0xffffffd7`.

3. **Unsigned wraparound.**  
   With an explicit selector and base bit pattern `0xffffffff`, adding one produces zero. The numeric gate accepts. Acceptance still depends on the later memory accesses and calls completing normally.

4. **Sentinel contribution after mutation.**  
   If the two context results compare equal and F-30529d2a9890 changes target offset `0xef4`, the subsequent contribution read observes the post-call value.

5. **Different sentinel contexts with no captured context.**  
   If the original context is zero and the later context result differs, contribution zero is used. No context-relative 16-bit contribution is read.

6. **Explicit forced mode.**  
   If the context byte at `0x18c` has bit zero set, the submission mode is `2`. F-be0967308708 is not called on that explicit path, including when the service predicate is nonzero.

7. **Predicate declines additional action.**  
   An accepted request with a zero service predicate still performs the final byte write, cleanup, and zero return. Slot `0x128` is not invoked.

8. **Flag changes between submission and action.**  
   If target offset `0xe3c` changes between initial submission and the later flag read, the conditional service operation receives the later normalized value.

## Uncertainties

### Inferred intent

“Eligibility,” “quantity,” “mode,” “submission,” and “cleanup” are descriptive labels inferred from control flow and argument use. They do not establish domain semantics.

The limit of 1000 may represent a capacity constraint, but its unit and rationale are unknown.

### documented_only

The following remain undocumented beyond their visible use:

- Dependency implementations, complete calling conventions, ownership rules, and global effects.
- The meaning and internal structure of the 32-byte local region.
- Whether the final byte denotes completion, invalidation, notification, or another state.
- Valid selector ranges beyond the special handling of −1.
- Whether dependencies can fail through exceptions, nonlocal control transfer, termination, or other mechanisms.
- Whether callers guarantee that arithmetic cannot wrap.
- Whether the two visible invocation forms of F-267f68fe54d6 reflect an unusual interface or incomplete signature information.
- Thread-safety, synchronization, and reentrancy guarantees.

No global-preservation or memory-release guarantee is assumed for any dependency.

## Evidence

The established behavior above is derived solely from the supplied static function representation: its branch structure, field-access widths, unsigned arithmetic, dependency arguments, indirect dispatch offsets, direct store, and return constants.

No dependency bodies or independent interface definitions were supplied. No runtime tests have been performed. The acceptance examples are derived conditions, not execution evidence.
