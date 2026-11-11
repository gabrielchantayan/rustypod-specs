<!-- Trusted provenance: F-2ca4d5ef4d4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ca4d5ef4d4b

## Purpose

F-2ca4d5ef4d4b produces a bounded sequence of two-channel frames. Each frame derives one interpolated value from byte-addressed source storage, selects shared or separate channel scaling, adds the resulting contributions to two existing signed 16-bit values, and writes two 16-bit outputs.

The routine also updates source-progress accounting, conditionally sets a completion marker, replaces the requested frame count with the number produced, and stores only the fractional part of its final local source position.

Audio interpolation and mixing are inferred intent. The established behavior is the arithmetic, memory access ordering, and state transition described below.

## Interface

The routine accepts five arguments and returns no value.

| Argument | Role | Representation |
|---|---|---|
| State address | Configuration and persistent progress | Address of the record described below |
| Destination address | Output channel pairs | Two consecutive 16-bit elements per frame |
| Existing-values address | Values to which contributions are added | Two consecutive signed 16-bit elements per frame |
| Source address | Interpolation input | Byte-addressed storage |
| Count address | Requested count on entry; produced count on exit | Address of an unsigned 32-bit value |

State and source addresses use 32-bit integer-valued representations in the supplied static body. A portable pointer interface, alignment requirements, native byte order, and concrete calling convention are not independently established.

### State layout

Offsets are byte offsets from the state address.

| Offset | Width | Use |
|---|---:|---|
| `0x44` | 32 bits | Position increment, loaded as signed |
| `0x4C` | 32 bits | Initial unsigned position; receives final fractional residue |
| `0x64` | 32 bits | Mode, compared with `1` |
| `0x70` | 32 bits | First-channel unsigned gain |
| `0x74` | 32 bits | Second-channel unsigned gain |
| `0x84` | 32 bits | Signed boundary |
| `0x88` | 32 bits | Signed progress storage |
| `0x8C` | 8 bits | Completion marker, conditionally written as `1` |

The local position has a 16-bit integer part and a 16-bit fractional part. Position advancement uses unsigned 32-bit arithmetic: the signed increment is converted to unsigned before addition.

A shared 32-bit global supplies output-substitution values and the equal-gain branch threshold. Its value, ownership, and immutability are unknown.

### Storage and ownership requirements

For a defined, non-faulting execution:

- All accessed state fields and the count must be readable; written locations must be writable.
- Destination and existing-values regions must cover every processed frame.
- For each processed frame, the source byte at the current integer index and its immediate successor must be readable. A zero fractional position does not eliminate the successor access.
- The target must support the address calculations, conversions, shifts, and storage accesses involved.
- Concurrent mutation must not be assumed safe.

The routine performs no demonstrated allocation, ownership transfer, or deallocation. All argument storage remains caller-provided. There is no explicit null, capacity, alignment, or source-range validation.

## Behavior

### Initial reads and gain-path selection

Before checking whether any frames are requested, the routine observes, in order:

1. The shared global, retained for positive out-of-range output substitution.
2. The first-channel gain.
3. The second-channel gain.
4. The initial position.
5. The increment.
6. The shared global again, retaining its bitwise complement for negative out-of-range substitution.

The two global observations are distinct. Their equality is not guaranteed by the supplied evidence.

The gains, position, increment, and substitution values are retained locally. Writes to their original storage later in the invocation do not refresh these retained values.

Three processing paths exist:

| Condition | Contribution calculation |
|---|---|
| Equal gains, common gain below the global threshold | Apply the common gain once and reuse the contribution |
| Equal gains, common gain at or above the global threshold | Use the interpolated value directly |
| Unequal gains | Apply each channel’s gain separately |

The threshold comparison is unsigned 32-bit and performs another global read. That read occurs only when the gains are equal. The direct path must not be described as a general “unity-or-greater gain” rule: that interpretation depends on the unknown threshold value.

### Count handling

The produced count begins at zero. At each normal loop-condition evaluation, the unsigned iteration index is compared with the value then stored at the count address. The requested count is not retained as a fixed entry snapshot.

With stable, nonoverlapping storage, the entry count bounds production. Aliasing or external modification of the count can change a subsequent loop bound.

A zero count skips all per-frame work, but not the initial reads or final count and position writes.

### Interpolation

Let:

- \(P\) be the current unsigned 32-bit local position.
- \(j=\lfloor P/2^{16}\rfloor\), the source index.
- \(f=P\bmod 2^{16}\), the fractional residue.
- \(w=\lfloor f/2\rfloor\), an integer from 0 through 32767.
- \(A\) and \(B\) be the interpreted current and successor source bytes, respectively, each multiplied by 256.

The interpolated value is:

\[
I=A+\operatorname{ASR}_{15}\bigl(w(B-A)\bigr).
\]

Here, \(\operatorname{ASR}_{15}\) denotes a signed arithmetic right shift by 15 bits. Under that target shift behavior, negative nonintegral results round toward negative infinity.

The lowest fractional-position bit has no effect on interpolation. Plain-character signedness is not established: source bytes `0x80` through `0xFF` may represent negative or nonnegative values, depending on the target. After either interpretation, the interpolation arithmetic remains within signed 32-bit range.

The source expressions reference the current byte more than once. The evidence does not establish a read-once snapshot or an exact order among reads inside an arithmetic expression. Stable source contents are therefore the safe precondition.

### Gain calculation

For a path applying gain \(g\), multiplication uses unsigned 32-bit arithmetic. The product is reduced modulo \(2^{32}\), converted to signed 32-bit, and shifted right by 15:

\[
G(I,g)=
\operatorname{ASR}_{15}
\left(
C_s\left((I g)\bmod 2^{32}\right)
\right),
\]

where \(C_s\) denotes the target’s conversion from unsigned 32-bit to signed 32-bit.

For values at most `0x7FFFFFFF`, that conversion preserves the value. For larger values, its result is target-dependent; a two’s-complement bit-pattern interpretation must be stated as an assumption where needed.

The channel contributions are:

| Path | First channel | Second channel |
|---|---|---|
| Equal gains below threshold | \(G(I,g)\) | The same retained contribution |
| Equal gains at or above threshold | \(I\) | \(I\) |
| Unequal gains | \(G(I,g_0)\) | \(G(I,g_1)\) |

No legal gain range is supplied. Multiplication does not use a wider retained product and does not saturate before the shift. Large gains can therefore produce wrapped products rather than proportionally larger contributions.

### Channel output

Each existing channel value is read as signed 16-bit and added to its contribution. Under the described signed-shift interpretation, these additions fit signed 32-bit range.

For each resulting sum:

- From −32768 through 32767 inclusive: retain the sum.
- At least 32768: select the initially retained global value.
- Below −32768: select the retained complement of the second global observation.

The selected value is narrowed to 16-bit destination storage. The usual target interpretation retains its low 16 bits; exact out-of-range signed narrowing semantics are target-dependent.

This is conventional signed 16-bit saturation only under suitable global values and narrowing behavior. In particular, stable global observations of `0x00007FFF` give the usual endpoint bit patterns `0x7FFF` and `0x8000`.

The observable channel order is:

1. Read the first existing value.
2. Write the first destination value.
3. Read the second existing value.
4. Write the second destination value.

Thus, the first destination write can affect the second existing-value read when the regions overlap.

### Progress accounting

Progress is updated after both channel outputs have been written.

Let \(D\) be the increment converted to unsigned 32-bit, and define:

\[
N=(P+D)\bmod 2^{32},
\qquad
j_0=P\gg16,
\qquad
j_1=N\gg16.
\]

Both shifted indices are unsigned. Their subtraction is consequently unsigned:

\[
d=(j_1-j_0)\bmod 2^{32}.
\]

Let \(S\) be the signed 32-bit progress value read from state. Adding it to \(d\) first converts \(S\) to unsigned. The update result is:

\[
R=\left(d+(S\bmod 2^{32})\right)\bmod 2^{32}.
\]

These subtraction and addition operations have defined modulo-\(2^{32}\) arithmetic. They are **not signed-overflow cases**.

The unsigned result \(R\) is then assigned to a signed 32-bit value:

\[
S'=C_s(R).
\]

The routine writes \(S'\) to the progress field and retains it for the boundary comparison. If \(R\) exceeds the signed maximum, the unsigned-to-signed conversion is target-dependent. This conversion uncertainty is distinct from arithmetic overflow.

The update is based on the difference between integer indices, not on a wrap-corrected number of source crossings. For example, movement from integer index 65535 to index 0 gives unsigned difference `0xFFFF0001`. Under a two’s-complement conversion, that difference represents −65535 when the old progress is zero.

The frame is counted as produced after the progress write.

### Boundary decision and position advancement

After output and progress updates, the routine tests:

- Whether the signed boundary is at most \(S'+1\).
- Only if that comparison succeeds, whether mode is different from `1`.

The expression \(S'+1\) is signed 32-bit addition. Unlike the progress update, it has a genuine signed-overflow uncertainty when \(S'\) is the signed maximum. Mode `1` does not avoid evaluating this earlier expression.

If both conditions hold:

- The completion byte is written as `1`.
- Processing stops immediately.
- The local position remains \(P\), excluding the current frame’s prospective increment.

Otherwise, the local position becomes \(N\), and processing continues subject to the next count check.

The boundary check is post-output. A boundary already reached on entry does not prevent production of the first frame when the requested count is nonzero.

### Finalization

Every normal exit writes:

1. The produced count to the count address.
2. The low 16 bits of the current local position to the position field, with the upper 16 bits zero.

On count exhaustion, the local position includes the last processed frame’s increment. On a boundary stop, it excludes that increment even though progress accounting already included its prospective integer-index change.

The completion marker is not cleared on entry, on zero-count calls, or on ordinary count exhaustion.

## Side effects

The routine explicitly writes:

- Two 16-bit destination elements per produced frame.
- The 32-bit progress field once per produced frame.
- The completion byte only on a boundary stop.
- The 32-bit count on every normal return.
- The 32-bit position field on every normal return.

There are no explicit writes through source or existing-values addresses, or to gains, increment, mode, boundary, or the shared global. Aliasing can nevertheless make a write affect storage reachable through another argument.

Exact destination/existing-values coincidence supports ordinary in-place processing with disjoint channel elements and frames. Partial overlap is order-sensitive; the routine does not snapshot an entire input frame before writing.

State and count aliasing are also observable. If the count location coincides with the position field, the final position write follows and supersedes the final count write.

No error status is returned. Invalid storage, unsupported arithmetic assumptions, or concurrent interference has no demonstrated recovery contract.

## Dependencies

No external function calls appear. There are no additional function dependencies requiring opaque IDs.

Direct dependencies are:

- Caller-provided memory and state.
- The shared 32-bit global.
- Target character interpretation, integer conversion rules, signed right shifts, address calculations, alignment, and storage representation.

No unknown callee is assumed to preserve globals or own or release memory.

## Acceptance examples

These are conditional mathematical acceptance criteria, not executed tests. Unless otherwise stated, assume stable nonoverlapping storage, valid source accesses, the described arithmetic shifts, conventional 16-bit storage narrowing, and no boundary stop.

### Zero requested frames

With count zero and initial position `0x12345678`:

- No source or existing-channel access occurs.
- No destination output is written.
- Progress and completion remain unchanged.
- Final count is zero.
- Final position is `0x00005678`.

Initial state and global reads still occur.

### Direct contribution

Assume every relevant global read is `0x00007FFF`, both gains are `0x00008000`, position is zero, and the current source byte is `1`.

The direct path is selected. The contribution is 256. Existing values 10 and −10 produce outputs 266 and 246.

### Fractional low-bit loss

On the direct path, with current byte `0`, successor byte `2`, and zero existing values:

- Fractional positions `0x8000` and `0x8001` both produce 256 per channel.
- Fractional positions `0x0000` and `0x0001` both produce zero.

Both source bytes remain required in every case.

### Separate gains

With unequal gains zero and `0x00008000`, position zero, current source byte `1`, and existing values 5 and 7, the outputs are 5 and 263.

### Endpoint substitution

With both endpoint global observations equal to `0x00007FFF`:

- Sum 32768 produces `0x7FFF`.
- Sum −32769 produces `0x8000`.
- Sums 32767 and −32768 remain unchanged.

Different global observations need not produce signed extrema.

### Boundary stop preserves the pre-increment fraction

Given one requested frame, position `0x0000F000`, increment `0x00002000`, progress zero, boundary 2, and mode zero:

- One frame is produced.
- Progress becomes 1.
- Completion becomes 1.
- Final count is 1.
- Final position is `0x0000F000`.

With mode `1`, the boundary does not stop processing; final position after count exhaustion is `0x00001000`.

### Unsigned progress arithmetic across position wrap

Given one requested frame, position `0xFFFF0000`, increment `0x00010000`, progress 65535, and mode `1`, with source indices 65535 and 65536 readable:

- The prospective position is zero.
- The unsigned index difference is `0xFFFF0001`.
- Adding unsigned-converted progress wraps to zero.
- The assigned signed progress is therefore unambiguously zero.
- The signed boundary expression evaluates zero plus one without overflow.
- Final count is 1 and final position is zero.

This update does not require a signed-overflow rule.

If the same position and increment instead start with progress zero, the unsigned update result is `0xFFFF0001`. Its conversion to signed progress is target-dependent; −65535 follows only under the two’s-complement conversion assumption.

## Uncertainties

The following remain **documented_only**:

- The shared global’s value, mutability, and ownership.
- Plain-character signedness.
- Target behavior for out-of-range unsigned-to-signed conversions, including progress assignment and gain-product conversion.
- Out-of-range signed narrowing to destination elements.
- Signed right-shift semantics, byte order, alignment, and concrete calling convention.
- Exact physical load counts and ordering within arithmetic expressions.
- Signed overflow in the boundary expression \(S'+1\).
- Intended legal ranges for gains, increment, boundary, and progress.
- Permitted aliasing, allocation sizes, and source-padding guarantees.
- The surrounding protocol for completion and source-position reconstruction.
- Behavior after invalid access or concurrent mutation.

The progress subtraction and addition are not listed as signed-overflow uncertainties: both are unsigned modulo-\(2^{32}\) operations. Their subsequent conversion to signed progress remains a separate target-dependent issue.

Audio mixing, fixed-point gain, and end-of-source tracking are inferred intent, not additional guarantees.

## Evidence

The supplied static body establishes the record offsets, initial observations, three gain paths, interpolation arithmetic, channel access order, unsigned progress-update expression, signed boundary expression, conditional completion write, and count-before-position finalization.

It contains no external calls, allocation operations, explicit error returns, or completion-marker clearing.

No runtime tests have been performed. All examples are conditional consequences of the supplied static representation.
