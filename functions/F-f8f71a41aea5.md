<!-- Trusted provenance: F-f8f71a41aea5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8f71a41aea5

## Purpose

F-f8f71a41aea5 expands three packed unsigned color components into four consecutive output bytes. The first three bytes contain expanded 5-bit, 6-bit, and 5-bit components, respectively. The fourth byte is always 255.

The apparent intent is conversion from an RGB565-style packed value to an opaque RGBA-style byte sequence. The component extraction and byte order are established by the supplied static evidence; the color-space interpretation and downstream use are inferred intent. No color-space conversion, gamma correction, interpolation, or premultiplication is present.

Expansion uses bit replication rather than exact proportional scaling with rounding. This distinction is part of the behavior and can affect intermediate component values.

## Interface

The function accepts two inputs:

- **Destination:** an address identifying the first byte of a writable region containing at least four consecutive bytes.
- **Packed value:** an unsigned integer passed by value. Its low 16 bits supply all three components. Higher bits do not affect the four output bytes under the integer-width interpretation described below.

There is no explicit return value. The result is delivered entirely through the destination.

The output layout is defined by increasing byte address:

| Byte offset | Meaning | Output width |
|---|---|---|
| 0 | Expanded component from packed bits 15–11 | 8 bits |
| 1 | Expanded component from packed bits 10–5 | 8 bits |
| 2 | Expanded component from packed bits 4–0 | 8 bits |
| 3 | Constant opaque value, 255 | 8 bits |

This is a byte layout, not a specification of a single 32-bit integer’s numeric value. Interpreting the four bytes as a native integer would introduce an endianness-dependent interpretation outside this function’s contract.

The supplied representation uses byte-sized output objects and an unsigned integer input. The arithmetic below uses an 8-bit-byte, 32-bit-unsigned interpretation. The displayed type notation does not independently establish the target ABI, so those physical widths remain a documented qualification of the contract. In particular, the large left shift requires an unsigned arithmetic width sufficient for that shift.

The caller must provide valid writable storage for all four output bytes. No input length, destination capacity, or validity flag is supplied. No alignment requirement beyond that needed for byte access is expressed.

The destination’s initial contents are not inputs: all four result bytes are assigned without reading their previous values.

## Behavior

### Component selection

The packed value supplies three unsigned fields:

- Bits 15 through 11 form a 5-bit first component, ranging from 0 through 31.
- Bits 10 through 5 form a 6-bit second component, ranging from 0 through 63.
- Bits 4 through 0 form a 5-bit third component, ranging from 0 through 31.

There is no sign extension. Bits above bit 15 do not contribute to the result under the stated width interpretation.

The function does not read a packed value from memory through the destination. The packed value is already an input scalar, so its interpretation does not depend on the byte order of any external packed representation.

### Component expansion

For a 5-bit component with numeric value \(x\), the output is:

\[
8x+\left\lfloor x/4\right\rfloor
\]

Equivalently, the five original component bits occupy output bits 7 through 3, and the original three most significant component bits are repeated into output bits 2 through 0.

For the 6-bit component with numeric value \(y\), the output is:

\[
4y+\left\lfloor y/16\right\rfloor
\]

Equivalently, the six original component bits occupy output bits 7 through 2, and the original two most significant component bits are repeated into output bits 1 through 0.

The additive descriptions are exact because the shifted component and repeated bits occupy disjoint positions. They do not describe an extra rounding step.

Each expansion maps zero to zero and the maximum source component to 255. Each mapping is strictly increasing over its component’s input range, but neither produces every possible 8-bit value.

This behavior is not interchangeable with rounding an exact multiplication by \(255/31\) or \(255/63\). For example, a 5-bit component of 3 produces 24 through bit replication, whereas exact proportional scaling rounded to the nearest integer would produce 25.

### Arithmetic widths and truncation

Under the 32-bit-unsigned interpretation, the third-component computation includes an unsigned left shift by 27 positions followed by an unsigned right shift by 24 positions. The left shift discards original bits above bit 4 through unsigned-width truncation. The subsequent right shift places the retained five bits in output positions 7 through 3.

That width-dependent expression must not be evaluated as an unbounded mathematical integer expression without also accounting for the byte conversion. Its established result under the stated model is the same 5-bit expansion used for the first component.

All final component values fit within an unsigned byte:

- The two 5-bit expansions range from 0 through 255.
- The 6-bit expansion ranges from 0 through 255.
- The fourth byte is exactly 255.

There is no saturation stage, error indication, or rejection of packed values whose higher bits are nonzero.

### Output order and completion

The displayed source establishes the following logical assignment order:

1. Replace destination byte 0 with the expanded first component.
2. Replace destination byte 1 with the expanded second component.
3. Replace destination byte 2 with the expanded third component.
4. Replace destination byte 3 with 255.

There are no conditional branches or dependency calls between these assignments.

On ordinary valid storage, successful completion leaves exactly the specified four-byte result. The source does not describe any writes outside those four bytes.

The logical assignment order does not, by itself, establish machine-instruction ordering, atomicity, or guarantees for concurrent observers. The destination is not presented as volatile or atomic storage, and the supplied evidence is not an execution trace.

## Side effects

The explicit side effect is replacement of four consecutive destination bytes. Existing contents are discarded rather than combined with the result.

The function does not explicitly:

- Allocate or release memory.
- Acquire or transfer ownership.
- Retain the destination address.
- Read or modify a named global object.
- Perform input/output.
- Invoke another function.
- Return a success or failure status.

The caller retains responsibility for the destination’s lifetime and accessibility. If the caller supplies an address into shared or global storage, the four writes modify that storage through the supplied address; the function does not distinguish storage ownership or allocation class.

No source-level read of the destination occurs. Storage neighboring the four-byte output region is not part of the specified result.

Invalid storage is outside the valid-input contract. There is no explicit null check, bounds check, or fault recovery. A null, dangling, read-only, or insufficiently sized destination does not have a defined error result here. A fault, termination, partial modification, or some other platform-dependent consequence must not be elevated into a promised behavior.

The operation is not specified as transactional. No rollback path is present, and callers must not rely on all-or-nothing publication of the four bytes.

## Dependencies

No opaque function dependencies are present in the supplied body.

The contract depends only on ordinary unsigned bit operations, byte conversion, address arithmetic for four consecutive bytes, and writable destination storage.

No callee behavior, global preservation, allocation policy, or ownership convention is needed to explain the visible transformation.

The execution environment’s memory-fault handling and concrete machine access behavior are outside the supplied evidence.

## Acceptance examples

The following are conditional acceptance examples, not runtime observations. They assume 8-bit output bytes, the stated unsigned arithmetic model, valid destination storage, and execution without an external memory fault.

Output sequences are listed in increasing destination-address order.

| Packed value | Extracted components | Expected output bytes |
|---|---|---|
| `0x0000` | 0, 0, 0 | `00 00 00 FF` |
| `0xFFFF` | 31, 63, 31 | `FF FF FF FF` |
| `0xF800` | 31, 0, 0 | `FF 00 00 FF` |
| `0x07E0` | 0, 63, 0 | `00 FF 00 FF` |
| `0x001F` | 0, 0, 31 | `00 00 FF FF` |
| `0x0821` | 1, 1, 1 | `08 04 08 FF` |
| `0x8410` | 16, 32, 16 | `84 82 84 FF` |
| `0x1800` | 3, 0, 0 | `18 00 00 FF` |

Additional acceptance conditions:

- **Ignored upper bits:** `0xABCD8410` produces the same four bytes as `0x00008410`: `84 82 84 FF`.
- **Independence from previous contents:** starting with arbitrary destination bytes does not change the expected result for a given packed value.
- **Fixed fourth byte:** the fourth byte is `FF` even when every packed component is zero.
- **Field separation:** changing only bits 4–0 changes only output byte 2; it does not change output bytes 0, 1, or 3.
- **Write extent:** for a destination inside a larger valid ordinary byte buffer, bytes outside offsets 0 through 3 are not modified by the source-described operation.
- **Replication rather than rounded scaling:** the `0x1800` example must produce first byte `18`, not `19`.

No acceptance expectation is assigned to invalid destination addresses or insufficiently sized storage.

## Uncertainties

The following remain **documented_only**:

- **Concrete ABI widths:** the displayed type notation supports the intended byte and unsigned arithmetic interpretation, but no independent target type definitions or ABI evidence were supplied.
- **Color semantics:** “RGB565” and “RGBA” describe the apparent layout intent. No supplied consumer establishes channel naming, color space, or alpha interpretation.
- **Physical memory accesses:** instruction selection, store merging, processor ordering, atomicity, and memory-mapped-device effects are not established.
- **Invalid-memory outcomes:** no portable failure mode or partial-write boundary is established.
- **Concurrent access:** no synchronization mechanism or concurrency guarantee is shown.

These unknowns do not alter the specified four-byte transformation for valid ordinary storage under the stated width interpretation.

## Evidence

The supplied static body establishes three component masks or equivalent bit selections, unsigned shifts and byte conversions, four sequential byte assignments, and a constant final byte of 255. It contains no explicit dependency calls, destination reads, conditional paths, allocation operations, or error-reporting mechanism.

The expansion formulas, output examples, field independence, and ignored-upper-bit behavior are mathematical consequences of those operations under the stated width interpretation.

The color-format interpretation is inferred intent, not independently established metadata.

No runtime tests have been performed.
