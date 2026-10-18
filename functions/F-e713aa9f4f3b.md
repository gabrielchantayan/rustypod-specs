<!-- Trusted provenance: F-e713aa9f4f3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e713aa9f4f3b

## Purpose

F-e713aa9f4f3b produces an unsigned 32-bit value corresponding to the middle 32 bits of the exact signed product of two 32-bit operands: product positions 16 through 47, with position zero denoting the least significant bit.

This is a **conditional behavioral contract**. The bit-selection model assumes 32-bit operands, a 64-bit product representation, and fixed-width two’s-complement interpretation. The supplied static material supports that model, but does not establish that every input has portable, defined execution behavior. Signed intermediate arithmetic introduces unresolved language-level restrictions, described under **Uncertainties**.

The function does not provide the complete product, saturation, rounding to nearest, or an overflow indication. Its result is an unsigned bit pattern, not a signed numerical result.

Selecting these product positions is consistent with multiplication of fixed-point values carrying 16 fractional bits. That interpretation is inferred intent only. No supplied caller contract establishes units, scaling, admissible magnitudes, or subsequent signed interpretation.

## Interface

**Function identifier:** F-e713aa9f4f3b

| Element | Value interpretation | Conditional width | Range |
|---|---|---:|---:|
| First operand | Signed integer | 32 bits | −2,147,483,648 through 2,147,483,647 |
| Second operand | Signed integer | 32 bits | −2,147,483,648 through 2,147,483,647 |
| Return value | Unsigned integer | 32 bits | 0 through 4,294,967,295 |

The operands are supplied by value. Neither is a pointer, storage extent, resource identifier, or ownership-bearing object.

There is one returned value and no separate status output. No output buffer, callback, error object, or explicit exceptional-result marker appears in the supplied body. In particular, zero and 0xFFFFFFFF are ordinary result patterns, not reserved failure indicators.

No application-level restriction to positive operands or small products is established. However, the absence of an explicit validation branch must not be interpreted as proof that all operand pairs have defined execution semantics in every language environment.

The value-level contract requires compatible operand and return widths. The supplied material does not independently establish argument registers, stack positions, alignment, byte order, or the physical return convention.

There is no serialized structure or aggregate layout. The layout specified here concerns numerical bit positions only.

## Behavior

### Mathematical result

Under the conditional model, form the exact mathematical product of the two signed operand values. The returned value is:

\[
\operatorname{result}
=
\left\lfloor
\frac{\text{exact product}}{65{,}536}
\right\rfloor
\bmod 4{,}294{,}967{,}296.
\]

The modulo operation here produces the unique result in the unsigned 32-bit range.

This equation specifies the observable result. It does not prescribe a sequence of machine operations or claim that the supplied expression has universally defined execution semantics.

The product must retain information beyond 32 bits. Reducing the product to 32 bits before selecting the result would change valid conditional outcomes.

For signed 32-bit operands, the exact product lies in:

\[
-2^{62}+2^{31}
\;\leq\;
\text{exact product}
\;\leq\;
2^{62}.
\]

Both endpoints fit in a signed 64-bit value. The upper endpoint occurs when both operands are −2,147,483,648. The lower endpoint occurs when one operand is −2,147,483,648 and the other is 2,147,483,647.

The representability of this exact product does not establish the validity of every intermediate operation used by the supplied body.

### Output layout

Interpret a negative product through its 64-bit two’s-complement representation. Number its bits from 0 at the least significant end through 63 at the most significant end.

| Returned positions | Product positions |
|---|---|
| 0–15 | 16–31 |
| 16–31 | 32–47 |

Product positions 0–15 and 48–63 do not contribute to the conditional result. The returned value has no additional fields, tags, or padding.

This selection is independent of byte order because it describes numerical significance rather than byte addresses. Any serialization of the returned value would require a separate byte-order contract.

### Discarded low information

The discarded low 16 product bits do not trigger a rounding increment.

Consequently, positive products from 1 through 65,535 return zero under the model. A product of 65,536 returns one.

For negative products, the mathematical floor is significant. Products from −65,535 through −1 produce 0xFFFFFFFF. Replacing the specified behavior with division rounded toward zero would incorrectly produce zero for those products.

Exact multiples of 65,536 have no discarded fractional remainder. Their quotients are nevertheless reduced to the unsigned 32-bit range, so divisibility alone does not prevent loss of upper information.

### Discarded upper information

The result does not preserve product information above position 47. There is no saturation at either signed or unsigned output limits.

Under the conditional model:

- A nonzero product can return zero.
- Either a positive or a negative product can return a result with its highest bit set.
- A negative product can return zero.
- Different products can return the same value.
- The original product generally cannot be recovered from the returned value.

These outcomes are consequences of the output layout, not reported errors.

The interface returns an unsigned value. A consumer may separately reinterpret the same bits as signed, but that consumer action is outside this function’s established interface.

### Symmetry and observable order

The conditional result is symmetric: exchanging the operand values preserves the returned bit pattern.

There are no explicit accesses to caller-addressed storage, shared state, or external services in the supplied body. Accordingly, there is no externally observable read/write ordering requirement between such operations.

Argument-expression evaluation belongs to the caller and is outside this contract. The mathematical description also does not promise any particular number or order of internal arithmetic evaluations.

## Side effects

The supplied static body contains no explicit:

- Read from or write to caller-provided storage.
- Global or thread-local state access.
- Allocation, deallocation, retention, or release.
- Resource acquisition or release.
- Input/output operation.
- Callable dependency.
- Error-state update or validation branch.

The return value is the only established caller-visible output. No persistent state transition is specified.

Because arguments are received by value, the function does not modify the caller’s operand objects through this interface. No ownership transfer takes place, and the caller receives no resource requiring cleanup.

These observations concern explicit behavior in the supplied body. They do not establish properties of instrumentation, runtime support, generated machine code, or hardware faults. In particular, no constant-time guarantee, timing bound, instruction count, or platform-wide absence of side effects has been demonstrated.

The unresolved arithmetic semantics must not be converted into an invented error protocol. The source does not establish a defined exception, fallback result, or recoverable status for problematic intermediate arithmetic.

## Dependencies

No callable dependency is present in the supplied body. There are therefore no dependency function IDs to enumerate.

The contract depends on arithmetic widths and representation rules, not on an identified external service. No assumption about an unknown callee preserving global state, owning storage, or releasing resources is needed.

Whether a particular compilation target introduces arithmetic support routines is **documented_only**. No generated executable or lowering information is supplied, so no such routine is assigned an identity or a side-effect contract here.

## Acceptance examples

All examples below are **conditional model expectations**, derived statically. They are not execution observations or assurances that the supplied expression has defined behavior for every listed pair.

Hexadecimal results denote unsigned 32-bit values.

| First operand | Second operand | Exact product | Conditional result |
|---:|---:|---:|---:|
| 0 | −2,147,483,648 | 0 | 0x00000000 |
| 1 | 1 | 1 | 0x00000000 |
| 65,535 | 1 | 65,535 | 0x00000000 |
| 32,768 | 2 | 65,536 | 0x00000001 |
| 65,535 | 65,535 | 4,294,836,225 | 0x0000FFFE |
| 65,536 | 65,536 | 4,294,967,296 | 0x00010000 |
| −1 | 1 | −1 | 0xFFFFFFFF |
| −65,536 | 1 | −65,536 | 0xFFFFFFFF |
| −65,537 | 1 | −65,537 | 0xFFFFFFFE |
| −65,536 | 65,536 | −4,294,967,296 | 0xFFFF0000 |
| 2,147,483,647 | 65,536 | 140,737,488,289,792 | 0x7FFFFFFF |
| 8,388,608 | 16,777,216 | 140,737,488,355,328 | 0x80000000 |
| −2,147,483,648 | 65,536 | −140,737,488,355,328 | 0x80000000 |
| −16,777,216 | 16,777,216 | −281,474,976,710,656 | 0x00000000 |
| −2,147,483,648 | −2,147,483,648 | 4,611,686,018,427,387,904 | 0x00000000 |

These examples distinguish the following observable requirements of the model:

- **Product width:** 65,536 multiplied by 65,536 produces 0x00010000, not zero. Narrowing the product prematurely loses required information.
- **No rounding to nearest:** 65,535 multiplied by one produces zero despite its large discarded remainder.
- **Negative remainder treatment:** −1 multiplied by one produces 0xFFFFFFFF, not zero.
- **Negative boundary transition:** −65,536 multiplied by one produces 0xFFFFFFFF; decreasing the first operand to −65,537 changes the result to 0xFFFFFFFE.
- **Unsigned interpretation:** 0x80000000 denotes 2,147,483,648 at this interface, not a negative return value.
- **No saturation:** A product of \(2^{47}\) produces 0x80000000 rather than a signed maximum.
- **Upper-information loss:** Both the listed negative product of \(-2^{48}\) and the positive product of \(2^{62}\) produce zero.
- **Symmetry:** Reversing either operand pair preserves its conditional result.

For negative products and sufficiently large positive products, these expectations especially require the stated qualification concerning signed intermediate semantics. They must not be presented as verified behavior of an arbitrary compilation.

## Uncertainties

### Widths and representation — documented_only

The supplied type categories support the proposed width model, but independent type definitions and a target specification are absent.

The 32-bit operand and return widths, 64-bit product width, and two’s-complement interpretation are therefore explicit conditions of this contract. A target with different relevant widths requires a separate analysis.

Physical ABI details and byte order remain unspecified. Byte order does not change the numerical bit-selection model.

### Intermediate arithmetic validity — documented_only

The supplied body uses signed intermediate arithmetic in constructing the result. Two unresolved issues prevent an unconditional portability claim:

- An intermediate unsigned value may be outside the representable range of a signed destination.
- Signed bit-positioning arithmetic may involve a negative value or a mathematical result outside the signed destination’s range.

The applicable language version and implementation determine the consequences. Depending on those rules, some cases may require implementation-defined behavior or fall outside defined execution semantics.

The exact product fitting in 64 bits does not remove these concerns. Likewise, the absence of an explicit overflow check does not prove that the implementation wraps predictably.

The conditional result model describes the intended bit-level outcome when the required fixed-width behavior applies. It does not assign a fallback value to executions for which that behavior is not established.

### Numerical interpretation — inferred intent

Sixteen fractional bits are a plausible application convention. The supplied material does not establish that callers use that convention, restrict their operands accordingly, or treat the unsigned result as signed.

No additional numerical preconditions are inferred from that possibility.

### Execution environment — documented_only

No runtime tests have been performed. Generated instructions, compiler choices, runtime support, fault behavior, and timing remain unverified.

No execution evidence resolves the portability questions or establishes behavior outside the conditional model.

## Evidence

**Established static observations**

- The interface has two signed integer operands and an unsigned integer return.
- The operands are widened for product computation.
- The returned expression selects information from a wider product.
- The body contains no explicit dependency calls, caller-storage accesses, persistent-state updates, resource operations, or error branches.
- Signed intermediate arithmetic is present, so its validity requires separate consideration from product representability.

**Derived under the stated width and representation conditions**

- The selected information is product positions 16 through 47.
- Every exact signed 32-bit operand product fits within signed 64-bit range.
- Floor division by 65,536 followed by unsigned 32-bit modular reduction describes the conditional result.
- Operand exchange preserves that result.
- The acceptance values follow from exact arithmetic and the specified output layout.

**Inferred intent**

- Fixed-point multiplication with 16 fractional bits.

**Not established**

- Defined, portable execution for every operand pair.
- Independently verified target widths or ABI properties.
- Actual returned values from an executable.
- Generated instruction sequences, timing, or platform-level effects.

The evidence is static only. The behavioral model, its arithmetic derivation, and the unresolved execution semantics are distinct claims.
