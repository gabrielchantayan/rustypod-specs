<!-- Trusted provenance: F-de3e63dee52d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de3e63dee52d

## Purpose

F-de3e63dee52d coordinates discovery, admission, and recording of variable-length units within a requested span. It maintains a cumulative consumption measure, writes 12-byte descriptors through a descending descriptor cursor, and associates recorded units with a progression value derived from context fields.

A zero mode permits recording without the explicit requested-span fit checks used by a nonzero mode. A nonzero mode can report a terminal condition when the requested span is reached, crossed, or otherwise terminated by particular dependency outcomes.

The return value is not a general success indicator. Descriptor writes and consumption updates can occur on a zero return; a return of one does not imply that any descriptor was written.

These behaviors are established by the supplied static representation. The interpretation of the units, progression value, and dependency statuses remains unknown.

## Interface

The function accepts five arguments:

| Argument | Width and interpretation |
|---|---|
| Context | A 32-bit address identifying readable and writable state. |
| Initial position | A 32-bit integer used as the initial candidate position. |
| Requested span | An unsigned 32-bit quantity. |
| Descriptor-count output | A writable address receiving a 32-bit count on every normal return. |
| Mode | A 32-bit value tested only for zero versus nonzero. |

The return is a 32-bit value, explicitly initialized to zero and changed only to one.

### Context layout accessed directly

Offsets below are byte offsets from the context address.

| Offset | Width | Direct use |
|---|---:|---|
| `0x18` | 32 bits | Input to F-1e0edc6e0bc9. |
| `0x20` | 32 bits | Existing cumulative state; subtracted when establishing a marker target and incremented at exit. |
| `0x24` | 32 bits | Unsigned multiplier in a two-word arithmetic calculation. |
| `0x28` | 32 bits | Both a numeric bound in position arithmetic and an address through which descriptors are written; also decremented by 12 on selected paths. |
| `0x2c` | 32 bits | Initial progression value captured for descriptor generation. |
| `0x48` | 32 bits | Unsigned multiplicand contributing the low part of a two-word calculation. |
| `0x4c` | 32 bits | Multiplicand contributing to that calculation’s high word. |
| `0x624` | 32 bits | Used with `0x20` to establish a marker target. |
| `0x628` | 32 bits | Conditionally overwritten with the current progression value. |

A descriptor consists of three consecutive 32-bit fields:

| Relative offset | Value |
|---|---|
| `0` | Current candidate position |
| `4` | Current candidate length |
| `8` | Current progression value |

Descriptor field writes occur in that order. The context field at `0x28` is read separately for each field write. A descriptor write therefore must not be modeled as a single atomic 12-byte write through a permanently captured address.

### Preconditions and arithmetic domain

The function performs no explicit null, range, alignment, capacity, or overflow validation. All directly accessed context fields, dependency outputs, descriptor destinations, and the count output must be valid for their actual accesses.

Arithmetic involving positions, lengths, consumption, progression, and cursor movement uses 32-bit quantities in the supplied representation. This contract describes wrapping arithmetic as arithmetic modulo \(2^{32}\), while preserving the representation’s explicit signed and unsigned comparisons. An implementation in a language with different overflow rules must not silently substitute checked arithmetic or unbounded integers.

The supplied representation does not establish the original language’s signed-overflow guarantees.

## Behavior

### Initial scan

The function begins with:

- Consumption equal to zero.
- Descriptor count equal to zero.
- Return value equal to zero.
- Candidate position equal to the initial position.

It computes an unsigned availability value by subtracting the initial position from the context field at `0x28`.

The initial scan budget is asymmetric:

- If unsigned availability is less than the requested span, the budget is availability minus three. This branch does not clamp underflow.
- Otherwise, the budget is requested span minus three. If that result, interpreted as signed 32-bit, is negative, the budget becomes zero.

Consequently, this is not universally equivalent to a nonnegative minimum minus three. For example, availability of two in the first branch produces the bit pattern `0xffffffff`.

F-50643035a312 receives the context, writable candidate-position and candidate-length outputs, this budget, and the consumption output.

If that dependency returns nonzero:

- No progression-step dependency or candidate-processing loop is entered.
- No descriptor is written directly by this function.
- The count remains zero.
- A nonzero mode selects return value one; zero mode leaves it zero.
- The common exit adds the dependency-provided consumption to context state and writes the count output.

A dependency’s nonzero result is a branch condition here, not an established semantic error code.

### Progression setup

After an initial zero result from F-50643035a312, the function prepares a two-word value:

- The low word is the low 32 bits of the unsigned product of fields `0x48` and `0x24`.
- The high word is the high 32 bits of that unsigned product, plus the low 32 bits of the product of fields `0x24` and `0x4c`, modulo \(2^{32}\).

F-1e0edc6e0bc9 receives the low word, high word, the field at `0x18`, and zero. Its 32-bit return becomes the progression increment.

The initial progression value is read from `0x2c` before that dependency call. After the call, the function captures a marker target as the field at `0x624` minus the then-current field at `0x20`.

This timing matters: the marker target uses state read after F-1e0edc6e0bc9. No preservation of context or other mutable state by that dependency is established.

### Candidate admission

Before each candidate is processed, F-8b94282ebf76 receives:

1. Context.
2. Current candidate position.
3. Current candidate length.
4. Current consumption.
5. Original requested span.
6. Original mode.

A zero result ends processing with return value zero and the descriptor count already accumulated.

A nonzero result allows F-670e5b39a6f4 to run. That dependency receives the context, current candidate position and length, and two writable outputs: a next-position value and a continuation value.

Its zero and nonzero outcomes select different progression paths.

### Nonzero result from F-670e5b39a6f4

This path records the current candidate, when permitted, and seeks the next candidate by scanning.

In zero mode, there is no requested-span fit check before recording.

In nonzero mode, the function compares the requested span against consumption plus candidate length using unsigned comparison and a wrapping 32-bit sum:

- If the candidate would cross the requested span, no descriptor for that candidate is written. Consumption becomes exactly the requested span, and the function returns one through the common exit.
- Otherwise, the candidate is recorded.

Before recording, if current consumption equals the captured marker target, the current progression value is written to `0x628`.

Recording writes the three descriptor fields and increases the reported descriptor count by one. Consumption then increases by the candidate length.

In nonzero mode, if this updated consumption equals the requested span, the function returns one immediately. On this path:

- The descriptor cursor is not decremented.
- The progression value is not advanced.
- Candidate position is not advanced.
- No further scan occurs.

Otherwise, candidate position advances by candidate length, and the function prepares another scan. The scan’s additional-consumption output is initialized to zero.

For this subsequent scan, the function computes:

- Remaining span: requested span minus consumption.
- Availability: the current field at `0x28` minus the advanced candidate position.

These quantities are compared as signed 32-bit values:

- If availability is signed-less than remaining span, the budget is availability minus three, without a clamp.
- Otherwise, the budget is remaining span minus three, clamped to zero if its signed interpretation is negative.

The descriptor cursor has not yet been decremented when this availability is read.

If the scan returns nonzero, its additional consumption is added to current consumption:

- In zero mode, processing ends with return value zero.
- In nonzero mode, processing ends with return value one if the pre-scan remaining span was signed-less-than-or-equal to the pre-scan availability. Otherwise, the return remains zero.

These nonzero-scan exits retain the descriptor just written and its count, but do not decrement the cursor or advance the progression value.

If the scan returns zero:

1. The progression value increases by the progression increment.
2. Additional scan consumption is added.
3. The current context field at `0x28` is decremented by 12.
4. The incremented descriptor count becomes the count carried into the next admission call.

### Zero result from F-670e5b39a6f4

This path uses the dependency’s continuation outputs rather than scanning directly for the next candidate.

In nonzero mode, the function first compares requested span with consumption plus candidate length, using unsigned comparison and a wrapping sum.

If that sum is greater than or equal to the requested span:

- A strictly greater sum produces no descriptor for this candidate.
- An equal sum writes the descriptor and increases the count.
- Both cases set consumption to the requested span and return one.
- Neither case performs the marker write, advances progression, decrements the cursor, or invokes the continuation dependency.

The missing marker update in the equality case is observable and differs from the nonzero-classification path.

For zero mode, or for a nonzero-mode candidate that remains strictly below the requested span:

1. If consumption equals the marker target, `0x628` receives the current progression value.
2. The descriptor is written and the count increases.
3. The progression value increases by its increment.
4. The current field at `0x28` is decremented by 12.
5. Consumption increases by candidate length.
6. Candidate position is replaced by the next-position output from F-670e5b39a6f4.
7. F-572311e78cc8 receives the context, the continuation value, a four-byte scratch output, and a writable candidate-length output.

If F-572311e78cc8 returns zero, admission repeats using the replacement position and updated length. A nonzero result ends processing with return value zero, retaining all preceding writes, count changes, and consumption changes.

The scratch output is not subsequently read by this function.

### Common exit

Every normal return performs these final writes in order:

1. Read the then-current field at `0x20`, add consumption, and write the result back to `0x20`.
2. Write the descriptor count to the caller’s count output.
3. Return zero or one as selected above.

The final cumulative update is not based on a saved entry value. Dependency changes to `0x20` can therefore affect the final result.

If the count output aliases context or descriptor memory, its final write can overwrite earlier state. No non-aliasing guarantee is established.

## Side effects

Direct effects are limited to:

- Writes of descriptor fields through the address held at `0x28`.
- Selected decrements of the field at `0x28` by 12.
- Conditional writes to `0x628`.
- The final increment of `0x20`.
- The final descriptor-count output.

Dependency effects are additional and unknown.

There is no direct allocation, release, or ownership-transfer operation in the supplied function. This does not establish that dependencies allocate nothing, release nothing, preserve globals, or preserve context fields.

Descriptor count and cursor movement are not interchangeable measures. Several terminal paths increase the count without decrementing the cursor.

No rollback is performed. A later admission failure, scan termination, or continuation termination preserves preceding direct writes.

The function has no explicit iteration bound. Termination depends on dependency outcomes, arithmetic state, and mode-dependent terminal comparisons.

## Dependencies

| Opaque ID | Inputs and outputs used here | Result interpretation here |
|---|---|---|
| F-50643035a312 | Context; writable position and length; 32-bit scan budget; writable consumption contribution | Zero permits continued processing; nonzero selects a termination path. |
| F-1e0edc6e0bc9 | Two words of a calculated value; context field at `0x18`; zero | Return bits become the progression increment. Arithmetic meaning is unknown. |
| F-8b94282ebf76 | Context, position, length, consumption, requested span, mode | Zero stops candidate processing; nonzero permits classification. |
| F-670e5b39a6f4 | Context, position, length; writable next-position and continuation outputs | Nonzero selects rescanning; zero selects continuation handling. |
| F-572311e78cc8 | Context, continuation value, writable four-byte scratch, writable length | Zero permits another admission iteration; nonzero terminates. |

Output validity on each dependency result is an external requirement wherever this function consumes that output. The supplied material does not establish the dependencies’ complete contracts.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless otherwise stated, assume valid storage, no dependency mutation of unrelated context fields, and no arithmetic wrap.

1. **Initial scan terminates in zero mode.**  
   If the first scan returns nonzero and reports consumption seven, the function writes count zero, adds seven to `0x20`, and returns zero. No later dependency runs.

2. **Initial scan terminates in nonzero mode.**  
   With the same scan outcome and any nonzero mode, the final consumption and count writes are unchanged, but the return is one.

3. **Admission rejects the first candidate.**  
   If the initial scan returns zero and admission returns zero, classification does not run. The count is zero, return is zero, and the scan-provided consumption is still added at exit.

4. **Exact fit on the rescanning path.**  
   Let requested span be 20, consumption eight, candidate length 12, mode nonzero, and classification nonzero. The descriptor is written and counted. Consumption becomes 20 and return is one. If the marker target is eight, `0x628` is written before the descriptor. The cursor is not decremented.

5. **Exact fit on the continuation path.**  
   Use the preceding values but classification zero. The descriptor is written and counted, and return is one with consumption 20. The marker is not updated, even if its target is eight. No continuation dependency runs, and the cursor is not decremented.

6. **Candidate crosses the requested span.**  
   With requested span 20, consumption eight, length 13, and nonzero mode, either classification outcome reaches a return of one with consumption forced to 20. The current candidate is not written or counted.

7. **Post-record scan terminates before cursor movement.**  
   Suppose a descriptor is recorded on the rescanning path, its following scan returns nonzero, and that scan reports two additional units consumed. The descriptor remains counted and the two units are included in final consumption. The direct cursor decrement and progression advance for that iteration do not occur. In nonzero mode, the return depends on the saved signed comparison of remaining span and availability.

8. **Continuation terminates after committed state changes.**  
   If a fitting candidate takes the continuation path and F-572311e78cc8 returns nonzero, the descriptor, count increment, cursor decrement, and candidate-length consumption remain committed. The function returns zero.

9. **Small initial availability is not clamped in both branches.**  
   Availability two and requested span ten select a scan budget of `0xffffffff`. Availability ten and requested span two select zero. Treating both cases as a clamped minimum would violate the observed arithmetic.

## Uncertainties

- **documented_only — Semantic purpose:** “Candidate,” “descriptor,” “consumption,” and “progression” describe dataflow, not an established file format, codec, timestamp domain, or application protocol.
- **documented_only — Dual-use field:** The numeric-bound and descriptor-address uses of `0x28` are both explicit. Whether this reflects an unusual shared representation or incomplete type recovery is unresolved.
- **documented_only — Dependency behavior:** Full status meanings, output guarantees, global mutations, memory ownership, and exceptional behavior are unavailable.
- **documented_only — Invalid inputs:** Faults, misalignment, insufficient descriptor storage, and invalid dependency outputs are not handled explicitly. Their platform-specific consequences are unspecified.
- **documented_only — Overflow fidelity:** Explicit unsigned operations and comparison signedness are visible; original signed-overflow semantics are not fully established.
- **documented_only — Evaluation order:** The order of separate calls, branches, and direct writes is visible. Relative ordering of independent reads inside a single arithmetic expression or argument list is not fully established.
- **Inferred intent:** The requested-span checks suggest bounded processing in nonzero mode. They do not justify normalizing all return-one paths into “the requested span was successfully consumed.”

## Evidence

The supplied static function representation establishes the five arguments, direct context offsets, 12-byte descriptor layout, dependency call sites, branch-specific arithmetic, return assignments, and common final writes.

The private dependency map supplies only the opaque dependency identities used above. It does not establish their semantics.

No implementation execution, runtime tests, or dynamic memory observations have been performed. Acceptance examples are deductions conditional on the stated inputs and dependency outcomes.
