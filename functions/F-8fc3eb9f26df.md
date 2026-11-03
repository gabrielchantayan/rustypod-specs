<!-- Trusted provenance: F-8fc3eb9f26df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8fc3eb9f26df

## Purpose

F-8fc3eb9f26df selects one of four byte-to-integer packing operations. One operation assembles four bytes into a 32-bit numeric value, most-significant byte first. The other three select high-order bits from three or four bytes and pack them into the low 16 bits of the result.

An unsupported selector returns zero without accessing the supplied byte pointer.

These transformations are established by static inspection of the supplied function body. Their resemblance to color-component packing is inferred intent only. No pixel format, channel meaning, or external protocol is established.

## Interface

The function accepts:

- **Selector:** an integer compared for exact equality with 4, 5, 6, and 7.
- **Byte pointer:** the starting address of the input sequence. The required readable extent depends on the selector.

It returns an unsigned integer. It does not return a pointer or a separate success indicator.

The following notation describes the input without assigning semantic channel names:

- **B0:** byte at offset 0 from the supplied address.
- **B1:** byte at offset 1.
- **B2:** byte at offset 2.
- **B3:** byte at offset 3.
- **Result bit 0:** the least-significant bit of the returned numeric value.

### Width model

The supplied body uses byte and unsigned-integer type names, but their definitions are not included. The concrete layouts in this contract use the apparent intended model:

- An input byte is an unsigned 8-bit quantity, ranging from 0 through 255.
- The unsigned result type has at least 32 value bits.
- Integer operations used for the masked component expressions can represent values through 65,535.

An unsigned 32-bit result is the likely platform interpretation, but its exact width is not independently established by the supplied fragment. A narrower unsigned result type would make the selector-4 left shifts unsuitable for this contract. A signed intermediate type unable to represent the component-packing values could also change language-level validity.

Under the stated width model, selector 4 can return any value from 0 through 4,294,967,295. Selectors 5, 6, and 7 return values from 0 through 65,535. If the result type is wider than 32 bits, all bits above bit 31 are zero; the packed 16-bit modes also leave bits 16 and above zero.

### Input extent and validity

| Selector | Required input bytes | Highest accessed offset |
|---|---:|---:|
| 4 | 4 | 3 |
| 5 | 3 | 2 |
| 6 | 4 | 3 |
| 7 | 4 | 3 |
| Any other value | 0 | None |

For a supported selector, the caller must supply storage from which every required byte can validly be read. There is no supplied length, null check, bounds check, or alignment check.

The accesses are byte accesses; the body does not impose a multi-byte-load alignment requirement. Validity of the pointer and its readable extent remain caller obligations.

For an unsupported selector, the body does not dereference or perform arithmetic on the byte pointer. In particular, a null pointer does not prevent the unsupported-selector path from returning zero.

## Behavior

### Selection and completion

The selector determines exactly one transformation. Selection uses equality, not a range, bit mask, or modulo operation. Values such as 0, 3, 8, and negative values therefore take the same fallback path.

For each recognized selector, the function obtains the relevant byte values, constructs the packed result, and returns it. It does not process a sequence of records, advance a caller-visible pointer, or retain progress between calls.

The fallback result is zero. This is not a unique error value: every supported transformation can also produce zero.

### Selector 4: four-byte, most-significant-first assembly

All eight bits of each of B0 through B3 are retained.

| Result bits | Source |
|---|---|
| 31–24 | B0 bits 7–0 |
| 23–16 | B1 bits 7–0 |
| 15–8 | B2 bits 7–0 |
| 7–0 | B3 bits 7–0 |

Equivalently, the numeric result is:

**B0 × 16,777,216 + B1 × 65,536 + B2 × 256 + B3.**

The fields do not overlap. Combining them with bitwise OR therefore has the same numeric result as adding their contributions.

This defines the returned number independently of host byte order. It does not define the memory byte order that would result if a caller later stored the returned integer.

No sign extension applies to the input bytes under the unsigned-byte model. For example, a leading byte with its top bit set contributes to result bit 31 rather than representing a negative component.

### Selector 5: 5/6/5-bit packing

This transformation consumes B0, B1, and B2. It does not access B3.

| Result bits | Source |
|---|---|
| 15–11 | B0 bits 7–3 |
| 10–5 | B1 bits 7–2 |
| 4–0 | B2 bits 7–3 |

The low three bits of B0, low two bits of B1, and low three bits of B2 are discarded. Changing only those discarded bits cannot change the result.

There is no rounding, scaling to preserve endpoints, saturation, or carry between fields. The operation selects existing bits and relocates them. The result occupies exactly 16 possible bit positions, with all higher positions zero.

### Selector 6: four-nibble packing

This transformation consumes B0 through B3 and retains the upper nibble of each.

| Result bits | Source |
|---|---|
| 15–12 | B0 bits 7–4 |
| 11–8 | B1 bits 7–4 |
| 7–4 | B2 bits 7–4 |
| 3–0 | B3 bits 7–4 |

The low four bits of every input byte are ignored.

The retained nibbles appear in input order, from the most-significant result nibble to the least-significant result nibble. The operation does not replicate a nibble, normalize component magnitudes, or inspect the discarded bits to round the retained values.

### Selector 7: 5/5/5/1-bit packing

This transformation consumes B0 through B3.

| Result bits | Source |
|---|---|
| 15–11 | B0 bits 7–3 |
| 10–6 | B1 bits 7–3 |
| 5–1 | B2 bits 7–3 |
| 0 | B3 bit 7 |

The low three bits of each of B0, B1, and B2 are ignored. Only the most-significant bit of B3 is retained.

Consequently, B3 values from 0 through 127 contribute zero, while values from 128 through 255 contribute one. This boundary follows directly from selecting bit 7; it is not an independently established semantic threshold.

### Read ordering and stability

The function body references each required byte once in the selected expression. It establishes no left-to-right order among those byte reads: the operands are combined using operations that do not sequence their evaluation.

Accordingly, this contract specifies **which offsets contribute**, but not an observable order such as “offset 0 is read before offset 1.” The different modes must not be distinguished by an assumed read order.

The logical byte references also do not establish the number, width, or ordering of machine-level memory transactions. Compiler behavior and the target memory system are outside the supplied evidence.

For a deterministic result, the required bytes must remain stable while they are being accessed. The body supplies no locking, atomic snapshot, or synchronization. Behavior involving concurrent modification, volatile device storage, or read-sensitive memory is not established by this fragment.

## Side effects

The supplied body performs no explicit writes through the input pointer and no explicit writes to global or static state. It allocates no memory, releases no memory, and makes no function calls.

The byte pointer is used only to obtain input values. Ownership is neither acquired nor transferred, and the function does not retain the pointer in visible storage. The caller remains responsible for the storage’s lifetime.

The only explicit result is the returned unsigned integer. There is no status output, diagnostic message, callback, or explicit error report.

Invalid memory on a recognized-selector path is not converted into a defined failure result. Such a call violates the readable-storage precondition; the resulting fault or other language/runtime behavior is not specified here.

## Dependencies

No called-function dependencies appear in the supplied body. No additional opaque function IDs are therefore needed.

The behavior depends on:

- The actual definitions and widths of the byte and integer types.
- The applicable integer-promotion and shift rules.
- Valid access to the required input bytes.
- The execution environment’s ordinary pointer and memory semantics.

No assumption about an unknown callee preserving globals, retaining pointers, or releasing storage is needed, because there are no calls in this body.

## Acceptance examples

These are conditional, statically derived acceptance examples, not runtime observations. They assume the width model above and stable, valid input storage wherever bytes are required. Byte values and results are hexadecimal unless stated otherwise.

| Selector | Input bytes beginning at offset 0 | Expected numeric result | Property exercised |
|---|---|---|---|
| 4 | 12 34 56 78 | 12345678 | Four-byte ordering |
| 4 | 80 00 00 00 | 80000000 | Preservation of result bit 31 |
| 4 | FF FF FF FF | FFFFFFFF | Full 32-bit range |
| 5 | 12 34 56 | 11AA | Placement of all three retained fields |
| 5 | F8 FC F8 | FFFF | Maximum 5/6/5-bit result |
| 5 | 07 03 07 | 0000 | Discarded low bits have no effect |
| 6 | 12 34 56 78 | 1357 | Upper-nibble ordering |
| 6 | F0 F0 F0 F0 | FFFF | Maximum four-nibble result |
| 6 | 0F 0F 0F 0F | 0000 | All lower nibbles are discarded |
| 7 | 12 34 56 78 | 1194 | Three five-bit fields with low result bit clear |
| 7 | F8 F8 F8 80 | FFFF | Maximum 5/5/5/1-bit result |
| 7 | 00 00 00 7F | 0000 | B3 below the bit-7 boundary |
| 7 | 00 00 00 80 | 0001 | B3 at the bit-7 boundary |

Additional acceptance conditions:

- With selector 5 and exactly three valid readable bytes, no fourth byte is required.
- With selector 5, replacing `F8 FC F8` with `FF FF FF` leaves the result unchanged at FFFF.
- With selector 6, replacing any byte’s lower nibble while preserving its upper nibble leaves the result unchanged.
- With selector 7, changing B3 from 80 to FF leaves its contribution unchanged at one.
- With any supported selector and all required bytes zero, the result is zero.
- With selector 8 or selector −1, the result is zero and no byte access is required, including when the supplied pointer is null.

## Uncertainties

The following items are **documented_only** rather than guessed:

- **Exact type definitions and ABI widths:** absent from the fragment. The concrete numeric contract is conditional on the stated width model.
- **Domain semantics:** the 5/6/5, four-nibble, and 5/5/5/1 layouts suggest packed component formats, but channel identities and meanings are not established.
- **Selector provenance:** no enumeration definition or caller context establishes why these particular selector values are used.
- **Input lifetime beyond the call:** the body does not establish who created the storage or how long the caller otherwise keeps it.
- **Physical memory-access behavior:** machine access widths, ordering, and suitability for memory-mapped devices are not established.
- **Invalid-input outcomes:** no portable fault type, recovery path, or diagnostic behavior is shown.

## Evidence

**Established static evidence:** exact selector comparisons; fallback zero; referenced byte offsets; masks and shifts; the resulting bit layouts; absence of explicit writes, calls, allocation, release, and pointer retention in the supplied body.

**Inferred intent:** conversion from byte-oriented components into packed numeric formats, including a likely 32-bit unsigned return convention.

**Not observed:** execution results, platform type definitions, callers, generated machine behavior, and runtime failure handling.

No runtime tests have been performed.
