<!-- Trusted provenance: F-e7c5183babcf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7c5183babcf

## Purpose

F-e7c5183babcf transfers data from a source address to a destination address. It selects one of two access patterns according to the starting addresses’ alignment:

- When both addresses are four-byte aligned, it copies complete words until an externally parameterized predicate selects a word for bytewise emission.
- Otherwise, it processes up to two output bytes per iteration, stopping after writing a zero byte.

**Inferred intent:** an optimized copy of a zero-terminated byte string, including the terminator.

**Established limitation:** ordinary string-copy semantics are conditional. The aligned path depends on an external detection value whose contents are not supplied. Both paths also depend on the relationship between a word’s numeric value and its bytes in memory. The unaligned path begins each iteration with a word-width source read.

This contract describes the represented accesses and state transitions without assuming the inferred string-copy intent is universally achieved.

## Interface

### Inputs and output

The function receives two address-valued arguments, in this order:

1. **Destination:** the starting address for writes.
2. **Source:** the starting address for reads.

The represented interface types both addresses as pointers to unsigned words, but portions of the operation access memory and advance positions one byte at a time.

There is no length, destination capacity, scan limit, or error-output argument. The interface specifies no return value. Observable results consist of memory writes, memory-read effects, and whether execution returns or encounters an environmental failure.

### Width and arithmetic model

The representation supports a model with:

- Eight-bit bytes.
- Four-byte, 32-bit unsigned words.
- Word-pointer advancement by four bytes.
- Alignment determined by the lowest two address bits.
- Logical right shifts of the captured unsigned word.
- Byte extraction from the low eight bits of that word.

The descriptions below use this model. Unsigned word subtraction is interpreted modulo \(2^{32}\). Address arithmetic is meaningful only while the resulting addresses and accesses remain valid.

One expression shifts a signed-typed external value left by seven positions. Its exact interpretation is unresolved for values whose shift is not defined by the applicable source-language rules. Where a mathematical detection predicate is given below, it explicitly uses a 32-bit bit-pattern interpretation; it does not establish that all possible signed source-language evaluations are defined.

### Preconditions

For ordinary completion, the caller must provide readable source memory and writable destination memory covering every access actually performed.

In particular:

- No null-address check is present.
- Destination capacity is not checked.
- A readable terminator byte alone is insufficient when the function reads a complete word containing it.
- The unaligned path requires the represented word reads to be supported at the addresses reached.
- The aligned path must eventually encounter a word satisfying its detection predicate.
- Neither source stability nor source–destination noninterference is enforced.

For conventional copying results, source bytes must remain unchanged until their relevant reads. Nonoverlapping source and destination regions, together with no external source mutation, are sufficient to exclude the interference discussed here.

No allocation, ownership transfer, or lifetime extension is established.

## Behavior

### Entry read and path selection

The function first reads an externally stored detection value. Let **K** denote its captured 32-bit bit pattern.

This read precedes alignment selection. It occurs even if the subsequently selected path does not use K. Later writes to the external storage do not replace the already captured value for this invocation.

The aligned path is selected only if both starting addresses have their lowest two bits clear. Otherwise, the unaligned path is selected immediately. There is no preliminary byte-copy phase to improve alignment, and the function does not switch paths later.

### Aligned path: word transfer

At each current source position, the function reads one complete unsigned word, **W**, before writing any part of that word to the destination.

Under the stated bit-pattern interpretation, the detection predicate is true exactly when the following quantity is nonzero:

\[
\bigl((W-K)\bmod 2^{32}\bigr)
\;\mathbin{\&}\;
\operatorname{NOT}_{32}(W)
\;\mathbin{\&}\;
\bigl((K\ll 7)\bmod 2^{32}\bigr).
\]

If the predicate is false:

- W is written to the current destination by a word-width store.
- Both positions advance four bytes.
- The next complete source word is read and tested.

If the predicate is true, that word is not written by a word-width store. Instead, the function enters the captured-word byte phase.

The predicate is not an unconditional zero-byte test. **If K is `0x01010101`**, its bit-pattern interpretation detects whether W contains at least one zero byte. The supplied material does not establish that K actually has that value.

No source word after the selected stopping word is read by the aligned path.

### Aligned path: captured-word byte emission

The final phase uses only the already captured word. It performs no further source reads.

Bytes are emitted from the least significant end of the captured numeric value:

- The current low byte is written to the destination.
- The destination position advances one byte.
- If that byte was zero, the function returns.
- Otherwise, the retained unsigned value is shifted logically right by eight bits, and emission continues.

The represented implementation determines whether the low byte is zero before its store. In either case, the terminating zero is written before ordinary return.

This phase makes between one and five byte writes:

- A zero in the least significant byte produces one write.
- A zero in a later byte produces writes through that byte.
- If all four captured bytes are nonzero, four nonzero bytes are emitted, followed by a fifth byte containing zero.

The fifth-byte case follows from exhausting the 32-bit captured value by repeated shifts. That zero need not correspond to any separately read source byte.

Byte order matters:

- On a little-endian target, least-significant-first emission follows increasing source addresses within the captured word.
- On a big-endian target, it reverses that within-word source order.

Earlier complete-word transfers do not use this extraction sequence. Under ordinary same-machine load/store semantics, they preserve the captured word’s byte representation.

### Unaligned path

Each iteration starts with a word-width read at the current source position. The captured word is converted to a character-sized value, which is written to the current destination position.

Under the eight-bit, low-byte conversion model, this first output is the word’s low eight bits. On a little-endian target it corresponds to the byte at the current source address; that correspondence does not generally hold on a big-endian target.

If the first written byte is zero, the function returns without a separate source-byte read at offset one.

Otherwise:

- It reads one byte at source offset one.
- It writes that byte at destination offset one.
- If the second byte is zero, it returns.
- If both bytes are nonzero, both positions advance two bytes and another iteration begins.

Thus a continuing iteration has this observable access order:

1. Word-width source read.
2. Byte-width destination write.
3. Byte-width source read at offset one.
4. Byte-width destination write at offset one.

A first-byte termination performs only the first two accesses. A second-byte termination performs all four.

Consecutive continuing iterations start two bytes apart, so their four-byte source reads overlap by two bytes. A word read can include bytes beyond a terminator even when only the terminator itself is ultimately written.

### Termination and errors

The bytewise portions return after writing a zero byte. The aligned word loop has no independent length bound or fallback stopping rule.

No explicit failure status, exception translation, recovery, or retry is represented. Invalid memory, unsupported alignment, or leaving an accessible region may cause an environmental fault. If no aligned stopping predicate is reached, word scanning and copying continue for as long as execution permits.

The representation does not establish how such failures are delivered to callers.

## Side effects

### Reads and writes

The represented reads are:

- One entry read of the external detection value.
- One word read for each aligned-path word examined.
- One word read at the start of each unaligned-path iteration.
- One additional byte read in an unaligned iteration when its first extracted byte is nonzero.

Writes are restricted to addresses derived from the destination argument:

- Word stores for aligned words whose predicates are false.
- Byte stores for the aligned stopping word’s emission.
- Byte stores throughout the unaligned path.

There is no explicit write to the external detection storage. However, a destination alias of that storage can modify the same underlying memory. Such modification does not change the already captured K.

The caller’s pointer objects are not updated. Position advancement changes only the function’s internal address state.

### Overlap and source mutation

No whole-source snapshot is taken.

On the aligned path, a destination word store can change a later source word before that later word is read. On the unaligned path, the first destination byte write can change the source byte that will be read separately at offset one during the same iteration.

Once an aligned stopping word has been captured, its emitted bytes are independent of subsequent source-memory contents. Destination writes cannot alter that retained numeric value.

Consequently, overlapping regions do not have general move semantics. Even an initially present terminator can be overwritten before the function reaches the read that would have observed it.

No synchronization, atomicity, volatile-access guarantee, or coherent snapshot under concurrent mutation is established.

### Ownership

No allocation, release, or ownership-management operation is visible. Caller-provided storage must remain valid for the required accesses. No callee is invoked whose memory or global-state behavior would need an additional assumption.

## Dependencies

No called-function dependencies are visible. There are therefore no additional opaque function IDs to list.

The non-function dependencies are:

- The entry-time external detection value.
- The execution environment’s word width, byte order, character conversion, alignment support, and memory-access behavior.
- Readability, writability, and lifetime of the accessed memory.
- Source stability and noninterference when conventional copying outcomes are required.

The detection value’s initialization, actual contents, and mutability outside this invocation are not supplied.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests.

**Shared noninterference condition:** except for the explicitly labeled overlap examples, assume source and destination accessed regions do not overlap, source contents remain stable throughout the invocation, all required reads and writes are valid, and the stated width and conversion model applies. These assumptions are essential wherever an expected later read relies on initial source contents.

### Aligned conventional detection

Given both addresses are four-byte aligned, a little-endian target, K equal to `0x01010101`, and initial source bytes:

`41 42 43 44 45 00 7A 7B`

the expected accesses and output are:

- The first source word is read and copied by a word store.
- The next complete source word is read.
- The captured-word phase writes `45 00`.
- No destination writes are performed for `7A` or `7B`.

The second source word read nevertheless includes those post-terminator bytes. The shared noninterference condition ensures the first destination store does not alter this later source word.

### Aligned immediate termination

With the same alignment, byte order, and K, source bytes beginning `00 AA BB CC` cause one complete source-word read followed by one destination-byte write containing `00`.

Providing only one readable source byte does not satisfy this access requirement.

### Unaligned first-byte termination

When either starting address is unaligned, on a little-endian target, source bytes beginning `00 AA BB CC` cause:

- A complete word read.
- One zero-byte destination write.
- Return without the separate offset-one source-byte read.

The external detection value is still read at entry, although it does not determine this path’s transfer.

### Unaligned second-byte termination

On the unaligned path with little-endian interpretation, source bytes beginning `51 00 AA BB` cause:

- A word read.
- A destination-byte write of `51`.
- A separate read of source offset one.
- A destination-byte write of `00`, followed by return.

The shared noninterference condition is necessary: the first write must not replace the offset-one source terminator before its separate read.

### Overlap changes the aligned result

Assume the conventional aligned example’s initial source bytes, K, and byte order, but place the destination four bytes after the source. This example intentionally does not satisfy the shared nonoverlap condition.

The first destination word store replaces source bytes at offsets four through seven with `41 42 43 44` before the next source word is read. The next read therefore does not capture the initial `45 00 7A 7B`.

The earlier prediction of final emission `45 00` no longer follows. Subsequent termination and output depend on later accesses and the effects of continued overlapping writes.

### Overlap changes the second byte

Assume the unaligned second-byte example’s initial source bytes and little-endian interpretation, but place the destination one byte after the source. Assume the starting addresses select the unaligned path.

The initial word read captures the original word. The first destination write then replaces source offset one with `51`. The separate offset-one source read observes `51`, not the initial zero, and the second destination write is also `51`.

The function does not return after that second write on account of the original terminator. Further behavior depends on later memory contents and writes.

### Detection extremes

If K is zero, the bit-pattern predicate is always false. On the aligned path, encountering source zero bytes does not itself cause termination; complete words continue to be read and copied.

If K and W instead make the predicate true for W equal to `0x44434241`, the final phase writes `41 42 43 44 00`. This conditional example assumes that predicate outcome rather than establishing a particular K. The fifth byte is synthesized from the exhausted captured value, without a fifth source-byte read.

## Uncertainties

The following remain **documented_only**:

- The actual detection value and whether it varies across invocations.
- Exact target ABI widths and byte order beyond the supported model.
- The precise machine realization of the represented unaligned word reads.
- Signed-left-shift behavior for detection values outside a defined source-language range.
- Exact character-conversion semantics where a captured value is not representable by the character type.
- Platform support for unaligned reads and reads extending beyond a logical terminator.
- Any caller-imposed overlap restriction.
- Fault delivery, surrounding recovery, and concurrent-memory behavior.
- Whether callers observe a return-register artifact despite the represented void interface.

The string-copy interpretation remains inferred intent. It does not justify assuming a particular K, replacing word reads with byte reads, or promising overlap safety.

## Evidence

Established static evidence from the supplied representation includes:

- Two address arguments and no specified return value.
- An external-value read before alignment selection.
- A joint four-byte alignment test.
- A complete-word read and predicate test before each conditional aligned word store.
- A captured-word emission phase using low-byte extraction and unsigned right shifts.
- An alternate path with a word read, first byte write, conditional separate byte read, and second byte write.
- Return after writing a zero byte on either bytewise portion.

The overlap examples follow directly from the represented ordering of destination writes before later source reads. They require no assumption that source memory retains its initial contents after an aliasing write.

No runtime tests, target inspection, dependency inspection, or implementation validation have been performed. All acceptance examples are static deductions under their explicit conditions.
