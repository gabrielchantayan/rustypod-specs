<!-- Trusted provenance: F-d3922c7fe0ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3922c7fe0ec

## Purpose

F-d3922c7fe0ec consumes a byte sequence through a caller-supplied cursor, updates that cursor in place, and returns an unsigned integer assembled from selected bits of the input.

For stable, non-overlapping input, its behavior resembles decoding the one-, two-, and three-byte forms of UTF-8. This resemblance is **inferred intent**, not a validation guarantee. The function does not validate continuation-byte prefixes, reject overlong encodings, reject surrogate values, or support four-byte decoding. Unsupported leading-byte patterns return zero after advancing the cursor by three bytes.

The contract below describes the supplied static representation. No runtime tests have been performed.

## Interface

### Input and mutable state

The sole argument identifies a writable cursor cell. On entry, that cell contains the address of the first input byte.

The supplied representation uses:

- A four-byte cursor cell.
- Byte-sized input elements, interpreted as unsigned values from 0 through 255.
- An unsigned integer return type, treated here as 32 bits under the represented target convention.

The actual platform ABI and pointer representation are not independently established. In particular, this representation is not evidence that the interface can be transplanted unchanged to a platform with wider pointers.

The function has no length argument, end pointer, output buffer, or explicit error-reporting argument.

### Output

The function returns an unsigned integer. Under stable-input conditions, the result is always between 0 and 65,535 inclusive.

It also replaces the cursor value with an address one, two, or three bytes beyond the original input address. The final advancement depends on the branch taken, not on how many bytes were actually read.

### Preconditions for ordinary use

To obtain a defined, useful result under the apparent interface:

- The argument must identify a valid cursor cell that can be read and written.
- The cell must initially contain a valid input address.
- Each byte actually read on the selected path must be accessible.
- The pointer advances performed on that path must be valid under the target’s address and object model.
- For the stable-input interpretation and acceptance examples below, the cursor cell must not overlap the input bytes, and the bytes must not change during execution.

There is no explicit check for any of these conditions.

The function does not receive enough information to establish that an input sequence is complete. Providing a truncated buffer does not cause a defined “incomplete sequence” result; it may instead cause an invalid memory access.

## Behavior

### Cursor origin and logical access order

The function first reads the input address from the cursor cell and retains that original address as the basis for subsequent accesses and cursor updates. It does not reload the cursor cell to locate the second or third input byte.

The supplied representation establishes the following logical sequence:

1. Replace the cursor with the original address plus one byte.
2. Read the first input byte and retain its unsigned value for subsequent prefix classification and result construction.
3. Read the first byte again to test its high bit.
4. If that high bit is clear, return the retained first-byte value.
5. Otherwise, replace the cursor with the original address plus two bytes.
6. Read the second byte.
7. Classify the retained first-byte value:
   - A two-byte prefix produces a result immediately.
   - Every other prefix first causes the cursor to be replaced with the original address plus three bytes.
8. After that third cursor position has been stored:
   - A three-byte prefix causes the third byte to be read and a result to be assembled.
   - Any other prefix returns zero without reading the third byte.

These are source-level logical accesses. The supplied material does not establish the exact number or timing of physical memory transactions in an executable. The first byte’s repeated appearance, in particular, is not evidence of two guaranteed hardware loads.

The cursor is advanced before the corresponding byte access. No rollback is present.

### Stable-input branch classification

When the first byte remains unchanged across its logical reads, behavior is:

| First byte | Final cursor advancement | Bytes read | Return construction |
|---|---:|---|---|
| 0x00–0x7F | 1 byte | First | First byte, unchanged |
| 0x80–0xBF | 3 bytes | First and second | Zero |
| 0xC0–0xDF | 2 bytes | First and second | Two-byte assembly |
| 0xE0–0xEF | 3 bytes | First, second, and third | Three-byte assembly |
| 0xF0–0xFF | 3 bytes | First and second | Zero |

“Bytes read” identifies distinct input positions, not physical load counts.

A zero byte takes the ordinary one-byte path. It is not treated as an end-of-input marker.

### One-byte result

If the high bit tested in the first byte is clear, the function returns the previously retained first-byte value.

For stable input, this is an identity mapping over 0 through 127, with a one-byte cursor advance.

### Two-byte result

The two-byte path is selected when the retained first byte has the bit pattern `110xxxxx`.

The returned value is the bitwise combination of:

- The low five bits of the first byte, shifted left by six positions.
- The low six bits of the second byte.

Equivalently, for unsigned byte values \(b_0\) and \(b_1\):

\[
((b_0 \mathbin{\&} 31) \ll 6)
\mathbin{|}
(b_1 \mathbin{\&} 63).
\]

The result range is 0 through 2,047. The shifted first-byte field occupies bits 6–10; the second-byte field occupies bits 0–5. These fields do not overlap.

The upper two bits of the second byte are ignored. They need not have the UTF-8 continuation pattern.

### Three-byte result

If the two-byte prefix test fails, the cursor is advanced to the original address plus three bytes before the next prefix decision.

The three-byte assembly is selected when the retained first byte has the bit pattern `1110xxxx`.

The returned value combines:

- The low four bits of the first byte, shifted left by twelve positions.
- The low six bits of the second byte, shifted left by six positions.
- The low six bits of the third byte.

For unsigned byte values \(b_0\), \(b_1\), and \(b_2\):

\[
((b_0 \mathbin{\&} 15) \ll 12)
\mathbin{|}
((b_1 \mathbin{\&} 63) \ll 6)
\mathbin{|}
(b_2 \mathbin{\&} 63).
\]

The result range is 0 through 65,535. The three fields occupy disjoint bit ranges: 12–15, 6–11, and 0–5.

The upper two bits of both trailing bytes are ignored. Neither byte is checked for a continuation prefix.

### Unsupported prefixes

After entering the high-bit-set path, any retained first byte that matches neither the two-byte nor three-byte prefix returns zero.

This path:

- Reads the second byte.
- Advances the cursor by three bytes.
- Does not read the third byte.

Under stable input, this covers leading values 0x80–0xBF and 0xF0–0xFF.

In particular, an apparent four-byte UTF-8 lead does not initiate four-byte decoding. It produces zero and a three-byte advance. No fourth byte is accessed.

### Validation and arithmetic boundaries

There is no separate validation phase.

The function accepts the payload bits of malformed trailing bytes, overlong encodings, and encodings whose numerical result lies in the surrogate range. It does not distinguish such cases from otherwise ordinary results.

The integer masks and shifts fit within a 32-bit unsigned result. No integer payload operation requires overflow or truncation beyond the documented masks.

Pointer addition is separate from payload arithmetic. The supplied representation does not establish behavior for address overflow, invalid pointer arithmetic, or an address that cannot be represented in the four-byte cursor cell.

## Side effects

The only explicit writes are replacements of the caller’s cursor cell:

- All paths store the original input address plus one byte.
- High-bit-set paths additionally store the original address plus two bytes.
- Paths other than the one-byte and two-byte return paths additionally store the original address plus three bytes.

The function does not explicitly write the input bytes.

Cursor and input overlap is not prevented. If they overlap, a cursor store can alter a byte before that byte is read. Consequently, the stable-input classification table is not a general aliasing guarantee. Exact overlapping behavior depends on the target’s pointer representation, byte order, memory rules, and any intervening changes.

The two logical first-byte reads also have distinct roles: the first supplies the retained value, while the second supplies the high-bit decision. If those reads observe different values, the first branch need not agree with the prefix of the retained value. Such mutation is outside the acceptance examples and is not assigned an additional synchronization contract.

No allocation, release, ownership transfer, explicit global access, callback, or external function call is shown. The caller supplies the cursor storage and input storage. The function contains no explicit mechanism for retaining either storage object after return.

Invalid memory access has no defined error return in the supplied material. A cursor store precedes each newly required byte access, and no recovery or restoration is shown. Whether any partial state is observable after a fault depends on the execution environment; no fault-handling guarantee is established.

## Dependencies

No called-function dependency is present in the supplied body.

F-d3922c7fe0ec depends only on the represented environment’s support for:

- Reading and writing the cursor cell.
- Reading unsigned bytes.
- Byte-addressed pointer advancement.
- Unsigned masking, shifting, and bitwise combination.

No assumptions about unknown callees preserving globals or managing memory are needed, because no callee is shown.

## Acceptance examples

These examples are conditional specifications derived from static evidence, not recorded test results. Unless stated otherwise, input bytes are stable, cursor storage does not overlap them, required accesses and pointer advances are valid, and the cursor initially points to the first listed byte.

| Input bytes, hexadecimal | Expected return | Expected advance | Behavior illustrated |
|---|---:|---:|---|
| `00` | 0 | 1 | Zero is an ordinary one-byte value |
| `7F` | 127 | 1 | Highest one-byte value |
| `C2 A2` | 162 | 2 | Ordinary two-byte assembly |
| `C0 80` | 0 | 2 | Overlong zero is not rejected |
| `DF FF` | 2,047 | 2 | Upper trailing-byte bits are ignored |
| `E2 82 AC` | 8,364 | 3 | Ordinary three-byte assembly |
| `E0 80 80` | 0 | 3 | Three-byte overlong zero is not rejected |
| `ED A0 80` | 55,296 | 3 | A surrogate-range result is returned |
| `EF FF FF` | 65,535 | 3 | Maximum three-byte payload result |
| `80 00` | 0 | 3 | Unsupported prefix still reads a second byte |
| `F0 90 80 80` | 0 | 3 | No four-byte decoding; only first and second bytes are read |

Additional conditional acceptance criteria:

- Replacing `C2 A2` with `C2 22` must leave the result at 162 and the advancement at two bytes. Both second-byte values have the same low six bits.
- Replacing `E2 82 AC` with `E2 02 2C` must leave the result at 8,364 and the advancement at three bytes.
- An unsupported leading byte must not require a third-byte read. This does not remove the requirement that its three-byte cursor advance be valid.
- A supported three-byte prefix requires a third-byte read after the cursor has already been advanced by three bytes.
- A two-byte prefix requires the second-byte read after the cursor has already been advanced by two bytes.
- A returned zero alone cannot identify an error or determine the advancement. Zero can arise from a one-byte zero, a two-byte payload, a three-byte payload, or an unsupported prefix.

No expected successful result is specified for a truncated input whose selected path reads beyond accessible storage.

## Uncertainties

### Established static evidence

The supplied body establishes the prefix masks, payload masks, shifts, branch structure, logical cursor updates, and absence of explicit bounds checks or external calls.

It also establishes that unsupported prefixes take a three-byte cursor advance without reading a third byte.

### Inferred intent

The bit patterns and payload assembly strongly suggest a limited UTF-8-like decoder. They do not establish that the surrounding system supplies valid UTF-8, that zero is an intended error marker, or that three-byte advancement on unsupported input is a recovery policy.

### documented_only

The following remain unestablished:

- The exact executable ABI and pointer encoding.
- Whether the reconstructed types match every original declaration.
- Physical memory-access counts and compiler transformations.
- Input validity guarantees imposed by callers.
- End-of-buffer conventions.
- Fault handling and post-fault observability.
- Concurrency, synchronization, and memory-mapped-I/O requirements.
- Whether cursor/input overlap is permitted by the wider interface.
- Any surrounding interpretation of the returned integer.

These unknowns do not justify adding validation, replacement characters, exceptions, rollback, four-byte support, or other behavior absent from the supplied representation.

## Evidence

This contract is based solely on the supplied static body for F-d3922c7fe0ec.

The decisive evidence is:

- A retained original input address used for all subsequent byte offsets.
- Cursor stores at offsets one, two, and three along the described paths.
- A high-bit decision for the initial return.
- A two-byte prefix mask of 0xE0 compared with 0xC0.
- A three-byte prefix mask of 0xF0 compared with 0xE0.
- Payload masks of 0x1F, 0x0F, and 0x3F.
- Payload shifts of six and twelve bits.
- An explicit zero result for the remaining high-bit-set cases.
- No third-byte access on that zero-result path.
- No explicit calls, length checks, continuation-prefix checks, or ownership operations.

No runtime execution, binary inspection, caller analysis, or external-document verification has been performed.
