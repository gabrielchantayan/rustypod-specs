<!-- Trusted provenance: F-dfd8935af90e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dfd8935af90e

## Purpose

F-dfd8935af90e scans values obtained from an input dependency, forwards accepted values to an output dependency, and stops when it encounters a configured delimiter outside the applicable quotation and escape conditions.

The function reports success only through its delimiter-recognition path. It reports failure when input readiness fails, decoding reports failure, the decoded low-order 16 bits are zero, or the output object's observed size-like field exceeds 1,024 after an ordinary forwarding call.

Quotation and escape handling affect delimiter recognition; they do not visibly remove quotation marks or escape markers from the forwarded sequence. The terminating delimiter also causes an output-dependency call before success is returned.

**Evidence classification:** These are static observations of the supplied representation. Describing the operation as scanning, decoding, and forwarding expresses inferred intent; the concrete dependency effects are not established. No runtime tests have been performed.

## Interface

The function accepts four inputs and returns a 32-bit result.

| Input | Visible representation | Required interpretation |
|---|---|---|
| Context | Pointer to storage accessed at byte offsets 0, 4, and 6 | Supplies an input handle, a delimiter, and a quotation-enable field. |
| Output reference | A 32-bit value used both as a dependency argument and as an address base | Identifies output-related state, including a 32-bit field at byte offset 4. |
| Unused input | 32-bit value | No visible read or forwarding occurs. |
| Initial decoder state | Unsigned 32-bit value | Copied into mutable per-invocation storage passed by address to the decoding dependency. |

The context layout visible to this function is:

| Byte offset | Width | Use |
|---|---|---|
| 0 | 32 bits | Input handle passed to input-related dependencies. |
| 4 | 16 bits, unsigned | Delimiter used in equality comparisons. |
| 6 | 8 bits | Quotation processing is enabled when this field is nonzero. |

These fields need not describe the complete object. No access to byte offset 7 or to a larger context layout is established.

The output-related layout visible to this function contains a 32-bit unsigned field at byte offset 4. The function compares this field with 1,024 after ordinary forwarding calls. Calling it a length or count is an inference; only its location, width, and use in the comparison are established.

### Return values

- **1:** A decoded value matched the current delimiter on the delimiter-eligible path, and the output dependency returned from the corresponding call.
- **0:** An input-related dependency reported failure, the decoded value was zero, or the post-forwarding output field exceeded 1,024.

There is no visible distinct result for end of input, malformed input, output overflow, or another dependency-reported failure.

### Preconditions

Execution requires that:

- The context supports the reads actually reached.
- The output reference supports the 32-bit read at byte offset 4 whenever an ordinary forwarding call returns.
- Each dependency can accept the arguments supplied on its call path.
- Storage passed by address to the decoder remains usable for the duration of that call.

No null check, bounds check on the supplied objects, alignment repair, or dependency-error recovery is visible. Invalid storage or incompatible dependency behavior is outside the established contract.

## Behavior

### Invocation state

Two internal flags begin clear:

- **Quoted-region flag:** Indicates whether delimiter recognition is currently suppressed by a quoted region.
- **Pending-escape flag:** Suppresses delimiter recognition on an applicable subsequent value.

A separate 32-bit decoder-state slot begins with the initial decoder-state input. The same slot is reused throughout the invocation. The decoder may modify it on each call.

None of these values is visibly written back into the context as persistent scanner state.

### Input acquisition and value selection

Each iteration begins with a 32-bit read of the context's input handle and a call to F-0e9d47e782bc.

If that dependency returns zero, the function immediately returns zero. That iteration performs no decoder call and no output call.

If the readiness result is nonzero, the function reads the input handle from the context again and calls F-51fbe285e59c with:

- The newly read handle.
- The address of the mutable 32-bit decoder-state slot.

The two handle reads are separate observable accesses. The function does not establish that they yield the same value.

The decoder's visible result is 64 bits. The low-order 32-bit portion determines success: zero causes an immediate zero return; any nonzero bit pattern permits processing to continue. A nonzero value with its top bit set is not rejected merely for appearing negative under a signed interpretation.

For a successful decoder result, the value used by the scanner is the unsigned low-order 16 bits of the decoder-state slot. The upper 16 bits of that slot do not participate in quotation, escape, or delimiter comparisons.

The high-order 32 bits of the decoder's return value do not affect any successful scanning decision. Although the representation extracts them, they are superseded by the low-order 16 bits of the decoder-state slot on success. On decoder failure, the function returns zero regardless of those high bits.

If the selected 16-bit value is zero, the function returns zero before consulting quotation configuration, comparing a delimiter, or calling the output dependency for that iteration.

### Quoted-region handling

When the quoted-region flag is already set:

- A value of 34, the double-quotation-mark value, clears the flag.
- Every other nonzero value leaves it set.
- No delimiter comparison occurs.
- The quotation-enable field is not consulted.
- The pending-escape flag is not changed.

The current value then follows the ordinary forwarding path, including when it closes the quoted region.

Consequently, disabling quotation processing in the context while already inside a quoted region does not itself exit that region. A value of 34 still closes it.

When the quoted-region flag is clear, the function reads the context's quotation-enable byte. If it is nonzero and the current value is 34, the function enters a quoted region and proceeds directly to ordinary forwarding.

This opening-quotation case takes precedence over both escape processing and delimiter comparison. In particular:

- A configured delimiter of 34 is not recognized while quotation handling is enabled.
- A pending escape does not prevent a value of 34 from opening a quoted region.
- Opening a quoted region does not clear a pending escape.

### Escape handling and delimiter recognition

Escape and delimiter handling apply only outside a quoted region and when the current value has not just opened one.

If the pending-escape flag is clear:

- A value of 92, the backslash value, sets the pending-escape flag and proceeds to ordinary forwarding without comparing the delimiter.
- Any other value is compared with a fresh 16-bit read of the context's delimiter.

If that comparison matches, the function makes the terminal output call described below and returns one. It does not perform the ordinary post-forwarding size check.

If the pending-escape flag was already set when this handling is reached, the function skips both backslash recognition and delimiter comparison. It clears the pending-escape flag and ordinarily forwards the current value.

A nonmatching, non-backslash value also leaves this handling with the pending-escape flag clear.

This is not an unconditional “escape exactly the next decoded value” rule. An intervening quoted region can preserve the pending escape: opening a quote, processing its contents, and closing it do not clear that flag. It is cleared when a subsequent value reaches the outside-quotation escape-handling path.

### Output calls and stopping condition

F-a03b19bdffef is called on two visibly different paths:

1. **Terminal delimiter path:** The supplied representation shows the output reference and a second argument equal to the decoder-state slot's low-order 16 bits. The function then returns one.
2. **Ordinary forwarding path:** The supplied representation shows only the output reference. The function then reads the unsigned 32-bit field at output byte offset 4.

For the ordinary path:

- A field value greater than 1,024 causes a zero return.
- A field value equal to 1,024 is allowed.
- Any smaller value is allowed.
- An allowed value leads to the next iteration.

The check occurs after the dependency call, not before it. The function does not prevent that call from producing state beyond the threshold, nor does it visibly undo the call's effects.

The differing visible argument lists prevent a complete output-content contract. In particular, the ordinary path cannot safely be specified as passing the decoded value in a second argument. See **Uncertainties**.

## Side effects

### Observable access order

For an iteration that reaches ordinary forwarding, the established order is:

1. Read the context input handle.
2. Call F-0e9d47e782bc.
3. If its result is nonzero, reread the context input handle.
4. Call F-51fbe285e59c with that handle and the decoder-state address.
5. Reject decoder failure or a zero selected value.
6. Perform applicable quotation-enable and delimiter reads, with those reads omitted on paths that do not need them.
7. Call F-a03b19bdffef.
8. Read the output field at byte offset 4.
9. Either return zero for a value above 1,024 or start another iteration.

On delimiter success, step 7 uses the terminal call form, and steps 8 and 9 are replaced by an immediate return of one.

The function does not visibly cache the context's handle, quotation-enable byte, delimiter, or output field across iterations. Their values may therefore affect later decisions if changed externally or by dependencies. No assumption that dependencies preserve global or shared state is justified.

### Mutation, ownership, and failure

The function visibly writes its own internal flags and decoder-state slot. It makes no direct store into the context or output object in the supplied representation.

Dependency calls may have additional effects, including input consumption or output modification, but their exact effects are not supplied. The function visibly ignores any result from F-a03b19bdffef and does not branch on an output-dependency status.

No allocation, deallocation, ownership transfer, cleanup, or rollback is visible in the caller. That does not establish that dependencies allocate nothing or release nothing.

A zero return does not imply that no work occurred. Earlier iterations may already have invoked the output dependency, and the threshold failure occurs only after such an invocation.

## Dependencies

| Opaque ID | Established caller-visible use | Not established |
|---|---|---|
| F-0e9d47e782bc | Receives a freshly read 32-bit handle. A zero result terminates the function with zero; a nonzero result permits decoding. | Whether it peeks, consumes input, blocks, changes the context, or distinguishes failure kinds. |
| F-51fbe285e59c | Receives a freshly reread handle and the address of mutable 32-bit state. Supplies a 64-bit result whose low-order 32 bits are used as a zero/nonzero status. | Encoding, input consumption, detailed state semantics, lifetime guarantees, and other side effects. |
| F-a03b19bdffef | Called once for each value reaching ordinary forwarding and once on a recognized delimiter. Ordinary calls are followed by an output-field check; terminal calls are not. | Complete calling convention, ordinary-call value source, exact output bytes or units, ownership effects, and error reporting. |

These are behavioral roles assigned from call-site evidence, not verified dependency implementations.

## Acceptance examples

The following are conditional static expectations, not executed tests. Unless stated otherwise, assume readiness and decoding succeed, dependencies return normally, context fields remain stable, and the post-ordinary-call output field stays at or below 1,024.

1. **Immediate readiness failure.**  
   If the first readiness call returns zero, the result is zero. No decoder or output call occurs.

2. **Decoder failure despite populated state.**  
   If the decoder writes a nonzero low-order 16-bit value but returns a result whose low-order 32 bits are zero, the result is zero with no output call for that iteration.

3. **Zero selected value.**  
   If decoding reports success but the decoder-state slot's low-order 16 bits are zero, the function returns zero. A configured zero delimiter does not turn this case into success.

4. **Immediate delimiter.**  
   With quotation handling disabled, no pending escape, and a nonzero delimiter other than 92, a matching first value causes the terminal output call and a result of one. The output field is not checked on that path.

5. **Quoted delimiter.**  
   With quotation handling enabled and delimiter 44, the sequence 34, 44, 34, 44 ordinarily forwards the first three values. The final 44 takes the terminal path. The internal 44 is not compared with the delimiter.

6. **Escaped delimiter.**  
   With delimiter 44 and quotation handling disabled, the sequence 92, 44, 44 ordinarily forwards the first two values. The second 44 terminates successfully.

7. **Escape survives quotation.**  
   With quotation handling enabled and delimiter 44, the sequence 92, 34, 34, 44, 44 ordinarily forwards the first four values. The quotation pair preserves the pending escape, so the first 44 clears it without delimiter recognition. The final 44 terminates successfully.

8. **Backslash configured as delimiter.**  
   A value of 92 never reaches the delimiter comparison with the pending-escape flag clear, because it first sets that flag. With the flag set, delimiter comparison is skipped. Thus a delimiter of 92 cannot produce success through the visible logic.

9. **Quotation mark configured as delimiter.**  
   With quotation handling enabled, 34 opens or closes a quoted region rather than matching the delimiter. With quotation handling disabled and no pending escape, 34 may match the delimiter.

10. **Threshold boundary.**  
    An ordinary output call followed by a field value of 1,024 permits another iteration. A field value of 1,025 returns zero after that call. A terminal delimiter call returns one without testing either value.

## Uncertainties

The following items are **documented_only**:

- The true signature and calling convention of F-a03b19bdffef. Its ordinary and terminal call forms have different visible argument counts.
- The exact data emitted or retained by an ordinary output call.
- Whether the output field measures bytes, characters, capacity, length, or another quantity.
- The meaning of the initial decoder state and its upper 16 bits.
- The character encoding, byte order of larger external data, and interpretation of values other than the numeric comparisons established above.
- Dependency behavior involving shared state, blocking, exceptions, nonlocal exits, memory ownership, or retained pointers.
- Whether the visible 32-bit address arithmetic faithfully represents all intended platform constraints.

A nonterminating execution is possible if dependencies continue succeeding, no value reaches a successful delimiter comparison, and the observed output field never exceeds 1,024. No separate iteration limit is visible.

## Evidence

The supplied static representation establishes the input widths and accessed offsets, two initially clear flags, repeated handle reads, dependency-call ordering, zero-result exits, 16-bit value selection, quotation and escape precedence, terminal success path, and unsigned post-forwarding threshold comparison.

Scanner terminology and the characterization of dependency roles are inferred intent. Dependency internals and unresolved output-call semantics remain unknown.

No runtime execution, tests, independent source inspection, or provenance verification has been performed.
