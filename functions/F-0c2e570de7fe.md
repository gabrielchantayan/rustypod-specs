<!-- Trusted provenance: F-0c2e570de7fe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0c2e570de7fe

## Purpose

F-0c2e570de7fe converts an input byte sequence into an unsigned 32-bit result selected by a format argument. The supported paths are:

- An inverted, weighted three-component intensity, returned through a lookup table or repeated across the result.
- A four-component conversion delegated to F-936c8a3c5816, followed by a specific bitwise expansion.
- A direct four-byte packing.
- Zero for unsupported format selections.

Bit 13 of the format argument is removed before selecting the conversion. On the supported four-component paths, that bit additionally requests reversal of the four result bytes. It does not change the results of the small-format paths.

The apparent intent is pixel-format conversion. Component meanings, lookup-table encodings, and the delegated conversion’s color semantics are not established by the supplied evidence.

## Interface

### Inputs

**Input address**

An address from which unsigned 8-bit components may be read. This contract names consecutive bytes at offsets 0, 1, 2, and 3 as A, B, C, and D, respectively. These names describe positions, not established channel semantics.

Required readable storage depends on the normalized format:

| Normalized format | Required input bytes |
|---|---:|
| 0 through 8, including unsupported selections in that range | A, B, C |
| 0x0555, 0x0565, or 0x1444 | A, B, C, D |
| 0x1888 | A, B, C, D |
| Any other value | No input bytes are read by this function |

**Format argument**

An unsigned 32-bit value. Its interpretation is:

- Clear bit 13, whose mask is 0x00002000, to obtain the **normalized format**.
- Retain the original bit-13 value as the byte-reversal request.
- Do not discard or normalize any other bits.

Consequently, setting an unrelated high bit can make an otherwise recognized format unsupported.

### Output

An unsigned 32-bit value. There is no separate success indicator or error object.

Zero is a valid conversion result and is also returned for unsupported formats. Callers cannot distinguish these cases from the return value alone.

### Width and arithmetic model

The supplied operations describe 8-bit input components and 32-bit unsigned result arithmetic. Left shifts used to construct the result discard bits beyond bit 31; unsigned right shifts insert zeros.

The small-format calculation never overflows 32 bits. Its weighted sum ranges from 0 through 8160.

No input buffer, table storage, or dependency-owned object is returned.

## Behavior

### 1. Format selection

The normalized format is the original format with bit 13 cleared.

Selection uses the entire normalized value:

1. Values below 9 enter the small-format path.
2. Values 0x0555, 0x0565, and 0x1444 enter the delegated conversion path.
3. Value 0x1888 enters the direct packing path.
4. All remaining values return zero.

A format rejected by step 4 causes no input-buffer access and no call to F-936c8a3c5816.

### 2. Small-format conversion

Every normalized format from 0 through 8 reads A, B, and C, even when the eventual result is zero because the selection is unsupported. D is not read on this path.

The function derives an inverted 8-bit weighted intensity:

\[
S = 8A + 20B + 4C
\]

\[
I = 255 - \left\lfloor S/32 \right\rfloor
\]

Since A, B, and C are each between 0 and 255, the shifted weighted value is between 0 and 255. The source’s complement-and-mask operation is therefore equivalent to the subtraction defining \(I\).

For normalized format \(n\), it next derives:

\[
Q = \left\lfloor I / 2^{8-n} \right\rfloor
\]

Here \(n\) is between 0 and 8, so the shift count is between 8 and 0. In particular, format 0 produces \(Q=0\).

The result then depends on the selection:

| Normalized format | Result |
|---|---|
| 1 | A 32-bit entry from the one-bit lookup table, indexed by \(Q\) |
| 2 | A 32-bit entry from the two-bit lookup table, indexed by \(Q\) |
| 4 | \(Q\) repeated into all eight 4-bit fields of the result |
| 8 | \(Q\) repeated into all four bytes of the result |
| 0, 3, 5, 6, 7 | Zero |

For format 1, \(Q\) is either 0 or 1. For format 2, it is between 0 and 3. For format 4, it is between 0 and 15. For format 8, it is between 0 and 255.

The lookup results are loaded as whole 32-bit words. They are not masked, expanded, or byte-reversed afterward.

#### Lookup-base access ordering

Two global slots provide the lookup-table base addresses. Their values and storage contents are not supplied.

The source establishes different access timing:

- For normalized format 1, the one-bit lookup-base slot is read before the component calculation. The resulting table entry is read after \(Q\) is calculated.
- For every other normalized format below 9, the two-bit lookup-base slot is read after the component calculation. This slot read occurs even for selections that ultimately use repetition or return zero.
- Only normalized format 2 dereferences an entry through the two-bit table base.

Thus, an unused table-base read must not be confused with an unused table-entry read. The supplied function reads the second base slot for formats such as 4 and 8, but it does not read a table entry for those formats.

The source does not establish a left-to-right order among the individual A, B, and C loads within the weighted expression.

#### Byte-reversal flag on small formats

Bit 13 is ignored after selection on this path. For example, formats 4 and 0x2004 have identical conversion behavior, including the same input and global-slot access pattern.

### 3. Delegated four-component conversion

For normalized format 0x0555, 0x0565, or 0x1444:

1. Read A, B, C, and D.
2. Call F-936c8a3c5816 with the normalized format followed by those four component values.
3. Form an intermediate 32-bit result by taking the bitwise OR of the dependency’s return value and that value shifted left by 16 bits.
4. Return the intermediate result unchanged if bit 13 of the original format is clear.
5. Otherwise, reverse the intermediate result’s four bytes and return it.

Each component is passed as its unsigned byte value, between 0 and 255. The input address itself is not passed to the dependency.

The individual component-load order within the call’s argument evaluation is not established. All four component values must be obtained before the call begins.

#### Exact expansion of the dependency result

Let the dependency’s unsigned 32-bit return have lower halfword \(L\) and upper halfword \(H\). Before optional byte reversal:

- The lower result halfword is \(L\).
- The upper result halfword is \(H \mathbin{\mathrm{OR}} L\).

This is not unconditionally “duplicate a 16-bit return.” It duplicates the lower halfword only if the dependency’s upper halfword is zero, or already contains no bits outside those set in the lower halfword.

The supplied evidence does not establish such a restriction on F-936c8a3c5816. Its upper return bits must therefore remain part of the contract.

### 4. Direct four-byte packing

For normalized format 0x1888, the intermediate numeric value has this layout:

| Result bit range | Input component |
|---|---|
| 31–24 | D |
| 23–16 | A |
| 15–8 | B |
| 7–0 | C |

Equivalently:

\[
P = D\cdot2^{24} + A\cdot2^{16} + B\cdot2^8 + C
\]

No dependency is called on this path.

If bit 13 is clear, return \(P\). If bit 13 is set, reverse its four bytes. The reversed numeric layout, from most significant byte to least significant byte, is C, B, A, D.

This describes the numeric return value, not a host-memory serialization.

### 5. Byte reversal

Byte reversal applies only to the delegated and direct packing paths.

For an intermediate numeric value whose bytes from most significant to least significant are \(X_3, X_2, X_1, X_0\), reversal returns the numeric value with bytes \(X_0, X_1, X_2, X_3\).

It reverses the complete intermediate result. It does not reorder input components before dependency invocation, alter the normalized format passed to the dependency, or change which conversion path is selected.

## Side effects

### Established direct effects

The function directly:

- Reads the selected number of input bytes.
- Reads the lookup-base slots as described for small formats.
- Reads one 32-bit table entry for normalized format 1 or 2.
- Calls F-936c8a3c5816 for normalized formats 0x0555, 0x0565, and 0x1444.
- Returns a 32-bit value.

No direct write to the input buffer, lookup tables, or global slots is shown. No direct allocation, release, or ownership-transfer operation is shown.

### Dependency effects

The effects of F-936c8a3c5816 are unknown. This contract does not assume that it preserves globals, avoids allocation, or avoids releasing memory.

It receives component values rather than the input pointer. That limits what this function explicitly supplies to it, but does not establish what other state the dependency can access.

### Preconditions and failures

For any path that reads input, the required bytes must be readable. For lookup paths, the relevant table base and indexed 32-bit entry must also be valid for the underlying execution environment.

The function does not check:

- Whether the input address is null.
- Whether enough input bytes are available.
- Whether table bases are valid.
- Whether table entries meet alignment requirements.

No recoverable failure behavior is established for invalid accesses. Returning zero for an unsupported format does not validate the input address.

No synchronization or coherent-snapshot guarantee is shown. Concurrent mutation of input bytes, table slots, or table contents is outside the established contract.

## Dependencies

### F-936c8a3c5816

**Invocation condition:** Normalized format is 0x0555, 0x0565, or 0x1444.

**Arguments, in order:**

1. Normalized format.
2. A as an unsigned component value.
3. B as an unsigned component value.
4. C as an unsigned component value.
5. D as an unsigned component value.

**Consumed output:** A value treated as an unsigned 32-bit word, expanded and optionally byte-reversed as specified above.

**Unknowns:** Internal packing rules, supported component semantics, return-value restrictions, failure behavior, global mutations, and memory-management effects are documented_only.

### Lookup storage

The one-bit table must provide readable 32-bit entries at indices 0 and 1. The two-bit table must provide readable 32-bit entries at indices 0 through 3.

Indexing advances by four bytes per entry. The result is the loaded word under the target’s native word interpretation; table byte order is not independently established here.

The actual base addresses and entry values are documented_only.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Direct packing and reversal

Given A = 0x12, B = 0x34, C = 0x56, and D = 0x78:

- Format 0x1888 must return 0x78123456.
- Format 0x3888 must return 0x56341278.
- Neither selection must call F-936c8a3c5816.

### Repeated intensity

Given A = B = C = 0:

- The inverted intensity is 255.
- Format 4 must return 0xFFFFFFFF.
- Format 8 must return 0xFFFFFFFF.
- Setting bit 13 must not change either result.

Given A = B = C = 255:

- The weighted shifted value is 255.
- The inverted intensity is zero.
- Formats 4 and 8 must return zero.

Given A = B = C = 128:

- The weighted shifted value is 128.
- The inverted intensity is 127.
- Format 4 must return 0x77777777.
- Format 8 must return 0x7F7F7F7F.

### Lookup boundaries

For format 1:

- A = B = C = 127 selects table index 1.
- A = B = C = 128 selects table index 0.

For format 2:

- A = B = C = 0 selects table index 3.
- A = B = C = 255 selects table index 0.

The returned value must equal the selected 32-bit table entry, without additional replication or reversal.

### Dependency return handling

For any supported delegated format, suppose F-936c8a3c5816 returns 0x12345678:

- With bit 13 clear, the result must be 0x567C5678.
- With bit 13 set, the result must be 0x78567C56.
- The dependency must receive the same normalized format and component values in both cases.

This example distinguishes the actual OR expansion from simply repeating the lower halfword.

### Unsupported selections and reads

For normalized format 3 and a valid three-byte input:

- A, B, and C are read.
- The two-bit lookup-base slot is read.
- No lookup entry is read.
- No dependency is called.
- The result is zero.

For format 0x00010008:

- The unrelated high bit remains after normalization.
- The selection is unsupported.
- The result is zero without reading the input or either lookup-base slot.

An unsupported selection requiring no input read must not be used as evidence that a null address is valid for recognized selections.

## Uncertainties

The following remain documented_only:

- Semantic names and color interpretation of A, B, C, and D.
- Actual lookup-table contents and their intended output encoding.
- Native byte order and alignment rules for table-word loads.
- The complete contract and side effects of F-936c8a3c5816.
- Whether the dependency normally returns only 16 meaningful bits.
- Detailed fault behavior for invalid input or table storage.
- Evaluation order among component loads within a single expression.
- Behavior under concurrent mutation or special memory mappings.

The constants resemble familiar packed color formats, but that resemblance is inferred intent, not evidence sufficient to specify the dependency’s conversion.

## Evidence

**Established static evidence:** The supplied function body establishes format masking, branch selection, input offsets, lookup indexing, weighted-intensity arithmetic, bitwise expansion, direct packing, optional byte reversal, and the absence of direct memory writes in this function.

**Inferred intent:** Pixel conversion, intensity quantization, and format-related meanings of the selector constants.

**Unavailable evidence:** Dependency implementation, lookup contents, platform memory-access rules, and runtime observations.

No runtime tests have been performed. All examples above are static acceptance conditions, with dependency and storage assumptions stated explicitly.
