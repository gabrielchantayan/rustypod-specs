<!-- Trusted provenance: F-d1955fffd5ee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d1955fffd5ee

## Purpose

F-d1955fffd5ee converts an unsigned flag value into another unsigned flag value through a fixed bit-selection and bit-remapping rule.

Nine input bits contribute to the result. Six retain their positions; three move to lower positions. Every other input bit is discarded. The function does not preserve flags merely because they occupy positions used in the output.

The established behavior is a numeric transformation. Interpreting the input and output as two different flag encodings is inferred intent; the supplied evidence does not identify the flags’ application-level meanings.

## Interface

### Input

One unsigned integer, supplied by value.

Bit positions in this contract are zero-based: bit 0 has numeric weight 1, bit 1 has numeric weight 2, and so forth.

The relevant input positions are:

- Bits 0, 1, 2, and 3.
- Bits 5 and 6.
- Bit 8.
- Bit 15.
- Bit 21.

All other input positions are ignored.

There are no established restrictions on combinations of relevant bits. Zero, individual flags, and combinations of flags are accepted by the visible transformation. No validation rejects combinations.

### Output

One unsigned integer, returned by value, containing exactly the contributions described below.

| Input bit | Input mask | Output bit | Output mask |
|---|---:|---|---:|
| 0 | `0x000001` | 0 | `0x000001` |
| 1 | `0x000002` | 1 | `0x000002` |
| 2 | `0x000004` | 2 | `0x000004` |
| 3 | `0x000008` | 3 | `0x000008` |
| 5 | `0x000020` | 4 | `0x000010` |
| 6 | `0x000040` | 5 | `0x000020` |
| 8 | `0x000100` | 8 | `0x000100` |
| 15 | `0x008000` | 7 | `0x000080` |
| 21 | `0x200000` | 21 | `0x200000` |

A listed output bit is set if and only if its corresponding input bit is set. All unlisted output bits are clear.

Hexadecimal padding in this document is for readability; it does not declare a storage size or byte layout.

### Width and representation

The supplied representation identifies the argument and result as unsigned integers but does not independently establish their exact ABI width. A 32-bit unsigned interpretation is consistent with the evidence, but the exact width remains **documented_only**.

The transformation explicitly uses bit 21. Its intended flag interpretation therefore requires a value domain capable of representing that bit. On an unsigned interface of at least 22 bits, the mapping above is fully representable without truncation.

The input and output are scalar values, not buffers or serialized records. No byte order, alignment, packed layout, or memory-backed field access is established. Behavior when an external caller converts a signed, oversized, or differently typed value into the argument belongs to that calling interface, not to this transformation.

## Behavior

### Selection and remapping

The result begins with only the contribution from input bit 0: numeric zero when that bit is clear, numeric one when it is set. Subsequent contributions are combined by bitwise union.

The remaining input bits are examined in this order:

1. Bit 1.
2. Bit 2.
3. Bit 21.
4. Bit 3.
5. Bit 5.
6. Bit 6.
7. Bit 8.
8. Bit 15.

Each examination independently determines whether its corresponding output contribution is included. There is no early return or combination-dependent rejection.

This order is established by the supplied static representation. It does not establish an externally observable sequence of memory reads: the function receives one value, and the visible operations do not access memory through it. The intermediate result is not exposed.

### Preserved positions

Input bits 0, 1, 2, 3, 8, and 21 retain their positions in the output.

For example, input bit 21 contributes `0x200000`, not a normalized Boolean value such as one. Only the initial contribution from input bit 0 is expressed through a Boolean conversion, whose numeric result is zero or one.

### Relocated positions

Three input flags change position:

- Input bit 5 becomes output bit 4.
- Input bit 6 becomes output bit 5.
- Input bit 15 becomes output bit 7.

These mappings are independent. Setting both input bits 5 and 6 produces both output bits 4 and 5.

Relocation does not imply that a flag already occupying the destination position in the input is preserved. Input bit 4, for example, contributes nothing despite output bit 4 being a valid result position.

### Ignored bits and output bounds

Any input bit absent from the mapping table has no effect on the result.

Consequently:

- Input bit 4 does not directly produce output bit 4.
- Input bit 7 does not directly produce output bit 7.
- Output bit 6 is always clear.
- Output bit 15 is always clear.
- All output bits above bit 21 are clear, if such positions exist in the interface width.

The union of all possible output contributions is `0x2001BF`. Thus every result contains only bits from that mask, and the maximum numeric result is `0x2001BF`.

All nine output positions have distinct input contributors. There are no collisions in which two selected input flags compete for the same output bit.

### Arithmetic and repeated application

The visible transformation uses flag tests, Boolean normalization, and bitwise union. It performs no addition, multiplication, division, or shift with a variable count. No arithmetic overflow, rounding, or carry behavior participates in the mapping when the stated bit positions are representable.

The function is not generally idempotent. Its output must not automatically be treated as an equivalent input encoding. For example, an input of `0x40` produces `0x20`; applying the function to `0x20` produces `0x10`; applying it to `0x10` produces zero.

Inputs differing only in ignored bits produce equal results. Conversely, differing values of any selected input bit produce a difference at its unique output position.

## Side effects

The visible body has no externally observable side effects beyond returning the result.

Specifically, it contains no:

- Global or static state accesses.
- Pointer dereferences or caller-memory writes.
- Allocation, release, or ownership transfer.
- I/O, logging, synchronization, or callbacks.
- Calls to other functions.

The input is passed by value and is not modified in caller storage. There is no persistent state transition and no retained reference to input data.

These statements describe the supplied function body, not unspecified instrumentation, calling machinery, or external execution conditions.

## Dependencies

No function dependencies are visible.

The transformation depends only on the supplied unsigned value and ordinary unsigned bitwise semantics. There is no unknown callee whose global-state effects, memory ownership, return value, or failure behavior must be assumed.

The exact machine-level calling convention and unsigned integer width are not established by the supplied evidence. No dependency identifier is needed for either absence of information.

## Acceptance examples

These examples are statically derived acceptance criteria, not recorded runtime observations. They assume an unsigned interface wide enough to represent bit 21 and every supplied input value.

| Input | Required result | Behavior distinguished |
|---:|---:|---|
| `0x000000` | `0x000000` | No flags contribute. |
| `0x000001` | `0x000001` | Bit 0 is preserved. |
| `0x00000F` | `0x00000F` | The lowest four bits are preserved together. |
| `0x000010` | `0x000000` | Input bit 4 is ignored. |
| `0x000020` | `0x000010` | Input bit 5 moves to output bit 4. |
| `0x000040` | `0x000020` | Input bit 6 moves to output bit 5. |
| `0x000060` | `0x000030` | The two adjacent remappings contribute independently. |
| `0x000080` | `0x000000` | Input bit 7 is ignored. |
| `0x000100` | `0x000100` | Bit 8 is preserved. |
| `0x008000` | `0x000080` | Input bit 15 moves to output bit 7. |
| `0x200000` | `0x200000` | Bit 21 is preserved. |
| `0x20816F` | `0x2001BF` | Every selected input bit contributes. |
| `0x2081FF` | `0x2001BF` | Additional ignored bits 4 and 7 do not change the result. |

If the interface is established separately as 32-bit unsigned, input `0xFFFFFFFF` must produce `0x002001BF`. This example is conditional on that width; it is not evidence establishing the width.

A conforming implementation must also satisfy these combination properties:

- Adding only ignored input bits leaves the result unchanged.
- Combining two input flag sets by bitwise union produces the union of their separate results.
- A result bit cannot appear without its mapped input contributor.

## Uncertainties

The following remain **documented_only**:

- Exact argument and return width, ABI, and register placement.
- Application-level names and meanings of the flags.
- Whether callers restrict otherwise representable flag combinations.
- The reason for the selected mappings and omitted bits.
- Behavior of external conversions before entry or after return.

The likely purpose is conversion between flag representations. That interpretation is inferred; it is unnecessary to establish the numeric contract.

There is no visible error channel. Zero is a normal result for zero input and for any input containing only ignored bits; it is not established as an error indication.

## Evidence

The contract is based on static inspection of the supplied representation of F-d1955fffd5ee.

That evidence establishes the individual flag tests, their order, their output contributions, the final returned value, and the absence of visible calls or external state accesses. The output mask, bounds, combination properties, repeated-application example, and acceptance values are deductions from those operations.

No runtime tests have been performed. No execution trace, binary-level ABI confirmation, caller behavior, or external flag definition was supplied.
