<!-- Trusted provenance: F-fbacd477f99e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbacd477f99e

## Purpose

F-fbacd477f99e conditionally updates an object using a selected entry, unconditionally decrements a separate object field, and conditionally invokes an indirect target obtained from global state.

The established sequence is:

1. Invoke a preparation dependency with an interior object address.
2. Invoke a selection dependency.
3. For a nonzero selection result, subtract an entry-associated quantity from an object field and invoke two entry-processing dependencies.
4. Decrement another object field regardless of whether an entry was selected.
5. Load an object pointer field and, if nonzero, read its first 32-bit word.
6. If both the pointer and that word are nonzero, read the current target from a separate global cell and invoke it.

Entry removal, consumption, accounting, and notification are plausible **inferred intent**. The supplied representation does not establish those meanings, a particular data structure, or any ownership transfer.

## Interface

### Arguments and result

The visible interface has four argument positions:

| Position | Established use |
|---|---|
| First | A 32-bit address-bearing value identifying the object. |
| Second | An opaque 32-bit value not explicitly used in the visible body. |
| Third | An opaque 32-bit value forwarded to F-217af407adee. |
| Fourth | An opaque 32-bit value forwarded to F-217af407adee. |

The function has no explicit result value. Return-register contents are not an established output.

F-217af407adee receives a second argument represented as a register-derived value following F-4a19e3742733. The evidence does not establish that this is the incoming second argument, an ordinary return value, or an intentional additional output of F-4a19e3742733. Its precise ABI meaning remains **documented_only**.

### Storage accessed

Offsets are byte offsets from the supplied object address or selected entry address.

| Location | Established access or use |
|---|---|
| Object + `0x08` | Interior address passed to F-33528ff50fcf. No direct access width is established at this location. |
| Object + `0x40` | Interior address passed to F-4a19e3742733; later read as a pointer field in the represented 32-bit environment. |
| Object + `0x48` | A 32-bit integer read, decremented, and written. |
| Object + `0x84` | A 32-bit integer conditionally read, reduced by an entry-associated quantity, and written. |
| Entry + `0x08` | Interior address passed to F-4610ab0b2d83. |
| Entry + `0x0c` | A 32-bit integer read as the subtraction operand. |
| Address loaded from object + `0x40` | Its first 32-bit word is read if the pointer is nonzero. |
| **DispatchTargetCell** | A separate global cell read to obtain the indirect target, only after both final conditions succeed. |

**DispatchTargetCell** is a stable abstract name for the global dispatch-target storage. It is not the object pointer field and is not the first word of the object-referenced storage.

The supplied representation uses a 32-bit address model. It does not establish a portable wider-pointer interface or independently establish the global cell’s physical storage width.

### Preconditions for ordinary completion

- The object address must support all direct object-field accesses reached on the selected path.
- A nonzero selected entry must provide readable storage at entry + `0x0c`.
- Dependencies must accept their supplied arguments and leave storage required by subsequent operations valid.
- A nonzero pointer loaded from object + `0x40` must permit a 32-bit read at that address.
- When dispatch is taken, DispatchTargetCell must be readable, and the target obtained from it must be callable under the applicable ABI.
- Address calculations must designate valid storage for the ordinary cases described here.

These are caller and environment obligations, not checks performed by this function. There is no visible null check for the object, range validation for a selected entry, or validity check for the target read from DispatchTargetCell.

## Behavior

### Preparation and selection

The function first invokes F-4a19e3742733 with object + `0x40`.

This passes the address of an object field, not the pointer value stored in that field. The dependency may therefore affect storage later read by this function. Its actual reads, writes, and other effects are unknown.

After that invocation returns, F-217af407adee receives:

- The original object address.
- The unresolved register-derived value associated with the preceding call.
- The incoming third argument.
- The incoming fourth argument.

Its result is treated as an address-bearing value. Zero skips entry processing; any nonzero result is used as an entry address without further visible validation.

### Nonzero selection

For a nonzero selection result, the function first updates object + `0x84`.

It reads the 32-bit object quantity and the 32-bit quantity at entry + `0x0c`, subtracts the entry quantity from the object quantity, and writes the result to object + `0x84`. Both operand reads precede the store; their relative order is not established by the supplied representation.

There is no visible operand-sign check, sufficiency check, saturation, or lower-bound clamp.

After the quantity store:

1. F-4610ab0b2d83 is invoked with the object address and entry + `0x08`.
2. After that dependency returns, F-33528ff50fcf is invoked with object + `0x08` and the selected entry address.

Both dependencies therefore observe entry processing after the direct quantity update has occurred, subject to their own behavior and any interference.

The entry addresses supplied to these calls are derived from the selection result. The function does not visibly reselect the entry or reload its address from the object between these calls. It makes no direct entry-field read after F-33528ff50fcf returns.

The call sequence does not establish that either dependency unlinks, destroys, or releases the entry.

### Zero selection

For a zero selection result, the function skips:

- The direct read-modify-write at object + `0x84`.
- F-4610ab0b2d83.
- F-33528ff50fcf.

It still proceeds to the decrement and dispatch conditions.

This branch does not guarantee that object + `0x84` remains unchanged throughout the invocation. Preparation or selection may have modified it.

### Unconditional decrement

After the selection branch completes, the function reads the current 32-bit value at object + `0x48`, subtracts one, and writes the result back.

This operation occurs on both selection paths, provided preceding operations return normally. It uses the value present at this point, including any changes made by earlier dependencies. It does not use a pre-call snapshot.

There is no visible zero check, saturation, or rollback.

### Final pointer and word checks

After storing the decremented value, the function loads the pointer field at object + `0x40`.

- If the pointer is zero, no pointee read or indirect dispatch follows.
- If the pointer is nonzero, the function reads the first 32-bit word at that address.
- If that word is zero, no indirect dispatch follows.
- If that word is nonzero, execution reaches the global target read.

The pointer is obtained after all earlier dependencies and the decrement. An earlier pointer snapshot is not established as equivalent.

The conditions use the loaded pointer and the word read through it. No second load of the object pointer field or second read of the pointee word is visible before dispatch. The pointee word is a condition, not the source of the dispatch target.

### Global target read and invocation

Only on the dispatch-taking path does the function read **DispatchTargetCell** to obtain the target.

The established ordering is:

1. Store the decremented value at object + `0x48`.
2. Load the final pointer from object + `0x40`.
3. Establish that the pointer is nonzero.
4. Read its first 32-bit word and establish that the word is nonzero.
5. Read the current target from DispatchTargetCell.
6. Invoke that target.

The function’s visible behavior does not read DispatchTargetCell on either dispatch-skipping path. This restriction concerns the function’s own direct access; unknown dependencies might independently access the same global state.

Earlier dependencies cannot be assumed to preserve DispatchTargetCell. If they change it, the later global read obtains the value present at that read. Caching the target before preparation, selection, entry processing, the decrement, or the final checks is not established as equivalent.

No additional null or validity check of the obtained target is visible. Nonzero guard values alone do not make an invalid target callable.

No explicit invocation arguments are recoverable. The target identity and ABI are **documented_only**. The supplied representation identifies the indirect transfer as incompletely recovered, so whether the underlying mechanism is an ordinary call, tail transfer, or another indirect control transfer also remains **documented_only**.

No further direct state operations are shown after the represented invocation returns.

### Arithmetic boundaries

The directly manipulated quantities are represented as signed 32-bit integers. For results within the signed 32-bit range, the established behavior is subtraction and decrement.

The source-facing evidence does not justify a portable overflow rule for out-of-range signed results. Wraparound, trapping, or another exact boundary behavior must not be selected without additional evidence. Such cases remain **documented_only**.

Address calculations likewise belong to the represented 32-bit environment. No address-overflow or range checks are visible.

## Side effects

The function’s direct writes are:

- One write to object + `0x84` on the nonzero-selection path.
- One write to object + `0x48` on either path that reaches the decrement.

Its additional observable actions are dependency invocations, the conditional pointee read, the conditional read of DispatchTargetCell, and the conditional indirect invocation.

The required ordering is:

- Preparation before selection.
- Conditional quantity store before both entry-processing calls.
- F-4610ab0b2d83 before F-33528ff50fcf.
- Both entry-processing calls, when reached, before the decrement.
- Decrement store before the final object-pointer load.
- Successful pointer and pointee-word checks before the global target read.
- Global target read before the indirect invocation.

There is no direct write to DispatchTargetCell in the visible body. That does not imply that dependencies or other execution contexts preserve it.

No direct allocation, deallocation, lock operation, error report, or status return is visible. Their absence from this body does not establish their absence inside dependencies.

No explicit recovery or rollback is shown. If a dependency does not return, subsequent operations are not reached. If an invalid access or other failure interrupts execution, earlier stores and dependency effects may already have occurred.

Ownership, storage lifetime, synchronization, and reentrancy are not established. In particular, unknown callees must not be assumed to preserve globals, preserve object storage, or own or release entry memory.

## Dependencies

| Opaque dependency ID | Visible contract |
|---|---|
| F-4a19e3742733 | Invoked first with object + `0x40`. Effects and the meaning of the subsequent register-derived selection argument are unknown. |
| F-217af407adee | Receives the object, unresolved second value, and incoming third and fourth arguments. Its zero/nonzero result controls entry processing. Selection policy and side effects are unknown. |
| F-4610ab0b2d83 | Invoked only for a nonzero entry, after the quantity store, with the object and entry + `0x08`. Ownership and structural effects are unknown. |
| F-33528ff50fcf | Invoked after F-4610ab0b2d83 with object + `0x08` and the selected entry address. Release or unlinking behavior is not established. |
| F-indirect-target | Contract-local opaque ID for whichever target is read from DispatchTargetCell on the dispatch-taking path. Identity, arguments, effects, and transfer mechanism are unresolved. |

F-indirect-target does not identify a fixed function across invocations. DispatchTargetCell supplies its identity at the late global read.

These are call-site observations, not specifications of dependency internals.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless specified otherwise, assume valid storage, normally returning dependencies, no interfering mutation, and representable signed arithmetic.

### Selected entry with ordinary quantities

If selection returns a valid entry, the object quantity is 100, and the entry quantity is 24 at their respective reads:

- The direct quantity store must write 76.
- That store must precede F-4610ab0b2d83.
- F-33528ff50fcf must follow F-4610ab0b2d83.
- The decrement must follow both calls.

If object + `0x48` contains 7 immediately before its read-modify-write, the stored result must be 6.

### No selected entry

If selection returns zero and object + `0x48` contains 3 at the decrement:

- No direct quantity update or entry-processing call occurs.
- The decrement stores 2.
- The final pointer condition is still evaluated.

### Negative operand and zero decrement value

If the object quantity is 10 and the entry quantity is −3, the quantity update stores 13. The negative operand is not visibly rejected.

If the decrement field is zero at its read, the store writes −1. Zero does not suppress the update.

### Late field mutation

If an entry-processing dependency changes object + `0x48` to 11 before returning, the subsequent decrement stores 10, assuming no later interference.

If a dependency changes object + `0x40`, the final condition uses the pointer loaded after the decrement, not an earlier value.

### Skipped dispatch

If the final pointer is zero, the function performs neither a pointee read nor its direct DispatchTargetCell read.

If the final pointer is nonzero but its first word is zero, the function reads that word but does not directly read DispatchTargetCell or invoke F-indirect-target.

These criteria do not forbid unknown dependencies from reading the global cell themselves.

### Dispatch after global mutation

Suppose DispatchTargetCell initially identifies target A. An earlier dependency changes it to target B. The final pointer and its first word are both nonzero, and the cell still identifies B when the global read occurs.

The function must obtain B from DispatchTargetCell at that point and dispatch to B. Dispatching to A using a target cached before the dependency is not equivalent to the established behavior.

### Invalid target despite successful guards

If both guard conditions succeed but DispatchTargetCell does not contain a callable target, ordinary completion is not guaranteed. No fallback, target validation, error return, or rollback is established.

## Uncertainties

The following remain **documented_only**:

- Concrete object and entry types and semantic field names.
- Precise ABI, including the register-derived second selection argument.
- Any unrecovered relationship between that value and the incoming second argument.
- Exact signed-overflow and address-overflow behavior.
- Dependency side effects, memory ownership, and lifetime changes.
- DispatchTargetCell’s physical width, initialization rules, and synchronization.
- Indirect target identity, argument contract, and underlying transfer mechanism.
- Concurrency, reentrancy, and behavior under interfering mutation.

These unknowns do not weaken the established late-read requirement: the visible dispatch path obtains its target from DispatchTargetCell after the decrement and successful final checks.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping. The prior draft and feedback are not independent behavioral authority.

Static evidence establishes the explicit access offsets, represented widths, conditional stores, dependency order, final guards, and separate global dispatch-target read. In particular, the global target access appears only within the branch reached after the final pointer and pointee word have both tested nonzero.

The representation explicitly marks the indirect transfer as incompletely recovered. It therefore supports the guarded target read and represented invocation without resolving the underlying transfer mechanism.

No dependency bodies, authoritative type definitions, instruction-level validation, or runtime observations were supplied. No runtime tests have been performed.
