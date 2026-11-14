<!-- Trusted provenance: F-a540fc83a3ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a540fc83a3ef

## Purpose

F-a540fc83a3ef initializes a caller-supplied, 20-byte output record and selects its contents from two integer inputs.

The primary input recognizes individual, exact selector values. The secondary input is considered only when the primary input is zero. The function does not combine selectors or accumulate results from multiple set bits.

Some selections copy two opaque, 32-bit values from external data into the output. Their meaning is not established: they must not be assumed to be pointers, callbacks, owned resources, or numeric constants.

**Evidence status:** The behavior below is established from the supplied static representation unless explicitly marked as inference or documented_only. No runtime tests have been performed.

## Interface

### Inputs

| Input | Contract |
|---|---|
| Primary selector | Signed, 32-bit integer. Exact equality determines recognized selections. |
| Secondary selector | Signed, 32-bit integer. Affects the result only when the primary selector equals zero. |
| Output address | Nullable address of writable storage covering at least 20 bytes when non-null. |

The primary selector’s high-bit-only case is the signed value −2,147,483,648, with the 32-bit representation `0x80000000`.

### Output layout

The following descriptive field names are used only to specify the observable record:

| Byte offset | Width | Field |
|---:|---:|---|
| 0 | 16 bits | Category |
| 2 | 16 bits | Qualifier |
| 4 | 32 bits | Mask |
| 8 | 32 bits | Code |
| 12 | 32 bits | Payload first |
| 16 | 32 bits | Payload second |

The fields occupy all 20 bytes; no gap appears in the supplied layout. Values below describe field values, not a prescribed byte serialization. Target endianness is not established.

### Return and validity

The function returns no value. It does not expose a separate success or failure indication.

A null output address is accepted and causes an immediate return. For non-null output, the caller must supply writable storage suitable for the shown 16-bit and 32-bit accesses. Insufficient size, invalid storage, or unsupported alignment is outside the established valid-input contract; the function performs no visible validation of those conditions.

The caller retains responsibility for the output storage. No allocation, release, ownership transfer, or retention of its address is visible.

## Behavior

### Initialization and common reads

For every non-null output address, the function first performs these writes in this order:

1. Category receives 1 through a 16-bit write.
2. Qualifier receives 0 through a 16-bit write.
3. Mask receives 0 through a 32-bit write.
4. Code receives 0 through a 32-bit write.
5. Payload first receives 0 through a 32-bit write.
6. Payload second receives 0 through a 32-bit write.

It then reads two external, 32-bit words, in order:

1. The first word of opaque data pair D.
2. The first word of opaque data pair A.

These reads occur before selector-specific handling, including handling that does not use either word. A null output address bypasses both the initialization and these reads.

The output is therefore initialized before selection, rather than assembled elsewhere and committed atomically. Previously stored output contents are not consulted.

### Primary selection

The table gives the final record for every recognized primary selector. Hexadecimal values are exact integer values. A zero means the field retains its initialized zero unless a later-write rule below states otherwise.

Pair designations identify external data sources, not literal numeric values.

| Primary selector | Category | Qualifier | Mask | Code | Payload first | Payload second |
|---|---:|---:|---:|---:|---|---|
| `0x00000001` | 3 | `0xFF` | `0x1` | 5 | A first | A second |
| `0x00000002` | 3 | `0xFF` | `0x8` | 6 | B first | B second |
| `0x00000004` | 3 | `0xFF` | `0x4` | 7 | C first | C second |
| `0x00000008` | 5 | 4 | `0x8000` | 8 | 0 | 0 |
| `0x00000010` | 8 | 5 | `0x10000` | `0x10` | 0 | 0 |
| `0x00000020` | 5 | 4 | `0x80` | `0xF` | 0 | 0 |
| `0x00000040` | 3 | `0xFF` | `0x10` | 9 | D first | D second |
| `0x00000080` | 3 | `0xFF` | `0x20` | `0xA` | E first | E second |
| `0x00000100` | 9 | 0 | `0x80000` | `0xB` | 0 | 0 |
| `0x00000200` | 5 | 3 | `0x40` | `0xC` | 0 | 0 |
| `0x00000400` | 5 | 5 | `0x4000` | `0xD` | 0 | 0 |
| `0x00000800` | `0xA` | `0xC` | `0x800` | `0xE` | 0 | 0 |
| `0x00001000` | `0xB` | 0 | 0 | 2 | 0 | 0 |
| `0x00002000` | 3 | `0xFF` | 0 | `0x11` | F first | F second |
| `0x00004000` | `0xB` | 0 | 0 | 3 | 0 | 0 |
| `0x00008000` | `0xB` | 0 | 0 | 4 | 0 | 0 |
| `0x00010000` | 4 | 0 | `0x100` | 0 | 0 | 0 |
| `0x00020000` | `0xD` | 0 | 0 | `0x12` | 0 | 0 |
| `0x00080000` | 3 | `0xFF` | `0x400` | `0x15` | G first | G second |
| `0x00100000` | `0xD` | 0 | 0 | 1 | 0 | 0 |
| `0x00200000` | 3 | `0xFF` | `0x200` | `0x17` | H first | H second |
| `0x00400000` | `0xA` | `0xC` | `0x1000` | 0 | 0 | 0 |
| `0x00800000` | `0xA` | `0xC` | `0x2000` | 0 | 0 | 0 |
| `0x01000000` | `0xD` | 0 | `0x20000` | 0 | 0 | 0 |
| `0x02000000` | 9 | 0 | 0 | `0x16` | 0 | 0 |
| `0x04000000` | 3 | `0xFF` | `0x200000` | `0x18` | I first | I second |
| `0x08000000` | 3 | `0xFF` | 0 | 7 | J first | J second |
| `0x10000000` | 5 | 0 | 0 | `0x19` | 0 | 0 |
| `0x20000000` | 5 | 6 | `0x2000000` | `0x1A` | 0 | 0 |
| `0x40000000` | 9 | 0 | `0x4000000` | `0x1B` | 0 | 0 |
| `0x80000000` | `0xC` | 0 | `0x400000` | 0 | 0 | 0 |

The `0x80000000` row denotes the signed minimum input, not a positive integer beyond the signed range.

Every recognized primary selection finishes without using the secondary selector. Notably, `0x00040000` has no recognized primary entry.

### Secondary selection

These selections apply only when the primary selector is exactly zero.

| Secondary selector | Category | Qualifier | Mask | Code | Payload first | Payload second |
|---|---:|---:|---:|---:|---|---|
| `0x00000001` | 5 | 3 | `0x800000` | `0x1C` | 0 | 0 |
| `0x00000002` | 4 | 3 | `0x1000000` | `0x1D` | 0 | 0 |
| `0x00000040` | `0xC` | 0 | 0 | 0 | 0 | 0 |
| `0x00000080` | 9 | 0 | 0 | `0x24` | 0 | 0 |
| `0x00000100` | 5 | 3 | `0x8000000` | `0x25` | 0 | 0 |
| `0x00000200` | `0xD` | 0 | 0 | 0 | 0 | 0 |
| `0x00000800` | 3 | `0xFF` | `0x10000000` | `0x26` | K first | K second |
| `0x00002000` | 3 | `0xFF` | 0 | `0x14` | L first | L second |
| `0x00008000` | 3 | 0 | 0 | 0 | 0 | 0 |
| `0x00010000` | 3 | 0 | 0 | 0 | 0 | 0 |
| `0x00020000` | 5 | 0 | 0 | 0 | 0 | 0 |
| `0x00400000` | 5 | 0 | 0 | 0 | 0 | 0 |
| `0x00800000` | 9 | 0 | 0 | 0 | 0 | 0 |

These are exact matches, not bit tests.

### Unrecognized inputs

If the primary selector is nonzero and not recognized, the secondary selector cannot select a result. The final output is the initialized record:

- Category: 1.
- All other fields: 0.

The same default results when the primary selector is zero and the secondary selector is unrecognized.

There is no distinct error record beyond this default and no reported error status.

### Order of selection-specific writes and reads

The following ordering applies after initialization and the two common external reads.

For selections without payload pairs, fields are written in layout order, omitting fields whose final table value is zero. Thus Category is written first; Qualifier, Mask, and Code follow when applicable. The payload fields receive no further writes on these paths.

For payload-bearing selections, the exact order differs:

| Selection | Ordered events after the common reads |
|---|---|
| Primary `0x1` | Write Category; write Qualifier; write Payload first using the previously read A first; read A second; write Mask; write Code; write Payload second. |
| Primary `0x2` | Write Category; write Qualifier; write Mask; write Code; read B first and write Payload first; read B second; write Payload second. |
| Primary `0x4` | Write Category; write Qualifier; write Code; read C first and write Payload first; read C second; write Mask; write Payload second. |
| Primary `0x40` | Write Category; write Qualifier; write Payload first using the previously read D first; write Code; read D second; write Mask; write Payload second. |
| Primary `0x80` | Write Category; write Qualifier; write Mask; write Code; read E first and write Payload first; read E second; write Payload second. |
| Primary `0x2000` | Write Category; write Qualifier; write Code; read F first and write Payload first; read F second; write Payload second. |
| Primary `0x80000` | Write Category; write Qualifier; write Mask; write Code; read G first and write Payload first; read G second; write Payload second. |
| Primary `0x200000` | Write Category; write Qualifier; write Mask; write Code; read H first and write Payload first; read H second; write Payload second. |
| Primary `0x4000000` | Write Category; write Qualifier; write Mask; write Code; read I first and write Payload first; read I second; write Payload second. |
| Primary `0x8000000` | Write Category; write Qualifier; write Code; read J first and write Payload first; read J second; write Payload second. |
| Secondary `0x800` | Write Category; write Qualifier; write Mask; write Code; read K first and write Payload first; read K second; write Payload second. |
| Secondary `0x2000` | Write Category; write Qualifier; write Code; read L first and write Payload first; read L second; write Payload second. |

Each field write uses the width in the interface layout. A pair-word read is 32 bits. No payload pair is fetched as one established atomic operation.

No arithmetic combines the input values or constructs the masks. The masks are selected constants, not shifts of the input. All listed constants fit their destination widths.

## Side effects

For valid non-null output, the visible effects are writes within the 20-byte record and reads of external words as specified above. No output-buffer reads are visible.

All output fields are initialized on every non-null invocation, even if their final value is zero. This overwrites prior payload contents without any visible attempt to release resources that those contents might represent.

No function calls, allocation, deallocation, external-data writes, I/O, or error-reporting operations are visible. There is no established guarantee of atomic publication, synchronization, or consistency between the two words of a pair.

If output storage overlaps external data, initialization and subsequent writes may affect later external reads. The tables describe the values actually observed at the specified read events, not necessarily values that existed before entry. Whether such overlap is valid under the target execution model is documented_only.

## Dependencies

There are no visible function dependencies, so no additional opaque function IDs are required.

The external-data dependencies are twelve opaque pairs of 32-bit words:

| Pair | Selection that copies it |
|---|---|
| A | Primary `0x1` |
| B | Primary `0x2` |
| C | Primary `0x4` |
| D | Primary `0x40` |
| E | Primary `0x80` |
| F | Primary `0x2000` |
| G | Primary `0x80000` |
| H | Primary `0x200000` |
| I | Primary `0x4000000` |
| J | Primary `0x8000000` |
| K | Primary zero, secondary `0x800` |
| L | Primary zero, secondary `0x2000` |

“Pair” is a documentation grouping; it does not establish a shared type, ownership relationship, or atomic read mechanism.

D first and A first are read on every non-null invocation. Other words are read only for their corresponding selection. A first and D first are not reread when copied into the output.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **Null output:** Given any selectors and a null output address, the function returns without output writes or external-word reads.

2. **Complete initialization:** Given a valid record containing arbitrary prior bytes, primary `0x10` produces Category 8, Qualifier 5, Mask `0x10000`, Code `0x10`, and two zero payloads. The prior contents do not survive.

3. **Primary precedence:** Given primary `0x1` and secondary `0x800`, the result uses pair A and Code 5. Pair K is not selected.

4. **No fallback for unsupported nonzero primary:** Given primary `0x40000` and secondary 1, the result is Category 1 with all remaining fields zero. The secondary selector does not produce its otherwise recognized record.

5. **No bitwise composition:** Primary 3 produces the default record, not a combination of the primary 1 and primary 2 records.

6. **Secondary exact matching:** Given primary zero and secondary 3, the result is the default record. Secondary 1 and secondary 2 are not combined.

7. **Signed boundary:** Given primary −2,147,483,648, the result is Category `0xC`, Mask `0x400000`, and zeros elsewhere. Other negative primary values produce the default record.

8. **Zero mask is meaningful:** Given primary `0x8000000`, the result has Category 3, Qualifier `0xFF`, Mask zero, Code 7, and pair J. The zero mask does not prevent payload selection.

9. **Opaque values are preserved:** If K first and K second yield particular 32-bit bit patterns at their respective reads, primary zero with secondary `0x800` copies those patterns unchanged into the two payload fields.

10. **Common-read behavior:** Given a valid output and an unsupported selector pair, D first and A first are still read, after initialization and in that order, even though neither value appears in the final record.

11. **Secondary category-only result:** Given primary zero and secondary `0x10000`, Category becomes 3 while Qualifier, Mask, Code, and both payloads remain zero.

## Uncertainties

### Inferred intent

The record appears to describe a selected option using a category, qualifier, mask, numeric code, and optional associated data. These names are descriptive conveniences, not established domain meanings.

The primary values resemble individual bit positions, but the established behavior is exact selection, not processing of a bitmask.

### documented_only

The supplied material does not establish:

- The application-level meaning of any category, qualifier, mask, or code.
- The numeric contents, mutability, initialization, or lifetime of the external words.
- Whether either payload field represents an address or resource.
- Any ownership or cleanup obligation associated with copied payload values.
- Target endianness, precise alignment requirements, or behavior on invalid memory access.
- Supported aliasing between output storage and external data.
- Thread-safety, synchronization, or behavior under concurrent modification.
- Machine-level access ordering beyond the supplied static representation.
- Caller expectations or downstream interpretation of the default record.

No missing semantics are supplied by assuming unknown routines preserve state or manage memory; no called routine is visible here.

## Evidence

The supplied static representation establishes:

- A null-output guard before all visible memory accesses.
- Six initialization writes covering a 20-byte record.
- Two unconditional external-word reads for non-null output.
- Exact primary-selector matching, including the signed-minimum case.
- Secondary selection restricted to primary zero.
- The complete field mappings and selection-specific access sequences above.
- No visible calls and no return value.

The external data definitions, callers, target execution details, and runtime observations were not supplied. No runtime tests have been performed.
