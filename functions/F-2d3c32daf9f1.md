<!-- Trusted provenance: F-2d3c32daf9f1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2d3c32daf9f1

## Purpose

F-2d3c32daf9f1 combines an unsigned 16-bit field from a caller-supplied object with results obtained from two ordered calls to F-d7be9cfb4000.

The established behavior is a fixed-width calculation, not an ordinary unbounded multiplication or division. It captures the object field before either dependency call, uses the upper 32 bits of the first dependency result to form the second call’s first argument, and uses the lower 32 bits to form an additional contribution to the return value.

A possible intent is to combine two components of a scaled quantity. That interpretation is inferred only. No unit, time base, rounding policy beyond the visible operations, or semantic meaning for either dependency argument is established.

## Interface

### Inputs

The visible interface has two arguments:

| Input | Visible representation | Established use |
|---|---|---|
| Object address | A value represented as a 32-bit integer in the supplied view | Locates an unsigned 16-bit field at byte offset 344, hexadecimal 0x158 |
| Dependency input | An uninterpreted 32-bit value | Passed unchanged as the first argument of the first dependency call |

The object address must identify storage for which the two-byte field at offset 344 can be read. Required alignment, address-space restrictions, byte order, and permitted address arithmetic depend on the target environment and are not established independently by the supplied material.

The first input is used as an address even though its visible representation is an integer. It is not dereferenced at offset zero.

### Output

The visible return interface is a 32-bit integer.

The result is determined by:

- The captured unsigned 16-bit field.
- Both 32-bit halves of the first dependency result.
- The 32-bit value retained from the second dependency result.
- Two multiplication results reduced to 32 bits.
- A logical right shift by three bits.
- A final addition reduced to 32 bits.

The returned bit pattern is specified below. Where its most significant bit is set, interpreting that pattern as a negative signed value requires the target’s signed-integer conversion and return conventions. The supplied material does not independently establish those conventions.

### Preconditions

For a defined completion of the visible sequence:

- Reading the two-byte field must be valid in the target environment.
- F-d7be9cfb4000 must accept the argument patterns supplied at both call sites.
- Both dependency invocations must return normally.

There is no visible validation of either input. In particular, the function does not visibly check the object address for null, verify the object’s extent, or validate the second input before forwarding it.

## Behavior

### Fixed-width interpretation

For describing the arithmetic, let:

- **S** denote the captured unsigned 16-bit field, ranging from 0 through 65,535.
- **L** denote the low 32 bits of the first dependency result.
- **H** denote the high 32 bits of that result.
- **R** denote the 32-bit value retained from the second dependency result.

These are descriptive mathematical labels, not additional program state.

“Reduced to 32 bits” means retaining the low 32 bits, equivalently taking the unsigned residue modulo 4,294,967,296.

### Observable order

The function performs the following ordered actions:

1. Reads the unsigned 16-bit field at byte offset 344 from the supplied object address.
2. Invokes F-d7be9cfb4000 with the original second input and the numeric value 1000.
3. Uses the high 32 bits of that result and the previously captured field to determine the first argument of a second invocation.
4. Invokes F-d7be9cfb4000 with that derived argument and the numeric value 8000.
5. Combines the retained second result with a contribution derived from the first result’s low 32 bits, then returns.

There is only one explicit read of the object field. The visible sequence does not reload it after either dependency call.

The final low-half contribution has no separately visible side effect. Its arithmetic does not introduce another dependency call or object access.

### First dependency invocation

The first invocation receives:

- The second input’s unchanged 32-bit pattern.
- The numeric value 1000.

Its result is captured in an eight-byte value. The subsequent operations distinguish that value’s low and high 32-bit halves.

The contract does not assign a signed numerical meaning to the entire eight-byte result. Extracting its upper half is performed through an unsigned 64-bit interpretation, so the extraction itself does not propagate a sign bit.

### Second dependency invocation

The first argument of the second invocation is the low 32 bits of the product of **S** and **H**.

Although the upper half is visibly converted through a signed 32-bit representation before multiplication, the other multiplication operand is unsigned 32-bit. Under the displayed 32-bit arithmetic model, the multiplication is therefore unsigned and wraps modulo 4,294,967,296.

The resulting argument is not:

- A full-width 48-bit product.
- A saturated product.
- A product divided or shifted before being passed.
- A product using the first result’s low half.

The second argument is the numeric value 8000.

The second call’s result is retained in a 32-bit integer value. The supplied view does not establish the dependency’s complete return convention; only this retained 32-bit contribution is used by the remaining computation.

### Low-half contribution

The additional contribution is obtained by multiplying **S** by **L**, retaining only the low 32 bits, and then logically shifting that retained value right by three bits.

The operation order matters: truncation to 32 bits occurs before the shift.

Equivalently, this contribution is the integer quotient by eight of the unsigned 32-bit residue of the product. It ranges from 0 through 536,870,911.

This is not generally equivalent to dividing the full mathematical product by eight and truncating afterward. It is also not an arithmetic right shift of a negative signed product.

### Final combination

The final return bit pattern is the low 32 bits of the sum of:

- **R**, interpreted by its 32-bit pattern.
- The low-half contribution described above.

The unsigned contribution causes the displayed addition to use unsigned 32-bit arithmetic. Carry beyond bit 31 is discarded. There is no visible saturation, overflow report, or special return code.

The visible function then returns that result through its 32-bit integer interface.

### Special cases

If **S** is zero, both multiplication-derived values are zero. Both dependency calls still occur. The second invocation receives zero and 8000, and its retained 32-bit result supplies the return bit pattern.

If **H** is zero, the second invocation’s first argument is zero, regardless of **S**. The low-half contribution can still be nonzero.

If **L** is zero, the low-half contribution is zero, regardless of **S**. The second invocation’s first argument can still be nonzero.

No first-result pattern visibly causes an early return or suppresses the second invocation.

## Side effects

### Direct accesses

The only explicit memory access is the two-byte object-field read at offset 344.

There are no explicit writes to the supplied object, other caller-owned storage, or global storage in the visible function body. Capturing dependency results and intermediate values does not establish externally observable storage writes.

### Dependency-mediated effects

F-d7be9cfb4000 is called twice, in the order specified above. Its effects are unknown.

In particular, this contract does not assume that either invocation:

- Preserves global state.
- Leaves the supplied object unchanged.
- Avoids allocation or deallocation.
- Avoids I/O.
- Is deterministic.
- Returns normally.

If the first invocation changes the object field, the current invocation of F-2d3c32daf9f1 still uses the field value captured before that call. Whether other dependency effects alter the second call’s behavior is dependency-defined.

### Ownership and lifetime

The visible function does not explicitly allocate, retain, release, or transfer ownership of memory.

The object address is used locally for the field read and is not explicitly passed to either dependency call. This does not establish that the dependency cannot access the same storage through other state.

No ownership or resource-management meaning is established for the second input.

### Errors and interruption

There is no explicit error branch or exception handling.

An invalid object-field read may fail according to the execution environment. If a dependency call does not return normally, the visible sequence does not provide a recovery path. A dependency result that might conventionally represent an error is not visibly recognized as such; its retained bits participate in the calculation.

## Dependencies

| Opaque function ID | Established interaction | Unknown contract |
|---|---|---|
| F-d7be9cfb4000 | Called first with the original second input and 1000; called second with the reduced high-half product and 8000 | Argument semantics, signed interpretation, full return convention, side effects, ownership effects, failure behavior, and determinism |

The constants 1000 and 8000 are established numeric arguments. Their meanings are not established. They must not be described as divisors, durations, limits, frequencies, or units without additional evidence.

## Acceptance examples

These examples are conditional arithmetic and sequencing requirements, not results from executed tests. Each assumes valid input storage and normally returning dependency calls with the stated results.

### Ordinary combination

Given:

- **S** is 16.
- The first dependency result has **H** equal to 3 and **L** equal to 40.
- The second dependency call supplies a retained result of 7.

Required observations:

- The first call receives the original second input and 1000.
- The second call receives 48 and 8000.
- The low-half contribution is 80.
- The returned 32-bit pattern represents 87.

### Truncation before shifting

Given **S** equal to 2 and **L** equal to hexadecimal 0x80000004:

- Their mathematical product is hexadecimal 0x100000008.
- The retained 32-bit product is 8.
- The low-half contribution is 1.

An implementation that instead shifts the full mathematical product would produce a different contribution and would not satisfy this contract.

### High-half product wrapping

Given **S** equal to 65,535 and **H** equal to hexadecimal 0x00010001:

- Their product is hexadecimal 0xFFFFFFFF.
- The second invocation’s first argument must have all 32 bits set.
- Its second argument remains 8000.

The signed meaning assigned to that first argument by the dependency remains unspecified.

### Zero field does not skip calls

Given **S** equal to zero:

- The first invocation still receives the original second input and 1000.
- The second invocation receives zero and 8000.
- The final return pattern equals the retained second result.

Returning zero without making both calls would not satisfy the visible behavior.

### Field mutation after capture

Given an initial field value of 5, a first-call result with **H** equal to 2 and **L** equal to 8, and a first dependency invocation that changes the field to 9:

- The second invocation receives 10, not 18.
- The low-half contribution is 5, not 9.

These requirements follow from the pre-call field capture.

### Final addition wrapping

Given **S** equal to 8, **L** equal to 1, and a retained second result with pattern hexadecimal 0xFFFFFFFF:

- The low-half contribution is 1.
- The returned 32-bit pattern is zero.

No overflow indicator is produced by the visible function.

## Uncertainties

The following are **documented_only**:

- Exact target ABI, byte order, alignment rules, and integer-to-address conversion.
- Whether the displayed dependency return representations accurately describe its complete interface.
- Interpretation of return patterns outside the positive signed 32-bit range.
- Semantic meaning of the object field, dependency input, and numeric constants.
- Dependency access to globals, memory, devices, or resources.
- Concurrency guarantees and behavior if another actor changes the field during its read.
- Valid object types, valid input ranges beyond the visible field width, and application-level failure conventions.

The scaling interpretation is inferred intent, not an established requirement. No unknown dependency behavior is replaced here with an assumed division or normalization operation.

## Evidence

Established static evidence consists of the supplied function representation:

- An unsigned 16-bit read at offset 344 precedes the dependency calls.
- Two calls target F-d7be9cfb4000 with numeric second arguments 1000 and 8000, respectively.
- The first result is captured as eight bytes and contributes separate upper and lower 32-bit portions.
- The visible unsigned operands establish 32-bit modular multiplication and a logical right shift.
- The final expression combines the retained second result with the shifted low-product contribution.
- No explicit validation, memory write, ownership operation, or error branch is present.

This contract distinguishes those observations from inferred intent and unresolved platform or dependency behavior. No runtime tests have been performed.
