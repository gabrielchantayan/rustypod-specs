<!-- Trusted provenance: F-afd88fe51e6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afd88fe51e6c

## Purpose

F-afd88fe51e6c accepts a source address, a 32-bit byte count, and a mutable context. Its behavior depends on mode bits in that context:

- In the ordinary mode, it transforms input bytes into a 64-byte context-resident buffer, invoking F-fc23781a6360 whenever a block becomes full.
- In the alternate mode, it may first complete an existing partial transformed block. Any remaining input is then submitted, without the byte transformation, to a separate 4096-byte buffering path.
- A null source or zero count causes an immediate return without accessing the context.

The established contract is a byte-processing and buffering operation. The meaning of the transformation, the downstream destination, and whether this represents encoding, transport, or another service are not established.

A zero return is the routine’s normal completion indication. It does not independently establish successful downstream delivery: several dependency results are ignored, and their contracts are unavailable.

## Interface

### Arguments and result

| Item | Width and interpretation | Contract |
|---|---|---|
| Source | 32-bit address-valued argument | Starting address of input bytes. Zero has explicit early-return behavior. |
| Count | Unsigned 32-bit integer | Requested number of input bytes. |
| Context | 32-bit address-valued argument | Mutable storage containing the fields described below. |
| Result | Unsigned 32-bit integer | Zero on ordinary completion; otherwise an unchanged dependency result, the original count on the null-source path, or an externally stored fallback value. |

There is no explicit validation of a non-null context, source extent, destination extent, pointer alignment, or nested pointer validity.

### Context layout

Offsets are byte offsets from the context address. Only accessed storage is specified.

| Offset | Access width | Observed use |
|---|---:|---|
| 0–63 | Individual bytes | Transformed-byte staging buffer. |
| 68 | 32 bits | Its low six bits select the initial position in the 64-byte buffer. |
| 92 | 32 bits | Pointer to an auxiliary object used by the alternate path. |
| 100 | 32 bits | Mode and flag word. |

The initial staging position is the word at offset 68 masked with 63. This routine does not directly write that word; any accounting changes are delegated.

If the context word at offset 92 is nonzero, the routine may read a 32-bit pointer at offset 4 of that auxiliary object. The resulting address identifies a downstream object with these accessed fields:

| Offset | Access width | Observed use |
|---|---:|---|
| 76 | 32 bits | Compared with the address of the same object plus 84. |
| 80 | 32 bits | Explicitly cleared after each full downstream-buffer submission. |
| 84 | Address only | Address used in the comparison; contents are not read here. |
| 104 | 32 bits | Downstream buffered-byte count. |
| 108 onward | Byte range | Base of a 4096-byte downstream staging region. |

These are access layouts, not complete object definitions.

### Preconditions for bounded, memory-safe use

For the ordinary 64-byte staging path, source storage must remain readable for the requested bytes, and context storage must accommodate the accessed fields and staging writes.

The alternate path additionally requires valid nested objects whenever their pointers are followed. If downstream buffering is selected, the downstream object must provide the accessed fields and sufficient backing storage for its 4096-byte region.

For conventional bounded behavior, relevant additions must not wrap, and dependencies must leave fields used after their calls in states compatible with subsequent accesses. The routine does not enforce those conditions.

## Behavior

### Entry and mode selection

Entry processing is ordered as follows:

1. If the source address is zero, return the original count immediately.
2. Otherwise, if the count is zero, return zero immediately.
3. Read the context mode word at offset 100.
4. Read the context word at offset 68 and retain only its low six bits as the initial staging position.

Thus, a null source with a positive count does not produce a fixed error code: it returns that positive count. Neither early-return case accesses the context.

The alternate path is selected only when both conditions hold:

- The low eight bits of the mode word equal 2.
- Bit 8, mask 256, is set.

All other mode combinations select the ordinary path. Bit 9, mask 512, has a further effect only after the alternate path is selected. Other mode bits are not tested here.

The mode decision and initial staging position are based on these entry reads. They are not recomputed after dependency calls.

### Byte transformation

Every byte written by this routine into the 64-byte context staging buffer is transformed.

For an input byte with value \(x\), the stored byte is:

\[
T(x) = (-99x - 20) \bmod 256.
\]

The distinction between signed and unsigned interpretation of the input character does not change the resulting low eight bits.

For each transformed byte, the source byte is read before the corresponding destination byte is written. Processing advances in increasing source order and increasing staging-buffer position within each block.

Input copied into the downstream 4096-byte buffer does not receive this transformation in this routine.

### Ordinary path

Let the initial staging position be \(p\), between 0 and 63, and the requested count be \(n\).

The routine first evaluates the unsigned 32-bit sum \(p+n\).

If that sum exceeds 63:

- Transform enough leading input bytes to fill staging positions \(p\) through 63.
- Invoke F-fc23781a6360 with the context address as both arguments.
- While at least another complete 64-byte block remains, transform that block into positions 0 through 63 and invoke F-fc23781a6360 again.
- Transform any final remainder into positions starting at 0.

The full-block invocation occurs before the next complete-block availability check. Consequently, the last completed block is submitted even when no input remains afterward.

If the initial sum does not exceed 63, no full-block invocation occurs; all requested bytes are transformed into staging positions beginning at \(p\).

After processing, invoke F-31d98408104a with the context and the original requested count, then return zero. The routine does not inspect either dependency’s result.

For nonwrapping, valid inputs, an exact block fill is submitted immediately. A partial final block is not submitted by this path.

### Alternate path: optional handling of an existing partial block

Initially, no input is considered consumed.

If bit 9 is clear and the initial staging position is nonzero, the routine first handles that partial block.

If the unsigned sum of staging position and requested count is less than 64:

- Transform all requested bytes into the existing staging buffer.
- Invoke F-31d98408104a with the context and the full requested count.
- Mark all input consumed.

Otherwise:

- Transform enough bytes to fill the existing staging buffer.
- Invoke F-fc23781a6360 with the context as both arguments.
- Invoke F-31d98408104a with the context and the number of bytes consumed in this fill.
- Retain any remaining input for the downstream path.

The order is significant: submission of the completed block precedes the accounting call.

If bit 9 is set, or the initial staging position is zero, this initial partial-block handling is skipped. In particular, bit 9 can cause the routine to proceed downstream without filling a nonempty context staging buffer.

If all input has been consumed at this point, return zero. No downstream readiness or object checks occur.

### Alternate path: readiness gate

If input remains, invoke F-d8a13f219e81 with the context and the literal value 1.

- A zero result permits further processing.
- Any nonzero 32-bit result is returned unchanged.

An error here does not roll back prior staging writes, block submission, or accounting.

Only after this call succeeds does the routine read the context pointer at offset 92. The readiness dependency can therefore affect the nested object subsequently selected.

### Alternate path: downstream eligibility

Advance the source address by the number of bytes already consumed and compute the remaining count using 32-bit arithmetic.

The downstream path requires all of the following:

1. The context pointer at offset 92 is nonzero.
2. The advanced source address is nonzero.
3. The downstream object’s buffered count at offset 104, interpreted unsigned, is less than 4096.
4. The word at downstream offset 76 equals the downstream object address plus 84.

The downstream pointer is obtained through offset 4 of the auxiliary object only when the auxiliary pointer and advanced source address are both nonzero.

The buffered-count read precedes the pointer-comparison read. The latter is skipped if the buffered count is at least 4096.

If eligibility fails, the routine returns a 32-bit value read from external storage. Its numeric value and meaning are unknown. It is not established that this value is nonzero.

### Alternate path: downstream buffering

After eligibility succeeds, the routine repeatedly reads the downstream buffered count and compares the unsigned 32-bit sum of that count and the remaining input against 4095.

When the sum is at least 4096:

- Invoke the external byte-transfer dependency, designated F-unknown-01, to transfer enough input bytes to fill the downstream buffer.
- Invoke F-d15c1be26a89 with the downstream object, its buffer address at offset 108, and a length of 4096.
- Write zero to the 32-bit field at offset 80.
- Read the buffered-count field at offset 104 again.
- Write zero to that buffered-count field.
- Advance the source and reduce the remaining input by 4096 minus the value just read.

The post-submission count read is material. It is not necessarily equal to the count used before submission: F-d15c1be26a89 and the transfer dependency have unknown side effects.

When the loop condition no longer holds:

- Invoke F-unknown-01 with the current buffer end, current source address, and remaining count.
- Read the buffered-count field again and add the remaining count to it.
- Store the resulting 32-bit count.
- Return zero.

This final transfer invocation can have a zero length, including after an exact full-buffer submission. No additional F-31d98408104a call accounts for these downstream bytes in this routine.

### Arithmetic boundaries

Lengths, positions, pointer-sized values, and stored words are 32-bit quantities in the supplied representation. Unsigned additions wrap modulo \(2^{32}\); they are not arbitrary-precision calculations.

The remaining downstream length is held in signed 32-bit storage, while the loop’s total-size comparison is unsigned. The supplied representation does not establish safe behavior for every count above the signed 32-bit range or every address overflow.

Large counts can wrap the staging threshold sum or the complete-block lookahead sum. This can invalidate ordinary buffer-bound reasoning. Such inputs must not be treated as automatically validated or safely handled.

## Side effects

Established direct effects include:

- Reading input bytes.
- Writing transformed bytes into the context staging buffer.
- Reading context mode, position, and nested-object pointers.
- Invoking dependencies in the orders described above.
- Reading and clearing downstream state fields.
- Updating the downstream buffered count.
- Reading an external fallback-result value on downstream ineligibility.

The routine has no direct allocation, deallocation, or ownership-transfer operation visible in the supplied body. That does not establish that its dependencies neither allocate nor release memory.

No rollback mechanism is visible. A nonzero return can follow partial consumption and observable state changes.

No overlap guarantee is established. Incremental staging writes may affect later source reads if memory overlaps. The external transfer dependency’s overlap semantics are unknown.

No locking or reentrancy protection is visible. Concurrent access and dependency-driven mutation require an external contract.

## Dependencies

| Opaque ID | Observed invocation and treatment |
|---|---|
| F-fc23781a6360 | Receives the context twice after a transformed 64-byte block is filled. Result ignored. |
| F-31d98408104a | Receives the context and a byte count after ordinary-path processing or alternate-path partial-block handling. Result ignored. |
| F-d8a13f219e81 | Receives the context and 1 before downstream processing. Zero continues; nonzero returns unchanged. |
| F-d15c1be26a89 | Receives the downstream object, its buffer address, and 4096 after a fill transfer. Result ignored. |
| F-unknown-01 | External byte-transfer dependency receiving destination, source, and length. Exact implementation, overlap behavior, failure behavior, and side effects are documented_only. |

The external fallback-result storage is a data dependency, not a function. Its initializer and mutability are unavailable.

None of these dependencies is established to preserve globals, context fields, downstream fields, source memory, or object lifetime.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume valid disjoint storage, nonwrapping arithmetic, and dependencies that return normally without unexpected mutation.

1. **Null source:** With source zero and count 17, return 17 without reading the context.

2. **Zero count:** With a nonzero source and count zero, return zero without reading the context or source bytes.

3. **Single transformed byte:** In ordinary mode with staging position zero and input byte 1, write byte 137 at context offset zero, invoke F-31d98408104a with count 1, and return zero. Do not invoke F-fc23781a6360.

4. **Exact ordinary fill:** With ordinary mode, staging position 60, and count 4, write four transformed bytes at positions 60–63, invoke F-fc23781a6360 once, then invoke F-31d98408104a with count 4.

5. **Multiple ordinary blocks:** With ordinary mode, staging position 10, and count 120, submit two completed blocks: one containing 54 new bytes and one containing 64 new bytes. Leave two transformed bytes at positions 0–1. The accounting dependency receives 120 once.

6. **Alternate partial-only completion:** With alternate mode, bit 9 clear, staging position 10, and count 5, write five transformed bytes and account for 5. Return zero without calling the readiness dependency.

7. **Alternate exact partial fill:** With alternate mode, bit 9 clear, staging position 60, and count 4, complete and submit the block, then account for 4. Return zero without downstream processing.

8. **Failure after partial progress:** With alternate mode, bit 9 clear, staging position 60, and count 7, process and account for four bytes first. If F-d8a13f219e81 then returns 9, return 9 without undoing those effects.

9. **Bit 9 bypass:** With alternate mode, bit 9 set, and a nonzero staging position, leave the existing partial staging block untouched by this routine before invoking the readiness dependency.

10. **Downstream exact fill:** With downstream count 4090 and six remaining bytes, transfer six bytes and invoke F-d15c1be26a89 for 4096 bytes. If the dependencies leave the count at 4090 until the explicit reset, subsequently perform a zero-length transfer and finish with count zero.

11. **Post-call count mutation:** In the preceding example, if the submission dependency instead changes the count to zero, subsequent source and remaining-length adjustment uses zero, not 4090. The unchanged-count outcome must not be assumed.

## Uncertainties

The following are documented_only:

- Semantic meaning of the mode flags, transformation, accounting, and submission operations.
- Dependency side effects, failure signaling, ownership behavior, and global-state preservation.
- Numeric value and interpretation of the fallback result.
- Full object layouts, allocation extents, alignment requirements, and lifetime rules.
- External transfer semantics, especially zero-length and overlapping transfers.
- Intended behavior for arithmetic overflow, high-bit-set counts, invalid pointers, concurrency, or reentrant mutation.

Buffering and submission are inferred intent from access patterns and argument shapes. They are not evidence of successful transport, persistence, or delivery.

## Evidence

This contract derives solely from the supplied static function representation and dependency mapping. Established evidence consists of branch predicates, access widths and offsets, byte arithmetic, argument values, return propagation, and observable operation order.

No dependency bodies, runtime traces, external data initializers, or complete type definitions were supplied. No runtime tests have been performed. The acceptance examples express consequences under stated conditions; they are not execution results.
