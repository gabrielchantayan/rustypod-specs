<!-- Trusted provenance: F-67cbddfbaae0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-67cbddfbaae0

## Purpose

F-67cbddfbaae0 converts one integer selector into a fixed 64-bit result. Recognized selectors produce a single-bit value; all other selectors produce zero. The mapping is sparse and nonmonotonic: the selector is not generally the position of the returned bit.

The established behavior is a numeric mapping, not validation against a contiguous range. In particular, selectors 68 and 69 have nonzero results even though many smaller positive selectors return zero.

**Inferred intent:** the results resemble capability, category, or feature masks. No meaning for the individual bits is established. Consumers must not infer names, permissions, or supported operations from this function alone.

## Interface

### Input

One integer selector, interpreted as a signed 32-bit value under the width model represented by the supplied source.

There is no pointer, length, buffer, context object, or explicit state parameter. Every representable selector has a defined numeric outcome within that model. The function imposes no visible requirement that a selector be positive or belong to a declared enumeration.

The supported input domain therefore includes:

- Negative values.
- Zero.
- Every explicitly mapped positive selector.
- Unmapped values between mapped selectors.
- Values greater than the largest mapped selector.

Conversion of a wider argument into this input type is a caller or ABI concern, not behavior specified by this function.

### Output

One signed 64-bit integer. Every established result is nonnegative and is either zero or a power of two.

The largest result is `0x0080000000000000`, equal to \(2^{55}\). It fits within a signed 64-bit integer without setting the sign bit. Results above bit 31 must not be truncated to 32 bits.

The return value is delivered by value. There is no returned allocation, borrowed reference, ownership token, or separately populated output parameter.

### Width and representation boundary

The supplied representation uses an integer input and a long-long integer result. This contract uses the conventional 32-bit input and 64-bit output interpretation. Exact platform type widths, calling convention, register placement, and byte order are not independently established by the supplied material.

The hexadecimal values below specify numbers, not byte sequences. No serialized or in-memory output layout is exposed.

## Behavior

### Exact nonzero mapping

The following table is exhaustive for nonzero results. Bit positions are zero-based: bit 0 has numeric value 1.

| Selector, decimal | Result, hexadecimal | Set bit |
|---:|---:|---:|
| 1 | `0x0000000000100000` | 20 |
| 2 | `0x0000000000000001` | 0 |
| 3 | `0x0000000000000002` | 1 |
| 4 | `0x0000000000000004` | 2 |
| 5 | `0x0000000000000008` | 3 |
| 6 | `0x0000000000000010` | 4 |
| 7 | `0x0000000000000020` | 5 |
| 8 | `0x0000000000000040` | 6 |
| 9 | `0x0000000000000080` | 7 |
| 10 | `0x0000000000000100` | 8 |
| 11 | `0x0000000000000200` | 9 |
| 12 | `0x0000000000000400` | 10 |
| 13 | `0x0000000000000800` | 11 |
| 14 | `0x0000000000080000` | 19 |
| 15 | `0x0000000000020000` | 17 |
| 16 | `0x0000000002000000` | 25 |
| 17 | `0x0000000000200000` | 21 |
| 18 | `0x0000000004000000` | 26 |
| 19 | `0x0000000008000000` | 27 |
| 20 | `0x0000000010000000` | 28 |
| 21 | `0x0000000000040000` | 18 |
| 22 | `0x0000000020000000` | 29 |
| 23 | `0x0000000040000000` | 30 |
| 24 | `0x0000000100000000` | 32 |
| 25 | `0x0000000200000000` | 33 |
| 29 | `0x0000004000000000` | 38 |
| 31 | `0x0000000080000000` | 31 |
| 34 | `0x0000008000000000` | 39 |
| 35 | `0x0000010000000000` | 40 |
| 37 | `0x0000020000000000` | 41 |
| 39 | `0x0000080000000000` | 43 |
| 40 | `0x0000100000000000` | 44 |
| 54 | `0x0000200000000000` | 45 |
| 68 | `0x0040000000000000` | 54 |
| 69 | `0x0080000000000000` | 55 |

There are 35 distinct nonzero mappings. Each produces exactly one set bit, and no two mapped selectors produce the same result.

### Zero-result selectors

Every selector absent from the table returns zero.

Within the inclusive interval from 0 through 69, the zero-result selectors are:

- 0.
- 26–28.
- 30.
- 32–33.
- 36.
- 38.
- 41–53.
- 55–67.

Every negative selector and every selector greater than 69 also returns zero. These are ordinary return outcomes, not a separately signaled error path.

Zero does not distinguish an explicitly handled empty mapping from an otherwise unrecognized value. A caller cannot use the result alone to recover which kind of zero-result selector was supplied.

### Selection and evaluation order

The supplied behavior first resolves mappings in the lower selector group, including selector 54. Paths that do not return a result there proceed to the exceptional mappings for selectors 68 and 69. Selector 68 is distinguished before selector 69.

These comparisons inspect only the supplied scalar argument. There are no externally visible intermediate reads, writes, or calls whose ordering a consumer could observe. Accordingly, this contract requires the exact numeric mapping, not a particular comparison sequence or lookup strategy.

### Width-sensitive results

Several mappings cross boundaries that a replacement implementation must preserve:

- Selector 31 produces \(2^{31}\), a positive 64-bit result. It is not a negative result obtained by sign-extending a signed 32-bit value.
- Selector 24 produces \(2^{32}\), which cannot survive conversion through a 32-bit result container.
- Selector 69 produces \(2^{55}\). The supplied arithmetic widens an unsigned value before shifting it left by 32 bits. Its result must not be computed as a 32-bit shift that loses the high bits or exceeds the operand width.

No result requires signed overflow, saturation, wraparound, or use of bit 63. An implementation may choose another calculation method, provided it returns the exact values above.

## Side effects

The supplied body shows no externally visible side effects beyond delivering the return value.

Specifically, it contains no visible:

- Reads from or writes to global, shared, or caller-owned memory.
- Pointer dereferences.
- Allocation, deallocation, retention, or ownership transfer.
- File, network, console, or device operations.
- Callback invocation or dependency call.
- Error-channel update or diagnostic emission.

There is therefore no exposed memory access sequence, buffer aliasing rule, synchronization protocol, or persistent state transition to preserve.

For a given input, the visible body determines the same output without consulting prior invocations or ambient state. This supports treating the numeric operation as deterministic and independent across calls. It does not establish properties of an unseen runtime wrapper, instrumentation layer, or ABI mechanism.

No cleanup obligation is transferred to the caller.

## Dependencies

No dependency calls are present in the supplied body, and the provided dependency map is empty. No additional opaque function IDs are needed to describe this behavior.

The function’s numeric behavior does not rely on an unknown callee preserving globals, supplying a value, or releasing memory. There is no evidence of such a callee.

Caller-side interpretation is outside this contract. In particular, whether a consumer combines these results with bitwise operations, uses them as lookup keys, or treats zero as an application error is not established.

## Acceptance examples

The following are conditional acceptance criteria derived from the static evidence. They are not reports of executed tests.

### Ordinary and reordered mappings

- Given selector 2, an accepting implementation returns `0x0000000000000001`.
- Given selector 13, it returns `0x0000000000000800`.
- Given selector 1, it returns `0x0000000000100000`, not bit 1.
- Given selector 14, it returns bit 19; given selector 15, it returns bit 17. Increasing the selector does not imply increasing the result.

These cases distinguish the specified mapping from a simple selector-based shift.

### Crossing the 32-bit boundary

- Given selector 23, the result is `0x0000000040000000`.
- Given selector 31, the result is `0x0000000080000000`, still positive.
- Given selector 24, the result is `0x0000000100000000`, not zero.
- Given selector 25, the result is `0x0000000200000000`.

An implementation that truncates to 32 bits or sign-extends the selector-31 result does not satisfy the contract.

### Sparse high mappings

- Given selector 53, the result is zero.
- Given selector 54, the result is `0x0000200000000000`.
- Given selector 55, the result is zero.
- Given selector 67, the result is zero.
- Given selector 68, the result is `0x0040000000000000`.
- Given selector 69, the result is `0x0080000000000000`.
- Given selector 70, the result is zero.

These cases require recognition of the two exceptional high selectors without extending their mapping to neighboring values.

### Unmapped and boundary inputs

Under the stated signed 32-bit input model:

- Selector 0 returns zero.
- Selector −1 returns zero.
- Selectors 26, 30, and 41 return zero.
- The minimum input, −2,147,483,648, returns zero.
- The maximum input, 2,147,483,647, returns zero.

No separate failure indication is required for any of these inputs.

### Repeated calls

Given a sequence of selectors 69, 0, 2, and 69, the required result sequence is \(2^{55}\), zero, 1, and \(2^{55}\). The visible behavior provides no basis for a previous call changing a later result.

## Uncertainties

### Established static evidence

The exact selector-to-result relationship, the absence of visible dependency calls, and the absence of visible external memory access are established by the supplied body. The table and zero-result rule capture its complete numeric behavior under the stated width model.

### Inferred intent

The single-bit outputs suggest translation from identifiers to masks. This is an interpretation, not an established semantic label. No selector names or bit meanings are specified.

### documented_only

The following remain unknown and are not guessed:

- The original platform’s verified type widths and ABI.
- Whether callers restrict the selector to a narrower semantic domain.
- Whether zero denotes unsupported, reserved, disabled, invalid, or intentionally empty.
- The meaning of any individual returned bit.
- Whether surrounding software attaches diagnostics or additional effects to invocation.
- Timing, generated instruction sequence, and machine-level calling behavior.

These unknowns do not alter the numeric mapping stated here.

## Evidence

This contract is based solely on static examination of the supplied function body and the empty dependency map.

The supplied body establishes explicit constant returns for the mapped selectors, zero for the remaining paths, and a widened left shift yielding the selector-69 result. It exposes no calls or external state operations.

No runtime tests, executable inspection, caller analysis, or independent ABI verification have been performed. The acceptance examples specify required behavior; they are not observed execution results.
