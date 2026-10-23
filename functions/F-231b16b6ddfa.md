<!-- Trusted provenance: F-231b16b6ddfa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-231b16b6ddfa

## Purpose

F-231b16b6ddfa expands the four nibbles of the low 16 bits of an unsigned input into four consecutive output bytes. Each nibble is duplicated into both halves of its output byte.

The most significant nibble of the selected 16-bit value becomes the first output byte; the least significant nibble becomes the fourth. For example, the selected value `0x1234` produces bytes `0x11`, `0x22`, `0x33`, and `0x44`, in that order.

This is a fixed-size, in-place write to caller-provided storage. It is not a textual hexadecimal conversion: it produces four binary bytes, without a terminator, prefix, separator, or length field.

The nibble expansion is established by the supplied static evidence. Any application-level interpretation—such as expanding packed channel values—is inferred intent, not part of the contract.

## Interface

### Inputs

| Input | Contract |
|---|---|
| Destination | Pointer to the first of at least four consecutive writable bytes. |
| Value | Unsigned scalar whose low 16 bits supply the four input nibbles. The supplied representation uses an unsigned integer conventionally interpreted as 32 bits. |

The byte layout below uses eight-bit bytes and a 32-bit unsigned input model. The supplied material does not independently establish the target’s type definitions or ABI; that limitation is recorded under **Uncertainties**.

For a 32-bit input, bits 31 through 16 do not affect any output byte. The complete input remains a by-value scalar; the function does not receive a pointer to its storage.

### Output

There is no return value. On normal completion with valid destination storage, the four-byte destination region contains the expanded nibbles.

Let `V` be the low 16 bits of the input. The output layout is:

| Byte offset | Selected bits of `V` | Output byte |
|---|---|---|
| 0 | 15–12 | Selected nibble repeated twice |
| 1 | 11–8 | Selected nibble repeated twice |
| 2 | 7–4 | Selected nibble repeated twice |
| 3 | 3–0 | Selected nibble repeated twice |

For a selected nibble `n`, where `0 ≤ n ≤ 15`, the output byte is `17 × n`. Thus every produced byte has equal high and low nibbles.

### Preconditions

The destination must identify a live writable region covering offsets 0 through 3. No size argument is provided, and no bounds check is present.

A null destination, an undersized region, a read-only region, or a pointer outside valid storage does not meet the interface preconditions. The function provides no defined error result for these cases.

The supplied byte-oriented accesses impose no visible multibyte alignment requirement. They do not establish suitability for device memory, concurrent access, or any particular atomicity requirement.

## Behavior

### Value transformation

The function selects the input’s low four nibbles in descending significance. Each selected four-bit value becomes an eight-bit value by occupying both the high and low nibble positions.

This is equivalent to an exact unsigned four-bit-to-eight-bit endpoint-preserving expansion:

- `0x0` becomes `0x00`.
- `0x1` becomes `0x11`.
- `0x7` becomes `0x77`.
- `0x8` becomes `0x88`.
- `0xF` becomes `0xFF`.

There is no rounding decision, saturation, lookup table, signed interpretation, or data-dependent branch. Every possible 16-bit selected value has a defined four-byte result under the stated type model.

The transformation is not ordinary zero-extension. A selected nibble of `0xA` becomes `0xAA`, not `0x0A`. It is also not a four-byte representation of the original integer.

### Write sequence and reads

At the source-level abstract-machine boundary, the destination assignments occur in this order:

1. Byte offset 0 receives the expansion of bits 15–12.
2. Byte offset 1 receives the expansion of bits 11–8.
3. Byte offset 2 receives the expansion of bits 7–4.
4. Byte offset 3 receives the expansion of bits 3–0.

No existing destination byte is read to compute its replacement. Initial destination contents therefore have no effect on the successful result.

There is no source-visible read from any other memory region. All output calculations depend on the scalar input supplied by value. Consequently, overwriting destination storage cannot change the input value used for later output calculations.

This logical ordering does not establish that a compiled implementation emits four distinct physical byte-store instructions in that order. The supplied source has no volatile qualification, synchronization operation, or other mechanism establishing externally observable intermediate stores.

### Arithmetic and widths

Under the 32-bit unsigned model, the visible masks, shifts, truncations, and bitwise combinations yield the layout above.

The fourth byte uses a left shift by 28 followed by a right shift by 24, together with the low input nibble. The left shift is unsigned: bits shifted beyond the 32-bit width are discarded rather than treated as signed overflow. After byte truncation, this contributes the original low nibble in the output’s high-nibble position. Combining it with the original low nibble produces the final replicated byte.

All shift counts are below 32. Every final result is between 0 and 255, inclusive. No signed arithmetic, negative shift count, division, or arithmetic error path is present.

### Byte order

The output order is defined by explicit byte offsets, not by storing a multibyte integer. For the same numeric input, the specified byte sequence does not depend on the host’s integer endianness.

For example, numeric input `0x1234` produces `11 22 33 44`, regardless of how that input would be represented in memory elsewhere.

## Side effects

The only explicit state changes are writes to destination offsets 0 through 3.

For valid ordinary memory:

- All four destination bytes are replaced.
- Bytes before and after the four-byte region are unchanged by this function.
- No allocation or deallocation occurs.
- No pointer is retained.
- No ownership transfer is expressed.
- No global state access, I/O operation, callback, or external function call is visible.

The caller remains responsible for the destination’s lifetime and storage management.

There is no success flag, error code, exception handling, rollback mechanism, or partial-completion report. If an invalid destination causes a fault, the supplied material does not establish which writes, if any, have become visible. Such a fault is outside the valid-input contract.

The function does not provide an atomic four-byte publication guarantee. Concurrent readers or writers require whatever synchronization the surrounding application and platform demand.

## Dependencies

No callable dependencies are present in the supplied body.

The function relies only on the execution environment’s unsigned integer operations, byte conversion rules, pointer-based byte access, and provision of valid writable storage.

No assumptions about another function preserving globals, retaining pointers, or releasing memory are needed: there are no visible callees.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Distinct nibble ordering

Given a valid four-byte destination and input `0x00001234`, normal completion must leave:

| Offset | Expected byte |
|---|---|
| 0 | `0x11` |
| 1 | `0x22` |
| 2 | `0x33` |
| 3 | `0x44` |

This distinguishes descending nibble order from reversed order and from a raw integer-byte encoding.

### Mixed boundary values

Given input `0x0000A50F`, the resulting bytes must be:

`AA 55 00 FF`

This checks a high nibble value, an interior value, zero, and the maximum nibble value within one result.

### Ignored upper input bits

Under the 32-bit input model, inputs `0x00001234` and `0xABCD1234` must produce identical output:

`11 22 33 44`

The upper 16 bits do not contribute to the result.

### Single selected nibble

Each of the following inputs must affect only the indicated output position:

| Input | Expected four bytes |
|---|---|
| `0x0000F000` | `FF 00 00 00` |
| `0x00000F00` | `00 FF 00 00` |
| `0x000000F0` | `00 00 FF 00` |
| `0x0000000F` | `00 00 00 FF` |

### Overwrite and region boundary

Given a valid six-byte region initially containing `7E CC CC CC CC 81`, use its second byte as the destination and supply input `0x00001234`.

Normal completion must leave the region as:

`7E 11 22 33 44 81`

The four destination bytes are overwritten independently of their previous contents; the neighboring bytes remain unchanged.

### Invalid storage

A null pointer or a destination with fewer than four writable bytes is outside the contract. No particular return behavior, diagnostic, fault type, or partial output is an acceptance requirement for those inputs.

## Uncertainties

- **documented_only — Target type definitions:** Eight-bit bytes and a 32-bit unsigned input are consistent with the supplied representation. No independent type declarations or ABI documentation were provided.
- **documented_only — Application meaning:** The destination may represent expanded packed components, but no caller context establishes a particular format, color model, or semantic role.
- **documented_only — Machine-level store behavior:** Store instruction widths, physical ordering, visibility to other agents, and atomicity are not established by the source-level assignments.
- **documented_only — Invalid-access outcome:** Platform fault handling and any observable partial modification are unspecified.
- **documented_only — Integration requirements:** Actual caller buffer sizes, destination lifetimes, and synchronization practices are not available.

These unknowns do not change the specified valid-memory transformation.

## Evidence

The established behavior comes from static inspection of the supplied function body: four consecutive destination assignments, input masks and shifts selecting the low four nibbles, byte-sized results, and no visible calls or validation.

The nibble-replication formula, ignored upper bits, output bounds, and acceptance examples are algebraic consequences of those operations under the stated width model.

No runtime tests, compilation checks, caller inspection, or machine-level observations have been performed.
