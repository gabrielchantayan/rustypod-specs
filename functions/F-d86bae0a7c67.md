<!-- Trusted provenance: F-d86bae0a7c67; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d86bae0a7c67

## Purpose

F-d86bae0a7c67 constructs a binary image in caller-supplied memory. The visible operation emits a header, three named records, associated attribute blocks, and four trailing 16-bit entries addressed relative to a length obtained from the input object.

The record labels are “Finder”, a six-byte slice of a static composite label, and “System”. The static composite label is presented as “RDNFSCAMReadMe”; its six-byte slice beginning at byte 8 would be “ReadMe” if that label accurately represents the underlying bytes.

The apparent intent is to construct a small filesystem-related catalog containing three entries. This interpretation is inferred, not an established format specification. In particular, the meanings of the attribute fields, fixed numeric identifiers, and trailing entries are not established by the supplied evidence.

This contract distinguishes direct memory operations from operations delegated to dependencies. It does not assume that apparent conversion, copying, or initialization helpers have their expected implementations.

## Interface

The function accepts two machine-level arguments, each represented as a 32-bit integer in the supplied representation:

- **Input-object address:** identifies storage from which configuration values are read.
- **Destination address:** identifies the memory in which construction occurs.

No meaningful return value is shown. Completion is by normal return after the final 16-bit store, subject to the behavior of called dependencies and the validity of memory accesses.

### Input-object fields

The visible body reads:

| Byte offset | Read width | Use |
|---|---:|---|
| `0x18` | 32 bits | Retain only the low 16 bits to obtain the destination-relative length, denoted **L** |
| `0x30` | 32 bits | Pass to F-c351b855a319; retain its 32-bit result, denoted **V** |

The read at `0x18` is a full 32-bit read, not a 16-bit read. Bytes outside its low half are discarded only after that read.

The initial sequence is:

1. Read and mask the length field.
2. Read the field at `0x30`.
3. Invoke F-c351b855a319 on that second value.
4. Invoke an unmapped external dependency with the destination address and L.

Thus, the object fields are read before the external destination-initialization call. Aliasing between input and destination does not retroactively change the already captured L or V.

### Memory preconditions

Successful execution requires all direct reads and writes, and all dependency accesses, to be valid. The function performs no visible bounds, null-pointer, alignment, or length validation.

The destination must support:

- Direct byte stores.
- Direct 16-bit stores.
- The accesses performed by the external initializer and other helpers.
- Stores addressed relative to `destination + L`, including negative offsets from that address.
- Attribute-block addresses derived from bytes reread from the destination.

Source metadata also requires readable storage preceding a saved metadata pointer. That pointer is not used solely as the beginning of an independent object.

No allocation, ownership transfer, or release is visible in this function. Dependency ownership behavior remains unknown.

## Behavior

### Representation conventions

All offsets below are byte offsets from the destination unless an attribute-block base is explicitly stated.

Two dependency-mediated operations recur:

- **16-bit encoding:** call F-0ec8efa9c769 with a numeric value, then directly store its 16-bit result.
- **32-bit emission:** call F-b5c826d2ab04 with a value, then call F-946cac4b59bc with the returned value and a destination address.

The first operation establishes a direct two-byte store. The second establishes a call sequence and destination argument, but not the number or order of bytes written by F-946cac4b59bc. Four-byte emission and byte-order conversion are plausible interpretations, not proven dependency contracts.

Native endianness of direct 16-bit stores is not established independently. The observable stored representation combines the returned 16-bit value with the platform’s store convention.

### Initial header

After obtaining L and V and invoking the external initializer, the function performs these operations in order:

1. Emit numeric value `2` at offset `4` through the 32-bit emission sequence.
2. Store byte `0xff` at offset `8`.
3. Encode `3` and store the result at offset `10`.
4. Store byte `1` at offset `9`.
5. Encode `0x0e` and store the result at offset `L − 2`.
6. Store byte `0x0d` at offset `14`.
7. Emit numeric value `2` at offset `16`.
8. Store byte `6` at offset `20`.
9. Call F-aa082a48ffa6 with the static “Finder” source, destination offset `21`, and count `6`.

The function then rereads the byte at offset `14`. Let its unsigned value be **A**. The first attribute-block base is:

**P = A + 15**

The reread is authoritative. Although the function previously stored `13` there, intervening dependencies are not proven to preserve that byte.

### First attribute block

Relative to P, operations occur in this order:

| Offset from P | Operation |
|---|---|
| `0` | Encode and directly store `0x0200` |
| `4` | Emit the first 32-bit word of the static composite-label storage |
| `8` | Emit its second 32-bit word |
| `12` | Encode and directly store `0x5100` |
| `20` | Emit `0x13` |
| `44` | Emit V |
| `48` | Emit V again, using a separate conversion and emission call |

The second static word is captured before its own conversion call. That captured value is retained for later use in the third attribute block. It is not reloaded for that later use.

Next, define the second record-header offset:

**H₂ = A + 117**

The function encodes H₂ and stores the result at offset `L − 4`.

### Second record header and attributes

At H₂, the function:

1. Stores byte `13` at relative offset `0`.
2. Emits `2` at relative offset `2`.
3. Stores byte `6` at relative offset `6`.
4. Calls F-aa082a48ffa6 with the static composite-label source advanced by eight bytes, destination relative offset `7`, and count `6`.
5. Rereads the byte at H₂.

Let the reread byte be **B**, interpreted unsigned. The second attribute-block base is:

**Q = H₂ + B + 1**

The function then performs:

| Offset from Q | Operation |
|---|---|
| `0` | Encode and directly store `0x0200` |
| `4` | Emit a 32-bit global metadata value |
| `8` | Emit another 32-bit global metadata value |
| `20` | Emit `0x12` |
| `44` | Emit V |
| `48` | Emit V again through a separate call sequence |

Unlike the first and third attribute blocks, this block has no visible direct store of encoded `0x5100` at relative offset `12`. The contents of that location depend on initialization and any dependency effects.

The function next loads a global metadata pointer and retains it as **M**. The first 16-bit metadata read is separately made through the global pointer expression; later reads use saved M. Therefore, the evidence does not require treating all metadata reads as an atomic snapshot.

The remaining second-block operations, in order, are:

| Offset from Q | Source and operation |
|---|---|
| `74` | Read a 16-bit value through the global metadata pointer, encode it, and store |
| `76` | Read 16 bits at `M + 2`, encode, and store |
| `30` | Read 32 bits at `M + 8` and emit |
| `26` | Read 32 bits at `M + 4` and emit |

The write at relative offset `30` precedes the write at `26`; address order does not determine execution order.

### Third record header and signed displacement

Define:

**H₃ = H₂ + B + 103**

The function encodes H₃ and stores the result at offset `L − 6`.

At H₃, it then:

1. Stores byte `13` at relative offset `0`.
2. Emits `2` at relative offset `2`.
3. Stores byte `6` at relative offset `6`.
4. Calls F-aa082a48ffa6 with the static “System” source, destination relative offset `7`, and count `6`.
5. Rereads the byte at H₃.

Let the reread byte be **C**, interpreted unsigned. The third attribute-block displacement, **R**, is obtained by taking the low 16 bits of `H₃ + C + 1` and interpreting those bits as a signed 16-bit integer.

This is a signed displacement, not an unsigned 16-bit offset. Under the visible bounds of A, B, and C, however:

- H₂ ranges from `117` through `372`.
- H₃ ranges from `220` through `730`.
- `H₃ + C + 1` ranges from `221` through `986`.

Consequently, with ordinary nonwrapping address calculations, the visible byte-derived arithmetic cannot reach the signed-negative range. The signed narrowing remains part of the operation even though these bounds make it immaterial in that case.

### Third attribute block and final trailing entry

Relative to R:

| Offset from R | Operation |
|---|---|
| `0` | Encode and directly store `0x0200` |
| `4` | Emit a third 32-bit global metadata value |
| `8` | Emit the previously captured second word of the static composite-label storage |
| `12` | Encode and directly store `0x5100` |
| `20` | Emit `0x14` |
| `44` | Emit V |
| `48` | Emit V again through a separate call sequence |
| `86` | Read 16 bits at `M − 12`, encode, and store |
| `88` | Read 16 bits at `M − 10`, encode, and store |
| `40` | Read 32 bits at `M − 4` and emit |
| `36` | Read 32 bits at `M − 8` and emit |

Finally, the function encodes `R + 102`, stores the result at offset `L − 8`, and returns.

The four trailing stores therefore occur at descending addresses in this chronological order:

1. `L − 2`
2. `L − 4`
3. `L − 6`
4. `L − 8`

No final length, checksum, status flag, or completion marker is visibly written afterward.

## Side effects

The function directly modifies caller-supplied destination memory and invokes dependencies that may have additional effects.

The direct stores are not a complete byte-by-byte initialization of the constructed region. Gaps exist between explicitly written fields. Their contents cannot be specified without the initializer and helper contracts.

No visible action:

- Allocates or frees storage.
- Validates a total buffer size.
- Checks dependency success.
- Reports an error.
- Rolls back partial construction.
- Prevents overlap between trailing entries and earlier fields.

If L is small, the trailing stores may overlap constructed fields or address memory before the destination. For example, `L = 0` places them at destination offsets `−2`, `−4`, `−6`, and `−8`, subject to machine address behavior. Such values are not rejected.

Dependency effects can influence later behavior. In particular, the three header bytes are reread after intervening calls, and metadata is read incrementally rather than copied into a complete local snapshot.

## Dependencies

| Opaque function ID | Established use | Not established |
|---|---|---|
| F-c351b855a319 | Receives the 32-bit input-object field at `0x30`; supplies retained 32-bit V | Meaning, global effects, failure behavior |
| F-0ec8efa9c769 | Receives values for encoded fields; its 16-bit result is directly stored | Byte order, parameter type, validation |
| F-b5c826d2ab04 | Receives each value in the 32-bit emission sequence | Whether it is a byte-order conversion or another transformation |
| F-946cac4b59bc | Receives the preceding conversion result and destination address | Write width, alignment handling, exact memory effects |
| F-aa082a48ffa6 | Receives static source, destination, and count `6` for each label | Exact copy semantics, termination behavior, other effects |

An additional external dependency receives the destination and L before construction. Its opaque function ID is absent from the supplied map. Its initialization behavior is **documented_only**; zero-filling, allocation, and bounds enforcement must not be assumed.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests.

### Conventional helper behavior

Assume the external initializer provides valid destination storage, the emission helper writes the expected field width, label calls copy exactly six bytes, and dependencies do not alter the header bytes after they are set.

Then:

- A, B, and C are all `13`.
- P is `28`.
- H₂ is `130`.
- Q is `144`.
- H₃ is `246`.
- R is `260`.
- The trailing encoded values, in write order, are `14`, `130`, `246`, and `362`.

The last direct third-block metadata store occupies offsets `348` and `349`. The value `362` in the final trailing entry does not itself cause a write at offset `362`.

### High bits of the input length

Input length fields `0x00000180` and `0xabcd0180` both produce L equal to `384`. With otherwise identical reads and dependency behavior, they supply the same initializer length and trailing-entry addresses.

### Header mutation by a dependency

If the byte at offset `14` is `14` when reread, while subsequent reread bytes remain `13`, then:

- P is `29`.
- H₂ is `131`.
- Q is `145`.
- H₃ is `247`.
- R is `261`.
- The final encoded value is `363`.

The function does not force the first reread value back to `13`.

### Retained versus reread data

If static composite-label storage changes after its second word has been captured, the third block still receives the captured word. By contrast, changes to metadata memory can affect later reads through M; there is no complete metadata snapshot.

### Overlapping trailing entries

A short L that places a trailing entry inside a record is not rejected. The observed result depends on chronological writes: the `L − 4` entry is written before second-record construction, and later construction can overwrite it. No nonoverlap guarantee is established.

## Uncertainties

The following remain **documented_only**:

- The external initializer’s identity and semantics.
- The actual format and interpretation of the binary image.
- Exact byte order and emission-helper write width.
- Dependency error, exception, ownership, and global-state behavior.
- Literal source bytes where only symbolic static-data labels are supplied.
- Concrete values of the global metadata words and metadata contents.
- Alignment requirements and behavior of invalid accesses.
- Machine behavior for address overflow or invalid destination arithmetic.

A clearing initializer, conventional endian conversions, and a four-byte emission helper would explain the visible construction pattern. These are inferred intent, not unconditional guarantees.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency map. Direct access widths, call order, literal arguments, pointer offsets, retained values, and byte rereads are established by that evidence.

Record-format interpretation, helper semantics, and descriptive static-data labels are not independent proof of runtime behavior.

No implementation access, dependency inspection, or runtime tests were performed.
