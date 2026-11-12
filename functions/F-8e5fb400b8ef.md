<!-- Trusted provenance: F-8e5fb400b8ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8e5fb400b8ef

## Purpose

F-8e5fb400b8ef produces two-channel, 16-bit output frames by interpolating an interleaved byte source and adding the resulting contributions to an existing signed 16-bit signal. Depending on the captured gains, it either scales each contribution or adds it directly.

The function updates a progress counter after each completed frame. A threshold and mode can stop processing after that update. On normal exit, it replaces the caller’s frame count with the number completed and saves a fractional source position.

Stereo resampling and mixing are **inferred intent**. The established contract is the storage layout, arithmetic, access ordering, and state transitions below. In particular, the equal-gain direct-add path covers a broad unsigned interval; it is not established as merely a near-unity optimization.

All evidence is static. No runtime tests have been performed.

## Interface

The function accepts five arguments and returns a 64-bit packed value.

| Argument | Representation | Meaning |
|---|---|---|
| State address | Unsigned 32-bit address value | Base of mutable state storage. |
| Destination | Pointer to 16-bit elements | Receives two output elements per completed frame. |
| Existing signal | Pointer to signed 16-bit elements | Supplies the two values to which source contributions are added. |
| Byte-source address | Signed 32-bit address-valued argument | Base of interleaved two-byte source frames. |
| Frame count | Pointer to an unsigned 32-bit word | Supplies a repeatedly read loop limit and receives the completed count. |

The address-valued argument widths do not establish portable pointer arithmetic, address validity, or behavior when address calculations overflow.

### State layout

Offsets are byte offsets from the state address.

| Offset | Width | Interpretation |
|---|---:|---|
| `0x44` | 32 bits | Source-position increment, loaded as signed. |
| `0x4C` | 32 bits | Initial unsigned source position; receives the saved fractional position. |
| `0x64` | 32 bits | Mode; exactly `1` suppresses threshold stopping. |
| `0x70` | 32 bits | Unsigned first-channel gain. |
| `0x74` | 32 bits | Unsigned second-channel gain. |
| `0x84` | 32 bits | Signed progress threshold. |
| `0x88` | 32 bits | Progress counter, accessed through signed storage. |
| `0x8C` | 8 bits | Completion marker; explicitly assigned `1` on threshold exit. |

The full state-record size and meanings of other bytes are unknown.

### Storage, validity, and ownership

Each source frame contains a first-channel byte followed by a second-channel byte. Producing an output frame requires both the selected source frame and its successor, even when the interpolation weight is zero.

Existing-signal and destination frames each contain two consecutive 16-bit elements and occupy four bytes. Their working pointers advance by four bytes per completed frame.

All reached addresses must support the required accesses. No explicit null, alignment, capacity, source-boundary, or address-overflow checks are visible. Threshold stopping occurs after output and cannot protect the triggering frame from invalid accesses.

No allocation, release, or ownership transfer is visible. No explicit non-overlap requirement is shown. Overlap nevertheless changes observable behavior because reads and writes are ordered rather than performed from a complete input snapshot.

### Return value

The 64-bit result concatenates two 32-bit words:

- High word: the first-channel gain captured at entry.
- Low word when the captured gains differ: the captured second-channel gain.
- Low word when the captured gains are equal: the state-address argument.

This applies even when zero frames are produced. The packed value is not established as a status code or completed-frame count.

## Behavior

### Entry capture and path selection

The function reads and captures these state fields in order:

1. First-channel gain.
2. Second-channel gain.
3. Initial source position.
4. Source-position increment.

Subsequent overlapping writes do not change these captured values.

The gains select one of three behaviors:

| Captured gains | Contribution processing |
|---|---|
| Unequal | Each channel uses its own gain. |
| Equal and unsigned gain below `0x7FFF` | Both channels use the shared gain, with a distinct second-channel arithmetic expression. |
| Equal and unsigned gain at least `0x7FFF` | Both channels bypass gain multiplication. |

The equal-gain scaling predicate covers exactly `0x00000000` through `0x00007FFE`. Its component unsigned conditions cover values below `0x7F00` and the following interval through `0x7FFE`.

Thus equal gains `0x7F00` and `0x7FFE` select scaling, whereas `0x7FFF`, `0x80000000`, and `0xFFFFFFFF` select direct addition. Unequal gains always select separate scaling.

### Loop limit

An unsigned iteration count begins at zero. Before each frame, it is compared with the current unsigned word at the frame-count pointer. The requested count is not captured once at entry.

Consequently, a destination or state write overlapping that word can change a later loop limit. For ordinary disjoint storage with a stable count, processing continues until that many frames have completed unless the threshold stops it earlier.

With a stable zero count, the processing loop does not read source samples, existing-signal samples, progress, threshold, mode, or the external replacement value. Entry capture and the final count and position writes still occur.

### Source selection and interpolation

Treat the captured position as an unsigned 32-bit word:

- Its upper 16 bits select a source-frame index.
- Its lower 16 bits represent a fractional position.
- The selected frame begins at the source base plus twice the frame index.
- The successor begins two bytes after the selected frame.

For each channel, let \(a\) and \(b\) be the selected and successor byte values after character promotion and multiplication by 256. Let \(w\) be the fractional position divided by two, discarding the remainder. The interpolated contribution is

\[
I=a+\operatorname{ASR}_{15}\bigl(w(b-a)\bigr),
\]

where \(\operatorname{ASR}\) denotes the represented signed arithmetic-right-shift operation.

The weight ranges from 0 through 32767. Fractions differing only in their least significant bit produce the same weight. Even the maximum fraction does not give the successor full weight.

The source is accessed as plain character data. Whether bytes `0x80` through `0xFF` promote to negative values or to 128 through 255 is **documented_only**. Under either usual 8-bit character interpretation, the interpolation product fits signed 32-bit range. Negative right-shift semantics remain an execution-model dependency; they must not silently be replaced by division truncating toward zero.

### Gain arithmetic

A gain is an unsigned 32-bit word. Multiplication by the interpolated contribution therefore uses unsigned 32-bit arithmetic, including unsigned conversion of negative contributions. The product retains only its low 32 bits.

**Unequal gains:** each channel’s product is converted to signed 32-bit, shifted right by 15, and added to that channel’s existing signed 16-bit value.

**Equal gains below `0x7FFF`:**

- The first channel follows the same product-conversion and right-shift-by-15 rule.
- The second channel first converts its unsigned product to signed 32-bit, then doubles that signed value, then shifts the doubled signed result right by 16 before adding the existing value.

Conversion precedes second-channel doubling. That doubling is not established as unsigned or modulo-\(2^{32}\) arithmetic. If the signed doubled value is not representable, its result is unresolved. When doubling is representable, the shifted contribution agrees with shifting the undoubled signed value right by 15.

**Equal gains at least `0x7FFF`:** the interpolated contribution is added directly to the existing signed 16-bit value. The captured gain is not multiplied into either channel.

Out-of-range unsigned-to-signed conversions and negative signed shifts require the recovered target interpretation. A portable-language interpretation has not been independently established.

### Output limiting

Each combined channel result is limited independently:

| Combined result | Destination representation |
|---|---|
| Less than `-32768` | `0x8000`, representing `-32768`. |
| From `-32768` through `32767` | The combined result as a 16-bit value. |
| At least `32768` | Low 16 bits of an external 32-bit replacement value. |

The external value is read during each channel’s limiting operation, even when positive replacement is unnecessary. There are two represented external reads per completed frame.

Positive limiting therefore behaves like saturation to 32767 only when the relevant external read has low half `0x7FFF`. Neither that value nor equality between the two reads is established.

### Observable ordering within a frame

The established access order includes the following:

- The selected first-channel source byte is read before the selected and successor second-channel bytes.
- Both second-channel source bytes and the successor first-channel byte are read before the first destination store.
- The first existing-signal value and first external replacement value are read before that store.
- The first destination store precedes the second existing-signal read.
- The second external replacement read precedes the second destination store.
- Both destination stores precede the progress-counter read and write.
- The progress-counter write precedes the threshold read.
- The mode is read only when the threshold comparison succeeds.

Some ordering within the first-channel expression, including between its existing-signal read and successor-source read, is not fully established.

All four required source bytes are captured before either destination store, but the two existing-signal values are not captured together. A first destination store overlapping the second existing-signal element can change the value used for the second channel of the same frame.

### Position, progress, and stopping

Let \(P\) be the current unsigned position and \(Q\) the position after adding the captured increment. The addition is unsigned 32-bit arithmetic and wraps modulo \(2^{32}\).

Let \(H\) extract a position’s upper 16 bits, and let \(C\) be the current progress-counter word. The progress update has unsigned arithmetic value

\[
U=\bigl(H(Q)-H(P)+C\bigr)\bmod 2^{32}.
\]

The counter is converted to unsigned for this expression. Both upper-half values are unsigned, so neither the subtraction nor the addition is an overflowing signed counter operation.

The result is then converted to signed 32-bit, retained as an updated signed temporary, and written to the progress field. Under a two’s-complement interpretation, high-bit-set results become negative. Exact out-of-range conversion semantics remain target-dependent.

After completing the output and progress update, the function counts the frame. It then compares the signed threshold with the updated signed temporary plus one. This later addition is signed; overflow at that addition remains unresolved.

If the threshold is at most that temporary plus one, the mode is read. A mode other than exactly `1` stops processing. The function explicitly writes marker byte `1` after the triggering frame has already been produced and counted.

If stopping does not occur, the local position advances to \(Q\), and processing returns to the loop-limit check.

### Exit state and marker qualification

Every normal exit writes:

1. The completed unsigned count through the frame-count pointer.
2. The current position’s low 16 bits, zero-extended into the full 32-bit saved-position field.

On count exhaustion, every completed frame has advanced the local position. On threshold exit, progress includes the triggering frame’s increment, but the local position has not yet received that increment. Threshold exit therefore saves the fraction used for the triggering frame.

There is **no explicit marker-clearing assignment**. This does not guarantee marker preservation: destination, progress, count, or saved-position writes can clear or otherwise change its byte if their storage overlaps it.

Likewise, a threshold exit guarantees an explicit marker write of `1` at that point, not necessarily a final marker value of `1` under arbitrary aliasing. The later count or saved-position write can overwrite it. Marker preservation outside the explicit threshold write requires all other writes to avoid its byte.

## Side effects

The directly represented writes are:

- Two 16-bit destination stores per completed frame.
- One 32-bit progress-counter store per completed frame.
- One marker-byte store of `1` on threshold exit.
- One 32-bit completed-count store on every normal exit.
- One 32-bit saved-position store on every normal exit.

These describe access purposes, not necessarily distinct storage locations. Aliasing can make a destination store modify state, a count store modify the marker, or a final position store overwrite an earlier count store.

Entry-captured gains, increment, and position remain locally captured despite overlapping writes. Progress, threshold, mode, existing-signal values, and loop limits are read at the points described above and may therefore reflect earlier writes.

No allocation, deallocation, external function call, synchronization, rollback, or explicit error return is visible. Invalid accesses and unresolved arithmetic have no shown recovery path. Concurrent mutation and memory-mapped access semantics are not established.

## Dependencies

- **F-8e5fb400b8ef:** the subject function. No calls to other functions are visible.
- **External replacement data:** a 32-bit value read independently for each channel’s limiting operation. Its initialization, ownership, mutability, and expected value are unknown.
- **Execution model:** character signedness, address arithmetic, signed right shifts, out-of-range integer conversions, signed-overflow behavior, and 16-bit storage representation.

There are no visible callees from which global preservation or memory ownership could be inferred.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless an example explicitly introduces overlap, assume valid disjoint storage, a stable requested count, and the represented arithmetic-shift behavior. Marker-preservation statements also require other writes not to overlap its byte.

1. **Zero frames.** With requested count zero and initial position `0x12345678`, no frame or progress update occurs. The final count is zero and saved position is `0x00005678`. The marker is unchanged under the non-overlap assumption.

2. **Gain boundaries.** Equal gains `0x7EFF`, `0x7F00`, and `0x7FFE` select scaling. Equal gains `0x7FFF`, `0x80000000`, and `0xFFFFFFFF` select direct addition. Unequal gains select separate scaling regardless of magnitude.

3. **Observable boundary difference.** With zero fraction, current source byte one, and existing value zero, equal gain `0x7FFE` produces 255 in each channel. Equal gain `0x7FFF` produces 256.

4. **Interpolation quantization.** Current byte zero, successor byte two, and fraction `0x8000` give expanded endpoint values zero and 512, weight 16384, and contribution 256. Fraction `0x8001` gives the same contribution.

5. **Direct addition still reads the successor.** With equal gains `0x7FFF`, zero fraction, current source byte one, and existing value 100, the channel result is 356. Successor source storage must still be readable.

6. **Negative limiting.** Assuming signed source characters, equal bypass gains, zero fraction, current byte `0x80`, and existing value `-1`, the combined result is `-32769`. The stored representation is `0x8000`.

7. **External positive replacement.** A combined result of 32768 selects the external value’s low half. Low half `0x7FFF` stores 32767; low half `0x1234` stores that representation instead.

8. **Threshold exit retains the triggering fraction.** With position zero, increment `0x00018000`, progress zero, threshold two, mode zero, and requested count at least two, the first frame updates progress to one and stops. Completed count is one, marker is one under disjoint storage, and saved fraction is zero rather than `0x8000`.

9. **Mode exactly one suppresses stopping.** Under the preceding conditions, but with mode one and requested count one, count exhaustion saves fraction `0x8000`. Progress is one. The marker remains unchanged under disjoint storage.

10. **Position-wrap progress arithmetic.** With position `0xFFFF0000`, increment `0x00010000`, and progress 65535, the unsigned upper-half difference is `0xFFFF0001`. Adding progress modulo \(2^{32}\) gives zero. With threshold two and requested count one, count exhaustion saves fraction zero. Source storage at the high selected index and its successor must be valid.

11. **Unresolved signed doubling.** Assuming unsigned source characters, byte 255 at zero fraction gives contribution 65280. Equal gain `0x7FFE` gives unsigned product 2138964480, which is representable as signed before doubling. Its signed double is not representable. No unconditional second-channel output is specified.

12. **Aliased destination clears the marker without threshold exit.** Let the destination begin at state offset `0x8C`, with its four writable bytes valid. Request one frame; use equal gains `0x7FFF`, zero source and existing-signal values, zero position, increment, and progress, and threshold 100. Keep other storage disjoint. The first 16-bit destination store writes zero over the marker byte, clearing an initially nonzero marker. The threshold does not trigger, and no explicit marker assignment occurs.

13. **Return packing.** Captured gains three and four return high word three and low word four. Captured gains three and three return high word three and low word equal to the state-address argument.

## Uncertainties

The following remain **documented_only**:

- Plain-character signedness for source bytes.
- External replacement data’s value, initialization, stability, and concurrency properties.
- Exact negative-shift and out-of-range conversion rules.
- Overflowing signed doubling on the equal-gain second-channel scaling path.
- Overflow of the signed updated-progress-plus-one expression.
- Address-overflow behavior, alignment requirements, and formally supported overlap.
- Higher-level meanings of progress, threshold, mode, and return packing.
- Whether another component explicitly clears the marker.
- Volatility, synchronization, and memory-mapped access guarantees.

The unsigned progress subtraction and addition are established, not unresolved signed overflow. Equal-gain branch boundaries are established. The absence of an explicit marker clear is also established, but marker preservation without a non-overlap condition is not.

## Evidence

The supplied static representation establishes the argument widths, accessed state offsets, entry-capture order, gain-path predicates, interpolation arithmetic, unsigned gain products, conversion-before-doubling order, channel access ordering, progress update, post-frame stopping, final writes, and return packing.

The marker behavior follows from distinguishing the dedicated threshold assignment from the other memory writes. A destination beginning at offset `0x8C` directly overlaps the marker byte; storing a zero first-channel output therefore clears it without executing any marker-specific clear.

Resampling and mixing are inferred intent. Target-specific arithmetic, unsupported memory configurations, and external-data properties remain unresolved as stated above. No execution, runtime tests, dependency inspection, or independent target-ABI verification has been performed.
