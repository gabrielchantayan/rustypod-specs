<!-- Trusted provenance: F-f3a53be97fda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3a53be97fda

## Purpose

F-f3a53be97fda reads unsigned 16-bit source elements, writes a derived sequence of 16-bit destination patterns, and then invokes a target obtained from a global function-pointer slot.

For conventional positive dimensions, each generated destination row combines values from two adjacent source rows. Most columns use two successive packed-field averages, giving greater weight to the later row. The last column uses one average.

The arithmetic separates each 16-bit pattern into fields of widths 5, 6, and 5 bits. Interpreting those fields as color channels is inferred intent, not an established interface requirement.

The final target is read from global storage at the invocation, after the transformation-loop writes. It is not established as a fixed target selected before the transformation. In particular, destination writes that alias the target-pointer storage could affect which target is subsequently invoked.

The dependency’s behavior is unresolved. Direct transformation results must therefore be distinguished from memory contents after the complete function returns.

## Interface

The function has four inputs and no explicit return value.

| Input | Established representation | Role |
|---|---|---|
| Source | Pointer to unsigned 16-bit elements | Supplies transformation operands. |
| Destination | Pointer to signed 16-bit elements | Receives 16-bit output patterns. |
| Width | Signed 32-bit integer | Controls source-row spacing, inner iteration count, and final dependency arguments. |
| Height | Unsigned 32-bit integer | Determines the outer bound; also supplies a row-end arithmetic operand when the inner loop is skipped. |

The destination’s signed element type does not make the ordinary packed-field calculation a signed-field operation. A stored pattern with bit 15 set may be interpreted as a negative signed 16-bit value.

No capacities, error-output pointers, or ownership flags are supplied. There are no explicit null, alignment, dimension, or capacity checks.

For width at least two and a positive outer bound, assuming valid arithmetic and storage:

- Each outer iteration writes exactly width destination elements.
- The number of generated rows is height minus one under the height interpretation described below.
- Destination writes are consecutive.
- Source accesses fit within the mathematical region of width times height elements.
- The first source element is not read by the transformation.

These bounds concern direct transformation accesses only. They do not bound the final dependency’s accesses.

Required storage includes every directly accessed element and every pointer position formed by the executed path. The global function-pointer slot must also be readable at final invocation. Valid invocation additionally depends on the slot containing a usable target and on that target’s unresolved requirements.

## Behavior

### Outer bound and initial pointer formation

The outer bound is formed by subtracting one from height using unsigned 32-bit arithmetic, then assigning the result to a signed 32-bit quantity.

Under the represented 32-bit signed interpretation:

| Height | Outer iterations |
|---|---|
| 0 or 1 | None |
| 2 through `0x80000000` | Height minus one |
| `0x80000001` through `0xffffffff` | None |

The out-of-range unsigned-to-signed conversion is target-dependent at the source-language level. These ranges describe the represented interpretation, not a portability guarantee for unrelated implementations.

A source pointer displaced forward by width elements is formed before the outer-loop decision. Consequently, an empty transformation does not remove all pointer-validity requirements.

After the loop, including when it executes zero times, the function proceeds to the global target-slot read and indirect invocation.

### Packed-field average

For two unsigned 16-bit patterns, one averaging stage combines:

- The bits set in both operands.
- Half the differing bits, after clearing bit positions 5 and 11.

Bit numbering begins at zero.

For ordinary 16-bit operands, this computes the floor of the mean independently in three unsigned fields:

| Bit positions | Width |
|---|---|
| 0–4 | 5 bits |
| 5–10 | 6 bits |
| 11–15 | 5 bits |

Clearing positions 5 and 11 prevents a shifted bit from crossing a field boundary. Bit zero is discarded by the right shift.

The first averaging stage also clears bit 16 of its result. For two 16-bit inputs, that bit cannot be needed, so this additional clearing does not change the ordinary result.

### Ordinary row mapping

For width at least two, number source rows and columns from zero. Generated row \(r\) combines source rows \(r\) and \(r+1\).

For destination column \(c\), where \(0 \le c < \text{width}-1\):

- The earlier-row operand comes from source position \((r,c+1)\).
- The later-row operand comes from source position \((r+1,c)\).
- Those operands undergo one fieldwise average.
- The intermediate result is averaged again with the same later-row operand.
- The second result is stored.

For an individual field with earlier-row value \(a\) and later-row value \(b\), the output is:

\[
\left\lfloor
\frac{b+\left\lfloor(a+b)/2\right\rfloor}{2}
\right\rfloor.
\]

This is approximately a one-quarter earlier-row contribution and a three-quarter later-row contribution. The two separate truncations are part of the behavior; a differently rounded weighted average is not interchangeable.

The final destination column is treated differently:

- Its earlier-row operand is the earlier row’s last element, retained from the final inner iteration.
- Its later-row operand is the later row’s last element.
- These operands undergo only one averaging stage.

Thus, the earlier row’s last element contributes to both the penultimate and final destination columns. The earlier row’s first element does not contribute to that generated row.

### Direct access order and overlap

Within an ordinary inner iteration, the represented order is:

1. Read the earlier-row source element.
2. Retain that value, including for possible row-end use.
3. Read the later-row source element.
4. Calculate the two-stage result.
5. Write the next destination element.

Each inner iteration completes before the next. The row-end destination write follows that row’s inner writes, and the next outer iteration follows the row-end write.

The row-end expression contains two references to the final later-row source element. Their relative evaluation order is not established. Nor does the supplied representation prove that execution performs two distinct physical memory transactions.

There is no source snapshot. If destination storage overlaps source storage, an earlier destination write can change a value consumed by a later source read. The original-input row formulas describe nonoverlapping storage, or overlap that does not alter subsequent operands. Otherwise, the ordered accesses govern the result.

The same aliasing concern extends to the global target-pointer storage: direct writes occur before the final target read.

### Width one

When width is one and the outer loop executes:

- The inner loop is skipped.
- One destination element is written per outer iteration.
- Outer iteration \(r\) reads source element \(r+1\).
- Source element zero is not read.
- The other row-end operand is the original unsigned 32-bit height.

Height is not replaced by a retained source value on this path. Its full 32-bit value participates in the exclusive-or and logical right-shift calculation. Truncating height to 16 bits before that calculation can change the stored result.

The row-end operation combines the source value with the low 16 bits of height for its common-bit contribution, but uses the full height for its differing-bit contribution. The resulting assignment produces a 16-bit destination value, subject to the target’s signed conversion rules.

### Width zero and negative widths

There is no rejection branch for nonpositive width.

For width zero, with a positive outer bound:

- The inner loop is skipped.
- One destination element is nevertheless written per outer iteration.
- Row-end source accesses advance consecutively from source element zero.
- The other arithmetic operand remains the original height.
- The final dependency receives identical first and second pointer arguments and scalar zero.

Width zero must not be modeled as an empty operation.

For negative widths, the inner loop is skipped when evaluating width minus one is representable. The source traversal used at row ends begins at the source displaced by width and advances one element per outer iteration. The destination still advances by one written element per outer iteration.

These observations do not establish a portable valid negative-width interface. Negative pointer displacements require suitable storage, and the final signed left shift of a negative width does not have a portable defined result. The most negative signed width also creates problems for width-minus-one and width-negation arithmetic.

### Final target selection and invocation

After all direct transformation writes, the function reads a global function-pointer slot and invokes the obtained target. This contract designates that dynamic invocation **F-indirect-01**.

The slot is storage containing a target pointer; it is not itself the unknown callee. F-indirect-01 identifies the invocation relationship, not a proven immutable function identity.

The three represented arguments are:

1. The destination pointer immediately after the last directly written element.
2. That pointer moved backward by width 16-bit elements.
3. Width shifted left by one bit.

For ordinary positive dimensions, these resemble a next-row pointer, a preceding-row pointer, and a byte count of twice width. This suggests a possible row-copy operation, but does not establish one.

If there are no outer iterations, the first argument is the original destination pointer. The second is width elements before that pointer, even though no row was generated.

No earlier snapshot of the target slot is established. If prior destination writes validly alter its storage, the final target selection depends on the slot contents after those writes. If those contents no longer represent a valid callable target, no successful invocation or recoverable outcome is established.

The supplied representation establishes no function-pointer width, global-slot alignment, atomicity guarantee, or supported mechanism for concurrent slot updates.

## Side effects

Directly established effects are:

- Source-element reads along the executed transformation path.
- Consecutive 16-bit destination writes.
- A read of global function-pointer storage after those writes.
- An indirect invocation using the target obtained from that storage.

No direct allocation, deallocation, or explicit ownership transfer appears. This does not establish anything about allocation, ownership, or release behavior inside F-indirect-01.

The function does directly access global storage to select its final target. It does not explicitly write that slot as a named global operation, but writes through an aliased destination could affect it. Nothing establishes that the dependency preserves the slot or any other global state.

No explicit error result, recovery branch, or validation failure is provided. Invalid accesses, invalid pointer arithmetic, unsupported signed arithmetic, an invalid target pointer, or dependency failure have no established recoverable outcome.

For positive width, a width no greater than `0x3fffffff` avoids overflow in the final signed 32-bit left shift. That bound alone is insufficient: all pointer operations, storage accesses, target selection, and dependency requirements must also be valid.

Normal return is conditional on the dependency returning normally. Its return value, if any, is not exposed as a result of this function.

## Dependencies

| Opaque ID | Established relationship | Unresolved behavior |
|---|---|---|
| F-indirect-01 | Invoked once after the loop, using a target read from global function-pointer storage at that point, with two destination-derived pointers and one width-derived scalar. | Target identity, additional ABI requirements, memory accesses, global effects, ownership behavior, error signaling, return value, and normal return. |

The global target slot is a separate data dependency, not another function dependency. Its initial contents, mutability, storage relationship to caller buffers, and pointer representation are not supplied.

No other external invocation is established by the supplied body.

Moving a pointer by width 16-bit elements and passing twice width as a scalar are distinct operations with distinct validity requirements. Their apparent relationship does not prove a copy contract.

## Acceptance examples

These are conditional static acceptance examples, not observed runtime results. Direct output expectations apply immediately before F-indirect-01 can modify memory.

### Two-column transformation

Given width 2, height 2, valid nonoverlapping storage, and source rows:

- Earlier row: `0x0000`, `0xffff`.
- Later row: `0x0000`, `0x0000`.

The directly generated row is:

- Column zero: `0x39e7`.
- Column one: `0x7bef`.

The first value uses two averages; the second uses one.

After these writes, the target slot is read. The resulting invocation receives the destination advanced by two elements, the original destination, and scalar 4. Whether the target writes another row is **documented_only**.

### Equal operands and field separation

When an ordinary inner operation reads the same 16-bit pattern from both positions, its direct result equals that pattern. Equal row-end operands likewise reproduce the pattern.

For a single averaging stage:

- Combining `0x0020` with zero produces zero, not a bit in the lower field.
- Combining `0x0800` with zero produces zero, not a bit in the middle field.

These outcomes follow from field-boundary masking.

### Width one

Given width 1, height 2, and source element one equal to zero:

- Exactly one direct destination element is written.
- Its pattern is `0x0001`.
- The dependency arguments are the destination advanced by one element, the original destination, and scalar 2.

For width 1 and height `0x00010002`, a zero source operand in the first row-end operation instead produces pattern `0x8001` under the represented 16-bit conversion behavior. The height bit at position 16 shifts into the stored low 16 bits. This distinguishes full-width height arithmetic from prematurely truncating height.

### Width zero

Given width zero, height 2, and source element zero equal to zero:

- One direct destination element is written with pattern `0x0001`.
- The target is then obtained from the global slot.
- Both pointer arguments equal the original destination advanced by one element.
- The scalar argument is zero.

### No outer iterations

Given height 1 and otherwise valid pointer arithmetic:

- The transformation performs no source-element reads or destination writes.
- Initial source-pointer displacement still occurs.
- The global target slot is still read.
- The invocation receives the original destination, that pointer moved backward by width elements, and width shifted left once.

### Target-slot aliasing

Suppose a valid destination alias causes direct writes to change the global target slot, and its contents at invocation form another valid callable target.

The function must select according to the slot contents at the final read, not according to an assumed pre-transformation snapshot. The chosen target receives the same destination-derived arguments specified above.

Whether a particular alias and pointer representation permit such a valid change is **documented_only**. Corrupting the slot does not imply a safe fallback.

## Uncertainties

The following remain **documented_only**:

- The target identity and behavior of F-indirect-01.
- The global target slot’s initial contents, pointer width, alignment, and mutability.
- Whether any supported caller arrangement aliases destination storage with that slot.
- Whether the dependency copies a final row.
- The intended semantic meaning of the packed fields.
- Intended support for zero, negative, or extreme dimensions.
- Exact target rules for out-of-range signed conversions.
- Whether repeated row-end source references become separate physical reads.
- Concurrent modification semantics for source, destination, or target storage.
- Post-invocation memory and global state.
- Dependency ownership, allocation, release, error, and return behavior.

The image-processing interpretation and possible final-row copy are inferred intent only. Neither supplies missing dependency guarantees.

## Evidence

The supplied static representation establishes the parameter widths, loop conditions, pointer progression, masks, direct access sequence, and final indirect invocation.

The 5/6/5 field interpretation follows algebraically from the boundary masks. The row mapping follows from the earlier-row traversal’s initial per-row increment and the later-row traversal’s advancement past each row-end element.

The final invocation obtains its target through global function-pointer storage after the transformation loop. This establishes a direct global read and provides no evidence of a target snapshot before the writes. Potential destination aliasing therefore cannot be excluded from target-selection behavior.

Only the embedded static representation was available. No underlying implementation, external dependency, or runtime behavior was inspected. No runtime tests were performed. The acceptance examples are arithmetic and ordering consequences of that representation, not execution evidence.
