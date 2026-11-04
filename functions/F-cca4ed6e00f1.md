<!-- Trusted provenance: F-cca4ed6e00f1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cca4ed6e00f1

## Purpose

F-cca4ed6e00f1 obtains a signed result from an object and an effective four-byte value, then performs an ordered sequence of object callbacks and finalization around that result.

An initial object query selects between two paths:

- Use the caller-supplied four-byte value directly.
- Require a nonzero third argument and obtain a replacement value through object callbacks.

After obtaining the result, the function conditionally requests an operation over an inclusive-looking range, propagates a fresh object-query result to another callback, issues a result-related callback, and invokes a direct dependency before returning.

This describes the visible orchestration, not the semantic purpose of the object. Interpretations such as insertion, selection, notification, or range invalidation are **inferred intent**, not established behavior.

## Interface

### Inputs

The function receives three arguments:

1. **Object reference.** A pointer to an object whose first represented word supplies a dispatch-table address. The next represented integer word is read later as a signed value.
2. **Supplied value.** An opaque four-byte value. Its interpretation is unknown. Depending on the initial query, it is either passed to a direct dependency or replaced before that call.
3. **Auxiliary argument.** A signed integer used in three ways:
   - Compared with zero on the replacement-value path.
   - Passed to the callback that obtains a replacement value.
   - Passed unchanged to a later callback alongside the computed result.

No pointee relationship, string encoding, ownership transfer, or allocation policy is established for either the supplied value or the auxiliary argument.

### Result

The represented return type is a signed integer.

There are two normal return cases:

- **Early refusal:** return −1 when the initial object query is nonzero and the auxiliary argument is zero.
- **Completed sequence:** return the result produced by F-a8f246f36c15.

The completed sequence does not treat a negative result, including −1, as an error requiring an early exit. Consequently, a returned −1 alone does not establish that the early-refusal path occurred.

### Widths and layout

The supplied representation explicitly identifies the supplied value as four bytes. It presents the other arguments, the result, and the object’s second word as integer or pointer-related quantities.

The following layout describes the apparent 32-bit model:

| Location | Represented role |
|---|---|
| Object offset 0 | Dispatch-table pointer |
| Object offset 4 | Signed integer read after F-a8f246f36c15 |
| Dispatch-table byte offsets | Callback entries identified below |

The supplied fragment does not independently establish the native ABI, pointer width, integer width, calling convention, or structure declaration. In particular, object offset 4 relies on the apparent four-byte integer model. Dispatch-table offsets are byte offsets explicitly present in the representation.

The signed comparisons and subtraction described below must be preserved. They must not be silently reinterpreted as unsigned arithmetic.

### Preconditions

Normal completion requires:

- A readable object reference and readable dispatch-table reference at every dispatch.
- Valid, callable entries for every callback reached on the selected path.
- A readable second object word at the time that word is sampled.
- Callback implementations compatible with the represented argument and return usage.
- Object and dispatch storage that remain usable after every preceding dependency call.

The wrapper performs no explicit null, bounds, lifetime, callback-presence, or object-validity checks.

## Behavior

The following sequence describes observable ordering when each dependency returns normally. Dependency-internal behavior remains outside this contract.

### 1. Query the initial condition

The function dispatches F-indirect-01 through table offset 0x60, passing the object.

Only whether its signed result equals zero controls the initial branch. Any nonzero result selects the replacement-value path; the result is not required to be positive.

If the query returns zero:

- The supplied four-byte value remains the effective value.
- The auxiliary argument is not checked for zero at this stage.
- The condition, preparation, and replacement callbacks in the next section are skipped.

### 2. Optionally obtain a replacement value

If the initial query returns nonzero, the auxiliary argument is checked.

If the auxiliary argument is zero, the function returns −1 immediately. No subsequent callback or direct dependency in this wrapper is invoked. The initial query has already occurred and may already have produced side effects.

If the auxiliary argument is nonzero:

1. Dispatch F-indirect-02 through offset 0x64, passing the object.
2. If that callback returns zero, dispatch F-indirect-03 through offset 0x68, passing the object.
3. Dispatch F-indirect-04 through offset 0x50, passing the object and the auxiliary argument.
4. Use the four-byte result of F-indirect-04 as the effective value.

A nonzero result from F-indirect-02 skips F-indirect-03, regardless of sign. The result of F-indirect-03 is not used.

On this path, the caller-supplied value is replaced in the function’s local argument state. This does not establish any write to caller-owned storage represented by that value, nor does it establish release of the original value.

### 3. Obtain the result

Call F-a8f246f36c15 with:

- The object.
- The effective four-byte value selected above.

Retain its signed result for the remainder of the function and for the eventual return.

There is no visible validation of this result. Negative values continue into later callbacks.

F-a8f246f36c15 executes before the second object word is read. Any changes it makes to that word are therefore relevant to the later range decision.

### 4. Sample the upper bound

Read the object’s second represented integer word and subtract one. Retain the resulting signed value as the sampled upper bound.

This sampling occurs:

- After F-a8f246f36c15 returns.
- Before F-indirect-05 is invoked.

The upper bound is not reread after the following callback. If that callback changes the object’s second word, the range decision still uses the previously sampled upper bound.

Subtraction at the minimum representable signed integer is unresolved; see **Uncertainties**. Ordinary, non-overflowing cases have the behavior described here.

### 5. Issue the unconditional pre-range callback

Dispatch F-indirect-05 through offset 0xbc, passing:

- The object.
- Integer value 1.

This call occurs on every path that reaches F-a8f246f36c15 and continues normally. It does not depend on whether the result is negative or whether a valid range exists.

Its return value is ignored.

### 6. Conditionally issue the range callback

Dispatch F-indirect-06 through offset 0xb8 only when both signed conditions hold:

- The sampled upper bound is nonnegative.
- The retained result is less than or equal to that upper bound.

Pass:

- The object.
- The retained result as the first bound.
- The sampled upper bound as the second bound.
- Integer value 1.

There is no independent requirement that the first bound be nonnegative. A negative result can therefore be passed to this callback when the sampled upper bound is nonnegative.

The callback is skipped if the sampled upper bound is negative, even if the retained result is still smaller.

The callback’s return value is ignored.

The argument arrangement suggests a range operation, but endpoint inclusivity and the meaning of the final integer are not established by this wrapper.

### 7. Query again and propagate that result

Dispatch F-indirect-01 again through offset 0x60, passing the object.

This is a fresh query after the pre-range callback and any range callback. It is not a reuse of the initial query result and need not return the same value.

Pass its four-byte result to F-indirect-07, dispatched through offset 0x88, together with the object.

No Boolean normalization is visible. The returned value is forwarded as represented, rather than reduced to zero or one.

The return value of F-indirect-07 is ignored.

### 8. Issue the result-related callback

Dispatch F-indirect-08 through offset 0xa8, passing:

- The object.
- The retained result from F-a8f246f36c15.
- The original auxiliary argument.

This call uses the original auxiliary argument, not the effective four-byte value.

Its return value is ignored.

### 9. Finalize and return

Call F-488fdf51ede2 with the object and the retained result.

Its return value, if any, does not influence the wrapper’s return.

After that call returns normally, return the retained result from F-a8f246f36c15. Later object changes do not cause the result to be recomputed.

## Side effects

### Explicit wrapper effects

The visible wrapper:

- Reads object and dispatch-table state.
- Reads the second object word once, after F-a8f246f36c15.
- Replaces a local effective-value variable on one branch.
- Invokes dependencies in the order described above.

No direct write to object fields, global storage, or caller-owned buffers is visible.

### Dependency-mediated effects

All invoked dependencies may have side effects not shown here. The fragment does not establish that any dependency:

- Preserves global state.
- Preserves the object’s fields or dispatch-table pointer.
- Allocates or releases memory.
- Acquires or transfers ownership.
- Emits events or performs I/O.
- Avoids reentrant calls.

The dispatch-table expression is obtained from the object for each indirect call; the wrapper does not visibly retain one dispatch-table pointer for the entire sequence. Consequently, a dependency that changes the object’s dispatch-table pointer may affect later dispatches.

Use of the same dispatch offset at the beginning and near the end does not prove that both invocations reach the same implementation.

### Lifetime and failure exposure

The wrapper continues using the object after nearly every dependency call. If a dependency destroys the object or invalidates required dispatch storage, no visible recovery mechanism repairs that condition.

The early-refusal path is not necessarily side-effect-free: it occurs only after the initial query returns.

There is no visible rollback if a later dependency fails, faults, or does not return.

## Dependencies

The two direct dependencies have supplied opaque identities. Indirect identities are unresolved; the F-indirect labels below are document-local opaque references, not claims about resolved implementation identity.

| Dependency ID | Dispatch or linkage | Arguments and observed result use |
|---|---|---|
| F-a8f246f36c15 | Direct | Object and effective four-byte value; signed result retained, forwarded, and returned |
| F-488fdf51ede2 | Direct | Object and retained result; result ignored |
| F-indirect-01 | Table offset 0x60 | Object; first result controls branching, later result is forwarded |
| F-indirect-02 | Table offset 0x64 | Object; zero causes preparation callback |
| F-indirect-03 | Table offset 0x68 | Object; result ignored |
| F-indirect-04 | Table offset 0x50 | Object and auxiliary argument; returns replacement four-byte value |
| F-indirect-05 | Table offset 0xbc | Object and integer 1; result ignored |
| F-indirect-06 | Table offset 0xb8 | Object, retained result, sampled upper bound, integer 1; result ignored |
| F-indirect-07 | Table offset 0x88 | Object and fresh query result; result ignored |
| F-indirect-08 | Table offset 0xa8 | Object, retained result, original auxiliary argument; result ignored |

These entries describe only the call-site contract. They do not establish the complete signatures, ownership policies, or internal behavior of the dependencies.

## Acceptance examples

These are conditional contract examples derived from the supplied representation, not executed tests. Unless stated otherwise, dependencies return normally, preserve the storage needed for subsequent operations, and arithmetic does not overflow.

### Early refusal

Given a nonzero initial query result and a zero auxiliary argument:

- Return −1.
- Do not call F-indirect-02, F-indirect-04, either direct dependency, or any later callback.
- Do not read the second object word for the range decision.

Side effects of the initial query remain possible.

### Supplied value used despite zero auxiliary argument

Given an initial query result of zero and an auxiliary argument of zero:

- Pass the supplied value to F-a8f246f36c15.
- Continue through the normal post-result sequence.
- Pass zero as the auxiliary argument to F-indirect-08.

Zero is not rejected unconditionally.

### Replacement with conditional preparation

Given a nonzero initial query result, a nonzero auxiliary argument, and a zero result from F-indirect-02:

- Invoke F-indirect-03 before F-indirect-04.
- Pass the replacement returned by F-indirect-04 to F-a8f246f36c15.
- Do not pass the original supplied value to F-a8f246f36c15.

If F-indirect-02 instead returns any nonzero value, omit only F-indirect-03 from that sequence.

### Single-position candidate range

Given that F-a8f246f36c15 returns 4 and the sampled second object word is 5:

- The sampled upper bound is 4.
- Call F-indirect-05 first.
- Then call F-indirect-06 with bounds 4 and 4 and final integer 1.

Equality satisfies the range condition.

### Negative first bound

Given that F-a8f246f36c15 returns −2 and the sampled second object word is 3:

- The sampled upper bound is 2.
- Invoke F-indirect-06 with bounds −2 and 2.
- Continue through the remaining callbacks and return −2.

The wrapper contains no nonnegative-result guard.

### No range when the sampled upper bound is negative

Given a sampled second object word of zero:

- The upper bound is −1.
- Skip F-indirect-06, regardless of the retained result.
- Still invoke F-indirect-05 and the subsequent query, propagation, result-related, and finalization calls.

### Snapshot retained across mutation

Given that the sampled second object word is 6 and F-indirect-05 subsequently changes that word to 100:

- Use upper bound 5 for the pending range decision.
- Do not substitute 99.

A later dispatch may nevertheless use a changed dispatch-table pointer if the object was also modified.

### Fresh query forwarded without normalization

Given an initial query result of zero and a later query result of 7:

- Take the supplied-value path initially.
- Later pass 7, not 1 and not the earlier zero, to F-indirect-07.

### Completed sequence returning −1

Given an initial query result of zero and a result of −1 from F-a8f246f36c15:

- Do not take the early-refusal path.
- Perform the remaining sequence, including the conditional range callback if its signed conditions hold.
- Return −1 after finalization.

## Uncertainties

The following items are **documented_only**:

- **Native widths and ABI:** the exact machine-level integer and pointer widths, calling convention, and complete object layout are not independently supplied.
- **Overflow behavior:** subtracting one from the minimum signed integer may wrap under a machine interpretation or be invalid under a language-level interpretation. The fragment alone does not settle that boundary.
- **Semantic names:** the object type, effective value, auxiliary argument, second object word, and callback purposes are unknown.
- **Range semantics:** the bounds’ meaning, endpoint interpretation, and validity requirements belong to F-indirect-06 and are not established here.
- **Error meanings:** −1 is an explicit early-return value, but neither that value nor any other result has a demonstrated domain-specific error definition.
- **Dependency effects:** global mutation, ownership, allocation, release, reentrancy, exceptions, and nonlocal exits are unspecified.
- **Concurrency:** no synchronization or atomicity guarantee is visible.
- **Indirect identity:** dispatch offsets are established; the implementations occupying those offsets at each call are not.

## Evidence

The established evidence is limited to static inspection of the supplied function representation:

- A zero/nonzero initial dispatch result selects the effective-value path.
- The early −1 return requires both a nonzero initial query and a zero auxiliary argument.
- F-a8f246f36c15 precedes the second-word read.
- The upper bound is sampled before the unconditional callback.
- The range predicate uses signed comparisons and does not reject a negative first bound.
- A fresh query result is forwarded to another callback.
- F-488fdf51ede2 is invoked before the retained result is returned.
- No explicit object-field write, allocation, deallocation, or rollback appears in the wrapper.

Semantic interpretations are identified as inferred intent. Unresolved behavior is documented rather than supplied by assumption. No runtime tests have been performed.
