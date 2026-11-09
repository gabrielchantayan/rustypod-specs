<!-- Trusted provenance: F-cff3dffe14df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cff3dffe14df

## Purpose

F-cff3dffe14df performs a row-major conversion from three byte-addressed image planes into packed four-byte pixels. Each output pixel contains three independently saturated color components followed by the constant byte 255. Two input planes are sampled at half the horizontal and vertical coordinate of the first plane.

The arithmetic is consistent with a limited-range, planar YUV-to-BGRA conversion. That format identification is **inferred intent**, not an independently established interface guarantee. The established contract is the byte layout, sampling pattern, arithmetic, and access order described below.

The supplied evidence leaves two important interface details unresolved:

- The third source-plane address and destination address come from stack-relative words whose origins are not established by the displayed four-argument interface.
- Two conversion coefficients are read from external storage, and their values are not supplied.

Consequently, this contract specifies a conditional conversion operation rather than claiming that the four displayed arguments alone constitute a usable calling interface.

## Interface

### Displayed arguments

The displayed interface contains four 32-bit arguments:

| Argument | Interpretation supported by the evidence |
|---|---|
| First | Signed width, denoted \(W\). |
| Second | Signed initial height, denoted \(H\). |
| Third | A 32-bit address value used as the first, full-resolution source-plane base. |
| Fourth | A 32-bit address value used as the first subsampled source-plane base. |

The function has no explicit return value. Converted data is returned through destination memory.

Pointers and address-bearing words are represented as 32-bit values in the supplied representation. This does not establish portability to a different pointer width or calling convention.

### Additional address inputs

For a positive initial height, the function obtains two more address-bearing words from stack-relative storage:

- The second subsampled source-plane base.
- The packed destination base.

Their positions relative to the local stride storage are visible, but their relationship to caller arguments, saved state, or another enclosing layout is not established. They must not be presented as verified fifth and sixth arguments.

For the behavioral descriptions below, the three source bases are called \(Y\), \(U\), and \(V\), and the destination base is called \(D\). These are descriptive roles, not a verified public parameter list.

### Strides and layouts

The full-resolution source stride is:

\[
S_Y = \bigl((W + 3)\bmod 2^{32}\bigr)\mathbin{\&}\texttt{0xFFFFFFFC}.
\]

The shared subsampled-plane stride is:

\[
S_C =
\bigl((\operatorname{trunc}(W/2)+3)\bmod 2^{32}\bigr)
\mathbin{\&}\texttt{0xFFFFFFFC}.
\]

Division by two is signed division with truncation toward zero in the displayed expression. For nonnegative widths, these are four-byte round-ups of \(W\) and \(\lfloor W/2\rfloor\), respectively.

The destination has no independently supplied pitch. Its row-start displacement is \(4Wr\) bytes for row \(r\), subject to the unresolved overflow behavior discussed below.

The source samples used for pixel \((r,c)\) are:

| Plane | Byte displacement from its base |
|---|---|
| Full-resolution | \(rS_Y+c\) |
| First subsampled | \(\lfloor r/2\rfloor S_C+\lfloor c/2\rfloor\) |
| Second subsampled | \(\lfloor r/2\rfloor S_C+\lfloor c/2\rfloor\) |

Each destination pixel occupies four consecutive bytes at displacement \(4(Wr+c)\).

### Preconditions for defined conversion use

A caller or surrounding execution environment must provide:

- Valid storage for the external coefficient reads.
- Accessible stack-relative words supplying the additional plane and destination addresses.
- Readable source bytes at every sampled address.
- Writable destination bytes at every output address.
- Address calculations and arithmetic within a range whose machine behavior is established.
- Any non-aliasing and stability guarantees needed to obtain an ordinary image conversion.

No validation enforces these conditions. Source/destination disjointness, coefficient stability, and ownership are not checked.

## Behavior

### Entry and empty dimensions

The function reads a signed 32-bit coefficient from external storage at entry and retains that value for later use. This occurs before the nonpositive-height return. Call this retained coefficient \(K\).

It also constructs local copies of the displayed arguments and the two derived strides.

If the initial height is less than one, the function returns without entering the row loop. In particular, it does not read source pixels or write destination pixels. This is not a guarantee of zero memory access: the entry coefficient read still occurs.

If the initial height is positive, row processing begins even when the width is zero or negative. A nonpositive width suppresses the pixel loop, but not the row-level address-word reads.

### Row processing

Rows begin at zero and advance by one. At the start of each row, the displayed sequence obtains:

1. The full-resolution stride.
2. The full-resolution source base.
3. The first subsampled source base.
4. The subsampled stride.
5. The second subsampled source base.
6. The destination base.

It then forms the row’s destination pointer using \(4Wr\).

For positive width, columns begin at zero and advance by one until all \(W\) pixels have been processed. After each row, the row counter is incremented and compared, as a signed value, with the height stored in the local argument-copy area.

Thus, the loop bound is visibly reloaded from that storage rather than expressed solely as an immutable entry value. Under ordinary execution in which this local storage is not overwritten, exactly \(H\) rows are visited. Effects of deliberate aliasing into that storage are not fully established.

### Per-pixel input acquisition

For each pixel, the visible sequence is:

1. Read the full-resolution source byte.
2. Subtract 16, producing \(y\).
3. Read an external coefficient and interpret its low 16 bits as signed. Call this per-pixel value \(L\).
4. Multiply the signed 16-bit interpretation of \(y\) by \(L\), producing \(t\).
5. Read the first subsampled source byte and subtract 128, producing \(u\).
6. Form the first-component numerator.
7. Read the second subsampled source byte.
8. Clamp and write the first component.

The source bytes are unsigned eight-bit quantities before subtraction:

\[
-16 \le y \le 239,\qquad
-128 \le u,v \le 127.
\]

The explicit signed 16-bit interpretation of \(y\) does not change its value over that range.

The coefficient \(L\) is obtained within the pixel loop, not snapshotted once per call. Stable coefficients therefore require an environmental guarantee. The access width of its backing storage is not established merely by the signed 16-bit interpretation.

### Component arithmetic

Subject to the arithmetic limitations below, define:

\[
t=yL
\]

\[
N_0=t+516u+128
\]

\[
N_1=t-100u-208v+128
\]

\[
N_2=Kv+298y+128.
\]

The first two components share the per-pixel product \(t\). The third does **not** reuse it: its full-resolution contribution uses the literal coefficient 298, independently of \(L\).

For a signed 32-bit numerator \(N\), the visible clamping operation is equivalent to:

\[
C(N)=
\begin{cases}
0 & N<0,\\
\lfloor N/256\rfloor & 0\le N<65536,\\
255 & N\ge65536.
\end{cases}
\]

The implementation uses signed right shifts, a comparison against 256, and a sign-derived mask to achieve this result. Negative intermediate values produce zero; large positive values produce 255. The added 128 is applied before shifting and saturation.

The four output bytes, in increasing address order, are:

\[
[C(N_0),\ C(N_1),\ C(N_2),\ 255].
\]

The likely channel interpretation is blue, green, red, and opaque alpha. Only the byte positions and computations are established.

### Arithmetic widths

The supplied representation uses signed 32-bit intermediates for the component numerators and a signed 32-bit retained coefficient \(K\).

The product \(yL\) fits in signed 32-bit range for all possible byte samples and signed 16-bit \(L\). The first two numerators also remain within signed 32-bit range.

The third numerator may overflow if \(K\) is unrestricted. For example, a sufficiently large magnitude of \(K\), multiplied by a nonzero \(v\), exceeds that range.

The supplied evidence does not establish a portable result for overflowing signed products or additions. In particular, it does not justify silently substituting unbounded arithmetic or asserting a language-level wrapping rule. Formula-based acceptance examples below require representable intermediate results.

The same limitation applies to overflowing address arithmetic and destination row displacement.

### Odd dimensions

Subsampled coordinates use division by two through right shifts of nonnegative row and column counters. An odd width therefore accesses \(\lceil W/2\rceil\) samples per visited chroma row, even though the chroma stride is derived from \(\lfloor W/2\rfloor\).

This distinction is observable:

- Width 1 produces chroma stride zero, while still sampling chroma column zero.
- Width 9 produces chroma stride four, while sampling chroma columns zero through four.

The function neither rejects these widths nor replaces the stride calculation with a ceiling-based layout. A caller must provide storage matching the actual addressing pattern rather than assuming a conventional odd-width planar layout.

An odd positive height visits the last row and uses chroma row \(\lfloor(H-1)/2\rfloor\).

## Side effects

### Destination writes

For each processed pixel, four one-byte writes occur in this order:

1. First component at pixel offset zero.
2. Second component at offset one.
3. Third component at offset two.
4. Constant 255 at offset three.

Pixels are processed left to right, then rows top to bottom. No destination read is visible.

All three source bytes for the current pixel are read before its first output-byte write. This does not make arbitrary overlap safe: a write can change source bytes needed by later pixels or rows.

### Other memory and state

The visible operation also:

- Reads the retained external coefficient once at entry.
- Reads the other external coefficient for every processed pixel.
- Writes local stack storage used for strides and argument copies.
- Reads stack-relative address words at row boundaries.

No explicit global write, allocation, release, ownership transfer, callback, or external function call appears.

There is no visible error code, exception construction, bounds check, null check, or rollback. If an access fails, the supplied evidence provides no recovery contract. Earlier output writes may already have occurred.

No ownership policy for the source or destination buffers can be inferred beyond their use as memory addresses.

## Dependencies

No called-function dependencies are visible in the supplied body. No additional opaque function IDs can therefore be assigned from this evidence.

The operation depends on external data and execution context:

| Dependency | Established use | Unresolved property |
|---|---|---|
| Retained coefficient storage | Read as a signed 32-bit value at entry; used in the third component | Value, initialization, validity, concurrent modification rules |
| Per-pixel coefficient storage | Read inside each pixel iteration; interpreted as signed 16-bit | Backing access width, value, initialization, stability |
| Additional stack-relative address words | Supply the second subsampled plane and destination bases | Calling convention and provenance |
| Source and destination memory | Byte reads and writes at calculated addresses | Sizes, lifetime, overlap policy, synchronization |

No assumption about unknown callees preserving global state or owning memory is required: there are no visible callees.

## Acceptance examples

These are conditional static acceptance examples, not runtime results. Unless stated otherwise, assume valid address inputs, stable local metadata, valid buffers, and no arithmetic overflow.

### Nonpositive height

For \(H=0\), the function performs its entry coefficient read and returns without processing a row. Source pixel bytes and destination pixel bytes remain untouched by the visible pixel-conversion path.

The same applies to negative initial heights.

### Positive height with zero width

For \(W=0\) and \(H=2\), the row-level address words are obtained for two rows, but neither row processes a pixel. There are no destination-byte writes and no per-pixel coefficient reads.

This case does not establish that inaccessible row-level address-word storage is harmless.

### Neutral black input

For a processed pixel with source bytes \(Y=16\), \(U=128\), and \(V=128\):

\[
y=u=v=0.
\]

All three numerators are 128, independently of the coefficient values. The output is:

\[
[0,\ 0,\ 0,\ 255].
\]

### Neutral white under a specified coefficient

If the per-pixel signed coefficient is \(L=298\), a pixel with \(Y=235\), \(U=128\), and \(V=128\) produces:

\[
[255,\ 255,\ 255,\ 255].
\]

Here \(y=219\), each numerator is 65390, and division by 256 discards the fractional part.

### First subsampled component contribution

For \(Y=16\), \(U=129\), and \(V=128\), the numerators are 644, 28, and 128. The output is:

\[
[2,\ 0,\ 0,\ 255].
\]

This example does not require a particular value of either coefficient.

### Second subsampled component contribution

For \(Y=16\), \(U=128\), and \(V=129\), the output is:

\[
[0,\ 0,\ C(K+128),\ 255],
\]

provided the displayed addition is representable. The first component is zero, and the negative second-component numerator clamps to zero.

### Shared chroma sampling

For width and height both equal to two, all four output pixels use chroma coordinate \((0,0)\). Each pixel still performs its own chroma-byte reads and per-pixel coefficient read; shared coordinates do not imply cached samples.

### Odd-width stride behavior

For \(W=9\), the full-resolution stride is 12 and the subsampled stride is four. Columns eight and zero of successive chroma rows can therefore address the same byte through different coordinate pairs. A conforming description must preserve this addressing behavior rather than assume a larger chroma stride.

## Uncertainties

- **documented_only — Complete calling interface:** The origins of the second subsampled-plane and destination addresses are unresolved.
- **documented_only — Coefficient values:** No numeric values are supplied for either external coefficient. Conventional conversion constants must not be substituted as facts.
- **documented_only — Overflow semantics:** Overflowing signed arithmetic and address calculations lack a fully established execution contract.
- **documented_only — Aliasing:** Overlapping source, destination, coefficient, or stack storage can affect later reads. No supported overlap policy is shown.
- **documented_only — Concurrency:** No synchronization is visible. Concurrent changes to coefficients or buffers have no established consistency guarantee.
- **Inferred intent:** The layouts and coefficients suggest planar subsampled YUV conversion into BGRA-like pixels.
- **Unknown:** Buffer lengths, caller validation, intended dimension restrictions, and fault behavior outside this function.

## Evidence

Established static evidence consists solely of the supplied function representation: argument widths, local stride expressions, loop conditions, byte-address calculations, coefficient uses, saturation expressions, and ordered byte stores.

The channel names and image-format interpretation are inferences from those operations. A complete calling convention, concrete coefficient values, memory ownership policy, and overflow behavior are not established.

No runtime tests, execution traces, or implementation-access checks have been performed.
