<!-- Trusted provenance: F-53be0ee86aef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-53be0ee86aef

## Purpose

F-53be0ee86aef writes a variable-length, most-significant-group-first representation of an unsigned 32-bit value into caller-provided storage and returns an unsigned 64-bit result.

The apparent encoding intent is seven payload bits per byte, with bit 7 marking a continuation byte. Values below 16,384 always occupy two bytes, including values that would fit in one seven-bit group.

That interpretation is fully established for the two-byte branch. For larger values, it depends on the scratch-storage layout and byte order described below. The supplied representation uses byte writes into and beyond a four-byte scratch object, followed by a numeric bit clear on that object. Those details prevent an unconditional, platform-independent encoding claim.

The returned value is not merely the number of bytes written. Its low 32 bits hold the length; its high 32 bits contain input-derived or scratch-derived information.

## Interface

### Inputs

- **Destination:** a pointer to writable byte-addressed storage.
- **Value:** an unsigned 32-bit integer, denoted \(N\), in the range 0 through 4,294,967,295.

Each output element is an eight-bit byte.

There is no capacity argument. The caller must provide enough writable storage for the selected output length:

| Input range, inclusive | Bytes written |
|---|---:|
| 0–16,383 | 2 |
| 16,384–2,097,151 | 3 |
| 2,097,152–268,435,455 | 4 |
| 268,435,456–4,294,967,295 | 5 |

A five-byte destination is sufficient for every input under the intended scratch-layout assumptions.

### Result

The result is an unsigned 64-bit integer. Define its low and high halves as the least-significant and most-significant 32 bits, respectively.

For \(N < 16,384\):

- Low half: 2.
- High half: \(N \mathbin{\&} 127\).

Equivalently:

\[
R = ((N \mathbin{\&} 127)\times 2^{32}) + 2.
\]

For \(N \ge 16,384\):

- Low half: the number of seven-bit extraction iterations, between 3 and 5.
- High half: the low 24 bits of a captured 32-bit scratch value, as defined in **Behavior**.

No signed interpretation of either half is established. In particular, callers must not treat the entire 64-bit result as a byte count.

### Preconditions and ownership

The destination must remain writable for the entire operation and must have sufficient capacity. No null-pointer, capacity, or accessibility check appears in the supplied representation.

The function does not allocate, release, or retain caller storage. Ownership of the destination remains with the caller. No dependency call is present.

The larger-value branch additionally requires a valid interpretation of the scratch-byte accesses. Five-byte values require an access at offset four from a four-byte scratch object. The supplied representation alone does not establish portable validity for that access.

## Behavior

### Branch selection

The original input is compared against the two-byte threshold through an unsigned right shift by 14 bits:

- A zero shifted result selects the two-byte branch.
- A nonzero shifted result selects the extraction-and-reversal branch.

For a 32-bit unsigned input, this is exactly the boundary between 16,383 and 16,384.

### Two-byte branch

The function performs two destination writes, in ascending address order:

1. At destination offset zero, it writes the upper seven payload bits of the 14-bit input, with bit 7 set.
2. At destination offset one, it writes the lower seven payload bits, with bit 7 clear.

Thus the byte values are:

\[
B_0 = (N \mathbin{\mathrm{>>}} 7) \mathbin{|} 128,
\qquad
B_1 = N \mathbin{\&} 127.
\]

The branch condition ensures that the shifted value fits in seven bits before the continuation bit is added. No payload information is lost through the eight-bit conversion.

The function then returns the two-half result described in **Interface**.

Consequences:

- Zero produces two bytes, not one.
- Inputs 0 through 127 receive a leading byte of 128.
- Inputs 128 through 16,383 use the usual two seven-bit groups.
- The returned high half contains the final seven-bit payload, not the first emitted byte and not the complete input.

This branch does not depend on scratch-word byte order.

### Larger-value branch: extraction

For the remaining inputs, the function initializes internal 32-bit scratch state from the original input and extracts seven-bit groups from least significant to most significant.

Define the extracted groups as:

\[
g_j = (N \mathbin{\mathrm{>>}} 7j) \mathbin{\&} 127.
\]

The extraction count \(k\) is the smallest positive integer such that shifting the original input right by \(7k\) bits has exhausted its significant bits. Within this branch, \(k\) is 3, 4, or 5.

Each iteration:

- Writes \(g_j \mathbin{|} 128\) to scratch-byte offset \(j\).
- Captures the current numeric value of the four-byte scratch word.
- Shifts the remaining unsigned input right by seven bits.
- Advances the extraction count.

The iteration that exhausts the input still writes its group and captures the scratch word. There is no extra zero group after it.

For five-byte inputs, the last group is between 1 and 15 because only four input bits remain after four seven-bit shifts. This final extraction writes at scratch-byte offset four, beyond the first four-byte scratch object in the supplied representation.

### Larger-value branch: finalization and destination writes

After extraction, the function clears numeric bit 7 of the first 32-bit scratch word.

It then reads scratch bytes in descending offset order, from \(k-1\) down to zero, and writes them to destination offsets zero through \(k-1\), respectively.

The destination write order is therefore ascending, even though scratch reads run in reverse order. All extraction writes occur before destination emission begins.

The scratch-word value used for the return is the last captured value from the extraction phase. It is not a fresh capture after clearing bit 7.

Let \(W\) denote that captured 32-bit value. The return is:

\[
R = ((W \mathbin{\&} \text{0x00FFFFFF})\times 2^{32}) + k.
\]

The return mask also clears bit 31 of the low half. Since \(k\) is only 3 through 5, that clearing does not change the length.

### Conventional little-endian interpretation

If the four-byte scratch word is little-endian, numeric bit 7 belongs to scratch-byte offset zero. Clearing that bit removes the continuation marker from the least-significant extracted group, which becomes the final emitted byte after reversal.

Provided all required scratch offsets are valid, the resulting output is:

- The most-significant seven-bit group first.
- Bit 7 set in every byte except the final byte.
- Bit 7 clear in the final byte.
- No extra leading zero group in this branch.

For these conditions, the returned high half can be expressed independently of the untouched scratch bytes:

\[
H =
(g_0 \mathbin{|} 128)
+ 2^8(g_1 \mathbin{|} 128)
+ 2^{16}(g_2 \mathbin{|} 128).
\]

All three groups exist because this branch requires at least three extraction iterations.

This high half retains the continuation bit in its least-significant byte, despite the later clearing of the corresponding emitted byte. It is a pre-finalization snapshot, not a packed copy of the final output.

### Arithmetic boundaries

The input shifts are unsigned, so they introduce zeros rather than sign bits.

No input requires more than five iterations. The extraction count and destination index remain small and do not approach 32-bit integer overflow.

The representation performs scratch address arithmetic through a signed integer conversion. Correct address reconstruction therefore depends on the target’s address model and conversion behavior. A portable guarantee for wider pointer models is not established.

## Side effects

### Caller-visible memory

On a successful execution under the stated preconditions, the function writes exactly the selected number of destination bytes.

The supplied representation contains:

- No explicit destination reads.
- No destination terminator beyond the encoded bytes.
- No capacity update.
- No clearing of destination bytes after the encoded region.
- No explicit access to global state.

Bytes following the output remain unwritten by the shown operations. An invalid destination or insufficient capacity is not handled through a defined error return.

The listed access order describes the supplied operations. It does not establish machine-level timing, atomicity, or volatile-device access semantics.

### Internal storage

The larger-value branch modifies internal scratch storage during extraction and finalization. Those modifications affect both the emitted bytes and the high half of the return value.

The supplied layout also shows additional initialized scratch words. Their initialization does not establish ownership of storage beyond the first word or portable permission to access it through an offset from that word.

### Errors and partial results

There is no explicit error path.

If a destination access fails, no rollback is shown. Earlier destination writes may already have occurred. Neither a normal return nor a particular fault mechanism is specified for invalid memory.

No validation rejects any unsigned 32-bit input value.

## Dependencies

No calls to other opaque functions are present in the supplied body. There are therefore no dependency function IDs to list.

In particular, this function does not visibly delegate allocation, encoding, validation, or error reporting. No assumptions about an unknown callee’s global-state preservation or memory ownership are needed.

Execution still depends on environmental properties that are not function calls:

- Eight-bit bytes and the represented integer widths.
- Writable destination memory.
- Scratch-storage addressability.
- Integer-to-address conversion behavior.
- Scratch-word byte order.

## Acceptance examples

All byte sequences below are hexadecimal. Returned values are shown as 16-digit hexadecimal unsigned 64-bit values.

The two-byte examples follow directly from the supplied operations and do not require a scratch-endianness assumption.

| Input | Destination bytes | Return value |
|---:|---|---|
| 0 | 80 00 | 0x0000000000000002 |
| 1 | 80 01 | 0x0000000100000002 |
| 127 | 80 7F | 0x0000007F00000002 |
| 128 | 81 00 | 0x0000000000000002 |
| 16,383 | FF 7F | 0x0000007F00000002 |

The following examples are conditional on little-endian scratch-word storage, valid scratch-byte access through offset \(k-1\), and usable address conversions:

| Input | Destination bytes | Return value |
|---:|---|---|
| 16,384 | 81 80 00 | 0x0081808000000003 |
| 2,097,151 | FF FF 7F | 0x00FFFFFF00000003 |
| 2,097,152 | 81 80 80 00 | 0x0080808000000004 |
| 268,435,455 | FF FF FF 7F | 0x00FFFFFF00000004 |
| 268,435,456 | 81 80 80 80 00 | 0x0080808000000005 |
| 4,294,967,295 | 8F FF FF FF 7F | 0x00FFFFFF00000005 |

Additional acceptance conditions:

- **Exact capacity:** With a valid destination whose capacity equals the selected length, only that many destination bytes are written.
- **Unused suffix:** If accessible sentinel bytes follow the output region, no shown operation writes those suffix bytes.
- **Minimum length:** Inputs below 128 still produce two bytes; a one-byte result would not match the supplied behavior.
- **Return interpretation:** For input 1, the result is not the integer 2, even though exactly two bytes are emitted.
- **Snapshot distinction:** For input 16,384 under the larger-branch assumptions, the final output byte is 00, while the returned high half retains 80 in its least-significant byte.
- **Invalid storage:** Null, inaccessible, or undersized destinations have no specified successful result or recovery behavior.

These examples are static acceptance expectations, not observed executions.

## Uncertainties

### documented_only — Scratch byte order

The supplied text does not independently establish the target’s byte order.

On a big-endian four-byte word, clearing numeric bit 7 affects scratch-byte offset three rather than offset zero. The conventional continuation-bit interpretation of the larger-value branch would therefore not follow. Its captured numeric return value would also differ.

The two-byte branch remains byte-order independent.

### documented_only — Fifth scratch byte

Five-byte inputs require scratch-byte offset four. The supplied representation places this access beyond the first four-byte scratch object and suggests neighboring internal storage, but it does not establish a portable five-byte backing object.

The intended five-byte behavior is documented conditionally rather than treating the access as universally valid.

### documented_only — Address model

Scratch-byte addressing includes conversion through a signed integer type. The supplied text does not establish the full pointer width, valid address range, or conversion semantics needed to make that operation portable.

### documented_only — Meaning of the high return half

The high return half has a mechanically established construction, but its consumer-facing purpose is unknown. It may reflect an interface convention or representation artifact; neither explanation is established.

No semantic name such as checksum, status, original value, or encoded length should be assigned to it without additional evidence.

### Inferred intent

The larger branch’s extraction, continuation markers, and reversal strongly suggest a seven-bit variable-length integer encoder. The two-byte minimum is established behavior, not evidence that the function implements a particular external format.

The reason for that minimum is unknown.

## Evidence

This contract is based only on the supplied static function representation:

- A 14-bit threshold selects the two-byte branch.
- That branch performs two explicit byte writes and constructs a masked 64-bit result.
- The other branch repeatedly extracts seven-bit groups from an unsigned 32-bit value.
- Each extracted group is written with bit 7 set before the scratch word is captured.
- A numeric bit clear precedes reverse-order scratch reads and ascending destination writes.
- The larger-branch return combines the captured scratch word with the extraction count, then masks the combined value.
- No function calls, explicit destination reads, capacity checks, allocation operations, or explicit global accesses appear.

Supplied annotations about omitted unreachable regions do not establish additional reachable behavior and are not treated as behavioral authority.

No runtime tests have been performed. The destination sequences, lengths, return values, and boundary cases above are static derivations, with target-dependent cases explicitly conditional.
