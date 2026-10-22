<!-- Trusted provenance: F-d8ce763ae82c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d8ce763ae82c

## Purpose

F-d8ce763ae82c subtracts two two-component signed-integer values and writes a two-component result into caller-provided storage. When the initially computed second component is negative, it adds 86,400 to that component and decrements the first component once.

The established behavior is subtraction with a single conditional borrow. The constant 86,400 suggests a representation involving seconds within a day, but the supplied evidence does not establish units, an epoch, or a public time-related type. Those interpretations are inferred intent, not requirements.

This is not a general normalization operation. It does not repeatedly borrow, carry an oversized positive second component, validate input ranges, or allocate result storage.

## Interface

### Inputs and output

The function receives three addresses:

| Address | Role | Required storage |
|---|---|---|
| First operand | Supplies the value being subtracted from | Two readable, consecutive signed-integer elements |
| Second operand | Supplies the value being subtracted | Two readable, consecutive signed-integer elements |
| Result | Receives the difference | Two writable, consecutive signed-integer elements |

For this contract, each pair is described as a **coarse component** followed by a **fine component**. These names distinguish positions and do not establish physical units.

The function has no returned value. Its result is communicated through writes to the result address.

### Width and layout

Each component has the implementation’s signed `int` width, designated **W** bits here. The second component is one such element after the first. Each pair therefore occupies two consecutive `int` elements, with no separate field or pointer between them.

The supplied source does not independently establish the target data model:

- Under a 32-bit `int` interpretation, each component occupies four bytes and each pair occupies eight bytes.
- Under another compatible data model, element width and byte offsets follow that model.
- Byte order is not established and is not used explicitly by the function.

The arithmetic uses signed-integer expressions. The evidence does not justify replacing these with guaranteed modulo-\(2^W\) arithmetic. The constant 86,400 must also be considered in the target’s integer model; it is representable in the ordinary 32-bit signed interpretation.

### Preconditions for a defined result

The supplied addresses must designate live, suitably aligned storage supporting the required reads and writes. No null-address, alignment, accessibility, or bounds checks appear.

For the ordinary signed-arithmetic interpretation used below:

- Both component differences must be representable in the arithmetic type used.
- If borrowing occurs, the coarse difference minus one must be representable.
- Any addition used to adjust the fine difference must be representable.

The contract does not assign a wrapped result to signed overflow. Exact overflow behavior on a particular compiled target is `documented_only`.

The function does not require the three address ranges to be distinct. Overlap can affect the values subsequently read, however, so the non-overlapping mathematical description is not sufficient for every overlapping arrangement.

## Behavior

### Logical access and update order

The supplied source establishes this logical sequence:

1. Read the first operand’s coarse component.
2. Read the second operand’s coarse component.
3. Write their difference to the result’s coarse component.
4. Obtain the two fine components and compute the first operand’s fine component minus the second operand’s fine component.
5. Write that fine difference to the result’s fine component.
6. If that fine difference is negative:
   - Write the fine difference plus 86,400 to the result’s fine component.
   - Write the previously obtained coarse difference minus one to the result’s coarse component.
7. Return without a value.

The relative order of the two fine-component operand reads is not established by the subtraction expression. Both contribute to the fine difference before its result is stored.

This sequence describes source-level accesses and their aliasing consequences. It is not a guarantee of distinct machine instructions, externally observable intermediate stores, atomicity, or memory-ordering behavior for concurrent observers.

### Result with stable, non-overlapping operands

Let the first operand be \((A_c,A_f)\) and the second be \((B_c,B_f)\). Let:

\[
D_c=A_c-B_c,\qquad D_f=A_f-B_f.
\]

Subject to the representability preconditions, the final result is:

| Condition | Final coarse component | Final fine component |
|---|---:|---:|
| \(D_f \geq 0\) | \(D_c\) | \(D_f\) |
| \(D_f < 0\) | \(D_c-1\) | \(D_f+86{,}400\) |

A zero fine difference does not trigger borrowing. A negative coarse difference alone does not trigger any correction.

The correction is performed once. If the original fine difference is less than −86,400, the final fine component remains negative. If the original fine difference is at least 86,400, it remains oversized because the function performs no positive carry.

If both input fine components lie in the range 0 through 86,399, their difference lies between −86,399 and 86,399. In that restricted case, the single correction leaves the final fine component in the range 0 through 86,399, assuming the coarse arithmetic is representable. That range is a sufficient caller condition, not a validation performed by the function.

### Overlap and in-place operation

The initial coarse values are obtained before the first result write. The borrow adjustment uses those previously obtained values rather than rereading the operands’ coarse components.

Consequently, when the operands otherwise occupy distinct ordinary storage, placing the result exactly over either operand supports the same final arithmetic result:

- Overwriting that operand’s coarse component cannot change its already obtained coarse value.
- Its fine component is read before the result fine component is written.
- The conditional coarse rewrite does not depend on rereading the overwritten storage.

This does not establish safety for every overlap pattern. In particular, the first result write occurs before either fine-component value is needed. If the result coarse component overlaps an operand’s fine component, the write can change the subsequent fine subtraction.

The source contains no copy of both complete operand pairs before output begins. Callers needing arithmetic on the operands’ original values must account for that distinction.

## Side effects

The function writes the result’s coarse component and then its fine component. On the negative-fine-difference path, it subsequently writes the fine component again and then the coarse component again.

For distinct operand and result storage, no operand is written. If storage overlaps, a result write can also modify memory designated as part of an operand.

No explicit allocation, deallocation, ownership transfer, global access, input/output operation, or dependency call appears in the supplied function body. The caller remains responsible for providing and maintaining the storage. The function does not release it.

There is no explicit error channel, validation failure, recovery path, or rollback. Invalid storage or unrepresentable signed arithmetic falls outside the defined-result preconditions. This contract does not specify whether a particular target traps, continues, or exhibits another outcome in those cases.

Nothing in the evidence establishes thread safety, atomic pair updates, or a synchronization protocol.

## Dependencies

No other opaque function ID is invoked by F-d8ce763ae82c in the supplied body. The provided dependency mapping identifies this function itself; it does not establish an additional callee.

The behavior depends on:

- Caller-provided storage, lifetime, alignment, and access permissions.
- The target’s signed-integer widths, representation, and expression semantics.
- The alias relationships among the supplied address ranges.

There is no unknown callee whose memory ownership, global-state behavior, or preservation guarantees need to be assumed. Compiler-generated support, if any, is not established by the supplied evidence.

## Acceptance examples

These are conditional expectations derived from the source, not runtime observations. Except for the overlap example, assume stable, distinct storage, ordinary signed arithmetic capable of representing every intermediate value, and sufficient writable result storage.

| First operand | Second operand | Expected final result | Condition illustrated |
|---|---|---|---|
| \((10,500)\) | \((3,200)\) | \((7,300)\) | Nonnegative fine difference |
| \((10,200)\) | \((3,500)\) | \((6,86{,}100)\) | One borrow |
| \((10,500)\) | \((3,500)\) | \((7,0)\) | Equality does not borrow |
| \((3,100)\) | \((5,200)\) | \((-3,86{,}300)\) | Borrow further decreases a negative coarse difference |
| \((10,0)\) | \((3,86{,}400)\) | \((6,0)\) | Fine difference exactly −86,400 |
| \((10,0)\) | \((3,90{,}000)\) | \((6,-3{,}600)\) | No repeated normalization |
| \((10,90{,}000)\) | \((3,0)\) | \((7,90{,}000)\) | No positive carry |

### Exact in-place result

Given distinct operands \((10,200)\) and \((3,500)\), placing the result exactly over the first operand produces \((6,86{,}100)\) in that operand’s storage. The second operand remains unchanged.

The same arithmetic outcome holds when the result instead exactly overlays the second operand, provided no additional overlap changes the relevant reads.

### Shifted overlap

Suppose a three-element backing region initially begins with 10 and 5. The first operand starts at the first element, while the result starts at the second element. The separate second operand is \((3,7)\).

The initial coarse subtraction writes 7 into the backing region’s second element. That element is also the first operand’s fine component. The subsequent fine subtraction therefore uses 7 minus 7, producing zero without borrowing.

The result pair becomes \((7,0)\), and the three backing elements become \((10,7,0)\). Arithmetic on the original operand values would instead have produced \((6,86{,}398)\). This difference follows from the established access order.

### Coarse underflow boundary

If the initial coarse difference is the minimum representable signed value and the fine difference is negative, the required decrement is not representable. Such an input does not have a defined result under the signed-arithmetic preconditions above; a wrapped result must not be accepted without additional target evidence.

## Uncertainties

- **`documented_only`: Exact integer width and byte layout.** The source identifies `int` elements, not a complete target ABI.
- **`documented_only`: Overflow behavior of the compiled artifact.** Signed wrapping is not established.
- **`documented_only`: Domain meaning.** Days and seconds are plausible interpretations of the components, not established interface requirements.
- **`documented_only`: Intended input ranges.** Canonical fine-component ranges are not checked or declared.
- **`documented_only`: Supported overlap policy.** Overlap effects can be derived from the body, but no separate interface promise requiring distinct storage is supplied.
- **`documented_only`: Machine-level access visibility.** Instruction selection, store elimination, atomicity, and concurrent observation are not established.

## Evidence

The established behavior comes solely from the supplied function body: two initial coarse reads, an initial coarse write, fine-component subtraction and write, a negative-result condition, a single addition of 86,400, and a coarse decrement using the earlier coarse values.

The single-borrow interpretation follows directly from that structure. The possible day-and-second interpretation follows only from the correction constant and remains inferred intent.

No runtime tests, compilation checks, ABI inspection, or external dependency analysis have been performed. The acceptance examples are static consequences under their stated conditions, not execution evidence.
