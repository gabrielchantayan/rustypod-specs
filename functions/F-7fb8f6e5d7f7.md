<!-- Trusted provenance: F-7fb8f6e5d7f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7fb8f6e5d7f7

## Purpose

F-7fb8f6e5d7f7 performs a count-bounded, forward byte transfer with a per-byte arithmetic transformation.

The transformation is **not limited to lowercase ASCII letters** under the ordinary unsigned-byte and signed-integer interpretation of the supplied source. Every input byte from `0x00` through `0x7A` is reduced by `0x20`, with the result wrapped into an eight-bit byte. Bytes from `0x7B` through `0xFF` are copied unchanged.

ASCII lowercase letters consequently become uppercase letters, but control characters, digits, punctuation, and existing uppercase letters also change. Describing the operation simply as “uppercase conversion” would omit observable behavior.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The function accepts three arguments, in this order:

| Argument | Role | Representation |
|---|---|---|
| Destination | Starting address for output writes | Pointer to byte-addressed storage |
| Source | Starting address for input reads | Pointer to byte-addressed storage |
| Count | Signed bound on the number of iterations | Integer |

The function returns no value. Output is communicated through destination memory.

For a positive count, the effective output length is exactly that count. For zero or a negative count, the effective output length is zero.

### Width and representation qualifications

The supplied source uses a byte type and an integer type without providing their definitions or target ABI.

The numerical byte behavior specified here uses the conventional interpretation of a byte as an **unsigned eight-bit quantity**, with values from 0 through 255. Under ordinary integer-promotion rules, such a byte is promoted to a signed integer before the threshold arithmetic. That signed arithmetic is important to the result.

The exact width of the count is **documented_only**: the supplied text does not independently establish that it is 32 bits. If the target integer is 32-bit signed, the representable count range is −2,147,483,648 through 2,147,483,647. The function does not impose a separate, smaller bound.

The pointer width, address-space size, alignment requirements beyond byte access, and concrete pointer representation are not established.

### Preconditions for defined memory behavior

For a positive count:

- Each source location reached during the transfer must be readable when its iteration executes.
- Each destination location reached during the transfer must be writable when its iteration executes.
- Advancing each pointer through the requested span must be valid under the applicable memory model, including forming its final advanced value.
- The caller must provide any synchronization needed to prevent invalid concurrent memory access.

Neither separate allocations nor nonoverlapping regions are required by the visible access pattern. Overlap is permitted as a behavioral case, but it can change which byte later iterations read.

For a nonpositive count, the function does not dereference or advance either pointer. No readable or writable byte span is required by the function body in that case.

## Behavior

### Count handling

The signed count is tested against an iteration position initially equal to zero.

If the count is zero or negative, the function returns without reading source memory or writing destination memory.

For a positive count, the function processes positions in increasing address order, starting at the supplied addresses. It does not stop at a zero byte or any other byte value.

There is no explicit count conversion to an unsigned length. A negative count therefore does not request a very large transfer.

### Per-byte transformation

Let an input byte’s numerical value be \(x\), with \(0 \le x \le 255\).

The established transformation, under the representation qualifications above, is:

| Input range | Output |
|---|---|
| `0x00`–`0x1F` | Input minus `0x20`, wrapped modulo 256; produces `0xE0`–`0xFF` |
| `0x20`–`0x7A` | Input minus `0x20`; produces `0x00`–`0x5A` |
| `0x7B`–`0xFF` | Input unchanged |

The threshold follows from a signed comparison of the byte value minus `0x61` against `0x1A`. For inputs below `0x61`, the subtraction produces a negative integer, which still satisfies that comparison. There is no separate lower-bound check.

The arithmetic remains within the range of an ordinary signed integer capable of representing all eight-bit unsigned values. Negative transformed values are converted back to the unsigned byte representation, producing the wraparound described above.

No locale, encoding table, Unicode rule, or character classification service participates.

### Observable access order

Each iteration:

- Reads one byte from the current source location.
- Determines its transformed value.
- Writes one byte to the current destination location.
- Advances both locations by one byte before the next iteration.

The read for an iteration precedes its write. The write completes in the source-level execution order before the following iteration’s read.

The function does not first capture the entire input span. Its behavior therefore depends on memory as encountered during the transfer, including changes caused by its own earlier writes.

### Overlap

With identical source and destination addresses, each byte is read and then replaced by its transformed value. Each position is visited once.

When the destination begins inside the source span at a higher address, an earlier write can replace a byte that a later iteration will read. That replacement is then transformed again. The result is not generally equivalent to transforming a snapshot of the original source.

When the destination begins at a lower address, ordinary overlapping spans do not cause the function’s earlier writes to overwrite later source positions. Other concurrent changes remain outside the guarantees of this contract.

### Boundary arithmetic

For a positive, representable count, the iteration position increases from zero to the count without needing a value greater than the count. The visible loop structure therefore does not require signed overflow of that position, even when the count is the signed integer type’s maximum value.

This does not establish that a memory span of that size exists or that the pointer advances are valid.

## Side effects

The function directly writes exactly one destination byte per completed iteration. For valid, uninterrupted execution with a positive count, the directly written addresses are precisely the consecutive destination span of that length.

It does not:

- Append a terminator.
- Clear unused destination storage.
- Allocate or release memory.
- Transfer ownership of either buffer.
- Change the caller’s stored pointer values merely by advancing its local pointer copies.
- Directly access a named global object in the supplied body.
- Report a status, processed length, or error code.

The source parameter is not used for direct writes. Nevertheless, memory reachable through it may change when destination writes alias that memory.

The caller retains responsibility for buffer lifetime and capacity. Nothing in the supplied body establishes recovery from invalid addresses, inaccessible pages, or other memory-access failures. Such failures are not specified as successful partial transfers or as any particular exception.

The implementation provides no explicit transaction or rollback mechanism. Its access order establishes progressive writes, not all-or-nothing output.

## Dependencies

No calls to other functions are visible in F-7fb8f6e5d7f7.

The behavior depends on the execution environment’s byte representation, integer promotions, conversion rules, and memory-access semantics. No independent dependency function requires an opaque identifier.

Because no callees are present, this contract makes no assumptions about unknown callees preserving globals, acquiring ownership, or releasing memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observations from executed tests. Unless otherwise stated, they assume unsigned eight-bit bytes, ordinary signed integer promotion, valid accessible spans, and no concurrent modification.

### Nonpositive count

For a count of zero or any representable negative value:

- No source byte is read.
- No destination byte is written.
- Existing destination contents remain unchanged.

This case does not require either pointer to designate an accessible byte, provided passing the pointer values themselves is valid in the execution environment.

### Transformation boundaries

With separate buffers and count seven:

| Input | Expected output |
|---|---|
| `00 1F 20 60 61 7A 7B` | `E0 FF 00 40 41 5A 7B` |

This distinguishes the actual lower-range behavior from a transformation restricted to ASCII lowercase letters.

### Existing uppercase and high bytes

With separate buffers and count five:

| Input | Expected output |
|---|---|
| `41 5A 80 FE FF` | `21 3A 80 FE FF` |

Existing ASCII uppercase letters are changed; bytes above `0x7A` are not.

### Embedded zero and exact length

For input `61 00 62 63` and count three, the destination’s first three bytes become `41 E0 42`.

The fourth destination byte is not written by this invocation. The embedded zero neither terminates processing nor remains zero.

### In-place operation

For a single buffer containing `61 62 7B`, using the same starting address for both arguments and count three produces `41 42 7B`.

### Forward-overlap propagation

For a buffer initially containing `61 62 63 64`, use its first byte as the source, its second byte as the destination, and count three.

The resulting complete buffer is `61 41 21 01`.

The second iteration reads `0x41` written by the first iteration, rather than the original `0x62`. The third similarly reads `0x21`. This demonstrates progressive overlap behavior rather than snapshot semantics.

## Uncertainties

- **Established static evidence:** Signed count control, forward traversal, one source read followed by one destination write per iteration, conditional subtraction, no explicit terminator handling, and no visible calls.
- **Representation qualification:** The exact numerical mapping assumes the byte type is unsigned and eight bits wide. Its declaration is not included.
- **documented_only — ABI:** Exact integer width, pointer width, calling convention, and binary-level layout are not independently established.
- **Inferred intent:** ASCII lowercase-to-uppercase conversion is a plausible purpose of the constants. It is not a sufficient description of the supplied arithmetic and must not replace the wider mapping.
- **documented_only — invalid memory:** Fault delivery, partial effects following an invalid access, and platform-specific recovery are unspecified.
- **documented_only — external memory effects:** No guarantees are established for concurrent mutation, memory-mapped devices, or observation requiring volatile or atomic access semantics.

## Evidence

The supplied body supports the count-bounded forward traversal and per-byte read-before-write order. Its signed threshold arithmetic includes values below `0x61`; its byte assignment supplies the wraparound under the stated unsigned eight-bit interpretation.

There is no visible validation, allocation, deallocation, dependency call, terminator check, or explicit error-reporting path.

All acceptance examples are statically derived. No runtime tests, binary inspection, or independent type-definition verification have been performed.
