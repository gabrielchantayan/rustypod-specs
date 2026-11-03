<!-- Trusted provenance: F-0a7faecb7263; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0a7faecb7263

## Purpose

F-0a7faecb7263 derives a numeric result from two endpoints, an evaluation position, and a caller-supplied record of 32-bit words.

The visible behavior has three paths:

- Equal endpoints that are zero select a zero intermediate without calling a dependency.
- Equal, nonzero endpoints select a difference formed from signed 16-bit conversions of two record values.
- Unequal endpoints call F-870c4fcf7295 and convert its result to a signed 16-bit intermediate.

Every path subsequently rereads one record word. The final calculation includes the difference between its initially captured value and its later value.

The interface returns a signed 32-bit integer, but **the evidence does not establish an unconditional signed-16-bit result range**. Signedness uncertainty in the final record operand propagates through the final subtraction, addition, multiplication, and right shift. Under one conventional arithmetic model, the result is between −32,768 and 32,767; under an unsigned evaluation model, it is between 0 and 65,535.

**Inferred intent:** The unequal-endpoint arguments resemble a scaled displacement or interpolation calculation. Neither that interpretation nor any particular division rule is an established contract for the dependency.

## Interface

### Inputs

Five inputs are accepted in this order:

| Position | Contract name | Representation and use |
|---|---|---|
| 1 | Opaque input | A 32-bit value with no visible use or forwarding. |
| 2 | Start endpoint | Signed 32-bit integer used in comparison-related subtraction and, on the unequal path, argument arithmetic. |
| 3 | End endpoint | Signed 32-bit integer used in comparison-related subtraction, the zero check, and dependency argument preparation. |
| 4 | Evaluation position | Signed 32-bit integer used only on the unequal-endpoint path. |
| 5 | Record address | Address interpreted as a sequence of 32-bit words. |

The relevant record layout is:

| Byte offset | Width | Visible use |
|---|---:|---|
| 0 | 32 bits | Read for dependency argument 5. |
| 4 | 32 bits | Captured initially; read again in the final calculation. |
| 8 | 32 bits | Appears in separate reads for dependency arguments 4 and 7. |
| 12 | 32 bits | Read for the direct difference or dependency argument 1. |

The record elements have an unresolved 32-bit type rather than an established signed numeric interpretation. The initial offset-4 value is assigned to signed 32-bit storage. Conversion details for record bit patterns outside the signed 32-bit range are not independently established.

These offsets do not imply semantic field names, ownership, or a larger object layout.

### Output

The declared result is a signed 32-bit integer. There is no separate success indicator, and no result value is explicitly reserved for failure.

Two conditional interpretations of the final expression are relevant:

| Arithmetic model | Conditional result interpretation | Conditional range |
|---|---|---|
| 32-bit wrapping arithmetic with an arithmetic final right shift | Signed interpretation of the combined value’s low 16 bits | −32,768 through 32,767 |
| Unsigned 32-bit final arithmetic with a logical right shift | Unsigned interpretation of the combined value’s low 16 bits | 0 through 65,535 |

In the unsigned model, every resulting value is representable in the declared signed 32-bit return type. The declaration therefore does not turn values from 32,768 through 65,535 into negative values.

Neither model is established unconditionally by the supplied evidence.

### Preconditions

The record must support all accesses reached on the selected path:

- Every path accesses offset 4 initially and again during final calculation.
- Equal, nonzero endpoints additionally access offset 12.
- Unequal endpoints access offsets 0, 8, and 12 as well.

Zero endpoints do not permit a null or otherwise inaccessible record: the initial offset-4 access precedes path selection.

No address validation, alignment check, bounds check, or arithmetic validation is visible. Accessible storage, suitable alignment, and any dependency-specific requirements are caller obligations. No defined recovery behavior is established when these obligations are violated.

## Behavior

### Widths and interpretation

The endpoints and evaluation position are signed 32-bit values. Record elements are 32-bit words with unresolved signedness. The nonzero-path intermediate is stored in a signed 16-bit quantity and then widened to signed 32 bits.

“Low 16 bits” below means reduction modulo 65,536. Their signed interpretation maps bit patterns 32,768 through 65,535 to −32,768 through −1.

Out-of-range conversions to signed 16 bits are treated as low-bit retention only where that conventional narrowing model is explicitly assumed. The source alone does not independently establish every conversion rule.

No unlimited-precision product or wider intermediate is visible. Argument arithmetic and final arithmetic must not be replaced with exact mathematical arithmetic without accounting for their widths.

### Initial capture and branch selection

The first expressed record access reads offset 4 into signed 32-bit storage. This captured value is reused later.

The function subtracts the start endpoint from the end endpoint and checks whether the difference is zero.

When that signed subtraction is representable, a zero difference is equivalent to equal endpoints. Under 32-bit wrapping subtraction, equality is also the resulting branch condition. Literal signed-language overflow semantics are not resolved by the supplied evidence.

The opaque first input is not consulted. The evaluation position does not affect branch selection.

### Equal endpoints that are zero

When the endpoint difference is zero and the end endpoint is zero:

- The selected signed 32-bit intermediate is zero.
- Offset 12 is not accessed by this path.
- F-870c4fcf7295 is not called.
- Execution proceeds to the final offset-4 access and return calculation.

This path does not assign or consume the signed 16-bit intermediate used by the other paths.

For a stable offset-4 value whose initial capture preserves its numeric value, the final expression returns zero under either conventional final-arithmetic model described above. A permitted change between the two observations can instead affect the result.

### Equal, nonzero endpoints

When the endpoint difference is zero but the end endpoint is nonzero, the function reads offset 12.

It converts both the offset-12 word and the captured offset-4 value to signed 16-bit quantities. Their difference is formed after integer promotion, assigned back to signed 16-bit storage, and then widened to signed 32 bits.

The promoted subtraction itself fits within a signed 32-bit integer: subtracting two signed 16-bit values produces a mathematical value between −65,535 and 65,535. Assignment back to signed 16 bits can be out of range.

Under conventional low-bit narrowing, the selected intermediate is the signed interpretation of the low 16 bits of the offset-12 value minus the captured offset-4 value.

The dependency is not called. The evaluation position is unused. High record bits do not affect this selected intermediate under that narrowing model, but the final returned representation still depends on the final arithmetic model.

### Unequal endpoints

When the endpoint difference is nonzero, the function makes one visible call to F-870c4fcf7295 with seven positional arguments:

| Position | Supplied value |
|---|---|
| 1 | The offset-12 word minus the captured offset-4 value, multiplied by the evaluation position minus the start endpoint. |
| 2 | The end endpoint minus the start endpoint. |
| 3 | The captured offset-4 value. |
| 4 | A read of offset 8. |
| 5 | A read of offset 0. |
| 6 | The same captured offset-4 value supplied in position 3. |
| 7 | A separate read of offset 8. |

Positions 3 and 6 reuse the capture; they do not introduce fresh offset-4 reads.

The two offset-8 expressions are distinct. Their evaluation order is not established, nor is their equality if the storage can legally change during argument preparation. Stable storage supplies equal values in those positions.

All argument preparation precedes the dependency invocation. The evidence does not establish a complete order among the record reads within that preparation.

The subtraction involving the offset-12 record word has unresolved signedness. This uncertainty also affects its product. The signed evaluation-position subtraction and endpoint subtraction have their own overflow boundaries. No widened multiplication is established.

On normal dependency return, its result is converted to signed 16 bits and then widened to signed 32 bits. Under conventional low-bit narrowing, only its low 16 bits determine that selected intermediate.

The caller does not visibly divide, clamp, or interpret the dependency result as an error status.

### Final observation and return calculation

After selecting the intermediate, every path reads offset 4 again.

The final expression combines the captured offset-4 value, minus the newly read offset-4 word, plus the selected signed 32-bit intermediate. It then multiplies the combination by 65,536 and shifts right by 16 bits.

The newly read record operand has unresolved signedness. Consequently, the final expression cannot unconditionally be described as signed arithmetic or its shift as arithmetic.

Under **32-bit wrapping arithmetic followed by arithmetic right shift**, the result is the signed interpretation of the low 16 bits of:

> selected intermediate + initially captured offset-4 value − final offset-4 value.

Under **unsigned 32-bit evaluation followed by logical right shift**, the result is the unsigned interpretation of those same low 16 bits.

For example, a resulting low-bit pattern of 65,535 yields −1 in the first model and 65,535 in the second.

If the initial and final offset-4 values cancel, the signed-shift model returns the selected intermediate. The unsigned model instead returns its unsigned 16-bit representation. Thus, even an unchanged record does not establish that a negative selected intermediate is returned as a negative integer.

Literal signed multiplication can overflow; wrapping behavior is a conditional machine-arithmetic interpretation, not an unconditional language-level guarantee.

## Side effects

The visible body contains record reads but no explicit store through the record address or another address.

No direct allocation, release, ownership transfer, registration, or pointer retention is visible. The caller supplies storage for access; the body does not explicitly dispose of it.

These observations do not establish that the overall operation is side-effect-free. F-870c4fcf7295 has an unknown effect contract:

- The record address is not explicitly passed as an argument.
- The dependency might nevertheless access the record through an alias or other state.
- Preservation of globals is not established.
- Allocation, release, ownership changes, and external effects are not ruled out.
- Normal termination is not guaranteed by the available evidence.

If the dependency changes offset 4 through a valid access mechanism and returns normally, the later observation can affect the result. If it invalidates the record, the evidence provides no safe fallback for the ensuing access.

If the dependency does not return normally, the final read and normal function return are not established.

The listed accesses describe the expressed behavior. Volatile access guarantees, atomicity, and exact generated memory transactions are not established.

## Dependencies

### F-870c4fcf7295

**Invocation:** One visible call on the unequal-endpoint path; no call on either equal-endpoint path.

**Arguments:** Seven values, with the positional meanings and capture-versus-read distinctions specified above.

**Result handling:** Conversion to a signed 16-bit intermediate, followed by signed 32-bit widening. Low-bit retention is conditional on the narrowing model.

**Unknowns:** Calculation, argument types and signedness expectations, valid domain, overflow policy, error signaling, state effects, ownership effects, and termination.

No other explicit function dependency appears in the supplied body.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. They assume accessible, suitably aligned storage; conventional low-bit narrowing; normal dependency return where relevant; and no overflowing endpoint or argument arithmetic unless stated.

Final results are distinguished between the wrapping/arithmetic-shift model and the unsigned/logical-shift model. These examples do not resolve which model applies.

### Zero endpoints still access the record

With both endpoints zero and a stable offset-4 value of 100:

- Offset 4 is accessed initially and during final calculation.
- Offsets 0, 8, and 12 are not accessed.
- The dependency is not called.
- The result is 0 in either final-arithmetic model.

The record remains required.

### Equal, nonzero endpoints select a direct difference

With both endpoints 7, stable offset 4 equal to 100, and offset 12 equal to 140:

- The selected intermediate is 40.
- No dependency call occurs.
- The result is 40 under either final-arithmetic model.
- Changing only the evaluation position has no effect.

### A negative selected intermediate does not establish a negative return

With equal, nonzero endpoints, stable offset 4 equal to 0, and offset 12 equal to 32,768:

- Conventional narrowing selects −32,768.
- The wrapping/arithmetic-shift model returns −32,768.
- The unsigned/logical-shift model returns 32,768.

A signed-16-bit return guarantee would incorrectly exclude the second result.

### Unequal endpoints supply seven specified values

With start endpoint 10, end endpoint 20, evaluation position 15, and stable record words at offsets 0, 4, 8, and 12 equal to 9, 100, 7, and 140:

- The dependency arguments are, in order, 200, 10, 100, 7, 9, 100, and 7.
- If the dependency returns 20, the selected intermediate is 20.
- If offset 4 remains 100, both final-arithmetic models return 20.

Nothing establishes that the dependency actually returns 20 for these arguments.

### Dependency narrowing and final interpretation are separate

If the dependency returns 65,535 and offset 4 remains unchanged:

- Conventional narrowing selects −1.
- The wrapping/arithmetic-shift model returns −1.
- The unsigned/logical-shift model returns 65,535.

Dependency results with the same low 16 bits select the same intermediate under the narrowing model. Equal final results additionally require the same effective offset-4 correction and final arithmetic interpretation.

### A later record change affects the result

If the initial offset-4 value is 100, the selected intermediate is 20, and the final offset-4 observation is 103, the combined value is 17. Both final-arithmetic models return 17.

If the selected intermediate is instead zero, the combined value is −3 before low-bit normalization. The wrapping/arithmetic-shift model returns −3; the unsigned/logical-shift model returns 65,533.

These examples require a valid mechanism for the changed observation. They do not establish that the dependency mutates the record or authorize unsynchronized concurrent access.

## Uncertainties

The following remain **documented_only**:

- **Final-expression signedness:** The unresolved record-word type participates directly in the final subtraction. Signedness uncertainty extends through addition, multiplication, and right shift.
- **Return range:** Neither the signed-16-bit range nor the unsigned-16-bit range is established unconditionally.
- **Overflow and shifts:** Signed subtraction and multiplication can overflow. Wrapping arithmetic and arithmetic right shift are conditional interpretations, not independently verified target behavior.
- **Conversions:** Initial conversion to signed 32 bits and out-of-range signed 16-bit conversions lack independently established rules for all bit patterns.
- **Argument arithmetic:** Record-word signedness affects the unequal-path subtraction and product. No exact wider product is established.
- **Read ordering:** Initial capture precedes path processing; argument preparation precedes the call; final observation follows normal dependency return. Ordering among argument-read expressions remains unresolved.
- **Changing storage:** Volatile semantics, atomicity, synchronization requirements, and generated transaction counts are unknown.
- **Dependency contract:** No interpolation, division, clamping, preservation, ownership, or error-handling promise is available.
- **Invalid inputs:** No defined recovery is established for inaccessible storage, unsuitable alignment, invalidated storage, or violated dependency preconditions.

## Evidence

**Established static evidence:** The supplied body supports the five-input interface, relevant 32-bit record offsets, initial offset-4 capture, three processing paths, seven-argument dependency call, signed 16-bit intermediate on the nonzero paths, final offset-4 expression, and multiply-by-65,536/right-shift-by-16 calculation. No explicit write or ownership operation appears in the body.

**Conditional arithmetic conclusions:** Signed low-16-bit normalization follows from wrapping arithmetic with arithmetic right shift. Unsigned low-16-bit normalization follows from unsigned 32-bit arithmetic with logical right shift. The unresolved final record operand prevents treating the former as an unconditional output contract.

**Inferred intent:** The dependency arguments suggest scaled displacement or interpolation. This does not establish the dependency’s computation.

**Runtime evidence:** None. No runtime tests, dependency executions, memory traces, or target-arithmetic checks have been performed.
