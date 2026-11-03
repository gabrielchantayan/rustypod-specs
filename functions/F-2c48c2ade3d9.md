<!-- Trusted provenance: F-2c48c2ade3d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c48c2ade3d9

## Purpose

F-2c48c2ade3d9 derives a 32-bit dispatch selector from its third input and several external values. It repeatedly evaluates that selector until it lies in the inclusive range 0 through 291, then invokes a code address selected by a four-byte stride.

The established behavior is narrower than a complete dispatch specification: the supplied representation exposes the selector arithmetic and loop condition, but does not expose the selected code bodies or establish that every selectable address is a valid callable entry point.

**Inferred intent:** the routine appears to implement a bounded, computed dispatch with an external-state-dependent retry path. Neither the meaning of the selectors nor the purpose of the external state is established.

## Interface

### Inputs

The routine has three apparent parameters:

| Input | Apparent width | Established use |
|---|---:|---|
| First input | 32 bits, with no established interpretation | Not explicitly read by the visible body |
| Second input | 32 bits, with no established interpretation | Not explicitly read by the visible body |
| Third input | Unsigned 32 bits | Participates in the initial selector calculation |

No explicit pointer dereference, buffer access, or ownership transfer is performed through these inputs.

The first two inputs must not be assigned semantic roles solely because they are present. They are not explicitly forwarded as arguments to the final invocation. Whether their original machine-level values remain observable to a selected target is unknown.

### External values

For this contract, the external values are named by their visible roles:

- **Mask:** used in a bitwise AND with the third input.
- **Adjustment:** added to that masked value.
- **Overlay:** used in a bitwise OR with the third input.
- **Selector origin:** subtracted when calculating each candidate selector.
- **Reload value:** sampled on every loop iteration and used as the working value for a possible next iteration.

These are descriptive names, not claims about the intended data model.

The supplied representation does not include declarations for these external objects. Their exact storage widths, signedness, volatility, atomicity, initialization requirements, and permitted writers are therefore **documented_only**. The arithmetic described below uses their effective 32-bit operand values, consistent with the visible 32-bit working values and selector.

### Outputs and return

There is no explicit return value.

If a candidate is accepted, the routine performs one visible indirect invocation with no explicit arguments. If that invocation returns normally, this routine then returns.

Normal return is not guaranteed:

- The candidate-selection loop can continue indefinitely.
- A selected address may not be a valid executable entry point.
- The selected code may not return.
- Other failures associated with external-state reads or execution are not handled by any visible error path.

### Preconditions

To reach a well-defined dispatch under the apparent machine model:

- The external values must be readable.
- Their effective values must support the described 32-bit operations.
- The selected computed address must be executable and suitable for the invocation convention.
- Any requirements imposed by the selected code must independently be satisfied.

No visible validation establishes those conditions.

## Behavior

### Arithmetic model

The visible selector is an unsigned 32-bit value. Comparisons against selector bounds are unsigned.

The initial working value combines the third input with the mask, adjustment, and overlay:

- One contribution is the bitwise AND of the third input and the mask, plus the adjustment.
- The other contribution is the bitwise OR of the overlay and the third input.
- The second contribution is subtracted from the first.

Under the apparent 32-bit arithmetic model, these operations retain the low 32 bits. A negative mathematical result is therefore represented by its corresponding 32-bit bit pattern; it is not clamped to zero.

Although the working storage is presented as signed, the candidate selector is unsigned. A working bit pattern with its high bit set does not make the candidate negative for the range check.

The external object declarations and original executable instructions are unavailable. Exact source-language promotion details beyond the apparent 32-bit model remain **documented_only**; signed-overflow behavior must not be invented from the recovered signed annotation alone.

### Initial external-state reads

Before the loop, the routine reads the mask, adjustment, and overlay to form the initial working value.

The representation establishes their participation in the same calculation, but does not establish a reliable order among those three reads. This contract does not promise that the mask is read before the adjustment, or that either is read before the overlay.

There is no visible atomic snapshot mechanism. If the values can change independently, the calculation is not established to use values from one coherent external-state version.

### Candidate calculation and reload

The loop executes at least once.

On each iteration:

1. The selector origin is read and subtracted from the current working value. The resulting low 32 bits form the candidate selector.
2. The candidate is evaluated against the acceptance conditions.
3. The reload value is read and assigned to the working value.
4. The loop either repeats or proceeds to dispatch using the candidate already calculated for that iteration.

The reload read occurs even on the iteration that exits the loop. It does not replace or recompute the candidate used for that iteration’s dispatch.

On a repeated iteration, the working value comes from the preceding iteration’s reload read. Consequently, a later candidate can combine a previously sampled reload value with a newly read selector origin. The representation does not establish that those two values are sampled together.

The mask, adjustment, and overlay are not visibly reread on repeated iterations.

### Acceptance range

The displayed conditions distinguish candidates below 256 from larger candidates and separately recognize the upper endpoint. Their combined effect is:

**A candidate is accepted exactly when its unsigned value is at most 291.**

Thus:

- 0 through 255 are accepted.
- 256 through 290 are accepted.
- 291 is accepted.
- 292 through 4,294,967,295 cause another iteration.

The internal equality check involving 256 does not make 256 a special observable dispatch case. Likewise, the separate equality check for 291 does not establish a distinct target behavior beyond accepting that selector.

This range result follows from unsigned subtraction and comparison. For candidates greater than 255, subtracting 256 produces a value from 0 through 4,294,967,039 without wraparound. A result through 34 accepts candidates through 290; the additional equality accepts 291.

### Retry behavior

A rejected candidate does not produce a visible error, return, or fallback dispatch. The routine repeats using the reload-based working value.

If the reload value and selector origin remain stable, all candidates after the initial one have the same value. Therefore:

- An initially rejected candidate can be followed by an accepted reload-based candidate.
- If the reload-based candidate is also outside the accepted range and both external values remain stable, the loop does not terminate.

There is no visible retry limit, timeout, delay, cancellation check, or recovery branch.

External changes may affect later iterations. The supplied representation does not establish memory-ordering guarantees or guarantee that concurrent updates will become observable in a particular iteration.

### Dispatch

For an accepted candidate, the routine computes a code address from a fixed dispatch base plus four times the candidate.

The accepted range implies:

- 292 possible selected addresses.
- Four-byte spacing between adjacent selected addresses.
- A minimum offset of zero.
- A maximum offset of 1,164 bytes.

The bounded selector multiplication does not itself overflow a 32-bit unsigned value.

The displayed invocation treats the computed address as executable code. It does **not** explicitly load a function pointer from a four-byte table element. Four-byte spacing alone is therefore insufficient evidence for a table of stored function pointers.

The routine invokes the selected address with no explicit arguments. If the selected code returns normally, the routine returns without further visible work.

The recovered indirect control transfer is uncertain. The exact machine-level distinction between a call and another form of transfer, including return-link handling and register preservation, is **documented_only**.

## Side effects

### Established visible effects

The visible body:

- Reads the initial mask, adjustment, and overlay.
- Reads the selector origin once per iteration.
- Reads the reload value once per iteration, including the terminating iteration.
- Performs one selected indirect invocation if candidate selection terminates.

No explicit writes to external objects, caller-provided memory, or global state are shown before that invocation.

Updating the working value is internal state only. It does not establish a write back to the external reload object.

### Read observability

The source representation establishes logical reads, not the memory attributes of the objects being read.

If the objects are ordinary memory, the reads may have no externally visible effect beyond influencing control flow. If they refer to volatile or device-backed storage, reads could themselves be observable. Those attributes are unknown, so neither interpretation is guaranteed.

### Ownership and dependency effects

The visible body does not explicitly allocate, retain, release, or transfer ownership of memory or another resource.

No such restriction can be placed on the selected target. Its effects on memory, global state, resources, registers, execution state, and control flow are unknown. In particular, this contract does not assume that it preserves globals or refrains from releasing memory.

## Dependencies

The supplied dependency information identifies only F-2c48c2ade3d9 itself. It supplies no opaque function IDs for the selected targets.

The routine depends on:

| Dependency | Established role | Unresolved contract |
|---|---|---|
| Initial external values | Form the first working value | Storage declarations, initialization, consistency |
| Selector origin | Determines each candidate | Mutation rules, read semantics |
| Reload value | Supplies working state for later iterations | Mutation rules, read semantics |
| Computed executable target | Performs the selected operation | Identity, arguments, effects, ownership, return behavior |

No direct helper invocation is visible before dispatch.

A complete end-to-end contract requires independent evidence for the reachable targets. Their behavior must not be inferred from selector numbers or address spacing.

## Acceptance examples

These examples are conditional consequences of the visible arithmetic and control flow. They are not runtime test results. Assume readable external state, the apparent 32-bit arithmetic model, and a valid target wherever dispatch is described.

1. **Lowest candidate.**  
   If the initial working value and the selector origin have identical 32-bit values, the first candidate is 0. The routine reads the reload value, then invokes the address at the dispatch base.

2. **Last candidate below 256.**  
   If the first candidate is 255, it is accepted. The selected address is 1,020 bytes beyond the dispatch base. A terminating-iteration reload read still occurs.

3. **First candidate at or above 256.**  
   A candidate of 256 is accepted. It selects the address 1,024 bytes beyond the base. It does not trigger a retry merely because it exceeds 255.

4. **Upper endpoint.**  
   A candidate of 291 is accepted and selects the address 1,164 bytes beyond the base.

5. **First rejected candidate.**  
   A candidate of 292 causes another iteration. It does not invoke an address 1,168 bytes beyond the base.

6. **Unsigned underflow.**  
   If the working value is 0 and the selector origin is 1, their 32-bit difference is 4,294,967,295. That candidate is rejected, rather than treated as a small negative selector.

7. **Recovery after rejection.**  
   Suppose the first candidate is 292. The first iteration then samples a reload value that, when reduced by the selector origin read on the second iteration, produces 7. The second iteration accepts 7, performs another reload read, and dispatches at an offset of 28 bytes.

8. **Stable nontermination.**  
   Suppose the first candidate is rejected and every later reload-based difference is 292. The routine continues looping. No visible target invocation or error return occurs.

9. **Unused explicit inputs.**  
   With the third input, external reads, and target behavior held identical, changing only the first or second input does not change the visible selector calculation. This does not prove that a selected target cannot observe residual machine state associated with those inputs.

10. **Nonreturning target.**  
    If an accepted candidate selects code that never returns, this routine does not subsequently perform its visible return.

## Uncertainties

The following items are **documented_only**:

- Exact external-object widths, signedness, qualifiers, and storage properties.
- Original calling convention and the meaning of the first two inputs.
- Instruction-level ordering among reads in the initial expression.
- Atomicity, synchronization, and visibility of concurrent external updates.
- The dispatch base’s executable layout and validity of every selected address.
- Whether the recovered indirect invocation accurately represents the original transfer mechanism.
- Target identities, input expectations, outputs, side effects, ownership behavior, and return guarantees.
- Fault behavior for invalid reads or invalid executable addresses.

The inferred dispatch intent does not resolve these uncertainties.

## Evidence

This contract is based solely on the supplied static representation of F-2c48c2ade3d9.

Established static evidence includes the three apparent inputs, the third input’s participation in the initial calculation, repeated selector-origin and reload reads, unsigned candidate comparisons, the inclusive accepted range of 0 through 291, and the four-byte-stride computed executable address.

The representation explicitly reports incomplete recovery of the indirect control transfer. Consequently, the selector behavior is more firmly supported than the selected targets’ interface or semantics.

No implementation access, executable inspection, runtime tests, or target-body analysis has been performed. No snapshot hash or independent provenance claim is supplied here.
