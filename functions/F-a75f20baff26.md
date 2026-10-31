<!-- Trusted provenance: F-a75f20baff26; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a75f20baff26

## Purpose

F-a75f20baff26 selects a return value through two ordered comparison gates and a final field-based fallback.

The first gate uses a value obtained independently of the input. A zero comparison result selects one shared return value immediately. Otherwise, a second gate uses a value obtained from the input’s region at byte offset 40. A zero result from that comparison selects a different shared return value. If neither gate selects a result, the function consults a field at byte offset 44 and may substitute a third shared value.

This is a source-facing contract derived from the supplied static representation. It does not establish that either comparison implements string equality, that the returned values are pointers, or that the input is a particular kind of object.

## Interface

### Input

The function takes one argument, represented as an integer and used as an address base.

Two offsets are relevant:

| Byte offset from input | Use |
|---|---|
| 40 | The computed address is passed to F-44362605ab79. This function does not explicitly read a field at that offset before the call. |
| 44 | A signed integer field is explicitly read on the final selection path. The supplied representation contains two reads of this location. |

The argument is not validated before address arithmetic or dependency calls.

For executions that reach the final selection path, the location at offset 44 must support the integer accesses required by the target platform. Any additional validity, alignment, lifetime, or accessible-size requirements imposed by F-44362605ab79 are unknown.

A complete input structure layout is not established. In particular, passing the address at offset 40 does not establish a four-byte field there, nor does it establish that the region accessed by F-44362605ab79 ends before offset 44.

### Output

The function returns an integer-valued result selected from:

- The shared return value associated with the first gate.
- The shared return value associated with the second gate.
- The first value read from the field at offset 44.
- A shared fallback value used when the second field read is zero.

No direct input value is returned merely because the input is zero or invalid. There is no established distinguished error result.

The meaning of the result is unknown. It may encode a scalar, handle, address, or another target-specific value. No ownership transfer is established.

### Width and arithmetic model

The supplied representation explicitly identifies the intermediate values returned by F-3ad65987df9a and F-44362605ab79 as four-byte quantities. It represents the function argument, comparison results, field accesses, and return value using an integer type.

A 32-bit integer and address model is consistent with that representation, but a separate target ABI declaration is not supplied. Consequently:

- Under a 32-bit integer ABI, the field reads and returned integer values are 32 bits wide.
- The offsets 40 and 44 are byte offsets, not counts of integers.
- Zero tests are exact comparisons with zero; they are not sign tests.
- Negative nonzero comparison results follow the same branch as positive nonzero results.
- No widening, saturation, bounds checking, or overflow handling is shown.

Behavior for input values whose offset calculations overflow the applicable arithmetic model is **documented_only**. The representation alone does not justify promising either modular address arithmetic or defined signed-overflow behavior.

## Behavior

### First gate: input-independent selection

The function first calls F-3ad65987df9a with no explicit arguments.

Its four-byte result becomes the first operand of an unmapped comparison dependency. The second operand refers to fixed named data associated with the marker `CntrlHistoryFn`.

The function tests the comparison dependency’s integer result:

- If it is zero, the function reads and returns the first-gate shared return value.
- If it is nonzero, execution proceeds to the second gate.

The zero-result path performs no explicit address calculation involving the input, no call to F-44362605ab79, and no explicit read of the field at offset 44.

This precedence is part of the contract. A first-gate selection cannot be overridden by the input field or by the second gate.

The named marker suggests a history-related purpose, but that is inferred intent rather than an established interpretation of the comparison or returned value.

### Second gate: selection using the offset-40 region

This gate is reached only after a nonzero result from the first comparison.

The function computes the input address plus 40 bytes and passes that computed address as the sole explicit argument to F-44362605ab79. It does not first load the contents of that address as an integer argument.

The four-byte result from F-44362605ab79 becomes the first operand of the same unmapped comparison dependency. The second operand is obtained from a shared comparison-operand location.

The ordering matters: F-44362605ab79 runs before the shared comparison operand is fetched for this call. If that dependency changes the shared operand, the subsequent comparison can observe the changed value. The contract does not assume that dependency preserves globals.

The second comparison result controls selection:

- If it is zero, the function reads and returns the second-gate shared return value.
- If it is nonzero, execution proceeds to the final field-based selection.

The second-gate shared return value is fetched after the second comparison completes. It must not be treated as an entry-time snapshot: the preceding calls may have changed it.

No explicit offset-44 field read occurs on the second-gate zero-result path.

### Final selection: field and fallback

This path is reached only when both comparison results are nonzero.

The supplied representation contains two reads from the integer field at input offset 44:

1. The first read supplies a candidate return value.
2. The second read determines whether the fallback is selected.

If the second read is zero, the function reads and returns the shared fallback value. If the second read is nonzero, it returns the value obtained by the first read.

For stable ordinary memory, this produces the simpler observable rule:

- A zero field selects the shared fallback.
- A nonzero field is returned unchanged.

That simplification depends on the two reads observing the same value. The represented behavior does not use the second read as the returned field value.

If a supported memory model permits the reads to differ:

- A first read of zero followed by a nonzero second read returns zero, without selecting the fallback.
- A nonzero first read followed by a zero second read returns the shared fallback.
- Two different nonzero reads cause the first value to be returned.

These distinctions describe the supplied representation. They do not establish machine-level access counts or make concurrent unsynchronized mutation valid. No volatile qualification, atomic access, synchronization, or memory-mapped I/O contract is supplied.

### Short-circuiting and errors

The selection order is strict: first gate, second gate, final field selection.

Only the selected shared return location is explicitly read as a result source. For example, a nonzero field on the final path does not require fetching the fallback value.

There is no explicit error handling around either dependency. The function does not:

- Check whether a dependency result is a valid address or handle.
- Reject a zero input before reaching an input-dependent path.
- Validate the offset-44 field.
- Catch or translate dependency failures.
- Produce a separately identified failure status.

If a dependency does not return normally, no subsequent normal return from this function is established. Invalid address accesses have no defined recovery behavior in the supplied evidence.

## Side effects

### Explicit effects

No explicit stores are shown.

The function explicitly performs dependency calls and conditional reads of shared data and input-relative memory. Its selected return value is the only directly represented outward result.

There is no explicit allocation, release, ownership handoff, reference-count adjustment, or initialization.

### Dependency-mediated effects

The absence of explicit stores does not establish purity.

Any invoked dependency may read or change shared state, mutate reachable input memory, allocate resources, release resources, or fail to return. None of those effects can be ruled out from the supplied material.

In particular:

- F-3ad65987df9a runs before the first comparison.
- The first comparison runs before the first-gate return value is fetched.
- F-44362605ab79 runs before the second comparison operand is fetched.
- The second comparison runs before either the second-gate return value or the final field is fetched.

Implementations must not assume that shared values or the input field retain their entry-time contents across those calls.

### Ownership and lifetime

The function contains no explicit copying or release of the four-byte values returned by its dependencies. It passes those values to the comparison dependency and makes no further explicit use of them afterward.

That does not establish whether those values are borrowed, owned, scalar, or pointer-like. Nor does it establish whether the comparison dependency retains or releases anything.

The input must remain suitable for the operations actually reached. Whether any dependency retains the offset-40 address beyond its call is **documented_only**.

## Dependencies

| Dependency | Established interaction | Unknown contract |
|---|---|---|
| F-3ad65987df9a | Called once at entry with no explicit arguments; its four-byte result supplies the first comparison operand. | Result interpretation, validity, ownership, global effects, and failure behavior. |
| F-44362605ab79 | Called once only after a nonzero first comparison; receives the input address plus 40 bytes; its four-byte result supplies the next comparison operand. | Required region layout, accessed extent, mutation, retention, result interpretation, ownership, and failure behavior. |

One additional comparison dependency appears twice. No opaque function ID for it was supplied. It therefore remains an unmapped dependency rather than being assigned a fabricated ID.

For that dependency, the established contract is limited to two operands, an integer result, and the caller’s distinction between zero and nonzero. Equality, ordering, string termination, encoding, null handling, operand mutation, and resource handling remain unknown.

The other data dependencies are:

- Fixed named data used by the first comparison.
- A shared operand used by the second comparison.
- Separate shared return values for the first and second gates.
- A shared fallback value.
- The input-relative integer field at offset 44.

No invariance, immutability, or relationship among these shared values is established.

## Acceptance examples

These are conditional acceptance cases, not executed tests. They assume normal dependency returns and valid accesses for the paths reached.

1. **First gate selects immediately.**  
   Given a zero first comparison result, return the first-gate shared value as read after that comparison. Do not call F-44362605ab79 or explicitly read the offset-44 field.

2. **Second gate selects.**  
   Given a nonzero first comparison result and a zero second result, call F-44362605ab79 with the input address plus 40 bytes, then return the second-gate shared value. Do not explicitly read the offset-44 field.

3. **Stable nonzero field.**  
   Given two nonzero comparison results and both field reads yielding 73, return 73. Do not fetch the shared fallback solely for this selection.

4. **Stable zero field.**  
   Given two nonzero comparison results and both field reads yielding zero, return the shared fallback value. If that fallback is itself zero, return zero; there is no further substitution.

5. **Negative comparison results.**  
   A comparison result of −1 follows the nonzero branch. It is not treated as a separate error or selection category.

6. **Negative field value.**  
   Under the 32-bit signed-integer model, a stable field value of −7 is returned as −7 after both gates fail to select. No positivity check is shown.

7. **Shared state changes during a call.**  
   If the second comparison changes its associated shared return value before returning zero, the function reads the resulting shared value afterward. Using a value cached before the comparison would not preserve the represented order.

8. **Distinct field observations, where supported.**  
   If the first field read yields 12 and the second yields 19, return 12. If the first yields 12 and the second yields zero, select the fallback. These cases do not authorize otherwise-invalid concurrent access.

9. **Input bypass.**  
   A zero first comparison result bypasses all explicit input-relative operations shown here. This does not establish that arbitrary input values are safe on paths reaching the second gate.

## Uncertainties

The following remain **documented_only**:

- The exact target ABI, integer width, pointer width, and address-overflow semantics.
- The input’s full layout and allocation extent.
- The identity and complete contract of the unmapped comparison dependency.
- Whether comparison operands are strings, addresses, handles, or other values.
- The contents and semantic meanings of the shared return values.
- Whether dependencies mutate or retain input-derived memory.
- Ownership and lifetime rules for dependency results and returned values.
- Alignment, concurrency, atomicity, and device-memory guarantees.
- Whether the two represented field reads correspond to two physical memory accesses.
- Behavior after invalid memory access or non-normal dependency completion.

The history-related interpretation suggested by the named marker is inferred intent only. The established operation is ordered selection based on zero versus nonzero comparison results.

## Evidence

The evidence consists solely of the supplied static function representation and dependency mapping.

That evidence supports the call order, byte offsets, branch precedence, zero-result decisions, conditional shared reads, two represented field reads, and absence of explicit stores or resource-management operations.

No dependency implementations, target ABI declaration, memory layout declaration, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are consequences of the represented control flow under their stated preconditions, not observations of an executing implementation.
