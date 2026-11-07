<!-- Trusted provenance: F-c0ab5b5d32b8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c0ab5b5d32b8

## Purpose

F-c0ab5b5d32b8 converts a sequential stream of four-byte pixel records into three separately addressed byte planes. It produces one luminance-like byte per processed pixel and two chrominance-like bytes per horizontal pair on alternating rows.

The coefficients and offsets are consistent with conversion from blue–green–red pixel records to limited-range luminance and centered chrominance. That interpretation is **inferred intent**, not a separately established format guarantee. The established behavior is the byte selection, arithmetic, addressing, and write order described below.

The conversion has two important characteristics:

- Every horizontal iteration processes two pixels, including when the requested width is odd.
- Chroma is calculated exclusively from the second pixel of each pair on even-numbered rows. It is not an average of adjacent pixels or rows.

The function performs no validation and returns no status.

## Interface

The function has six positional inputs:

| Position | Meaning | Representation |
|---|---|---|
| 1 | Requested width, \(W\) | Signed 32-bit integer |
| 2 | Requested height, \(H\) | Signed 32-bit integer |
| 3 | Source position, \(S\) | Pointer to readable bytes |
| 4 | First-plane base, \(Y\) | Address carried in a 32-bit integer |
| 5 | Second-plane base, \(U\) | Address carried in a 32-bit integer |
| 6 | Third-plane base, \(V\) | Address carried in a 32-bit integer |

The return type is void. No source position, destination position, processed count, or error code is returned.

The supplied representation uses 32-bit integer address calculations. A caller must supply addresses meaningful in the target environment; passing wider host addresses through these integer arguments is not established as supported.

### Source layout

Each pixel occupies four consecutive bytes:

| Byte within a pixel | Established use | Inferred interpretation |
|---|---|---|
| 0 | Input channel \(a\) | Blue |
| 1 | Input channel \(b\) | Green |
| 2 | Input channel \(c\) | Red |
| 3 | Not read | Unused or alpha-like field |

The fourth byte does not affect the conversion. Its meaning is unknown.

There is no source stride argument. Successive rows begin immediately after all pairs consumed for the preceding row.

### Destination layout

The destination row strides are derived internally:

\[
S_Y=\operatorname{align4}(W)
\]

\[
S_C=\operatorname{align4}(\operatorname{trunc0}(W/2))
\]

Here, the implemented alignment operation adds 3 using 32-bit unsigned arithmetic and then clears the lowest two bits. The division by 2 is signed division with truncation toward zero.

For ordinary positive dimensions without address wraparound, these are the width rounded upward to a multiple of four, and half the width rounded upward to a multiple of four after integer division, respectively.

The second and third planes use the same stride. The function neither accepts nor discovers caller-selected strides.

## Behavior

### 1. Entry and empty dimensions

The function computes the two destination strides and prepares the first-plane row address before checking the height.

If \(H<1\), it returns without reading source bytes or writing destination bytes.

If \(H>0\) but \(W\leq0\), it traverses the row-control loop but performs no pixel reads or destination writes. Thus a nonpositive width produces no conversion output, regardless of positive height.

These are control-flow observations, not general assurances that arbitrary invalid address values are acceptable in every execution environment.

### 2. Row and pair traversal

Rows are processed in increasing order, beginning with row zero.

For positive width, define the number of processed pairs per row as:

\[
P=\left\lceil W/2\right\rceil
\]

Each row consumes \(8P\) source bytes and emits \(2P\) first-plane bytes.

For pair index \(j\), where \(0\leq j<P\), the pair begins at horizontal position \(x=2j\). Its first pixel occupies source offsets 0–3 relative to the pair start; its second occupies offsets 4–7.

For non-overlapping storage and ordinary, non-wrapping address arithmetic, the source pair begins at:

\[
S+8(rP+j)
\]

where \(r\) is the row number.

The first-plane output locations are:

\[
Y+rS_Y+2j
\quad\text{and}\quad
Y+rS_Y+2j+1
\]

There is no special treatment of a final unpaired pixel. An odd requested width therefore still requires a complete second pixel in the final pair and causes an additional first-plane byte to be written beyond the requested width.

### 3. First-plane conversion

For a pixel with source channels \(a,b,c\), the output byte is:

\[
L(a,b,c)=
\left(
\left\lfloor
\frac{25a+129b+66c+128}{256}
\right\rfloor+16
\right)\bmod256
\]

Each channel is read as an unsigned byte, so its input range is 0 through 255.

The numerator is nonnegative and small enough that the first-plane calculation does not overflow 32-bit unsigned arithmetic. Across all possible channel values, its output lies between 16 and 235 inclusive. No explicit clamping is performed.

The first-plane conversion is applied independently to both pixels of every processed pair.

### 4. Chroma conversion and row selection

Chroma output is written only on rows whose zero-based row number is even: rows 0, 2, 4, and so on.

Both chroma values come from the **second** pixel of the current pair. For that pixel’s channels \(a,b,c\), the resulting bytes are:

\[
C_U(a,b,c)=
\left(
\left\lfloor
\frac{112a-74b-38c+128}{256}
\right\rfloor+128
\right)\bmod256
\]

\[
C_V(a,b,c)=
\left(
\left\lfloor
\frac{-18a-94b+112c+128}{256}
\right\rfloor+128
\right)\bmod256
\]

For the stated byte-input domain, both results lie between 16 and 240 inclusive.

These formulas express the observable low-byte result. The underlying expressions use unsigned 32-bit arithmetic, including wraparound for negative coefficient contributions, followed by a logical right shift by eight. Taking the stored low byte produces the formulas above; the negative numerators must not instead be divided with truncation toward zero.

For even row \(r\), the destinations are:

\[
U+(r/2)S_C+j
\]

\[
V+(r/2)S_C+j
\]

Odd rows produce no chroma writes. An odd positive height still produces chroma for its final row, because that final row has an even zero-based index.

There is no horizontal or vertical filtering. The first pixel of the pair contributes no chroma, and an intervening odd row contributes no chroma.

### 5. Observable access order

Within each pair, the established access sequence is:

1. Read the first pixel’s three used channel bytes and write its first-plane output.
2. Read the second pixel’s byte 1, then byte 2, then byte 0.
3. Advance the internal source position by eight bytes.
4. Write the second pixel’s first-plane output.
5. On an even row, write the second-plane chroma byte, then the third-plane chroma byte.

The supplied expression does not establish a relative evaluation order among the first pixel’s three channel reads. All of them contribute to the first output before its store.

The second pixel’s channel values are retained for both its first-plane output and its chroma outputs. They are not reread after the second first-plane store.

These distinctions matter if source and destination ranges overlap. For example, the first first-plane store can alter a second-pixel channel before that channel is read. Conversely, a later store cannot change a second-pixel value already captured for the current pair.

Writes in one pair may also affect later source reads or other destination locations. The function contains no overlap detection, staging buffer, or protection against such interference.

### 6. Padding and row overlap

The function does not initialize padding bytes between the active portion of one row and the next.

The first-plane stride normally accommodates the extra output caused by an odd width. The chroma stride does not always accommodate the number of chroma samples actually written.

In particular, for positive widths of the form \(8k+1\):

\[
S_C=4k,\qquad P=4k+1
\]

A chroma row therefore writes one byte more than its nominal stride. Consecutive chroma rows overlap at their boundary.

Width 1 is the limiting case: \(S_C=0\), and every chroma-producing row writes to the same byte in each chroma plane. This behavior is established by the addressing, not corrected by the function.

### 7. Arithmetic and valid execution domain

A safe caller must provide readable and writable storage covering the actual accesses, not merely storage sized from the nominal width or chroma stride.

For a positive height and ordinary positive width, the source footprint is \(8PH\) bytes of sequential record storage. Within each record, only its first three bytes are read.

Without address wraparound, the highest written offsets are:

- First plane: \((H-1)S_Y+2P-1\).
- Each chroma plane: \((\lceil H/2\rceil-1)S_C+P-1\).

These describe addressed spans, not a promise that every byte inside each span is written.

The horizontal position advances by two using a signed 32-bit quantity. A width equal to the maximum signed 32-bit integer requires an increment beyond that signed range. Portable, well-defined behavior for that case is not established.

Large dimensions can also make row-address products, pointer advances, or address additions wrap or reach inaccessible memory. There are no overflow or capacity checks. The ordinary layout formulas above are conditional on representable, accessible address ranges and a terminating horizontal loop.

## Side effects

The function writes directly into caller-provided destination memory. It does not allocate, resize, initialize wholesale, or release any buffer.

The source pointer is advanced internally, but the caller’s pointer object is not updated through an output parameter.

No explicit global-state access or dependency call appears in the supplied body. Consequently, there is no callee-preservation or callee-ownership assumption needed for this function.

Memory permissions, invalid addresses, insufficient storage, and harmful aliasing are not checked. Such conditions can cause access faults or memory corruption; the function supplies no structured error result or rollback behavior. Writes completed before a failure are not undone.

The three destination regions may alias at the address level, but independent-plane output is only assured when their write ranges do not interfere. If destinations share a location within a pair, the later store in the specified order determines the final byte at that location, subject to any subsequent iterations.

## Dependencies

No called function is present in the supplied body. There are therefore no external opaque function IDs to enumerate.

F-c0ab5b5d32b8 depends on the execution environment for valid byte-addressed memory and the represented 32-bit arithmetic and address model. No external format descriptor, stride metadata, allocator, or validation service is consulted.

## Acceptance examples

The following are conditional acceptance examples derived from static analysis. They are not runtime test results. Unless stated otherwise, assume valid, disjoint buffers and non-wrapping addresses.

### Black and white pair

Given width 2, height 1, a first pixel whose three used bytes are all zero, and a second pixel whose three used bytes are all 255:

- The first-plane bytes at offsets 0 and 1 are 16 and 235.
- The second-plane byte at offset 0 is 128.
- The third-plane byte at offset 0 is 128.
- The unused fourth byte of either pixel has no effect.
- No padding byte is written.

Both destination strides are four bytes.

### Chroma selects the second pixel

Given width 2 and height 1, make the first pixel black and the second pixel have \(a=0,b=0,c=255\):

- First-plane output is 16, then 82.
- Second-plane output is 90.
- Third-plane output is 240.

Changing only the first pixel changes its first-plane output but does not change either chroma output.

### Alternating-row chroma

Given width 2 and height 3:

- Source consumption is 24 bytes.
- First-plane pairs are written at offsets 0–1, 4–5, and 8–9.
- Chroma is calculated from the second pixels of source rows 0 and 2.
- Each chroma plane receives bytes at offsets 0 and 4.
- The second pixel of source row 1 has no effect on either chroma plane.

### Odd width consumes another pixel

Given width 3 and height 1:

- Two pairs are processed.
- Four source records, totaling 16 bytes, are consumed.
- Four first-plane bytes are written, including offset 3.
- Two bytes are written to each chroma plane.
- No absent-pixel substitution or edge replication occurs.

Supplying only three source records does not satisfy the actual access requirement.

### Width 1 overwrites earlier chroma

Given width 1 and height 3:

- Every row consumes two source records and writes two first-plane bytes.
- The chroma stride is zero.
- Row 0 writes offset 0 of both chroma planes.
- Row 2 writes those same locations again.
- Final chroma values come from the second pixel of row 2.

### Width 9 overlaps chroma rows

Given width 9 and height 3:

- Five pairs are processed per row.
- The first-plane stride is 12.
- The chroma stride is 4.
- Row 0 writes chroma offsets 0 through 4.
- Row 2 writes chroma offsets 4 through 8.

The row-2 write at offset 4 replaces row 0’s final chroma sample in each plane.

### Nonpositive dimensions

With height zero or negative, no source or destination bytes are accessed.

With positive height and width zero or negative, no source or destination bytes are accessed, although row-control iteration still occurs.

## Uncertainties

- **Inferred intent:** Blue–green–red channel naming and a planar 4:2:0-style interpretation fit the coefficients and sampling geometry. No supplied external contract establishes a particular named media format.
- **documented_only:** Whether callers are required to provide positive, even dimensions is unknown. The function does not enforce that restriction.
- **documented_only:** No caller context establishes actual buffer sizes, permitted aliasing, or whether chroma row overlap for certain odd widths is intentional.
- **documented_only:** The meaning of the fourth byte in each source record is unknown; only its non-use is established.
- **documented_only:** Behavior beyond valid memory and defined arithmetic bounds is not established. In particular, no portable guarantee is made for signed horizontal-counter overflow or invalid address formation.
- **documented_only:** No independent runtime observation confirms the supplied representation’s exact behavior on a particular target.

## Evidence

The supplied function body is the sole static evidence for this contract. It establishes the six inputs, stride arithmetic, paired source traversal, coefficient calculations, alternating-row chroma writes, store order, and absence of explicit validation or dependency calls.

Format names and channel semantics are identified separately as inferred intent. Caller requirements, target-specific behavior outside the stated execution domain, and unspecified surrounding contracts remain documented unknowns.

No runtime tests have been performed.
