<!-- Trusted provenance: F-b37fcadfad8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b37fcadfad8d

## Purpose

F-b37fcadfad8d conditionally populates three 32-bit fields in a caller-supplied mutable record and marks that work complete.

A byte-sized completion marker controls whether any processing occurs. If it is already nonzero, the function returns without invoking dependencies or directly modifying the record. Otherwise, a second byte selects either fixed default values or a dependency-driven calculation.

The resulting fields appear intended to represent a year, month, and day. That interpretation is inferred from the default values, output adjustments, and arithmetic structure; no calendar model, epoch, timezone, or date-validity guarantee is established by the supplied evidence.

This contract distinguishes direct behavior visible in the supplied static representation from behavior that depends on unknown dependency semantics.

## Interface

### Argument and result

The function accepts one pointer to a mutable record. It has no explicit return value and exposes its direct results through writes to that record.

The observed accesses use a 32-bit address model and 32-bit record words. The directly relevant layout is:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 32 bits | First input word for the calculation dependency |
| 4 | 32 bits | Second input word for the calculation dependency |
| 8 | 32 bits | First output field; inferred year |
| 12 | 32 bits | Second output field; inferred month |
| 16 | 32 bits | Third output field; inferred day |
| 40 | 8 bits | Completion marker |
| 42 | 8 bits | Calculation-selection flag |

Offsets are relative to the supplied record pointer. The byte at offset 41 is not directly accessed. Neither flag is accessed as a full 32-bit field.

The two input words are passed onward without an established interpretation. They could jointly encode a wider value, but the caller’s record format and the receiving dependency’s contract are not supplied.

### Preconditions

For ordinary completion:

- The pointer must identify accessible storage for every access taken by the selected branch.
- Offset 40 must always be readable.
- If the completion marker is zero, offset 42 must also be readable, the output fields must be writable, and offset 40 must be writable.
- If calculation is selected, offsets 0 and 4 must be readable, the referenced external constant storage must be accessible, and all invoked dependencies must return normally.
- Word alignment and other platform-specific memory requirements must be satisfied.

The function performs no explicit pointer, length, alignment, or record-consistency validation.

A nonzero completion marker short-circuits the function before the input words, selection flag, or output fields need to be accessed by this function.

### Ownership and lifetime

No direct allocation or deallocation is visible. No direct transfer of record ownership is expressed.

On the calculation path, the first dependency receives the contents of the first two words, not an explicit record pointer. Whether those contents themselves encode addresses or ownership-bearing objects is unknown. Dependency allocation, retention, release, and global-state behavior are not established.

## Behavior

### 1. Completion gate

The first direct record access reads the byte at offset 40.

- If that byte is nonzero, the function returns immediately.
- If it is zero, the function reads the byte at offset 42 and selects one of the two initialization paths below.

Any nonzero completion-marker value inhibits processing; the test is not limited to the value 1. The selection flag likewise distinguishes zero from nonzero rather than requiring a particular nonzero value.

### 2. Default initialization

When both the completion marker and selection flag are zero, the function directly performs these writes in order:

1. Writes the 32-bit value 2000 at offset 8.
2. Writes the 32-bit value 1 at offset 12.
3. Writes the 32-bit value 1 at offset 16.
4. Writes the byte value 1 at offset 40.

It then returns.

This path does not invoke any dependency and does not read the two input words. It overwrites any previous contents of the three output fields. It does not directly change the selection flag.

### 3. Calculation path: external operands

When the completion marker is zero and the selection flag is nonzero, the function performs a fixed sequence of dependency calls and integer calculations.

Six pairs of externally stored 32-bit words supply operands to those calls. This contract names them **pair A** through **pair F** in order of first use. These names identify storage roles, not known numeric values.

Whenever a dependency produces a 64-bit result used by a later dependency, the caller separates that result into:

- its low 32 bits, supplied first; and
- its high 32 bits, supplied second.

This describes argument significance, not byte order in memory. The external operand pairs are also supplied as two separate 32-bit arguments in their observed order.

The numeric contents and types represented by pairs A–F are unavailable. In particular, the evidence does not establish that they are floating-point constants, integer divisors, or calendar coefficients.

### 4. Calculation path: ordered dependency protocol

The following sequence is established. The named intermediate results are explanatory labels, not record fields.

1. Invoke **F-1b22d7e17791** with the two input words followed by pair A.
2. Invoke **F-baa332e06e32**, obtaining a signed 32-bit result, **first result**.
3. Invoke **F-b3a54d881220**, obtaining a 64-bit result. No explicit argument is visible at this call site.
4. Invoke **F-8a693c103d1c** with that 64-bit result split low word first, followed by pair B.
5. Invoke **F-596e18a33aec** with the preceding 64-bit result split the same way, followed by pair C.
6. Invoke **F-baa332e06e32**, obtaining a signed 32-bit result, **second result**.
7. Form a 32-bit **initial aggregate** from the first and second results as described below.
8. Invoke **F-b3a54d881220** with the initial aggregate as its visible argument.
9. Invoke **F-8a693c103d1c** with the resulting split 64-bit value followed by pair D.
10. Invoke **F-596e18a33aec** with the preceding split 64-bit result followed by pair E.
11. Invoke **F-baa332e06e32**, obtaining a signed 32-bit **year basis**.
12. Invoke **F-b3a54d881220** with no explicit argument visible at this call site.
13. Invoke **F-c4b34006a13c** with the resulting split 64-bit value followed by pair E.
14. Invoke **F-baa332e06e32**, obtaining a signed 32-bit **first deduction**.
15. Subtract the first deduction from the initial aggregate to obtain a 32-bit **residual**.
16. Invoke **F-b3a54d881220** with the residual as its visible argument.
17. Invoke **F-596e18a33aec** with the resulting split 64-bit value followed by pair F.
18. Invoke **F-baa332e06e32**, obtaining a signed 32-bit **month basis**.
19. Invoke **F-b3a54d881220** with no explicit argument visible at this call site.
20. Invoke **F-c4b34006a13c** with the resulting split 64-bit value followed by pair F.
21. Invoke **F-baa332e06e32**, obtaining a signed 32-bit **second deduction**.

The absence of a visible argument at several calls does not establish that those dependencies operate without inputs. The surrounding sequence may rely on register conventions, implicit machine state, or shared state not represented by an ordinary function signature.

Likewise, where no return value is explicitly consumed, the call remains part of the required sequence. Its state effects may be essential to the next call.

### 5. Calculation path: arithmetic and output writes

The initial aggregate is formed from the first result, the second result, and the constant 1525.

The quarter-term calculation applies a sign-dependent bias before a signed right shift by two bits. With 32-bit arithmetic right shifts, this produces the second result divided by four, rounded toward zero, including for negative values. Thus, where the surrounding additions and subtraction do not overflow, the aggregate is:

**first result + second result − truncation-toward-zero(second result / 4) + 1525.**

The residual is the initial aggregate minus the first deduction.

After the final dependency returns, the function directly performs the following record writes in order:

1. **Third output field, offset 16:** residual minus second deduction.
2. **Second output field, offset 12:** month basis minus 1 if the signed month basis is less than 14; otherwise month basis minus 13.
3. **First output field, offset 8:** year basis minus 4715 if the adjusted month is less than 3; otherwise year basis minus 4716.
4. **Completion marker, offset 40:** byte value 1.

The year decision uses the adjusted month, not the original month basis.

All output fields are written as 32-bit words. The month comparison and adjusted-month comparison are signed comparisons. There is no explicit restriction to positive values or conventional calendar ranges.

The calculation path writes the output fields in the reverse order of the default path: third, second, then first. In both paths, the completion marker is the final direct write.

### 6. Widths and overflow limits

The visible integer intermediates and output fields are 32 bits. Dependency results identified as 64 bits are retained as 64-bit quantities before being divided into argument words.

The supplied representation does not establish a portable overflow contract for signed 32-bit addition or subtraction. Consequently:

- The arithmetic descriptions above give exact mathematical results when intermediate values remain representable.
- On a target using wrapping 32-bit arithmetic, overflowing operations retain their low 32 bits, with subsequent signed comparisons interpreting the resulting 32-bit pattern.
- Portable behavior for signed overflow is **documented_only**, pending confirmation of the target execution semantics.

The sign-bias operation used for division by four does not itself require an overflowing addition: it adds three only for negative second results. The surrounding aggregate arithmetic and output adjustments can overflow for unconstrained dependency results.

## Side effects

### Directly established effects

The function directly modifies only the three output words and the completion byte, and only when the initial completion byte is zero.

It does not directly:

- clear or normalize the selection flag;
- initialize intervening record bytes;
- validate a computed date;
- check a dependency result for an error indication;
- report a status code;
- roll back partial writes.

The initial completion marker is not rechecked before the final writes. A dependency-induced change to that byte therefore does not introduce a second completion gate.

### Dependency-mediated effects

The calculation path invokes 19 dependency calls. Those calls may have externally observable effects not described by this caller.

No assumption is made that dependencies preserve global values, leave the record untouched through aliases, avoid allocation, or preserve implicit computational state. External operand storage is referenced at its respective call sites; the evidence does not justify treating all operand pairs as an immutable snapshot taken at function entry.

The directly visible record writes begin only after the dependency sequence completes. This does not prove that the record remains unchanged during that sequence, because dependency-mediated aliasing is unknown.

### Failure and concurrency

There is no explicit error-handling path. If an access faults or a dependency does not return normally, ordinary completion and the final marker write are not guaranteed.

The completion byte is not shown as an atomic synchronization operation. The routine supplies no established lock, memory-ordering guarantee, or protection against concurrent readers and writers. Its ordered accesses alone do not make initialization transactional or thread-safe.

## Dependencies

| Opaque function ID | Observed caller-facing role | Calls on calculation path |
|---|---|---:|
| F-1b22d7e17791 | Receives the two input words and pair A; no explicit result is consumed | 1 |
| F-baa332e06e32 | Supplies signed 32-bit results used in arithmetic | 6 |
| F-b3a54d881220 | Supplies 64-bit results; two sites visibly pass a 32-bit intermediate, three show no explicit argument | 5 |
| F-8a693c103d1c | Receives a split 64-bit value and an operand pair; supplies a 64-bit result | 2 |
| F-596e18a33aec | Receives a split 64-bit value and an operand pair; no explicit result is consumed | 3 |
| F-c4b34006a13c | Receives a split 64-bit value and an operand pair; no explicit result is consumed | 2 |

These roles describe this caller’s use, not complete dependency interfaces. Conversion rules, rounding behavior, hidden inputs, exceptional values, and state preservation remain unknown.

## Acceptance examples

These are conditional contract examples, not executed tests.

### Already completed

Given a readable completion byte equal to 255, the function returns without directly reading the selection flag, reading either input word, invoking dependencies, or writing the record.

Existing output values remain untouched by this function.

### Default initialization

Given a zero completion byte, zero selection flag, and valid writable output storage, the resulting output words are 2000, 1, and 1 at offsets 8, 12, and 16 respectively. The completion byte becomes 1.

The required direct write order is offset 8, offset 12, offset 16, then offset 40. Existing output contents do not affect the result.

### Controlled calculation results

Suppose the calculation path is selected and the six calls to F-baa332e06e32 return, in order:

- 100;
- 8;
- 6800;
- 1600;
- 14;
- 1.

Assuming all dependencies return normally and the visible arithmetic has its stated 32-bit signed interpretation:

- The quarter term is 2.
- The initial aggregate is 1631.
- The residual is 31.
- The third output is 30.
- The month basis of 14 selects subtraction by 13, producing month 1.
- Month 1 selects subtraction of 4715 from the year basis, producing year 2085.

The direct output writes are therefore 30 at offset 16, 1 at offset 12, 2085 at offset 8, and finally 1 at offset 40. This example constrains caller arithmetic; it does not assert that real dependencies can produce this sequence for any particular input.

### Month and year boundaries

For a month basis of 13, the adjusted month is 12 and the year adjustment is subtraction by 4716.

For a month basis of 14, the adjusted month is 1 and the year adjustment is subtraction by 4715.

For a month basis of 3, the adjusted month is 2 and the year adjustment is subtraction by 4715.

For a month basis of 4, the adjusted month is 3 and the year adjustment is subtraction by 4716.

These boundaries apply without an additional calendar-validity check.

### Negative quarter term

If the second result is −3, the quarter term is zero, not −1, under the arithmetic-right-shift interpretation. This distinguishes the observed sign-adjusted calculation from unadjusted floor division.

## Uncertainties

The following are **documented_only**:

- The semantic meaning and encoding of the two input words.
- Whether the outputs truly represent a date, and, if so, the calendar, epoch, timezone, and supported range.
- The values and numeric interpretation of operand pairs A–F.
- Full dependency signatures and implicit argument conventions.
- Dependency rounding, error, exception, ownership, and state-preservation behavior.
- Signed-overflow behavior beyond the conditional target interpretation stated above.
- Whether external operand storage or record storage can alias dependency-visible state.
- Any synchronization protocol imposed by callers.
- Validity guarantees for negative, oversized, or otherwise unconventional output values.

The apparent date-conversion intent does not justify filling in any of these missing contracts.

## Evidence

Established static evidence consists of the supplied function representation: its byte and word accesses, branch conditions, dependency-call order, 32-bit integer operations, 64-bit result splitting, output-write order, and final completion-byte write.

Date terminology is inferred intent. Dependency mathematics, operand values, and hidden-state conventions are not established.

No runtime tests have been performed. The acceptance examples are analytical consequences of the visible control flow and explicitly stated conditions, not observations from execution.
