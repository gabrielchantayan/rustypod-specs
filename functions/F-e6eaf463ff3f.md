<!-- Trusted provenance: F-e6eaf463ff3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6eaf463ff3f

## Purpose

F-e6eaf463ff3f transfers byte samples from three source regions to three destination regions, processing the regions in order. Destination geometry determines the number of transfers. Two values obtained from F-d7be9cfb4000 determine which source coordinates are selected.

The first region uses the supplied dimensions directly. The second and third use logically halved destination dimensions and source row strides. Destination rows have alignment-derived spacing; source rows do not receive corresponding padding.

**Inferred intent:** the three regions represent image planes, with two half-resolution secondary planes, and the operation performs nearest-sample resizing. Neither an image format nor a color interpretation is established. The dependency’s operation is also unknown, so a particular resize ratio cannot be promised from dimensions alone.

This contract describes established static behavior separately from conditional arithmetic interpretations and unresolved external behavior. No runtime tests have been performed.

## Interface

The sole explicit input is a pointer to a descriptor containing at least ten consecutive 32-bit words. The function has no explicit return value.

| Word index | Byte offset | Meaning within this function |
|---|---:|---|
| 0 | 0 | First source base address |
| 1 | 4 | Second source base address |
| 2 | 8 | Third source base address |
| 3 | 12 | Source width and source-stride basis |
| 4 | 16 | Source height |
| 5 | 20 | First destination base address |
| 6 | 24 | Second destination base address |
| 7 | 28 | Third destination base address |
| 8 | 32 | Destination width |
| 9 | 36 | Destination height |

Address-bearing fields are 32 bits. The interface does not establish support for native pointers wider than those fields. Descriptor byte order and external structure declarations are not supplied.

Each sample is one byte. The operation neither interprets sample values nor combines neighboring samples.

### Caller obligations

For an invocation with defined, memory-safe behavior, the caller must provide readable descriptor storage and accessible storage for every actual byte access. This requires accounting for the selected coordinates, destination row spacing, arithmetic behavior, and any changes caused by overlapping storage or dependency effects.

There is no explicit validation of:

- Null or invalid addresses.
- Buffer capacities.
- Dimension signs or zeros.
- Arithmetic overflow.
- Overlap between buffers or with the descriptor.
- Preconditions of F-d7be9cfb4000.

The visible body does not allocate output storage or establish an ownership transfer. Dependency ownership behavior remains unknown.

### Arithmetic scope

The representation distinguishes signed and unsigned 32-bit operations. Loop eligibility uses signed comparisons; source-bound comparisons use unsigned comparisons. Logical halving and byte-address calculations must not be replaced indiscriminately by signed arithmetic.

Some signed operations require additional execution-model assumptions, particularly left shifts used in dependency arguments, products used for coordinates and destination offsets, and right shifts of negative values. This contract does not promise portable wraparound for signed overflow.

Where a result below is explicitly described under **32-bit machine interpretation**, it assumes two’s-complement interpretation, arithmetic right shift of negative signed values, and the stated fixed-width behavior. Such conditional descriptions do not resolve the original executable’s overflow semantics.

## Behavior

### Dependency interaction and descriptor timing

Before any plane transfer, F-d7be9cfb4000 is invoked for the horizontal increment, using the source width shifted left by twelve bits and the destination width. After that call returns, it is invoked for the vertical increment, using the source height shifted left by twelve bits and the destination height.

The two returned values are retained and used across all three planes. They participate as signed 32-bit increments.

The vertical-call descriptor inputs are read after the horizontal call returns. The relative evaluation order of the two arguments within an individual call is not established.

These calls are not conditional on positive output dimensions. Even an invocation that ultimately transfers no bytes reaches both calls if both return normally.

The dependency may modify memory or global state; preservation is not established. In particular, plane setup uses descriptor contents available after both calls, rather than a guaranteed entry-time descriptor snapshot.

### Plane geometry

Each plane obtains its own snapshot of base addresses, source stride, and output dimensions when that plane begins.

| Property | First plane | Second and third planes |
|---|---|---|
| Source and destination bases | Corresponding address fields | Corresponding address fields |
| Source row stride | Source width word | Source width word logically shifted right once |
| Active output width | Destination width word | Destination width word logically shifted right once |
| Active output height | Destination height word | Destination height word logically shifted right once |
| Ordinary destination row alignment | Eight bytes | Four bytes |

For the first plane, descriptor reads occur in this order: source base, destination base, source width, destination height, destination width.

For each secondary plane, the order is: source base, destination base, source width, destination width, destination height.

These snapshots are independent. A store during an earlier plane can alter the descriptor values later planes read if destination storage overlaps the descriptor.

Logical halving treats the original field as unsigned. For example, a high-bit-set dimension does not remain negative after halving: it becomes a potentially very large nonnegative value. For ordinary nonnegative odd dimensions, halving discards the remainder.

A plane has active rows only if its snapshotted output height is positive as a signed 32-bit value. An active row has sample transfers only if its snapshotted output width is positive under the same interpretation. Positive height with nonpositive width still permits row iteration, but no sample accesses.

### Destination layout and traversal

Planes finish in first-to-third order. Within a plane, samples are transferred in row-major order, starting with row zero and column zero.

For ordinary nonnegative widths without problematic intermediate arithmetic:

- First-plane destination pitch is the active width rounded upward to a multiple of eight bytes.
- Secondary-plane destination pitch is the active width rounded upward to a multiple of four bytes.

The destination byte for a sample is at its snapshotted destination base, plus its row’s pitch offset, plus its column index.

Only active columns receive direct stores. Padding between the active width and destination pitch is not initialized or copied. This does not guarantee that padding remains unchanged if it aliases another active destination or memory modified by the dependency.

The alignment expressions are not an unlimited mathematical round-up operation. Under a 32-bit machine interpretation, with alignment \(a\) equal to eight or four, their effective pitch is:

\[
a\;\operatorname{trunc}_0\!\left(
\frac{\operatorname{signed32}(w+a-1)}{a}
\right),
\]

where the addition inside \(\operatorname{signed32}\) is modulo \(2^{32}\), and \(\operatorname{trunc}_0\) means truncation toward zero.

Thus a positive width near the signed limit can produce a negative pitch under that interpretation. Ordinary alignment guarantees apply only within a suitable arithmetic range. Row-offset multiplication and destination address addition impose further representability requirements.

### Source-coordinate selection

The same horizontal and vertical increments serve every plane; the secondary passes do not recompute or halve them.

Each output column determines a horizontal product with the horizontal increment. Each active output row determines a vertical product with the vertical increment. The vertical product is formed once for that row, while the vertical bound is checked separately for each transferred byte.

The coordinate conversion has twelve fractional bits, but its negative-product behavior is not conventional nearest-integer rounding.

For a representable signed product \(p\), under the stated 32-bit machine interpretation, the provisional coordinate is the unsigned 32-bit representation of:

\[
\operatorname{trunc}_0(p/4096)
+
\operatorname{bit}_{11}
\!\left(
\operatorname{uint32}\bigl(p+2047\,\mathbf{1}_{p<0}\bigr)
\right).
\]

Here, bit 11 contributes either zero or one, and \(\mathbf{1}_{p<0}\) is one only for a negative product. This equation describes the observable coordinate mapping, not a requirement to implement it through particular intermediate operations.

For nonnegative products, the mapping is ordinary half-up selection:

\[
\left\lfloor\frac{p+2048}{4096}\right\rfloor.
\]

For negative products, that simplification is invalid. Under the conditional machine interpretation, products of \(-1\), \(-2048\), and \(-4096\) produce provisional signed-coordinate values of \(0\), \(1\), and \(-1\), respectively. The last is subsequently treated as unsigned for comparison.

The bit-selection expression uses unsigned arithmetic in the supplied representation. Its behavior should not be described as arbitrary signed multiplication overflow. The uncertainty instead includes the preceding signed coordinate products and the interpretation of signed right shifts.

### Bounds and actual byte access

Every sample uses fresh source-height and source-width reads from the descriptor. These bounds are not part of the plane snapshot.

The observable memory-access order for a sample is:

1. Read the current source-height field.
2. Read the current source-width field.
3. Read the selected source byte.
4. Store that byte at the selected destination address.

The vertical comparison and possible adjustment precede the width-field read. Both adjustments precede the source-byte read.

Each provisional coordinate is compared with its corresponding bound as an unsigned 32-bit quantity. If the coordinate is greater than or equal to the bound, it is reduced by exactly one.

This is not saturation or a general clamp:

- A coordinate far beyond the bound remains far beyond it.
- A zero coordinate with a zero bound becomes \(2^{32}-1\).
- A coordinate represented as negative in signed form participates as a large unsigned value.
- There is no independent lower-bound correction.

All planes use the unhalved descriptor source-width and source-height fields for these comparisons. Secondary source strides are halved, but their comparison bounds are not.

The source byte address combines the snapshotted source base, the selected vertical coordinate times the snapshotted source stride, and the selected horizontal coordinate. The source-address expression uses unsigned 32-bit arithmetic. Address wraparound therefore does not itself establish that the resulting address is valid storage.

No whole-row or whole-plane snapshot is made. Each byte is read before its corresponding store, and the next sample follows afterward. Earlier stores can consequently affect later source bytes or later descriptor reads.

## Side effects

The function’s direct writes are one-byte stores to active destination locations. It has no direct descriptor-field assignment, allocation, release, buffer clearing, diagnostic output, or error-result construction.

Aliasing can broaden those effects. A destination store may modify:

- A source byte needed later.
- Another plane’s destination.
- Descriptor fields consulted by later samples or plane setup.
- Other storage reachable through the supplied addresses.

Overlapping buffers therefore have sequential transfer behavior, not guaranteed snapshot-copy behavior.

The two dependency calls may have additional effects on memory, globals, ownership, or control flow. Their effects are not constrained by the visible body.

Invalid accesses and unsupported arithmetic conditions are not converted into a defined failure result. Platform fault behavior and dependency failure behavior remain unresolved. Completion is not guaranteed if a dependency fails to return.

## Dependencies

### F-d7be9cfb4000

| Invocation | First argument | Second argument | Retained result |
|---|---|---|---|
| First | Source width shifted left twelve bits | Destination width | Horizontal increment |
| Second | Source height shifted left twelve bits | Destination height | Vertical increment |

Exactly two calls are reached before plane processing if both return normally. No other explicit dependency calls are present.

The dependency’s mathematical operation is **documented_only**. Division is a plausible interpretation, not an established property. Its behavior for zero, negative, or overflow-sensitive arguments is unknown, as are its memory effects, global-state effects, allocation or release behavior, and return guarantees.

The caller of F-e6eaf463ff3f cannot infer valid increments merely from valid-looking dimensions.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observed executions. Unless an example says otherwise, assume stable descriptor contents, nonoverlapping valid storage, suitable arithmetic semantics, and dependency calls that return without disruptive effects.

### Identity-coordinate transfer

With source and destination dimensions of eight by four, and both dependency results equal to 4096:

- The first plane transfers eight bytes on each of four rows, with source stride and destination pitch both eight.
- Each secondary plane transfers four bytes on each of two rows, with source stride and destination pitch both four.
- Each active output position selects the source position with matching coordinates.

The dimensions alone do not establish those dependency results.

### Downsampling and destination padding

With source dimensions ten by six, destination dimensions five by three, and both increments equal to 8192:

- First-plane source columns are 0, 2, 4, 6, and 8.
- First-plane source rows are 0, 2, and 4.
- Its destination pitch is eight, leaving three bytes per row without direct stores.
- Each secondary plane has active dimensions two by one, source stride five, and destination pitch four.
- Its selected source columns are 0 and 2; its two padding bytes receive no direct stores.

### Half-step selection

With horizontal increment 2048, output columns zero through three have provisional source columns 0, 1, 1, and 2.

If the source-width field remains two, their adjusted columns are 0, 1, 1, and 1.

### Negative-product distinction

Under the conditional 32-bit machine interpretation, a horizontal product of \(-2048\) produces provisional coordinate one. With a current unsigned source-width bound greater than one, that coordinate remains one.

Acceptance must not require symmetric rounding to zero or minus one.

### Upper-bound adjustment is only one decrement

A provisional column of seven with current source width three becomes six, not two. If storage exists at the resulting address, that is the byte accessed. The function performs no further range correction.

Likewise, provisional coordinate zero with bound zero becomes \(2^{32}-1\); it does not cause a checked error return.

### Secondary bounds remain unhalved

With source width six, destination width four, and horizontal increment 12288, a secondary plane has source stride three and active width two.

Its second output column selects provisional source column three. Because the comparison bound is six, that column is not decremented. Acceptance must not substitute the secondary stride, three, as the horizontal bound.

### Empty output still invokes dependencies

If destination width and height are zero and remain zero, both dependency calls are reached if they return normally. No plane performs direct sample stores. Whether the dependency accepts those arguments remains unknown.

### Sequential overlap

If an earlier destination byte aliases a source byte used by a later sample, that later read observes the value then present, including the earlier store. Acceptance must not require preservation of entry-time source contents.

## Uncertainties

The following remain **documented_only**:

- The dependency’s calculation, error behavior, side effects, and ownership behavior.
- External descriptor declarations, byte order, buffer capacities, and intended image format.
- The legal dimension range imposed by callers.
- Original-executable behavior for signed overflow and other unsupported arithmetic cases.
- Whether the supplied signed-shift expressions fully capture the execution model.
- External restrictions on overlap, descriptor aliasing, or negative dimensions.
- Fault behavior for inaccessible addresses.
- Any concurrency or memory-mapped-I/O guarantees.

The apparent image-resizing purpose is inferred intent. It does not override the observable logical halving, unsigned bounds, one-step decrements, or sequential byte transfers.

## Evidence

Established static evidence is the supplied function representation and the opaque dependency mapping. It supports the descriptor layout, two ordered dependency calls, three ordered plane passes, per-plane snapshots, per-sample bound reloads, signed loop tests, logical halving, alignment-derived destination spacing, coordinate-selection rule, and byte-at-a-time transfer order.

The representation also establishes the absence of explicit validation, direct allocation or release, and a returned error status in this body. It does not establish corresponding absences inside the dependency.

No dependency implementation, caller contract, runtime trace, or runtime test result was supplied. No runtime tests have been performed.
