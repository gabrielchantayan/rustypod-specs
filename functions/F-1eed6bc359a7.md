<!-- Trusted provenance: F-1eed6bc359a7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1eed6bc359a7

## Purpose

F-1eed6bc359a7 derives a centered rectangular mapping from dimensions and a fixed-point-like value stored in a caller-supplied context, submits that mapping to an associated object, updates additional object properties, and performs a final sequence of object operations.

The supplied static evidence establishes the arithmetic, field accesses, conditional calls, and call order. It does not establish the semantic meaning of the objects or their properties. Descriptions such as “width,” “height,” “opacity,” “lock,” or “render” are therefore interpretations unless explicitly qualified below.

On the visible normal-return path, the function returns the 32-bit value zero. It does not inspect dependency return values for errors, except that the results of F-d7be9cfb4000 are used in subsequent arithmetic.

## Interface

### Input and result

The function accepts one address-sized argument represented as a 32-bit integer in the supplied representation. It is used as the base address of a mutable context.

The result is a 32-bit value. Every visible normal-return path returns zero. No distinct error result is shown.

No null checks, bounds checks, object-validity checks, or arithmetic-overflow checks are visible.

### Context layout

The following byte offsets are relative to the context base. Unless otherwise specified, each access is four bytes.

| Offset | Established use |
|---|---|
| +0x18 | Value captured late in execution and conditionally passed to F-40f10dd4a12e. |
| +0x1c | Second argument to F-1d891874186d. |
| +0x20 | Object reference passed to F-460b2abbbbbe and F-1d891874186d. |
| +0x24 | Object reference passed to F-a05963f65233, F-460b2abbbbbe, and the final operation sequence. |
| +0x28 | First original extent; initially loaded as an unsigned 32-bit value. |
| +0x2c | Second original extent; initially loaded as a signed 32-bit value. |
| +0x3c | One-byte flag tested for equality to zero. |
| +0x44 | Reference to a readable structure whose four-byte member at +4 is copied to context +0x4c. |
| +0x48 | Reference to a readable structure whose four-byte member at +4 is copied to context +0x50. |
| +0x4c | Four-byte destination for the first copied member. |
| +0x50 | Four-byte destination for the second copied member. |

The object referenced through context +0x24 is also accessed at:

- +0x0b: one-byte discriminator, compared with 3.
- +0x78: address passed to F-338d16f9ce72 and F-1333331b2da1.

These offsets establish required accesses, not complete structure definitions. The two object references may or may not designate different objects; distinctness is not checked.

### Memory and arithmetic preconditions

For ordinary defined operation, the context and all transitively accessed addresses must support the listed reads and writes. Dependency-specific object validity, alignment, lifetime, and synchronization requirements are unknown.

The visible arithmetic uses 32-bit quantities. Its comparisons mix signed and unsigned interpretations. A mathematical implementation using unbounded integers is not automatically equivalent.

The evidence contains no explicit guard against zero divisors supplied to F-d7be9cfb4000. Its behavior for zero, extreme values, or signed-looking bit patterns is undocumented here.

## Behavior

### 1. Initial dependency calls and member copies

The function first calls:

1. F-d31472986b5e with no explicit arguments.
2. F-6db6c6bf494a with the context address.

Only after both calls does it read the references at context +0x44 and +0x48.

It reads the four-byte member at +4 of the first referenced structure, writes that value to context +0x4c, then reads the four-byte member at +4 of the second referenced structure and writes it to context +0x50.

The first copied value also supplies the initial arithmetic input described below. The second copied value has no further direct use in this function.

The context writes occur in this order. If context fields, referenced structures, or their members overlap, the earlier write can affect the later read. No independent snapshot of all inputs is taken before these operations.

### 2. Initial extent calculation

For explanation, call the original extents **W** and **H**, corresponding to context +0x28 and +0x2c. These are descriptive labels, not assertions about physical geometry.

The function reads H and W after completing the two copies.

From the first copied member, interpreted as a signed 32-bit quantity, it derives an initial first target extent:

- Add 32,767.
- Shift right arithmetically by 16 positions.
- Clear the lowest two bits of the resulting 32-bit value.

For nonnegative, nonoverflowing inputs, this produces a multiple of four from a fixed-point-like quantity. The representation does not establish the unit or intended rounding policy beyond those operations.

It then calls F-d7be9cfb4000 with:

1. The low 32 bits of the product of that initial target extent and H.
2. W.

The returned 32-bit value becomes the initial second target extent. A quotient interpretation is plausible, but is not established by the caller alone.

### 3. Independent extent bounds

The function bounds the first target extent using W:

- It compares four times W against the target using an unsigned 32-bit comparison.
- If four times W is smaller, the target becomes W shifted left by two bits.
- Otherwise, it calculates W divided by four using the signed interpretation of W, rounded toward zero, and raises the target to that value if required by an unsigned comparison.

The second target extent is bounded similarly using H:

- Four times H is treated as a 32-bit value and compared unsigned against the second target.
- If that value is smaller, the target becomes H shifted left by two bits.
- Otherwise, the target is raised, when necessary, to signed H divided by four rounded toward zero, interpreted as a 32-bit unsigned value for comparison.

For positive dimensions and products that do not overflow, these operations bound each target between one quarter and four times its corresponding original extent, with truncation at the lower bound.

That conventional range description does not apply generally to negative values, high-bit-set values, or wrapped products. The unsigned comparisons and signed quarter calculations remain material in those cases.

### 4. Conditional symmetric adjustment

The initial retained extents are W and H. Two additional adjustment values start at zero.

Only the unsigned comparison “W is less than the bounded first target” controls entry to this adjustment branch. The second target is not independently tested.

If the branch is entered:

1. The first excess half-extent is calculated as the logical half of the bounded first target minus signed W divided by two, rounded toward zero.
2. F-d7be9cfb4000 is called with the 32-bit product of that excess and W, followed by the bounded first target.
3. Its result becomes the first adjustment value.
4. Twice the first excess is subtracted from the first target.
5. Twice the first adjustment is subtracted from W to produce the first retained extent.
6. The second excess half-extent is calculated as the logical half of the bounded second target minus signed H divided by two, rounded toward zero.
7. F-d7be9cfb4000 is called with the 32-bit product of that excess and H, followed by the bounded second target.
8. Its result becomes the second adjustment value.
9. Twice the second excess is subtracted from the second target.
10. Twice the second adjustment is subtracted from H to produce the second retained extent.

These are 32-bit calculations. No subsequent nonnegativity check is applied to retained extents or adjustment values.

For ordinary positive dimensions, the branch resembles symmetric cropping after enlargement. This is inferred intent. It must not replace the actual branch condition or arithmetic, especially for odd dimensions or independently bounded target extents.

### 5. Centering values and mapping submission

Two centering values are then calculated:

- Signed W divided by two, rounded toward zero, minus the logical half of the current first target.
- Signed H divided by two, rounded toward zero, minus the logical half of the current second target.

Each subtraction result is interpreted as signed for a negative-result check. A negative result is replaced with zero.

The function next calls F-460b2abbbbbe with the object currently read from context +0x20 and the value 255.

It then calls F-a05963f65233 with the object currently read from context +0x24, followed by these eight 32-bit values in order:

1. First centering value.
2. Second centering value.
3. Second adjustment value.
4. First adjustment value.
5. Current first target extent.
6. Current second target extent.
7. First retained extent.
8. Second retained extent.

The reversed order of the two adjustment values is established and must be preserved. The dependency’s actual parameter meanings are unknown.

### 6. Flag-selected property update and association

After the mapping call returns, the function reads the byte at context +0x3c:

- Zero selects the 32-bit value zero.
- Any nonzero byte selects the 32-bit value 255.

It calls F-460b2abbbbbe with the object freshly read from context +0x24 and that selected value.

It then calls F-1d891874186d with the values read from context +0x20 and +0x1c.

The two field reads supply that call, but their relative evaluation order is not established by the supplied representation.

### 7. Final object-operation sequence

After F-1d891874186d returns, the function captures the value at context +0x18, then captures the object reference at context +0x24.

Using that captured object reference, it performs this sequence:

1. Call F-338d16f9ce72 with the object address plus 0x78.
2. Read the byte at object +0x0b.
3. If that byte equals 3, call F-40f10dd4a12e with the captured object and the previously captured context +0x18 value.
4. Call F-9afb51fb96af with the captured object and four addresses in the caller’s stack frame.
5. Call F-e2303c7cb41c with the captured object and four additional 32-bit arguments whose values are not reliably established by the supplied representation.
6. Call F-1333331b2da1 with the captured object address plus 0x78.
7. Return zero.

The four stack addresses passed to F-9afb51fb96af are distinct and separated by four bytes. In argument order, each successive address is four bytes lower than the preceding address.

The evidence does not establish that the four arguments to F-e2303c7cb41c are values written through those addresses. That relationship must remain unresolved.

## Side effects

The directly visible context mutations are the four-byte writes at +0x4c and +0x50. No other direct context writes are shown.

All dependency calls may have additional side effects. Their signatures and placement do not prove that they preserve globals, context fields, object references, object contents, or ownership.

Several reads deliberately occur after intervening calls:

- The flag is read after F-a05963f65233.
- The association arguments are read after the second property call.
- The final captured values are read after F-1d891874186d.
- The object discriminator is read after F-338d16f9ce72.

An implementation must not hoist those reads across calls merely on an assumption that dependencies are nonmutating.

Once the final object reference and context +0x18 value have been captured, subsequent operations use those captured values rather than reloading the context fields.

No allocation, explicit release, ownership transfer, or lifetime extension is directly visible. Passing a reference to a dependency does not establish any of those effects.

There is no visible cleanup branch for dependency failure. The final call on object +0x78 follows the preceding calls only when they return normally.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-d31472986b5e | First operation; no explicit arguments. |
| F-6db6c6bf494a | Receives the context before field copying and arithmetic. |
| F-d7be9cfb4000 | Receives two 32-bit arithmetic operands; called once normally and twice more in the enlargement branch. Results affect extents and adjustments. |
| F-460b2abbbbbe | Called first on context +0x20 with 255, then on context +0x24 with zero or 255. |
| F-a05963f65233 | Receives the mapping object and eight ordered geometry-related values. |
| F-1d891874186d | Receives context +0x20 and +0x1c after both property updates. |
| F-338d16f9ce72 | Receives the captured final object address plus 0x78. |
| F-40f10dd4a12e | Conditional call when the subsequently read object discriminator equals 3. |
| F-9afb51fb96af | Receives the captured final object and four stack addresses. |
| F-e2303c7cb41c | Receives the captured final object and four unresolved argument values. |
| F-1333331b2da1 | Final dependency call, using the same captured object-relative address as F-338d16f9ce72. |

The bracketing calls on object +0x78 may implement synchronization, and the zero/255 property may represent opacity. Both interpretations are inferred intent, not established contracts.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results. Numerical examples assume F-d7be9cfb4000 returns the ordinary integer quotient for the positive operand pairs shown, and that intervening dependencies do not alter the relevant context values.

### Unchanged target extents

Given W = 100, H = 80, and first copied member = 6,553,600:

- The initial first target is 100.
- The initial second target is 80 under the quotient assumption.
- Neither bound changes either target.
- The enlargement branch is not taken.
- Both centering and both adjustment values are zero.

F-a05963f65233 receives, after its object argument:

0, 0, 0, 0, 100, 80, 100, 80.

### Smaller centered target

Given W = 100, H = 80, and first copied member = 3,145,728:

- The initial first target is 48.
- The second target is 38 under the quotient assumption.
- The enlargement branch is not taken.
- The centering values are 26 and 21.

The eight mapping arguments are:

26, 21, 0, 0, 48, 38, 100, 80.

### Enlargement with symmetric adjustment

Given W = 100, H = 80, and first copied member = 13,107,200:

- The bounded targets are 200 and 160.
- The enlargement branch is taken.
- The excess half-extents are 50 and 40.
- The adjustment values are 25 and 20 under the quotient assumption.
- The final target extents are 100 and 80.
- The retained extents are 50 and 40.
- Both centering values are zero.

The eight mapping arguments are:

0, 0, 20, 25, 100, 80, 50, 40.

### Flag and discriminator decisions

If the flag byte is zero when read, the second F-460b2abbbbbe call receives zero. A nonzero byte, including a byte with its high bit set, selects 255.

If the final object’s discriminator is 3 after F-338d16f9ce72 returns, F-40f10dd4a12e is called. Any other byte skips that call without skipping the remaining final operations.

## Uncertainties

The following are **documented_only**:

- The semantic identities and ownership rules of the context and referenced objects.
- The exact arithmetic contract of F-d7be9cfb4000, including signedness, rounding, zero-divisor behavior, and exceptional outcomes.
- The source-language overflow model. The representation exposes 32-bit operations but does not establish portable behavior for every signed-overflow case.
- Valid input ranges and whether zero or negative extents can occur.
- Whether dependencies mutate the context, referenced structures, shared globals, or object lifetimes.
- Whether the calls on object +0x78 acquire and release synchronization.
- The meaning of the object discriminator and the property values zero and 255.
- The dataflow and semantic values of the four extra arguments to F-e2303c7cb41c.
- Error signaling, nonlocal exits, and effects when a dependency does not return normally.

These unknowns do not justify adding validation, fallback dimensions, alternate cleanup, or guessed argument values.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes field widths and offsets, direct context writes, visible arithmetic and comparison forms, conditional call sites, argument ordering, and the zero normal-return value.

Geometric terminology, fixed-point interpretation, quotient behavior, property semantics, and synchronization semantics are either explanatory labels or explicitly conditional inferences.

No runtime tests have been performed. No dependency implementation or independent ABI-level evidence was supplied.
