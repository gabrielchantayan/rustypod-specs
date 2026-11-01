<!-- Trusted provenance: F-3a6f7fda3c0d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a6f7fda3c0d

## Purpose

F-3a6f7fda3c0d computes a signed, byte-sized control value from three numeric inputs and one externally stored byte, limits that value to the inclusive range −60 through 12, encodes it in seven bits, and submits it to F-21cbfe3b29e6 together with the first input.

The function returns the 32-bit value zero after the dependency returns normally. It does not propagate the dependency’s result.

The arithmetic resembles fixed-point interpolation followed by a bounded control update. That interpretation is inferred intent, not an established domain meaning. The evidence does not identify the quantity, its units, or the object represented by the first input.

A critical behavioral distinction is that **signed-byte reduction precedes limiting**. This is not equivalent to limiting the full arithmetic result directly to −60 through 12.

## Interface

The interface has five inputs, in this order:

| Input | Evidenced representation | Use |
|---|---|---|
| First input | Opaque 32-bit value | Forwarded unchanged as the first argument to F-21cbfe3b29e6. |
| Second input | Opaque 32-bit value | Unused by this function. |
| Scale input | Signed 16-bit integer | Multiplies a signed 16-bit intermediate before a right shift by eight bits. |
| Adjustment input | Signed 16-bit integer | Subtracted while forming that intermediate. |
| Position input | Signed 32-bit integer | Used to form the initial value −60 minus this input. |

The scale and adjustment inputs are used with signed 16-bit interpretation. Their exact calling-convention representation outside the function, including treatment of wider caller-provided values, is not established.

The result is a 32-bit zero on every evidenced normal return. There is no separate error result.

The function also requires access to external state:

- An externally stored base address.
- A readable unsigned byte at offset one from that address.

The external byte is an additional computational input, not an output buffer.

The first input’s semantic type is unknown. Although it is forwarded to another function, this function does not itself dereference it or establish whether it denotes a pointer, handle, identifier, or ordinary number.

## Behavior

### Numeric stages

The calculation has several distinct width boundaries. They must not be collapsed into a single unbounded arithmetic expression.

For clarity, the terms below name conceptual values rather than implementation identifiers:

1. **Base value:** −60 minus the signed 32-bit position input.
2. **Narrowed base:** the base value reduced to a signed 16-bit value.
3. **Signed difference:** 12 minus the narrowed base minus the signed adjustment input, with the resulting difference reduced to signed 16 bits.
4. **Scaled contribution:** the signed difference multiplied by the signed scale input, then shifted right by eight bits using a signed 32-bit right shift.
5. **Combined value:** the base value plus the scaled contribution plus the external byte interpreted as an unsigned value from 0 through 255.
6. **Signed-byte candidate:** the low eight bits of the combined value interpreted as a signed value from −128 through 127.

For the signed reductions, the evidenced target-style interpretation retains the low bits and interprets the retained top bit as the sign bit. Thus a 16-bit pattern representing 65535 as unsigned represents −1 when used as a signed 16-bit value.

The product in the fourth stage fits within signed 32-bit range: both factors are signed 16-bit values, and the largest possible positive product is 1,073,741,824. The product itself therefore does not require overflow to explain the observed behavior.

The signed right shift supplies division by 256 with negative values rounding downward under the arithmetic-shift interpretation. For example, a product of −1 contributes −1 rather than zero. This distinction is observable and is not interchangeable with division that truncates toward zero.

The source representation forms the signed-byte candidate by moving the relevant byte into the high byte of a 32-bit word and shifting it back with signed interpretation. Its behavioral consequence is low-byte selection followed by sign extension. Bits above the low eight bits do not directly survive this stage.

### Limiting and encoding

The signed-byte candidate is limited as follows:

| Candidate | Limited value |
|---|---:|
| Less than −60 | −60 |
| −60 through 12, inclusive | Unchanged |
| Greater than 12 | 12 |

The limited value is then masked to its low seven bits before dispatch.

Consequently:

- Limited values 0 through 12 are passed as 0 through 12.
- Limited values −60 through −1 are passed as 68 through 127.
- The dependency’s second argument cannot be 13 through 67 on the evidenced normal path.
- The lower limit, −60, is passed as 68.
- The upper limit, 12, is passed as 12.

This is a seven-bit representation of the limited value, not an unsigned saturation into the interval 0 through 127.

### Dispatch and return

After the arithmetic and limiting have completed, F-21cbfe3b29e6 is invoked exactly once on the evidenced straight-line normal path:

- Its first argument is the original first input.
- Its second argument is the seven-bit encoded value.

The second input to F-3a6f7fda3c0d has no effect on the computation or the explicitly supplied dependency arguments.

After the dependency returns normally, F-3a6f7fda3c0d returns zero. There is no subsequent external-byte read, recalculation, or explicit state update.

### Preconditions and exceptional execution

Normal execution requires that the external base address can be obtained and that the byte at offset one is readable. No null check, bounds check, validity check, or recovery path is evidenced.

Normal return also requires that F-21cbfe3b29e6 return normally. If it faults, terminates execution, or transfers control elsewhere, the evidence does not establish a zero return from this function.

The source representation contains signed 32-bit arithmetic whose mathematical result can exceed signed 32-bit range for extreme position inputs. Exact behavior at those extremes is qualified under **Uncertainties**; an independent implementation must not silently choose saturation, widening, rejection, or wrapping as though that choice were fully established.

## Side effects

The function’s directly evidenced external effects are a read and a dependency invocation.

### Reads

The function obtains an externally stored base address and reads one byte at offset one from that address. That byte is used as an unsigned quantity.

The address must be available before the dependent byte can be read. The byte read occurs before the dependency invocation, because its value contributes to the second argument.

There is only one explicit byte read in the supplied body. The function does not explicitly reread the byte after invoking the dependency. No synchronization or atomicity guarantee is evidenced.

No pointed-to data is read through either opaque 32-bit input by this function itself.

### Writes and state changes

There are no explicit writes to caller-provided memory or external storage in the function body. Its computation uses temporary values and passes the final encoded value onward.

This does **not** establish that the overall operation is free of state changes. F-21cbfe3b29e6 may modify memory, external state, or the storage involved in the earlier read. Its effects are unknown here.

### Ownership and lifetime

No allocation, release, ownership transfer, or reference-count operation is directly evidenced.

Forwarding the first input does not prove that the dependency borrows it, retains it, consumes it, or releases anything associated with it. The dependency’s ownership contract remains unknown.

Because the dependency is invoked only after the byte has been read and the encoded value has been formed, a change it makes to that byte cannot retroactively affect the argument already supplied by this invocation.

## Dependencies

### F-21cbfe3b29e6

This is the only explicitly invoked dependency.

Established call contract from this function:

- One invocation per normal execution reaching dispatch.
- Two arguments.
- First argument: unchanged original first input.
- Second argument: a nonnegative encoded value belonging to either 0–12 or 68–127.
- Any ordinary return value is ignored.

Not established:

- The meaning or required validity of the first argument.
- The dependency’s return type or success convention.
- Whether it modifies globals or caller-accessible memory.
- Whether it retains or releases resources.
- Whether it can fail, block, reenter this function, or avoid returning.

The external base-address storage and the byte at offset one are additional data dependencies. Their initialization, ownership, and update rules are not supplied.

## Acceptance examples

These are conditional acceptance examples derived from the static arithmetic, not results of executed tests. Each assumes readable external state, ordinary target-style signed narrowing and arithmetic right shifts, and a normally returning dependency.

For every example, the dependency receives the original first input unchanged, and the function then returns 32-bit zero.

### Direct byte interpretation when the base is zero

With position input −60 and scale input zero, the base and scaled contribution are both zero. The adjustment input does not affect the result.

| External byte | Signed-byte candidate | Limited value | Dependency’s second argument |
|---:|---:|---:|---:|
| 0 | 0 | 0 | 0 |
| 12 | 12 | 12 | 12 |
| 13 | 13 | 12 | 12 |
| 127 | 127 | 12 | 12 |
| 128 | −128 | −60 | 68 |
| 196 | −60 | −60 | 68 |
| 255 | −1 | −1 | 127 |

In particular, increasing the byte from 127 to 128 changes the encoded argument from 12 to 68. This demonstrates why full-value saturation is not an equivalent substitute for signed-byte reduction followed by limiting.

### Lower boundary with zero scaling

With position input zero, scale input zero, and external byte zero:

- The base is −60.
- The signed-byte candidate is −60.
- The dependency’s second argument is 68.

Changing only the position input to one makes the base −61. The candidate is below the lower limit, so the encoded argument remains 68.

Changing only the position input to −1 instead makes the base −59. The encoded argument is then 69.

### Scale value 256

With position input zero, adjustment input zero, scale input 256, and external byte zero:

- The signed difference is 72.
- Scaling and shifting contributes 72.
- The combined value is 12.
- The dependency’s second argument is 12.

Changing only the adjustment input to one makes the signed difference 71 and the combined value 11. The dependency’s second argument becomes 11.

These examples illustrate the fixed-point character of the scaling without asserting a domain-specific meaning for the scale.

### Negative shift behavior

With position input −60, adjustment input 13, scale input one, and external byte zero:

- The signed difference is −1.
- Its product with the scale is −1.
- An arithmetic right shift by eight produces −1.
- The dependency’s second argument is 127.

An implementation that substitutes truncation-toward-zero division would instead produce zero and would not preserve the evidenced arithmetic-shift behavior.

### Sixteen-bit narrowing before scaling

With position input −60, adjustment input −32768, scale input one, and external byte zero:

- The mathematical difference before 16-bit reduction is 32780.
- Its signed 16-bit interpretation is −32756.
- Arithmetic shifting the product right by eight contributes −128.
- The signed-byte candidate is −128.
- The dependency’s second argument is 68.

Retaining a wider positive difference instead would produce a different result and would miss the explicit 16-bit reduction.

### Unused input

For otherwise identical inputs and external-byte state, changing only the second input does not change the arithmetic or either explicitly supplied dependency argument.

This does not constrain unrelated environmental behavior inside the dependency.

## Uncertainties

The following are **documented_only**, rather than guessed requirements:

- **Extreme signed arithmetic:** The base subtraction can exceed signed 32-bit range. Adding the scaled contribution can also exceed that range before the unsigned-byte term is incorporated. The supplied source representation does not independently settle strict language-level overflow behavior versus actual target machine arithmetic. The acceptance examples avoid those overflow cases.
- **Signed representation and shifting:** The behavioral reading uses two’s-complement narrowing and arithmetic signed right shifts, consistent with the displayed operations. No independently supplied instruction-level or execution evidence confirms those target properties.
- **External storage identity:** The meaning, size, initialization, lifetime, and mutability of the object containing the byte are unknown.
- **Concurrent changes:** The function contains no explicit synchronization. No promise is established about concurrent replacement of the base address or modification of the byte.
- **Dependency effects:** Global preservation, ownership handling, error behavior, and all other internal effects of F-21cbfe3b29e6 are unknown.
- **Domain meaning:** The constants, scale, adjustment, and encoded result have no established physical or application-level interpretation.
- **ABI details:** Register assignments, stack layout, and handling of mismatched callers are outside the supplied evidence.

## Evidence

The established static evidence consists of one function body and its supplied dependency mapping.

That evidence supports:

- Five inputs with the displayed widths and signed interpretations.
- No use of the second input.
- A single unsigned-byte read at offset one from an externally stored address.
- Explicit signed 16-bit reductions during the arithmetic.
- A signed product followed by a right shift of eight.
- Low-byte signed interpretation before limiting.
- Inclusive limiting to −60 through 12.
- A seven-bit mask before a single call to F-21cbfe3b29e6.
- Forwarding the first input unchanged.
- A constant zero result after normal dependency return.

The interpolation interpretation is inferred intent. Dependency internals, external-state semantics, and the qualified extreme-arithmetic cases remain unknown.

No runtime tests have been performed.
