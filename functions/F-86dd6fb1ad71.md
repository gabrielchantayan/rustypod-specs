<!-- Trusted provenance: F-86dd6fb1ad71; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-86dd6fb1ad71

## Purpose

F-86dd6fb1ad71 conditionally computes three output component bytes and one output alpha byte from two four-byte records and a 32-bit control value. It delivers the result through a dependency rather than returning it directly.

The established behavior has two routes:

- Invoke F-0c619ddcbcea with the context argument when the computed effective alpha is zero, the first record’s alpha is zero, or the effective alpha is exactly 256.
- Otherwise, combine the records’ components and invoke F-2355a01d30b6 with the context and four byte-valued results.

The arithmetic suggests fixed-point alpha compositing of straight, rather than already premultiplied, components. That interpretation is inferred intent. Component order, color space, the meaning of the context, and the effects of both output dependencies are not established.

In particular, the alternate dependency must not be described as a no-op, a copy, or an equivalent implementation of the computed route. Only its invocation is established.

## Interface

The function accepts four arguments:

| Argument | Established interpretation |
|---|---|
| Context | An opaque 32-bit value, passed unchanged to whichever terminal dependency is selected. It is not directly dereferenced by this function. |
| First record | A byte-addressed input whose offsets 0–2 supply three components and whose offset 3 supplies alpha. |
| Second record | A byte-addressed input with the same four-byte layout. |
| Control | An unsigned 32-bit integer used to scale the second record’s alpha. |

Each record field is an unsigned 8-bit value in the range 0–255. No component names or channel ordering are established. References below to the first, second, and third components mean offsets 0, 1, and 2, respectively.

There is no explicit return value and no explicit status result. The observable result consists of dependency calls and any effects of those calls.

For an invocation to complete normally, every byte actually read must be readable when accessed. The function performs no explicit pointer, length, alignment, or range validation. Four readable bytes per record are sufficient for its own accesses, but some routes require fewer:

- The second record’s alpha is always read.
- The first record’s alpha is read only when the effective alpha is nonzero.
- Component bytes are read only on the computed route.

No ownership transfer, allocation, or release is visible in this function. The records are not directly modified. Nothing here establishes whether a dependency retains references, changes external state, or releases resources associated with the context.

## Behavior

### Numeric model and byte expansion

The arithmetic uses a 256-based fixed-point scale. Define byte expansion as:

\[
E(x)=
\begin{cases}
x,&x\le127,\\
x+1,&x\ge128.
\end{cases}
\]

Thus 0 remains 0, 127 remains 127, 128 becomes 129, and 255 becomes 256. Expanded byte values never equal 128.

When converting an intermediate back toward byte scale, the function subtracts one if that intermediate is strictly greater than 127. For component arguments, it then retains only the low eight bits. This is not saturation: an out-of-range result is masked, not clamped.

The supplied representation uses 32-bit integer arithmetic. Unsigned expressions wrap modulo \(2^{32}\). Signed right shifts are represented as arithmetic shifts, corresponding to division by a power of two rounded toward negative infinity. Exact behavior in cases involving signed overflow remains an uncertainty rather than a mathematical unbounded-integer guarantee.

### Effective alpha and route selection

The second record’s alpha is read first. The control is increased by one when its unsigned value exceeds 127. This addition is unsigned and therefore wraps at the 32-bit boundary. The adjusted control is multiplied by the expanded second alpha, again with unsigned 32-bit wraparound.

The resulting 32-bit product is converted to a signed integer and shifted right by eight bits. Call the result the **effective alpha**.

For control values in the conventional byte range 0–255, this reduces to:

\[
q=\left\lfloor
\frac{E(\text{control})E(\text{second alpha})}{256}
\right\rfloor,
\]

with \(0\le q\le256\).

The function does not enforce that control range. Outside it, the unsigned adjustment, multiplication, signed conversion, and shift remain relevant. Such inputs must not be silently treated as clamped byte values.

Route selection is short-circuited in this order:

1. If the effective alpha is zero, invoke F-0c619ddcbcea with the context. Do not read the first record.
2. Otherwise, read the first record’s alpha. If it is zero, invoke F-0c619ddcbcea with the context.
3. Otherwise, if the effective alpha is exactly 256, invoke F-0c619ddcbcea with the context.
4. Otherwise, use the computed route.

Consequently, an effective alpha of 256 still requires reading the first record’s alpha. No component bytes are read on any of the alternate routes.

### Component sampling

On the computed route, the saved first-record alpha is expanded. The function then reads and expands:

- First-record components at offsets 0, 1, and 2, in that order.
- Second-record components at offsets 0, 1, and 2, in that order.

These reads all occur before any call to F-870c4fcf7295. The second alpha is not read again.

The alpha values and component values are sampled at separate points; there is no established atomic snapshot of either record. Aliasing between the records is not explicitly prohibited, and the function itself performs no intervening writes to them.

### Fully opaque first record

When the first record’s expanded alpha is 256, corresponding to an input alpha byte of 255, no call to F-870c4fcf7295 occurs.

For each component, let \(d\) be its expanded first-record value and \(s\) its expanded second-record value. The intermediate is:

\[
v=d+\left\lfloor\frac{q(s-d)}{256}\right\rfloor.
\]

The component argument delivered to F-2355a01d30b6 is:

\[
\bigl(v-\mathbf{1}_{v>127}\bigr)\bmod256.
\]

The output alpha argument is exactly 255.

The right-shift rounding matters when the second component is smaller than the first. A negative product is rounded downward by the arithmetic shift, not truncated toward zero.

### Partially opaque first record

Otherwise, the first record’s alpha is nonzero and not 255. Let its expanded value be \(a\). For ordinary byte-range controls on this route, both \(a\) and \(q\) lie between 1 and 255.

The function computes a combined expanded alpha:

\[
t=q+\left\lfloor\frac{a(256-q)}{256}\right\rfloor.
\]

It then calls F-870c4fcf7295 with two integer arguments:

- 4096;
- the combined expanded alpha \(t\).

Call its signed integer result \(r\). The dependency’s actual operation is unknown. Its use resembles a reciprocal normalization factor, but that does not establish that it performs division.

Each expanded first-record component is reduced by the first alpha:

\[
p=\left\lfloor\frac{da}{256}\right\rfloor.
\]

Using the corresponding expanded second-record component \(s\), the normalized component intermediate is:

\[
v=\left\lfloor
\frac{r\bigl(q(s-p)+256p\bigr)}{4096}
\right\rfloor.
\]

These formulas describe the represented arithmetic when the signed operations remain within range. They are not authorization to widen overflowing operations and thereby change results.

Each output component uses the same subtract-and-mask conversion as the opaque route. The output alpha is:

\[
\bigl(t-\mathbf{1}_{t>127}\bigr)\bmod256.
\]

For byte-range controls reaching this branch, the combined expanded alpha lies between 1 and 255. That bound does not constrain the dependency’s return value or independently guarantee valid normalized components.

### Final delivery

On either computed subroute, F-2355a01d30b6 receives five arguments:

1. The unchanged context.
2. The first output component, restricted to 0–255.
3. The second output component, restricted to 0–255.
4. The third output component, restricted to 0–255.
5. The output alpha, restricted to 0–255.

There is no direct store of the computed result into either input record.

## Side effects

The function’s directly established memory effects are byte reads from the two records. It contains no direct writes to either record or to identified global storage.

If all called dependencies return normally:

- An alternate-route invocation calls F-0c619ddcbcea once.
- An opaque-first-record computed invocation calls F-2355a01d30b6 once.
- A partial-first-record computed invocation calls F-870c4fcf7295 once, followed by F-2355a01d30b6 once.

All input-record reads on the partial route precede F-870c4fcf7295. The later component calculations use the previously sampled values rather than rereading the records.

No preservation of globals, context state, or memory ownership is assumed for any dependency. Likewise, there is no evidence that the terminal dependencies are pure or that they modify a particular destination.

No error reporting or recovery is present. If a required read fails or a dependency does not return normally, the supplied evidence provides no recovery behavior. In particular, delivery through F-2355a01d30b6 is conditional on the normalization dependency returning.

## Dependencies

| Opaque ID | Established role and arguments | Not established |
|---|---|---|
| F-0c619ddcbcea | Alternate terminal operation; receives only the unchanged context. Its return value, if any, is unused. | Output semantics, memory effects, ownership effects, and error behavior. |
| F-870c4fcf7295 | Called only for a nonzero, nonopaque first alpha on the computed route. Receives 4096 and the combined expanded alpha; returns an integer used in component normalization. | Division semantics, rounding, valid argument domain, global effects, and failure behavior. |
| F-2355a01d30b6 | Computed-result terminal operation; receives context and four byte-valued integer arguments in component/component/component/alpha order. Its return value, if any, is unused. | Whether it stores, emits, transforms, or otherwise consumes those values; ownership and error behavior. |

The contract does not replace unknown dependency behavior with guessed implementations.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless otherwise stated, they assume 32-bit arithmetic with arithmetic signed right shifts, readable required bytes, and normally returning dependencies.

### Zero control avoids the first record

With control 0 and any second alpha byte:

- The effective alpha is zero.
- Only the second record’s alpha is required from the records.
- F-0c619ddcbcea receives the context.
- Neither other dependency is called.

The first record need not be readable for this function’s own accesses on this route.

### Zero first alpha selects the alternate operation

With control 255, second alpha 128, and first alpha 0:

- The effective alpha is 129.
- The first alpha is read and found zero.
- No component bytes are read.
- F-0c619ddcbcea receives the context.

### Maximum effective alpha still reads first alpha

With control 255 and second alpha 255, the effective alpha is 256.

The function reads the first alpha before taking the alternate route. Even if that first alpha is nonzero, no component bytes are read and F-0c619ddcbcea is selected.

### Opaque interpolation preserves fixed-point rounding

Suppose:

- First record: components \((0,255,127)\), alpha 255.
- Second record: components \((255,0,128)\), alpha 255.
- Control: 127.

The effective alpha is 127. The first alpha is opaque, so no normalization dependency is called.

F-2355a01d30b6 receives the context followed by:

\[
(127,\ 128,\ 127,\ 255).
\]

The decreasing second component demonstrates downward rounding of a negative intermediate.

### Partial alpha depends on the normalization result

Suppose:

- First record: components \((255,0,0)\), alpha 127.
- Second record: components \((0,255,0)\), alpha 255.
- Control: 127.

The effective alpha is 127 and the combined expanded alpha is 190. F-870c4fcf7295 receives \((4096,190)\).

**If** that dependency returns 21, F-2355a01d30b6 receives the context followed by:

\[
(83,\ 165,\ 0,\ 189).
\]

The assumed return value is a condition of this example, not an established division result.

### Control is not clamped

With control \(2^{32}-1\), the unsigned control adjustment wraps to zero. The effective alpha is therefore zero for every second alpha byte.

The function selects F-0c619ddcbcea without reading the first record. Treating this control as 255 would not reproduce the established unsigned arithmetic.

## Uncertainties

The following are **documented_only**, not behavior filled in by assumption:

- Whether callers are required to restrict control to 0–255.
- The semantic names and color-space interpretation of the component bytes.
- The meaning, lifetime, and ownership of the context.
- The behavior of F-0c619ddcbcea and F-2355a01d30b6 beyond their observed arguments.
- The algorithm, rounding, exceptional behavior, and side effects of F-870c4fcf7295.
- Exact target behavior for signed overflow and for out-of-range unsigned-to-signed conversion. The represented 32-bit signed interpretation is clear enough to describe the operation, but portability across language or execution models is not established.
- Concurrency guarantees for externally changing input records.
- Whether callers permit invalid pointers on routes that do not dereference them.

The compositing interpretation and reciprocal-normalization interpretation are inferred intent only. Neither overrides the explicit alternate-route calls, integer boundaries, or unknown dependency behavior.

## Evidence

The supplied static representation establishes argument widths, byte offsets, short-circuit route selection, input-read order, integer operations, dependency arguments, and the absence of direct input writes.

The 256-based expansion, alpha-weighted component calculations, and 4096-based normalization support—but do not prove—the inferred compositing purpose.

No dependency implementation, caller contract, external format definition, or runtime observation was supplied. No runtime tests have been performed. The acceptance examples are arithmetic consequences of the static representation under their stated conditions.
