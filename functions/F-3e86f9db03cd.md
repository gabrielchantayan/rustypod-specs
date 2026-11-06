<!-- Trusted provenance: F-3e86f9db03cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e86f9db03cd

## Purpose

F-3e86f9db03cd converts up to 256 source bytes into scaled, signed 32-bit destination elements and replaces a destination header according to a backward examination of that destination.

The payload is organized from the end of the input toward its beginning. Each element contains up to four input bytes, with the last byte of a group occupying its least significant eight bits. Payload elements occupy increasing destination indices starting at one. Index zero is reserved for the header.

The final header depends on configuration and destination contents, not solely on the bytes supplied to this invocation. In particular, the examined destination range can extend beyond the newly generated payload.

**Established static evidence:** the unsigned input limit, byte-consumption order, packing and scaling, destination-store order, configuration uses, two header stores, and product-based scan condition.

**Inferred intent:** the payload and header resemble a signed, multiword numeric representation with removal of insignificant high-order elements. Neither that interpretation nor a particular numeric format is required by the established behavior.

The contract below describes the represented behavior under its stated arithmetic and address-model conditions. It does not establish portable pointer semantics or recovery from invalid memory accesses. No runtime tests have been performed.

## Interface

### Arguments and return value

| Argument | Represented type and meaning |
|---|---|
| Source address | Signed 32-bit integer participating in numeric address formation |
| Destination | Address of signed 32-bit elements, four bytes per element |
| Requested count | Unsigned 32-bit number of source bytes |

Source bytes are unsigned eight-bit values.

A requested count greater than 256 produces the configured rejection result, a signed 32-bit value. That result is not constrained to be negative or nonzero.

An accepted request that completes normally returns signed zero. No separate result is provided for invalid storage, unsupported pointer behavior, or exceptional arithmetic.

### External configuration

Five signed 32-bit configuration values have distinct roles:

| Role | Observable use |
|---|---|
| Payload multiplier | Scales packed source groups |
| Rejection result | Supplies the return value for an oversized request |
| Header multiplier | Participates in the examination extent and both header values |
| Extent multiplier | Participates in the examination extent |
| Examination multiplier | Scales each examined destination value for the stopping test |

These names describe uses, not permitted values or defaults.

The payload multiplier is obtained before rejection is decided. Accepted execution obtains examination-related configuration only after payload construction.

There are three represented uses of the header multiplier with separate accesses: one captured for the final header, one used to calculate the extent, and one used to calculate the initial header. The contract does not require these accesses to produce a coherent snapshot.

### Required storage and arithmetic conditions

For an accepted count \(n\), the generated payload contains

\[
\left\lceil n/4\right\rceil
\]

elements, between zero and 64 inclusive.

The destination must support writes to index zero and every generated payload index. It must also support reads at every index actually examined afterward. The latter requirement is configuration-dependent and is not bounded by 64.

Source storage must support each byte access actually reached. Destination values examined by the scan must be usable signed 32-bit values; allocating storage without supplying usable contents is not sufficient when those contents can be read.

Normal completion also depends on the pointer interpretation, signed conversions, and signed arithmetic described under **Uncertainties**.

No allocation, release, ownership transfer, or retention of either supplied address is visible. Caller storage is used directly.

## Behavior

### Oversized requests

The count limit is an unsigned comparison. Counts from 257 through \(2^{32}-1\) are rejected; 256 is accepted.

On rejection, the payload multiplier has already been obtained. The rejection-result configuration is then obtained and returned. No source byte, destination element, or examination-related configuration is accessed on this path.

Thus, the rejection path does not require usable source or destination storage. Its configuration accesses must still be valid.

### Source addressing and payload layout

Accepted execution consumes source offsets from \(n-1\) down to zero. A zero count causes no source access.

For any consumed offset, the numeric address is the unsigned 32-bit representation of the source-address argument plus that offset, reduced modulo \(2^{32}\). The unsigned count operand makes this addition unsigned. Crossing the numeric boundary is therefore modular address arithmetic, not signed-addition overflow.

That numeric result is converted to a byte address. Modular arithmetic alone does not establish that the resulting address maps to readable memory.

For a complete group whose bytes, in ascending source-address order, are \(a,b,c,d\), the unscaled packed value is

\[
a\,2^{24}+b\,2^{16}+c\,2^8+d.
\]

The group nearest the end of the requested input becomes destination element one. Earlier groups occupy progressively higher destination indices.

If the count is not a multiple of four, the incomplete group comes from the beginning of the input and becomes the last payload element:

| Group in ascending source-address order | Unscaled value |
|---|---|
| One byte \(a\) | \(a\) |
| Two bytes \(a,b\) | \(a\,2^8+b\) |
| Three bytes \(a,b,c\) | \(a\,2^{16}+b\,2^8+c\) |

Unused high bits in an incomplete group are zero. These formulas describe numeric element values, not the memory byte order of signed destination elements.

The four-byte grouping follows from the represented address-based bookkeeping under four-byte element scaling. Its strict pointer-model validity is unresolved; the layout above is conditional on that bookkeeping having the represented numeric behavior.

### Payload scaling and stores

Each packed value is multiplied by the captured payload multiplier using unsigned 32-bit arithmetic. Equivalently,

\[
\text{scaled bits}
=
\bigl(
\operatorname{u32}(\text{payload multiplier})
\times
\text{packed value}
\bigr)\bmod 2^{32}.
\]

A negative payload multiplier does not change this multiplication into signed arithmetic.

The unsigned result is converted to signed 32-bit form. Results no greater than \(2^{31}-1\) are directly representable. The outcome for larger results is target-dependent and cannot unconditionally be described as two’s-complement reinterpretation.

The represented computation includes a multiplication and signed conversion after every consumed byte, not only at group boundaries. An incomplete final group also has a multiplication and conversion associated with its eventual store. Consequently, an unresolved out-of-range conversion can arise before any store for that group.

Payload stores occur at indices one through the payload count, in increasing order. A complete group is stored before the first source read belonging to the next group. An incomplete final group is stored after its last source read.

Payload construction neither writes index zero nor clears unused destination elements.

### Extent and initial header

After payload construction, the examination multiplier is captured, followed by the header multiplier retained for the final header.

The signed examination extent is

\[
\text{extent}
=
(\text{extent multiplier}\times
 \text{header multiplier used for extent})
\times
\text{payload count}.
\]

Both products are signed 32-bit operations, with the displayed grouping. The first product is evaluated even when the payload count is zero. The relative evaluation order of the two configuration accesses in that first product is not established.

The function then writes the initial header:

\[
\text{initial header}
=
\text{header multiplier used for initial header}
\times
\text{payload count}.
\]

This is also signed 32-bit multiplication.

The initial header store precedes both obtaining a negative extent’s positive magnitude and reading any destination element for examination. This ordering matters if forming that magnitude is not defined.

### Destination examination and final header

Provided the extent and its magnitude are representable, the first possible examined index is the absolute value of the extent.

Only positive indices are read. A destination value stops the examination when its signed 32-bit product with the captured examination multiplier is defined and nonzero. A defined zero product permits examination at the next lower index.

Exhausting all positive indices gives a stopping index of zero without reading destination element zero. An initial extent of zero therefore causes no examination reads.

The starting index is not limited to the generated payload count. Preexisting destination values at larger indices can be read first and can determine the result.

For a valid nonnegative starting index, the exhaustion check can calculate negative one at the lower boundary without overflowing. The problematic magnitude boundary is the minimum signed extent, whose positive magnitude is not representable in signed 32-bit form.

The final header replaces the initial header. Its value is:

- the captured final-header multiplier times the stopping index when the extent is positive;
- the captured final-header multiplier times the negated stopping index when the extent is zero or negative.

These are signed operations. A zero stopping index produces a zero final header when the operations are defined.

The sign adjustment follows the extent, not the payload sign. A negative final-header multiplier can change the resulting sign again.

No payload store follows this replacement header store.

### Empty input

An accepted zero count causes no source reads and no payload stores, but it does not bypass configuration processing.

If the intermediate extent-multiplier/header-multiplier product is defined, the calculated extent is zero. The destination header receives zero as its initial value and zero again as its final value. No destination element is examined, and normal completion returns zero.

Multiplication by the zero payload count does not eliminate the earlier signed multiplication or make an overflowing intermediate product valid.

## Side effects

### Access ordering

The represented ordering is significant:

- The payload multiplier is obtained before any source access or size rejection.
- Source reads and payload writes are interleaved at group boundaries.
- Examination-related configuration is accessed after payload construction.
- The initial header is stored before any examination read.
- The replacement header is stored after the examination ends.

The two header stores are distinct represented writes, including when both values are zero. This is not a guarantee that an unsynchronized observer can safely observe either store. Atomicity and concurrency rules are unavailable.

All configuration needed for the extent and both header calculations has been accessed before the initial header store. No later configuration access is represented.

### Overlap and aliasing

There is no source snapshot or overlap check.

A payload store can change source bytes that have not yet been consumed if the regions overlap. In that case, the result depends on the interleaved accesses, not merely on the source contents at entry.

Destination stores may also affect configuration when storage aliases. Payload stores precede examination-related configuration accesses and can therefore affect their values. The captured payload multiplier is not refreshed.

Any such overlapping execution remains conditional on valid memory access and the applicable memory model. The absence of an explicit overlap check does not establish that every overlapping arrangement is valid.

### Failure behavior

No validation of capacity, alignment, source readability, configuration range, or arithmetic suitability is visible.

No transactional protection or rollback is provided. There is no established guarantee that earlier modifications are restored after a later fault or arithmetic failure. Nor is there an established recoverable return status for such failures.

## Dependencies

No callable dependency is visible. There are no additional opaque function IDs to enumerate and no basis for assumptions about a callee preserving configuration or releasing storage.

The external dependencies are:

- the five configuration roles;
- readable source bytes at the calculated addresses;
- writable destination header and payload elements;
- readable, usable destination values throughout the actual examination range;
- target rules for integer-to-address conversion and address-based bookkeeping;
- target rules for unsigned-to-signed conversion, signed arithmetic, alignment, and memory access.

These are requirements or unresolved semantics, not validations performed by F-3e86f9db03cd.

## Acceptance examples

These examples are conditional deductions from static evidence, not runtime observations. Unless an example addresses a boundary explicitly, assume stable configuration, valid accesses, the described four-byte grouping, and defined arithmetic and conversions.

### Rejection boundary

A count of 257 obtains the payload multiplier and returns the configured rejection result without accessing source or destination storage.

The same applies to \(2^{32}-1\). A configured result of zero remains a rejection result; the return value alone need not distinguish rejection from successful completion.

A count of 256 is accepted and generates exactly 64 payload elements if payload construction completes.

### Empty input

For count zero and a representable intermediate extent-factor product:

- no source byte is read;
- no payload element is written;
- destination index zero receives zero twice;
- no destination element is examined;
- the return value is zero.

Writable header storage is still necessary.

### Complete and incomplete groups

For ascending source bytes `01 02 03 04 05 06` and payload multiplier one:

| Destination index | Numeric payload |
|---|---|
| One | `0x03040506` |
| Two | `0x00000102` |

With unit header, extent, and examination multipliers, the initial header is two, examination stops at index two, and the final header is two.

### Removing a zero high-order element

For ascending source bytes `00 00 00 00 01 02 03 04` and unit multipliers, element one receives `0x01020304` and element two receives zero.

The initial header is two. Examination reads index two and then index one. The final header becomes one. Element two remains stored as zero; it is not erased or physically removed.

### Unsigned payload multiplication

For four `FF` bytes and payload multiplier two, the final unsigned product is `0xFFFFFFFE`. This product does not itself constitute signed multiplication overflow.

If the target converts that unsigned result by preserving its bit pattern as a two’s-complement signed value, the stored value is negative two. That conclusion, including successful continuation through intermediate conversions, requires the additional target conversion rule.

### Negative extent

For one nonzero payload element, unit header and examination multipliers, and extent multiplier negative one, the extent is negative one.

Examination reads index one. If its product is defined and nonzero, the final header is negative one.

### Zero examination multiplier

For a valid finite range and examination multiplier zero, every examined product is zero. Examination reaches the lower boundary and the final header becomes zero, even if some destination values are nonzero.

### Examination beyond generated payload

Suppose one payload element is generated, the header multiplier is one, and the extent multiplier is two. The extent is two.

If preexisting element two has a defined nonzero product with the examination multiplier, examination stops there. The final header is two despite only one payload element having been generated.

### Numeric address wraparound

For source-address argument negative one and count two, the consumed numeric addresses are zero followed by \(2^{32}-1\).

These addresses follow from unsigned addition. Successful access still requires a suitable integer-to-address mapping and readable storage.

### Minimum signed extent

If the extent equals the minimum signed 32-bit value, a representable initial-header calculation and its store occur before the unresolved magnitude operation.

Normal examination and a final return value are not established without additional target semantics for that operation.

## Uncertainties

### documented_only: configuration domain and intent

Actual configuration values, allowed ranges, and encoding conventions are unavailable. Unit values in examples are not defaults. The numeric-representation interpretation remains inferred intent.

### documented_only: pointer and address semantics

The represented packing cadence depends on address-based bookkeeping with four-byte element scaling. Arithmetic originating from a null address is not established as valid under a strict pointer model.

The numeric payload layout is therefore not a portability guarantee. Integer-to-address mapping, destination pointer validity, and overlapping-access legality remain separate requirements.

### documented_only: unsigned-to-signed conversion

Unsigned payload multiplication is modulo \(2^{32}\). Conversion of a result above the signed maximum is unresolved, including conversions represented after intermediate bytes. No unconditional signed result or successful continuation is promised for those cases.

### documented_only: signed arithmetic

No general wrapping rule is established for the extent products, header products, examination products, or negation of the minimum signed value.

In particular, the examination must not be simplified unconditionally to a nonzero-element test. A zero multiplier changes the result, and an overflowing product has no established behavior. A separately specified wrapping model could introduce further zero products, but that model is not supplied here.

### documented_only: memory observation and faults

Alignment, destination byte order, access atomicity, synchronization, concurrent mutation, and fault delivery are not specified.

Separate represented configuration accesses do not imply permission for unsynchronized modification. The payload limit likewise does not establish that the destination is large enough for examination.

## Evidence

Static evidence establishes the unsigned rejection boundary, descending source offsets, unsigned packing and payload multiplication, signed result conversions, increasing payload stores, post-payload configuration accesses, grouped signed extent calculation, initial and replacement header stores, and descending examination controlled by signed products.

Unsigned source-address addition follows from the unsigned offset operand. Unsigned payload multiplication follows from the packed operand. Neither establishes pointer validity, out-of-range signed conversion behavior, or wrapping semantics for separate signed operations.

The represented four-byte grouping and its pointer-model qualification are retained together. The initial header’s position before magnitude formation is also significant to the unresolved minimum-signed-extent case.

No runtime tests have been performed. All acceptance examples are conditional static deductions.
