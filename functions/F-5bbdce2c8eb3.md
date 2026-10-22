<!-- Trusted provenance: F-5bbdce2c8eb3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5bbdce2c8eb3

## Purpose

F-5bbdce2c8eb3 computes a 64-bit scalar from two 32-bit input words and a 32-bit control. The words occupy the conceptual lower and upper halves of an input bit pattern.

**Inferred intent:** shift that combined pattern left, discarding bits beyond its 64-bit boundary. This interpretation describes controls 1–63 when the relevant operations have the qualified semantics described below. It is not a complete contract for the entire control domain.

**Established static behavior:** the control selects between a result using both input words and a result using only the low input word. Selection and displacement use different portions or interpretations of the control. Consequently, a large control does not necessarily produce zero, and controls that differ by 256 can produce the same result.

This contract does not establish an identity operation at control zero. Nor does it establish normal completion for every input. Out-of-width shifts, signed arithmetic, and conversion to the signed return type introduce unresolved semantic requirements.

## Interface

| Item | Width | Contract role |
|---|---:|---|
| Low input word | 32 bits, unsigned | Supplies the lower input half and any contribution crossing into the upper output half. |
| High input word | 32 bits, signed | Supplies the upper input half when the selected result form uses both words. |
| Control | 32 bits, unsigned | Determines the result form and effective word displacements. |
| Return value | 64 bits, signed annotation | Carries the constructed result, subject to the return-conversion qualification below. |

Hexadecimal results in this document identify **64-bit patterns**, not an independently verified signed numerical interpretation. If return conversion preserves those bits and the representation is conventional two’s complement, a pattern with its highest bit set represents a negative value.

The logical lower output half comprises positions 0–31; the logical upper half comprises positions 32–63. These positions do not specify byte order, register allocation, stack layout, or a memory representation.

There are no pointer arguments, output buffers, lengths, handles, or ownership-bearing inputs. No storage capacity or aliasing requirement is visible.

No explicit input validation or caller restriction appears in the supplied body. In particular, the control is not visibly restricted to 0–63, and the high input is not visibly restricted to nonnegative values. Conditions used below to obtain definite results are semantic qualifications, not evidence of enforced caller obligations.

## Behavior

### Result selection

Two result forms are visible:

- **Combined form:** both words can affect the result.
- **Upper-only form:** only the low word affects the result; a completed, bit-preserving construction places zeros in the lower half.

Selection involves a subtraction at 32-bit unsigned width, followed by interpretation as a signed 32-bit value. Under conventional signed interpretation of that wrapped difference, the complete selection table is:

| Control, inclusive | Selected form |
|---|---|
| 0–31 | Combined |
| 32–`0x8000001F` | Upper-only |
| `0x80000020`–`0xFFFFFFFF` | Combined |

Thus the selection is not a full-domain unsigned comparison against 32. The second transition, at `0x80000020`, matters even though such controls are far beyond a conventional 64-bit shift range.

The table is conditional on the signed interpretation just stated. The unsigned subtraction itself wraps at 32 bits. An execution environment with different rules for the subsequent unsigned-to-signed conversion requires separate treatment.

### Displacement domain

The control remains a 32-bit input, but only eight bits of each derived displacement are retained. Every retained displacement is therefore in 0–255. That reduction does **not** establish an additional reduction to the valid count range of a 32-bit word.

The following table describes the count relationship without imposing processor-specific shift behavior:

| Selected form | Displacements used |
|---|---|
| Combined | A leftward displacement equal to the control’s remainder on division by 256, and a rightward displacement given by the eight-bit residue of the difference between 32 and the control. |
| Upper-only | A leftward displacement given by the eight-bit residue of the difference between the control and 32. |

Both result selection and these displacement rules are needed. Equal eight-bit residues do not necessarily imply equal result forms.

A displacement of 32 remains 32 at the level of the supplied operations. No source evidence selects modulo-32 masking, zero production, trapping, or another treatment for a 32-bit shift with a count outside 0–31.

### Combined form: established bit placement

The ordinary-count region for this form is a control residue of 1–31. Within that region, both displacements are between 1 and 31.

Subject to the signed-high-word qualification below, the result has the bit placement of a left shift of the conceptual 64-bit input by that residue:

- Low input bits that remain below position 32 occupy the lower output half.
- Low input bits crossing position 32 occupy the bottom portion of the upper output half.
- Surviving high input bits occupy the remaining upper positions.
- Positions opened below the displaced input are zero.
- Bits displaced beyond position 63 are discarded.

The two upper-half contributions are joined by bitwise OR. In this ordinary-count region they occupy disjoint bit positions, so the joining operation does not introduce arithmetic carry.

The lower-half displacement is unsigned and confined to a 32-bit word. It cannot retain bits that leave that word except through the separately formed contribution to the upper half.

The high input is signed, and its leftward displacement occurs as a signed 32-bit operation. A sufficient condition for the ordinary bit-placement description is that the high input is nonnegative and its mathematically displaced value does not exceed 2,147,483,647. For example, at displacement 1, a high input of `0x00000001` meets this condition; `0x40000000` does not.

For negative high inputs or mathematically nonrepresentable signed results, an unconditional bit-vector interpretation is **documented_only**. The supplied evidence does not resolve the applicable execution semantics.

Large controls selecting this form can repeat the ordinary-count behavior. The displacement is the eight-bit residue, not the full control interpreted as a 64-bit shift distance.

Outside residues 1–31, this form contains at least one out-of-width word shift. At residue zero, the rightward displacement is 32. At residue 32, the leftward displacement is 32. Other residues likewise do not provide a pair of ordinary 32-bit shift counts. No unconditional result pattern is specified for those cases.

### Upper-only form: established bit placement

When this form’s effective displacement is 0–31, the result is defined at the bit-pattern level as follows:

- The high input has no influence.
- The low input is displaced within a 32-bit word.
- Bits leaving that word are discarded.
- The surviving word occupies the upper output half.
- The lower output half is zero.

The truncation occurs before placement in the 64-bit result. A bit lost at the 32-bit boundary is not recovered by the later widening.

For controls 32–63, this placement agrees with shifting the conceptual two-word input left by the full control. At those distances, none of the original high input remains within the retained 64-bit boundary.

This agreement does not extend to all larger controls. Control 288, for example, selects the upper-only form with displacement zero. It therefore repeats the defined bit placement of control 32 rather than establishing a shift by 288 positions.

Effective displacements 32–255 are unresolved. If the initial word operation has a defined result, execution completes normally, and subsequent conversions preserve the constructed bits, that word is placed in the upper half and the lower half is zero. These conditions do not establish what the word is, or that an execution with an unresolved word shift must reach the return.

### Boundaries and completion

There is no visible special result for zero, 64, or controls larger than 64.

- **Control zero:** selects the combined form and requires a rightward word displacement of 32.
- **Control 64:** selects the upper-only form and requires a leftward word displacement of 32.
- **Control `0x80000020`:** under conventional signed classification, selects the combined form with leftward displacement 32 and rightward displacement zero.

These cases have neither explicit rejection nor an explicit substitute result.

The visible control flow has one selection and one selected scalar return. There is no loop, retry, callback, or continuing state machine. Normal completion is established only to the extent that the operations and conversions involved have established semantics.

## Side effects

The supplied body contains no explicit memory dereference, memory store, global access, object-state mutation, I/O, synchronization, allocation, or release.

Its visible interface consumes scalar values and returns a scalar. There is no ownership transfer, borrowed storage, retained reference, or resource-lifetime transition.

Accordingly, there is no observable sequence of memory reads or writes to specify. The result-form selection controls which arithmetic result is used. No externally observable ordering among independent arithmetic contributions is established or needed by the visible body.

The upper-only form does not use the high input in its result computation. This does not imply that a caller can omit that argument or violate the physical calling convention.

No error code, auxiliary status, explicit exception construction, or diagnostic channel appears. That absence is not a guarantee of successful return for semantically unresolved inputs.

Physical calling-convention activity, compiler-generated support operations, and processor-specific failure behavior were not inspected. They are not additional established side effects of this contract.

## Dependencies

No separately callable dependency is visible. F-5bbdce2c8eb3 has no exposed callee contract to specify.

Assembly of two logical 32-bit halves into a 64-bit pattern is a result-construction operation, not evidence of an external call, memory access, allocation, or ownership action.

The conditional results depend on these semantic foundations:

- The supplied 32-bit argument widths and signedness.
- Wrapping unsigned arithmetic at 32 bits.
- Conventional signed interpretation for the full-domain selection table.
- Ordinary logical behavior for unsigned shifts with counts 0–31.
- Defined signed behavior for the qualified high-word displacement.
- Preservation of constructed bits when producing the signed 64-bit return.

No unknown callee is assumed to preserve globals, release memory, or complete normally. None is visible.

## Acceptance examples

These are **static acceptance conditions**, not recorded executions. Result values denote bit patterns. Signed return interpretation remains qualified as described under Interface.

### Defined-count examples

| Low input | High input | Control | Required pattern under the stated semantic conditions |
|---|---|---:|---|
| `0x80000001` | `0x00000001` | 1 | `0x0000000300000002` |
| `0x00000003` | `0x00000000` | 31 | `0x0000000180000000` |
| `0x89ABCDEF` | Any 32-bit value | 32 | `0x89ABCDEF00000000` |
| `0x80000001` | Any 32-bit value | 33 | `0x0000000200000000` |
| `0x00000001` | Any 32-bit value | 63 | `0x8000000000000000` |
| `0x89ABCDEF` | Any 32-bit value | 288 | `0x89ABCDEF00000000` |
| `0x80000001` | `0x00000001` | `0xFFFFFF01` | `0x0000000300000002` |

The first example exercises transfer across the boundary between halves without signed high-word overflow. The second exercises the largest ordinary combined-form displacement.

The control-33 example distinguishes word-width truncation from widening before displacement: the low input’s highest bit is discarded, leaving only `0x00000002` for placement in the upper half.

The control-63 example produces a pattern whose conventional signed interpretation is the minimum signed 64-bit value. Acceptance of that signed numerical result additionally requires the return-conversion semantics.

The control-288 example demonstrates that large controls do not imply zero. The final example additionally depends on conventional signed classification and demonstrates repetition of combined-form behavior at a large control.

### Conditional boundary examples

**Control zero, low input 2, high input 1.** If an execution environment additionally reduces word-shift counts modulo 32, the constructed pattern would be `0x0000000300000002`. If instead the rightward shift by 32 yields zero, it would be `0x0000000100000002`. Neither rule is established here, and these alternatives are not exhaustive.

**Control 64, low input 1.** Additional modulo-32 count behavior would produce `0x0000000100000000`; a rule that makes the initial word shift produce zero would produce zero. The static evidence does not choose between them or establish normal completion.

**Control `0x80000020`.** Conventional classification requires the combined form, not the upper-only form. Its out-of-width leftward displacement prevents an unconditional result expectation.

**Control 1 with a negative high input.** The counts are ordinary, but the signed high-word operation remains unresolved. A definite pattern must not be inferred solely from the intended 64-bit shift.

**Control 1 with high input `0x40000000`.** The mathematically displaced high word exceeds the positive signed 32-bit range. This case falls outside the sufficient conditions for the combined-form bit-placement guarantee.

## Uncertainties

The following are **documented_only**:

1. **Out-of-width shifts.** Word counts 32–255 occur. Their treatment is not supplied.
2. **Signed high-word displacement.** Negative inputs and nonrepresentable mathematical results lack a complete execution-semantic contract.
3. **Signed control classification.** The range table assumes conventional interpretation after the wrapped unsigned subtraction.
4. **Signed return conversion.** Construction of a 64-bit pattern does not independently establish the signed numerical result for every pattern.
5. **Caller domain.** No evidence restricts controls to an ordinary shift range or imposes high-word value constraints.
6. **Physical interface.** Byte order, registers, stack layout, and physical return delivery are unavailable.
7. **Exceptional execution.** No observation establishes a particular failure mode or normal completion for unresolved operations.

The inferred shift purpose does not resolve these items. In particular, it supplies no basis for adding identity-at-zero, zero-for-large-controls, validation, or target-specific count masking to the contract.

## Evidence

Established static evidence consists of the supplied scalar annotations and visible operations: two 32-bit words, an unsigned 32-bit control, a signed 64-bit return annotation, signed classification after a wrapped control difference, eight-bit displacement reduction, and two distinct result constructions.

The combined form supports cross-boundary transfer from the low word and incorporation of the displaced high word. The upper-only form excludes the high word and confines the low-word displacement to 32 bits before widening.

The conventional 64-bit-left-shift description is an inference from those bit placements in qualified regions. It is not independent evidence about exceptional counts, signed conversions, caller restrictions, or physical machine behavior.

No runtime tests have been performed. No implementation was executed, and no target-specific shift semantics, external dependencies, or ABI behavior were verified.
