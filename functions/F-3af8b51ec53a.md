<!-- Trusted provenance: F-3af8b51ec53a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3af8b51ec53a

## Purpose

F-3af8b51ec53a constructs an unsigned flag value from selected bits of one unsigned input. Four low-order flags and one higher-order flag retain their positions; four other low-order flags move to different output positions. All other input bits are discarded.

The established behavior is a fixed, independent mapping of individual bits. The function does not interpret combinations as enumerated values, reject unsupported flags, or preserve unrecognized bits.

A likely intent is conversion between two flag representations. That interpretation is inferred only: the supplied evidence provides no flag names, domain meanings, or consuming interface.

## Interface

### Input

One unsigned integer value, interpreted as a bit field. Bit positions in this contract are zero-based, with bit 0 the least significant bit.

The supplied declaration identifies the type as unsigned but does not supply its underlying type definition or ABI. The highest examined position is bit 21, corresponding to mask `0x00200000`.

Consequently:

- The mapping is fully specified for an unsigned representation with at least 22 value bits.
- A 32-bit unsigned interface is a plausible interpretation, not independently established by the supplied material.
- Examples involving bit 31 or an eight-digit all-ones value are conditional on a 32-bit interface.
- For a wider interface, additional high input bits are also ignored.

No pointer, buffer, length, context object, or explicit state handle is an input. There is no byte-oriented layout or alignment requirement established for callers.

### Output

One unsigned integer containing only mapped flags. Every possible result is a subset of the output mask `0x0020816F`.

The output uses positions 0, 1, 2, 3, 5, 6, 8, 15, and 21. Every other output position is zero.

The result is a value, not an allocation or a reference to retained storage. There is no ownership transfer, release obligation, or indicated lifetime dependency.

### Preconditions and errors

There is no content-based precondition on the input. Zero, individual flags, mixed supported and unsupported flags, and all representable input bits set are handled by the same mapping.

Unsupported input bits are silently discarded. The function exposes no separate error indicator, error result, diagnostic, or failure branch. Zero is an ordinary mapped result; it must not be interpreted as an error merely because it is zero.

## Behavior

### Exact mapping

Each output flag is present if and only if its corresponding input flag is present.

| Input position | Input mask | Output position | Output mask |
|---|---|---|---|
| 0 | `0x00000001` | 0 | `0x00000001` |
| 1 | `0x00000002` | 1 | `0x00000002` |
| 2 | `0x00000004` | 2 | `0x00000004` |
| 3 | `0x00000008` | 3 | `0x00000008` |
| 4 | `0x00000010` | 5 | `0x00000020` |
| 5 | `0x00000020` | 6 | `0x00000040` |
| 6 | `0x00000040` | 8 | `0x00000100` |
| 7 | `0x00000080` | 15 | `0x00008000` |
| 21 | `0x00200000` | 21 | `0x00200000` |

The complete set of recognized input positions is represented by `0x002000FF`.

The mapping is independent for each row. Selecting multiple recognized input bits produces the union of their corresponding output bits. No recognized input bit suppresses another, and no output position has multiple input sources.

### Input and output positions are distinct roles

An input bit is not preserved merely because its position appears in the output mask.

For example:

- Input position 5 maps to output position 6, not output position 5.
- Input position 6 maps to output position 8, not output position 6.
- Input position 8 is ignored, despite position 8 being a possible output.
- Input position 15 is ignored, despite position 15 being a possible output.

All mappings depend on the original input value. Newly selected output bits do not become additional input flags during the same invocation. In particular, input `0x00000010` produces `0x00000020` only; it does not trigger a second mapping to `0x00000040`.

### Ignored bits and information loss

Any input bit outside `0x002000FF` has no effect on the result. Inputs differing only in ignored positions therefore produce identical results.

This conversion loses information whenever ignored bits are present. A consumer cannot reconstruct the complete original input from the return value.

Within the recognized input positions, the mapping has no collisions: every recognized input flag has a distinct output flag. Thus the recognized portion of the input is uniquely represented in the result, even though the full input is not.

### Width and arithmetic

The transformation uses bit selection and bit union. It does not rely on addition, multiplication, signed comparison, or arithmetic wraparound.

For an unsigned type capable of representing bit 21:

- No mapped result exceeds `0x0020816F`.
- No recognized input requires a position above bit 21.
- There is no overflow-dependent case.
- Every output bit above position 21 is zero.

If the calling environment accepts a wider value and narrows it before entry, that conversion is outside this function’s established behavior. Likewise, parsing negative text or converting a signed caller value is not specified here.

### Evaluation and observable order

The supplied static evidence considers input positions in this order: 0, 1, 2, 21, 3, 4, 5, 6, and 7, then returns the constructed value.

That order does not create a caller-visible sequence of flag updates. The input is a value parameter; the function contains no explicit external memory reads, external writes, or calls through which intermediate construction could be observed. The externally exposed result is the completed return value.

This contract does not require a particular machine-instruction sequence, branch layout, timing profile, or number of register operations.

## Side effects

The supplied body establishes no explicit side effects beyond returning the mapped value:

- No global or caller-provided storage is accessed.
- No memory is allocated, released, or retained.
- No input object is mutated.
- No I/O, callback, synchronization operation, or external service invocation appears.
- No persistent state is updated.

Identical unsigned inputs produce identical results under the stated width interpretation. There is no visible state whose prior contents influence the mapping.

These statements describe the supplied function body, not unspecified instrumentation or platform machinery. Register preservation, stack layout, tracing inserted elsewhere, and the broader runtime’s behavior are not established.

## Dependencies

No function dependencies are present in the supplied body. No additional opaque function IDs are needed.

The transformation depends only on unsigned bitwise semantics and a representation capable of holding its stated masks. There is no unknown callee whose effects on globals, ownership, or memory release must be assumed.

F-3af8b51ec53a is the sole function identified by this contract.

## Acceptance examples

These are static acceptance criteria derived from the supplied mapping. They are not reports of executed tests.

| Condition | Input | Required output | Distinguishing behavior |
|---|---|---|---|
| No flags | `0x00000000` | `0x00000000` | Empty selection remains empty. |
| Preserved low flags | `0x0000000F` | `0x0000000F` | Positions 0–3 retain their positions. |
| First moved flag | `0x00000010` | `0x00000020` | Mapping occurs once, without cascading. |
| Second moved flag | `0x00000020` | `0x00000040` | Input position 5 is not preserved. |
| Third moved flag | `0x00000040` | `0x00000100` | Input position 6 moves to position 8. |
| Fourth moved flag | `0x00000080` | `0x00008000` | Input position 7 moves to position 15. |
| Higher preserved flag | `0x00200000` | `0x00200000` | Position 21 remains independent. |
| Output-only position supplied | `0x00000100` | `0x00000000` | Input position 8 is ignored. |
| Another output-only position supplied | `0x00008000` | `0x00000000` | Input position 15 is ignored. |
| Mixed recognized and ignored bits | `0x00000111` | `0x00000021` | Ignoring position 8 does not affect positions 0 and 4. |
| All recognized flags | `0x002000FF` | `0x0020816F` | Every mapped output position is selected. |
| All bits set, if input is 32-bit | `0xFFFFFFFF` | `0x0020816F` | Unsupported bits do not pass through. |
| Only unsupported bits, if input is 32-bit | `0xFFDFFF00` | `0x00000000` | No recognized input position is selected. |

For any representable input, adding or removing only ignored flags must leave the output unchanged. Combining recognized flags must combine their mapped outputs without introducing additional positions.

## Uncertainties

The following remain **documented_only**:

- **Exact ABI width:** The underlying unsigned type definition is absent. A 32-bit interpretation is inferred, while the mapping requires only enough width to represent bit 21.
- **Calling convention and binary interface:** Register assignments, argument passing, return transport, and symbol linkage are not supplied.
- **Flag meanings:** No semantic labels or domain-specific validity rules are established.
- **Caller expectations:** Whether callers use this conversion for serialization, permissions, platform adaptation, or another purpose is unknown.
- **Runtime characteristics:** Timing, instruction selection, compiler transformations, and execution behavior have not been measured.

These unknowns do not alter the established bit mapping. They must not be replaced with assumptions about flag ownership, external state, validation, or errors.

## Evidence

The supplied static body establishes an unsigned value parameter, nine independent input-bit selections, construction of a result using only the listed output masks, and return of that result. It contains no explicit calls or external storage operations.

The mask bounds, information-loss properties, non-cascading behavior, and acceptance examples follow from that static mapping. The proposed flag-conversion purpose and possible 32-bit ABI are interpretations, not independently verified facts.

No runtime tests have been performed. No external documentation, caller behavior, binary execution, or type-definition evidence was supplied.
