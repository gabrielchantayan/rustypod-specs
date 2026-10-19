<!-- Trusted provenance: F-60dea9ea67ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-60dea9ea67ea

## Purpose

F-60dea9ea67ea performs an indirectly selected call, copies one four-byte value into a caller-supplied object, writes a second four-byte field with all bits set, and returns the four-byte value 1.

The ordering is part of the contract: the indirect call precedes the source-value read and both object-field writes. The copied value therefore reflects the source memory after the indirect call returns, not necessarily its value on entry.

An initialization or reset role is plausible, but not established. The meanings of the copied field, the all-bits-set field, and the indirect call are unknown.

## Interface

### Inputs

The function has two explicit inputs:

| Input | Representation established by the supplied source | Use |
|---|---|---|
| Object address | Integer address carrier used to access four-byte fields | Supplies the dispatch reference and identifies the two destination fields |
| Source address | Pointer to a four-byte quantity | Supplies the value copied after the indirect call returns |

The source represents the object address and the intermediate dispatch address using an integer type consistent with a 32-bit address model. Four-byte data accesses are explicit. The exact target ABI, pointer representation, and alignment requirements are not independently established.

The relevant object layout is:

| Byte offset from object address | Width | Access by this function |
|---|---:|---|
| +4 | 4 bytes | Read to obtain the dispatch-base address |
| +12 | 4 bytes | Written with the source value |
| +16 | 4 bytes | Written with all bits set |

At byte offset +12 from the dispatch-base address, the function reads a callable target. Under the apparent 32-bit address model, this is a four-byte function-pointer entry. Its exact ABI representation remains unverified.

These offsets are byte offsets, not indices into an array of four-byte elements. The dispatch entry at dispatch-base +12 and the destination at object +12 belong to separate address calculations; they need not identify the same storage.

### Output

On the visible normal-return path, the function returns a four-byte value whose bit pattern is `0x00000001`.

There is no visible alternate result, status propagation, or conditional return. The indirect target’s return value does not determine the enclosing function’s result.

### Preconditions

Normal completion requires:

- A readable dispatch reference at object +4.
- A readable callable-target entry at dispatch-base +12.
- A target callable under the actual execution environment’s calling convention.
- A readable four-byte source value when execution resumes after the indirect call.
- Writable four-byte storage at object +12 and object +16 when the respective writes occur.
- Address calculations and accesses valid under the actual machine and memory model.

The wrapper does not visibly check these conditions. In particular, it does not reject null inputs, validate a dispatch table, or check whether the indirect call invalidated memory needed later.

No distinctness requirement between the source, object, and dispatch storage is visible. Aliasing affects the result through the access order described below.

## Behavior

### 1. Resolve the indirect target

The function reads the four-byte field at object +4 and uses its value as the dispatch-base address.

It then obtains the callable target from byte offset +12 relative to that dispatch base.

These dispatch-related reads occur before the source-value read and before either direct destination write. The source does not show an earlier capture of the source value.

No additional object fields are visibly consulted to select the target.

### 2. Invoke the target

The selected dependency is invoked before either destination field is updated by this function.

The displayed call has no explicit arguments and no explicit receiver argument. That establishes the absence of visible argument expressions, not a complete machine-level ABI contract. It does not establish whether the target relies on registers, global state, or other implicit context.

The target’s return value is not visibly inspected or propagated. If the target returns normally and the subsequent accesses succeed, this function proceeds to both writes regardless of that value.

The indirect target’s other behavior is unknown. It may alter the source memory, destination fields, dispatch storage, global state, or resource ownership. This contract assumes none of those are preserved.

### 3. Read and copy the source value

After the indirect call returns, the function reads exactly four bytes from the supplied source address.

It then stores that four-byte value at object +12.

Consequences of this ordering include:

- A source change made by the indirect target is reflected in the copied value.
- A change made by the target to object +12 is overwritten by this store, provided the store completes.
- The source value is read before this store changes any overlapping destination bytes.
- The function does not visibly read a second source element or copy a larger structure.

The transferred quantity is a four-byte bit pattern. No signedness conversion, range restriction, or semantic interpretation is visible.

### 4. Write the all-bits-set field

After the source-copy store, the function writes the four-byte bit pattern `0xFFFFFFFF` at object +16.

This is a separate store following the store at object +12. Under ordinary valid contiguous layout, the two destination fields are adjacent and do not overlap.

The established value is “all 32 bits set.” Calling it a negative number, invalid identifier, reset marker, or other sentinel would require information not present here.

### 5. Return

After both stores, the function returns `0x00000001`.

No visible read-back verifies the stores. The source provides no transactional boundary, rollback, or synchronization operation.

### Exceptional and interrupted paths

The visible body contains no error-handling branch. It does not establish a defined result for an invalid address, an invalid target, an access fault, or an indirect call that does not return.

Progress may be partial. For example, if the source-copy store completes but the final store cannot complete, the source provides no mechanism that restores the earlier destination value. Target-specific fault delivery and nonlocal control transfer are outside the established contract.

## Side effects

### Direct memory effects

On normal completion, the direct writes are limited to:

1. Four bytes at object +12, containing the post-call source value.
2. Four bytes at object +16, containing `0xFFFFFFFF`.

The function directly reads the dispatch reference, the callable-target entry, and the source value. It does not visibly write object +4 or the dispatch entry.

These limits apply to the wrapper’s explicit accesses, not to the indirect target.

### Observable ordering

The visible sequence is:

1. Read the object’s dispatch reference.
2. Read the selected callable target.
3. Invoke that target.
4. Read the source value.
5. Write object +12.
6. Write object +16.
7. Return 1.

This sequence is not a guarantee of atomicity, cross-thread visibility, or externally observable ordering on every architecture. No locks, barriers, or atomic operations are established. Concurrent modification requires an external memory-model contract.

### Ownership and lifetime

The wrapper contains no explicit allocation or release operation and performs no visible ownership bookkeeping.

That does not establish that ownership is unchanged across the whole invocation. The indirect target may have resource-management effects. Its ownership contract is unknown.

For normal completion, memory needed after the call must remain accessible after that call. If the target releases or otherwise invalidates the source or destination storage, the visible wrapper supplies no recovery mechanism.

## Dependencies

### F-INDIRECT-01

F-INDIRECT-01 is a document-local opaque label for the dynamically selected callee, not an established identity for a particular implementation.

- **Selection:** callable-target entry at dispatch-base +12, where dispatch base comes from object +4.
- **Timing:** invoked once on the visible straight-line path, before the source read and destination writes.
- **Explicit arguments:** none shown.
- **Return handling:** result unused by F-60dea9ea67ea.
- **Required continuation behavior:** must return control for the later accesses and enclosing return to occur.
- **Unknown contract:** ABI details, implicit inputs, side effects, global-state preservation, ownership effects, and failure behavior.

No named concrete implementation of this dependency can be established from the supplied material. No other direct call is visible.

## Acceptance examples

These are conditional acceptance cases derived from static evidence. They have not been executed.

### Ordinary completion

Given valid dispatch storage, a normally returning target, accessible destination fields, and a source containing `0x12345678` after the target returns:

- Object +12 receives `0x12345678`.
- Object +16 receives `0xFFFFFFFF`.
- The function returns `0x00000001`.

### Source changed during the call

Given a source containing `0x11111111` on entry, if F-INDIRECT-01 changes it to `0x22222222` and returns normally:

- Object +12 receives `0x22222222`, assuming no intervening modification.
- Copying the entry-time value would contradict the established read order.

### Target return value

Given a target that returns a zero-like value normally, with all later memory accesses valid:

- Both stores still occur.
- F-60dea9ea67ea returns 1.

The target result is not a visible success gate.

### Source aliases object +16

Given that the source address equals object +16 and that field contains `0xAABBCCDD` after the target returns:

- The source read obtains `0xAABBCCDD`.
- Object +12 receives `0xAABBCCDD`.
- Object +16 is subsequently overwritten with `0xFFFFFFFF`.

### Source aliases object +12

Given that the source address equals object +12 and all accesses are valid:

- The field is read after the target returns.
- The same four-byte value is stored back at object +12.
- Object +16 is then set to `0xFFFFFFFF`.

This does not imply that object +12 retains its entry-time value; the target may have changed it.

### Target does not return

If F-INDIRECT-01 does not return control:

- Execution does not reach this wrapper’s source read, destination stores, or normal return.
- Any writes performed by the target itself remain outside the wrapper’s direct-write description.

## Uncertainties

The following are **documented_only**, not resolved by assumption:

- The application-level purpose and meanings of the two destination fields.
- The exact target architecture, endianness, calling convention, and alignment rules.
- The concrete identity and complete contract of F-INDIRECT-01.
- Whether the indirect call uses implicit object context.
- Whether the dependency preserves globals, memory validity, or resource ownership.
- The intended interpretation of `0xFFFFFFFF`.
- Behavior for null, misaligned, unmapped, wrapped, or otherwise invalid addresses.
- Concurrency guarantees and access atomicity.
- Exception handling or fault handling supplied outside the visible function.

Offsets +4, +12, and +16 are established. Defined address wraparound is not. This contract does not extend successful operation to arithmetic overflow or wrapped addresses merely because a machine might compute them.

## Evidence

The supplied static body establishes the dispatch lookup, indirect invocation, subsequent four-byte source read, two ordered four-byte stores, and constant four-byte return value.

It does not establish application semantics, a concrete dependency identity, ownership guarantees, or runtime fault behavior. Initialization/reset terminology is inferred intent only.

No runtime tests, execution traces, ABI verification, or external dependency inspection have been performed. The acceptance examples specify conditional behavior; they are not reports of observed executions.
