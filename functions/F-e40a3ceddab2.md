<!-- Trusted provenance: F-e40a3ceddab2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e40a3ceddab2

## Purpose

F-e40a3ceddab2 records a requested 32-bit position in a caller-supplied object, then determines success through either an installed callback or an unsigned comparison against an object field.

The position write occurs before validation or callback execution. A rejected request is therefore not automatically rolled back: without intervening changes by a dependency, the object retains the rejected position.

Established behavior:

- Return the 32-bit value `0` when the callback returns zero, or when no callback is installed and the requested position is no greater than the comparison field.
- Otherwise, invoke diagnostic dependency F-4457a9a44a68 and return the 32-bit value `0x00000055`, decimal 85, if that dependency returns normally.

The apparent intent is a seek-like operation on a stream-like object. Those semantic labels are inferred; the visible accesses and control flow establish the contract below without requiring a particular public object type.

## Interface

### Inputs

The function accepts two arguments:

1. **Object address.** An address identifying an existing object whose relevant layout is given below.
2. **Requested position.** An unsigned 32-bit value, ranging from 0 through 4,294,967,295.

The object address is represented by a 32-bit argument in the supplied source view. It is used as an address rather than as an ordinary signed numeric quantity. Valid addressing, alignment, and accessible storage are caller preconditions; the function does not check them.

### Relevant object layout

Offsets are byte offsets from the supplied object address.

| Offset | Width | Role established by this function |
|---|---:|---|
| `+4` | 32 bits | Unsigned comparison value, read on the no-callback path and after a nonzero callback result |
| `+8` | 32 bits | Position field, overwritten with the requested position on every normally entered path |
| `+20` | Pointer-sized callable slot in the depicted 32-bit layout | Optional callback; a null value selects local comparison behavior |

These are the only direct object accesses shown. The object may contain additional fields; their meaning is not established here.

The comparison field can be described as a bound for this operation because the no-callback path accepts positions less than or equal to it. The source does not independently establish whether it is a byte length, an end offset, a capacity, or another quantity.

### Output

The normal return is a 32-bit value:

| Value | Meaning within this function |
|---|---|
| `0x00000000` | Accepted by the applicable decision rule |
| `0x00000055` | Rejected by that rule, followed by a normally returning diagnostic call |

No other normal return value is visible.

A zero return does not prove that the final position field still equals the requested position: an invoked callback may have modified it. Likewise, return 85 does not establish any particular final object state after the diagnostic dependency has executed.

### Callback interface

When the callable slot is nonnull, the callback receives four arguments, in this order:

1. The original object address.
2. The original unsigned 32-bit requested position.
3. A zero-valued argument.
4. A zero-valued argument.

The callback result is tested only for zero versus nonzero. No distinction between positive and negative nonzero results affects this function’s decision. The source view represents the result as a 32-bit integer.

The semantic types of the two trailing zero arguments are not established. They must not be described as particular flags, lengths, pointers, or output parameters without additional evidence.

## Behavior

### 1. Record the request before choosing a validation path

The first visible action is a 32-bit write of the requested position to object offset `+8`.

This write is unconditional with respect to the callback slot and comparison field. In particular:

- The old position is not read or saved.
- The requested position is not checked before being stored.
- No clamping, truncation below 32 bits, or normalization is shown.
- No direct restoration of the old position appears on any path.

The function then examines the callable slot at offset `+20`.

These ordering statements describe the visible accesses in this function, not a synchronization or atomicity guarantee for concurrent observers.

### 2. If the callback slot is null

The function reads the unsigned 32-bit comparison field at offset `+4`.

It compares the original requested position against that value using an unsigned less-than-or-equal comparison:

- If the requested position is less than or equal to the comparison value, return zero immediately.
- If the requested position is greater than the comparison value, proceed to the rejection diagnostic.

Equality is accepted. A position of zero is accepted for every possible comparison value on this path.

There is no subtraction, addition, or conversion to a signed position in the comparison. Values with the high bit set retain their unsigned ordering.

On this path, the comparison field is read after the position write and callback-slot check. The diagnostic, if needed, receives the comparison value captured by that read.

### 3. If the callback slot is nonnull

The function invokes the callback with the four arguments described in the interface.

The invocation occurs after the position write. Consequently, the callback can observe the recorded request through offset `+8`, assuming no other actor has changed it.

The callback result controls the next action:

- **Zero result:** return zero immediately. The function does not read the comparison field at offset `+4` on this path.
- **Nonzero result:** read the unsigned 32-bit comparison field at offset `+4`, then proceed to the rejection diagnostic.

There is no fallback local bounds check after callback execution. In particular:

- A zero callback result accepts the request even if the requested position exceeds the comparison field.
- A nonzero callback result rejects the request even if the requested position is within the comparison field.
- The actual nonzero callback result is not propagated to the caller or passed to the diagnostic.

Because the comparison-field read follows the callback, it can observe a value written by the callback. A pre-call value must not be substituted for this post-call read.

The function does not directly rewrite the position field after the callback. Any callback mutation of that field is therefore left in place by the visible remainder of the function, subject to possible effects of the diagnostic dependency.

### 4. Report rejection and return the fixed failure value

Both rejecting paths invoke F-4457a9a44a68 with:

1. A reference to fixed diagnostic text identifying an invalid seek position.
2. The original requested position.
3. The comparison value read on the rejecting path.

The fixed diagnostic reference is not caller-supplied. Its storage ownership and the dependency’s treatment of it are not established here.

The diagnostic call’s result is not used. If it returns normally, F-e40a3ceddab2 returns `0x00000055`.

No visible rollback, retry, alternate callback, or additional validity check occurs between the diagnostic call and that return.

## Side effects

### Direct object mutation

The function directly writes only the 32-bit position field at offset `+8`.

Absent changes by dependencies or other actors, that field contains the requested position on both success and rejection. Rejection is not a transactional failure that preserves the prior position.

The function does not directly modify the comparison field or callable slot.

### Indirect effects

The callback receives the object address and can potentially modify the object, other memory, or external state. Its effects are unknown from this source.

F-4457a9a44a68 is invoked only on rejection. Its externally observable behavior is also unknown. The call alone does not establish that a message is printed, a file is written, a global error is set, or execution continues.

Neither dependency may be assumed to preserve globals, preserve the object, or leave unrelated memory unchanged.

### Ownership and lifetime

No direct allocation, deallocation, resource acquisition, or resource release appears in this function.

The function uses caller-provided storage; it does not visibly create a replacement object or return ownership of anything. However, the callback’s and diagnostic dependency’s ownership behavior is undocumented. Their invocation is not evidence that they retain, release, or refrain from retaining or releasing any supplied address.

For ordinary execution, the object must remain valid through every access actually taken. A callback that invalidates the object before returning a nonzero result would leave the subsequent comparison-field read without its required valid storage.

### Invalid inputs and abnormal termination

There are no visible checks for:

- A null or invalid object address.
- Insufficient object storage.
- Misalignment.
- An invalid nonnull callback target.

Such inputs have no established graceful error outcome.

If a dependency fails to return normally, the normal return contract does not apply. No recovery mechanism is shown.

## Dependencies

### Dynamically selected callback

**Identification:** Runtime callable value stored at object offset `+20`; no fixed opaque function ID is supplied.

**Invocation condition:** The slot is nonnull.

**Inputs:** Original object address, original requested position, zero, zero.

**Result consumed:** Zero means success; any nonzero value means rejection.

**Unknowns:** Full calling convention, argument semantics beyond the visible values, side effects, ownership effects, reentrancy, and abnormal exits.

### F-4457a9a44a68

**Invocation condition:** Either the null-callback path finds the requested position greater than the unsigned comparison field, or the callback returns nonzero.

**Inputs:** Fixed diagnostic-text reference, original requested position, and the comparison value read on that rejecting path.

**Result consumed:** None.

**Unknowns:** Output destination, formatting rules, persistence, global mutations, ownership effects, and whether every invocation returns normally.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not runtime observations. Unless stated otherwise, assume valid storage, no concurrent mutation, normally returning dependencies, and no dependency changes beyond those specified.

### No callback: equality succeeds

Given a null callback slot and a comparison value of 120, a requested position of 120:

- Writes 120 to the position field before reading the comparison field.
- Returns zero.
- Does not invoke F-4457a9a44a68.

### No callback: an excessive position remains recorded

Given a null callback slot, an initial position of 7, and a comparison value of 120, a requested position of 121:

- Replaces 7 with 121 before validation.
- Invokes F-4457a9a44a68 with the fixed diagnostic reference, 121, and 120.
- Returns 85 if the diagnostic returns normally.
- Leaves the position at 121 if the diagnostic makes no relevant modification.

### Unsigned boundary behavior

With no callback and a comparison value of `0x7FFFFFFF`, requesting `0x80000000` is rejected. The request is larger under unsigned ordering; it is not treated as a negative position.

With no callback and a comparison value of `0xFFFFFFFF`, every possible unsigned 32-bit request is accepted, including `0xFFFFFFFF`.

### Zero callback result overrides the local bound

Given a nonnull callback, a requested position of 900, and a comparison field containing 10:

- The function first records 900.
- The callback receives the original object address, 900, zero, and zero.
- If the callback returns zero, the function returns zero without reading the comparison field itself or invoking the diagnostic.

### Nonzero callback result rejects an otherwise in-range request

Given a requested position of 5 and a comparison field of 100, a callback result of any nonzero value leads to the diagnostic call and then return 85. Being within the comparison field does not reverse the callback’s rejection.

### Callback changes the diagnostic value

Suppose the callback changes the comparison field from 100 to 25 and then returns nonzero for a request of 40.

The function reads 25 after the callback and supplies 40 and 25 to F-4457a9a44a68. It does not use the earlier value of 100.

### Callback changes the recorded position

Suppose the callback changes the position field from the requested value to 17, then returns zero.

The function returns zero without restoring the requested position. Under the stated assumptions, the final position is 17.

## Uncertainties

### Established static evidence

The position write, object offsets, unsigned comparison, callback argument order, zero/nonzero decision, post-callback comparison-field read, diagnostic invocation, and two normal return values are directly supported by the supplied source.

### Inferred intent

“Position,” “bound,” and “seek-like operation” are descriptive interpretations of the visible behavior and diagnostic context. They do not establish physical I/O, file positioning, byte units, or a complete stream abstraction.

### Documented-only unknowns

The following remain **documented_only**:

- The complete object type and layout.
- Endianness and ABI details not settled by the source view.
- Public symbolic meanings of return value 85.
- The callback’s identity and full contract.
- The diagnostic dependency’s full contract.
- Thread safety, synchronization, and reentrant behavior.
- Whether either dependency mutates globals, changes object ownership, or terminates execution.
- Whether a successful call corresponds to any external device or file movement.

No runtime tests have been performed.

## Evidence

This contract is based solely on the supplied static function body and the provided opaque dependency mapping.

The strongest evidence is the operation ordering: the position is stored before either decision path; the callback’s zero result returns immediately; a nonzero callback result is followed by a fresh comparison-field read; and both rejection paths call F-4457a9a44a68 before returning the same fixed value.

No implementation of either dependency, runtime trace, external type definition, or execution result was supplied. Conclusions about their internal behavior are therefore intentionally excluded.
