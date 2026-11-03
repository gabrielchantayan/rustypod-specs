<!-- Trusted provenance: F-13bcc08bc7a3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-13bcc08bc7a3

## Purpose

F-13bcc08bc7a3 moves a caller-maintained byte cursor backward by one, two, or three positions and returns an unsigned integer assembled from the bytes encountered.

The established behavior resembles reverse decoding of a restricted UTF-8 representation. That interpretation is inferred intent, not a validation guarantee. The function recognizes certain leading-byte bit patterns but does not validate continuation bytes, reject overlong encodings, exclude surrogate values, or implement four-byte decoding.

The cursor is updated during processing, before each newly selected byte is read. It is not committed only after a successful decode. A zero result can represent either an assembled value of zero or a failed prefix-pattern check; there is no separate error indication.

## Interface

### Input

The function receives one pointer to a mutable integer-sized cursor slot. The slot contains an address-like integer identifying a position in byte-addressable memory.

Let **P** denote the cursor value read at entry. Bytes are addressed relative to that saved value:

- **B₁** is the byte at P − 1.
- **B₂** is the byte at P − 2.
- **B₃** is the byte at P − 3.

The slot is both an input and an output. The function reads its initial value once and then overwrites it as traversal proceeds. Later byte addresses are derived from the saved entry value, not from rereading the slot.

No buffer base, byte count, lower bound, encoding mode, or error-output parameter is supplied.

### Output

The return type is an unsigned integer. Every normal return is in the inclusive range 0 through 65,535:

- One-byte returns range from 0 through 127.
- Two-byte assembly ranges from 0 through 2,047.
- Three-byte assembly ranges from 0 through 65,535.
- A failed three-byte prefix check returns zero.

The supplied representation uses integer cursor storage and an unsigned integer return type. A 32-bit interpretation is consistent with it, but exact integer and pointer widths require external ABI confirmation. The byte masks and shifts establish the result ranges independently of whether the enclosing unsigned type is wider.

### Preconditions

For a defined normal execution:

- The cursor slot must be readable and writable.
- Every byte selected by the executed branch must be readable.
- Subtracting the selected offsets from P must produce valid addresses under the execution environment.
- The cursor representation must preserve the relevant addresses.
- Concurrent access or overlap between the cursor slot and byte storage must not invalidate the execution model.

The function itself does not check these conditions.

There is no demonstrated ownership transfer. The caller supplies the cursor slot and byte storage. The function contains no allocation, deallocation, or dependency call.

## Behavior

### Initial backward step

The function first reads P from the cursor slot. It then writes P − 1 to that slot before accessing B₁.

B₁ supplies the initial candidate result. Its high bit determines whether traversal stops:

- If B₁ is below 128, the function returns B₁.
- The final cursor value on this path is P − 1.
- No earlier byte is examined on this path.

This behavior includes a zero byte: it returns zero and consumes one position. No terminator-specific handling is present.

At the source-expression level, B₁ is read once for the candidate value and again for the high-bit test. Stable ordinary memory makes these reads equivalent in value. Exact machine-level load count is not established.

### Two-byte decision

If the high bit of B₁ is set, the function writes P − 2 to the cursor slot, then reads B₂.

A two-byte result is selected when the three highest bits of B₂ are `110`. Equivalently, masking B₂ with hexadecimal E0 yields hexadecimal C0.

On that path:

- The low five bits of B₂ become result bits 6 through 10.
- The low six bits of the initially read B₁ become result bits 0 through 5.
- The final cursor value is P − 2.
- B₃ is not accessed.

The assembled value is:

**64 × (B₂ masked with 31) + (B₁ masked with 63).**

The fields do not overlap, so this addition describes the same numeric result as combining the shifted fields by bitwise OR.

The function does not require B₁ to have a continuation-byte prefix. Its high bit must have caused traversal to continue, but its next-highest bit is unrestricted. Consequently, trailing bytes with leading patterns `11` are accepted on this path.

### Three-byte decision

If B₂ does not match the two-byte prefix pattern, the function writes P − 3 to the cursor slot and reads B₃.

A three-byte result is selected when the four highest bits of B₃ are `1110`. Equivalently, masking B₃ with hexadecimal F0 yields hexadecimal E0.

On that path:

- The low four bits of B₃ become result bits 12 through 15.
- The low six bits of B₂ become result bits 6 through 11.
- The low six bits of the initially read B₁ become result bits 0 through 5.
- The final cursor value is P − 3.

The assembled value is:

**4,096 × (B₃ masked with 15) + 64 × (B₂ masked with 63) + (B₁ masked with 63).**

Neither B₂ nor B₁ is checked for the continuation-byte prefix. B₂ is constrained only by having failed the earlier two-byte-prefix test. That branch precedence matters: if B₂ matches `110`, the function returns through the two-byte path without considering B₃, even if B₃ has a three-byte prefix.

If B₃ does not match `1110`, the function returns zero. The cursor nevertheless remains at P − 3.

### Traversal limits and malformed input

The function moves backward at most three positions per call. It does not search arbitrarily far backward for a leading byte, restore the original cursor after a mismatch, or return a distinct malformed-input status.

Four-byte UTF-8 sequences are not decoded as four-byte units. For a well-formed four-byte sequence immediately preceding P, the function examines its final three continuation bytes. The third byte encountered does not match the required three-byte leading pattern, so the function returns zero and leaves the cursor three bytes before P, immediately after the sequence’s leading byte.

No canonicality checks are present:

- An overlong two-byte encoding can return zero or another small value.
- An overlong three-byte encoding can return a value below 2,048.
- Values in the surrogate range can be returned.
- Values up to 65,535 can be returned.
- No normal result exceeds 65,535.

These outcomes follow from the bit operations, not from a claim that the inputs are valid text.

## Side effects

The only explicit writes are to the caller’s cursor slot.

On a stable, non-overlapping, normally returning execution, the cursor-write sequence is one of:

| Selected path | Cursor writes, in order | Final cursor |
|---|---|---|
| One byte | P − 1 | P − 1 |
| Two bytes | P − 1, then P − 2 | P − 2 |
| Three-byte assembly | P − 1, then P − 2, then P − 3 | P − 3 |
| Three-byte prefix mismatch | P − 1, then P − 2, then P − 3 | P − 3 |

Each write precedes the corresponding byte access. Thus, if an environment exposes an access failure after a cursor write, the supplied control flow contains no rollback. Whether such a failure is recoverable, and which state a handler can observe, depends on the execution environment.

The function has no explicit byte-buffer writes. However, if the cursor slot overlaps the byte region, updating the slot can change bytes that subsequent reads encounter. The ordinary decoding description therefore assumes non-overlapping storage. Results under overlap require the slot’s width, byte order, alignment rules, and memory model to be known.

No global access, I/O, synchronization, allocation, or release operation is explicit in the supplied body. Invalid addresses and arithmetic outside the representation’s defined range do not produce a documented function-level error return.

## Dependencies

No dependency calls are present. There are therefore no opaque dependency IDs to enumerate beyond F-13bcc08bc7a3 itself.

Execution relies on the surrounding environment for:

- The integer and address representation used by the cursor slot.
- Byte-addressable readable storage.
- Writable cursor storage.
- Defined address arithmetic and conversions.
- Memory-access and fault semantics.

There is no callee whose preservation of globals, ownership behavior, or memory-release behavior needs to be assumed.

## Acceptance examples

These are conditional acceptance cases derived from static inspection, not runtime observations.

For each row, the listed bytes occupy consecutive addresses immediately before P, in increasing address order. Storage is assumed readable, stable, and separate from the writable cursor slot. Arithmetic and address conversion are assumed valid. Byte and result values are hexadecimal.

| Bytes immediately before P | Expected result | Final cursor | Reason |
|---|---:|---|---|
| `41` | `0041` | P − 1 | The final byte has its high bit clear. |
| `00` | `0000` | P − 1 | Zero is returned directly; it is not treated specially. |
| `C2 A2` | `00A2` | P − 2 | The preceding byte selects two-byte assembly. |
| `DF BF` | `07FF` | P − 2 | Maximum result from the two-byte bit fields. |
| `E2 82 AC` | `20AC` | P − 3 | The second byte examined fails the two-byte prefix test; the third selects three-byte assembly. |
| `EF BF BF` | `FFFF` | P − 3 | Maximum three-byte result. |
| `C0 80` | `0000` | P − 2 | An overlong representation is not rejected. |
| `E0 80 80` | `0000` | P − 3 | Three-byte assembly can also produce zero. |
| `ED A0 80` | `D800` | P − 3 | A surrogate-range value is returned without exclusion. |
| `C2 FF` | `00BF` | P − 2 | The final byte is not validated as a continuation byte. |
| `E2 41 FF` | `207F` | P − 3 | The middle byte need not have a continuation prefix. |
| `E2 C2 A2` | `00A2` | P − 2 | The two-byte match takes precedence; the earliest listed byte is not read. |
| `41 80 80` | `0000` | P − 3 | The third byte examined fails the three-byte prefix check. |
| `F0 9F 98 80` | `0000` | P − 3 | No four-byte path exists; the leading byte remains before the final cursor. |

Branch-local memory requirements are also part of acceptance:

- With a high-bit-clear B₁, no readable B₂ or B₃ is required by the executed path.
- With a matching two-byte prefix in B₂, no readable B₃ is required.
- Otherwise, B₃ is accessed even when it will cause a zero return.
- A zero return alone cannot identify how far the cursor moved or whether prefix recognition succeeded.

## Uncertainties

### Established static evidence

The supplied body establishes backward offsets of one through three, incremental cursor writes, prefix-mask branch conditions, payload assembly, a zero fallback, and absence of explicit dependency calls.

It also establishes that the cursor is not restored on prefix mismatch and that no continuation-byte, canonicality, or Unicode scalar-value validation is performed.

### Inferred intent

The prefix patterns and payload widths strongly suggest a reverse decoder for a subset of UTF-8-like encodings. Treating the result as a Unicode character, code unit, or application-specific character value requires caller context not supplied here.

The reason for limiting traversal to three bytes is unknown.

### documented_only

The following remain unverified:

- Exact ABI widths, signed-integer behavior, and integer-to-address conversion rules.
- Whether the supplied representation exactly preserves the original machine-level memory accesses.
- Physical load and store counts, compiler reordering, and behavior for volatile or concurrently modified memory.
- Supported behavior for overlapping cursor and byte storage.
- Fault handling, invalid-address behavior, and arithmetic at address-space boundaries.
- Caller-side bounds checks, encoding guarantees, and interpretation of zero results.

No numeric wraparound contract is established for cursor subtraction. In particular, the description does not promise that subtracting past the representable or mapped address range safely wraps.

## Evidence

This contract is based solely on static inspection of the supplied function body. No implementation files, callers, external documentation, or runtime environment were inspected. No runtime tests were performed.

The behavioral claims follow from the visible cursor assignments, byte accesses, masks, shifts, branch order, and returns. Acceptance examples are calculated consequences under their stated preconditions, not observed executions. No snapshot hash or independently verified provenance is asserted.
