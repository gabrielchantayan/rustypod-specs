<!-- Trusted provenance: F-f0e2047addf0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f0e2047addf0

## Purpose

F-f0e2047addf0 consumes eight streams of byte samples and writes six output rows per processing group: one copied row and five rows produced by fixed-point weighted combinations.

The established behavior is byte-oriented. The routine does not establish a pixel format, channel interpretation, image dimensions, or allocation ownership. A resampling or interpolation role is inferred from the neighborhood access and weighted outputs, not established by the available evidence.

The contract below distinguishes directly visible behavior from dependency-defined behavior. In particular, the meanings of the stream initialization arguments and the arithmetic helpers remain unknown.

## Interface

The routine accepts two address-valued arguments represented by 32-bit quantities:

- **Context address.** The routine reads a 32-bit field at byte offset 4 and passes the context address to a dependency.
- **Descriptor address.** The routine accesses a sequence of 32-bit words and passes its address to a dependency.

There is no explicit return value or status result.

The descriptor fields directly used are:

| Word index | Byte offset | Visible use |
|---|---:|---|
| 0 | 0 | First destination base, before addition of a dependency-provided adjustment |
| 1 | 4 | Second destination base, before addition of the same adjustment |
| 3 | 12 | Factor in the processed byte-count calculation |
| 4 | 16 | Destination row stride, captured before the adjustment dependency is called |
| 8 | 32 | Other factor in the processed byte-count calculation |

These roles describe use, not a complete structure definition. The dependencies may access additional fields or mutate these fields.

The routine requires accessible storage for every read and write it actually performs. It contains no explicit null checks, range checks, destination-capacity checks, or pointer-validity checks.

### Arithmetic conventions

The visible computation uses:

- Unsigned 8-bit source samples.
- 32-bit address, counter, product, sum, and coefficient operations.
- A 64-bit result from F-d7be9cfb4000.
- Signed 32-bit comparisons for the outer-loop bound, byte-loop bound, and fixed-point saturation.
- A signed 16-bit interpretation of selected coefficient values in the first filtered output.

Addresses and arithmetic results must be understood at the visible 32-bit width. Unbounded mathematical arithmetic is not an equivalent contract where overflow changes a result.

For the formulas below, let **wrap32** retain the low 32 bits of a result and interpret those bits as a signed 32-bit value. This models the visible fixed-width computation; the supplied source representation alone does not establish portable language-level behavior for signed overflow.

## Behavior

### 1. Establish the processing bounds and destination state

The routine performs these actions in order:

1. Reads the 32-bit context field at byte offset 4.
2. Calls F-d7be9cfb4000 with that field and the integer argument 5.
3. Forms an outer-loop bound by multiplying the low 32 bits of the returned value by 5 at 32-bit width.
4. Captures descriptor word 4 as the destination row stride, denoted **S**.
5. Extracts a stream-selection value, **U**, by logically shifting the unsigned 64-bit dependency result right by 33 bits.
6. Calls F-c8a7bc15a71d with the descriptor address.
7. Reads descriptor word 0 and adds the returned adjustment to obtain destination base **A**.
8. Reads descriptor word 1 and adds that same adjustment to obtain destination base **B**.
9. Reads descriptor words 8 and 3 to calculate the processed byte count.

Consequently, descriptor words 0, 1, 3, and 8 are used after F-c8a7bc15a71d returns, whereas the row stride is captured before that call. The contract does not assume that dependency leaves the descriptor unchanged.

Let **P** be the signed interpretation of the low 32 bits of the product of descriptor words 8 and 3. The byte count **W** is P divided by 8 with truncation toward zero. The visible computation implements this through a sign-dependent bias followed by an arithmetic right shift.

A positive W controls the number of byte positions processed per group. A zero or negative W suppresses all source-byte reads and destination-byte writes, but does not suppress stream setup or, when the outer loop executes, the per-group dependency calls.

### 2. Initialize eight stream records

Eight separate local stream records are supplied to F-4f672b3add3a, in this order:

| Stream | Arguments after the record address |
|---|---|
| 0 | Context address, 2, U − 1 |
| 1 | Context address, 2, U |
| 2 | Context address, 2, U + 1 |
| 3 | Context address, 2, U + 2 |
| 4 | Context address, 2, U + 3 |
| 5 | Context address, 2, U + 4 |
| 6 | Context address, 2, U + 5 |
| 7 | Context address, 2, U + 6 |

The selection arithmetic is 32-bit unsigned arithmetic. In particular, U equal to zero makes the first selection value 4,294,967,295. The routine does not clamp or reject that value.

Each visible local record occupies four adjacent 32-bit words. The caller later uses the first word as mutable state and the fourth word as a stepping quantity. A dependency may also use the other words. Initialization of all eight records occurs even when the outer loop will execute zero times.

### 3. Process groups

The outer counter starts at zero. While it is signed-less-than the computed outer-loop bound, the routine processes one group and then increases the counter by 5.

For an ordinary nonnegative low-word result q from F-d7be9cfb4000, where multiplication and loop progression do not overflow, this produces q groups. That interpretation must not be extended to wrapped bounds: the actual bound is the 32-bit result of multiplication by 5, and the comparison is signed.

At the beginning of every group:

1. F-91678318bed2 is called once for each stream record, in stream order 0 through 7.
2. Each result is retained as a source-byte address for that group.
3. F-870c4fcf7295 is called with the 32-bit product of the outer counter and 6, followed by the integer argument 10.
4. Its returned value, denoted **R**, selects the destination row offset.

The destination row starting addresses for the group are:

| Output | Starting address |
|---|---|
| Copied row | A + S × R |
| First filtered row | B + S × R |
| Second filtered row | A + S × R + S |
| Third filtered row | B + S × R + S |
| Fourth filtered row | A + S × R + 2S |
| Fifth filtered row | B + S × R + 2S |

Address calculations use the visible 32-bit arithmetic. A negative stride is not explicitly prohibited. Distinctness or non-overlap of these rows is not checked.

The meaning of F-870c4fcf7295 is unknown. Its arguments do not by themselves justify describing R as a quotient or predicting consecutive destination groups.

### 4. Process each byte position

For byte positions from zero through W − 1, the routine obtains one unsigned sample from each stream. Denote these samples **x₀** through **x₇**.

The source-byte read order is:

1. x₀
2. x₁
3. x₂
4. x₇
5. x₄
6. x₅
7. x₃
8. x₆

All eight source bytes for the current position are read before any destination byte for that position is written.

There are nine coefficient storage locations, named **K₀** through **K₈** here solely to describe their roles. Their values and initialization are not supplied. Except where a signed 16-bit interpretation is specified below, their visible arithmetic contribution is at 32-bit width.

Define the fixed-point conversion **Q(z)**:

- Interpret the wrapped 32-bit accumulator as signed.
- Replace values below zero with zero.
- Replace values above 267,386,880 with 267,386,880.
- Shift the resulting nonnegative value right by 20 bits.
- Store the resulting byte.

The upper bound is 255 × 2²⁰. Thus Q produces bytes from 0 through 255, without an added rounding bias.

The six destination writes occur in the following order:

1. **Copied row:** x₁.
2. **First filtered row:** Q of the wrapped sum  
   K₃x₃ + K₂x₂ + signed16(K₀)x₀ + 64 × signed16(K₁)x₁.
3. **Second filtered row:** Q of the wrapped sum  
   K₅x₃ + K₄x₁ + 393,216x₂ − 131,072x₄.
4. **Third filtered row:** Q of the wrapped sum  
   K₇(x₃ + x₄) + K₆(x₂ + x₅).
5. **Fourth filtered row:** Q of the wrapped sum  
   K₄x₆ + K₅x₄ − 131,072x₃ + 393,216x₅.
6. **Fifth filtered row:** Q of the wrapped sum  
   K₂x₅ + K₃x₄ + 64 × K₁x₆ + 8 × K₈x₇.

In the first filtered row only, K₀ and K₁ use signed interpretations of their low 16 bits. The fifth filtered row uses K₁ at full 32-bit width. Replacing that distinction with a uniform coefficient width would change behavior.

The coefficient symbols in these formulas do not imply a single snapshot for the whole call or group. The visible access sequence is more specific:

- The copied-byte store occurs before the coefficient reads used for filtering.
- K₄ is captured before calculation of the first filtered row and reused for the second and fourth filtered rows at this byte position.
- K₅ is read for the second filtered row.
- K₅ is read again after that row is stored; this later value is retained for the fourth filtered row.
- K₇ and K₆ are read for the third filtered row.
- K₃ is captured after the third filtered row is stored and retained for the fifth filtered row.
- K₂, K₁, and K₈ are read for the fifth filtered row.

The supplied expression representation does not establish a complete ordering among coefficient reads within a single arithmetic expression.

### 5. Advance stream state

After all byte positions in a group have been handled, the routine updates the stream records in order 0 through 7.

For each record, it adds 20 times that record’s fourth 32-bit word to its first 32-bit word. The remaining record words are not directly written by this advancement.

These updates use the record contents present after F-91678318bed2 has returned. That dependency is not assumed to preserve the first or fourth word.

Advancement occurs after every executed group, including groups with W less than or equal to zero. No explicit stream-finalization or release call follows the loop.

## Side effects

Directly visible side effects are:

- Eight stream-initialization calls.
- Eight stream-fetch calls per executed group.
- One destination-row-selection call per executed group.
- Six byte stores per processed position per group.
- Per-group mutation of the first word in each local stream record.
- Reads from context, descriptor, source storage, and coefficient storage.

The routine itself does not explicitly allocate or release heap storage. This does not establish that its dependencies do neither.

Source and destination ownership are not transferred by any visible operation. Returned source addresses are read, not explicitly released. Whether they refer to borrowed, owned, transient, or internally managed storage is unknown.

Aliasing can affect results:

- The current position’s eight samples are captured before its first store.
- A destination write may nevertheless overwrite a source byte needed at a later position or group.
- Overlapping destination rows retain the specified write order; a later write may replace an earlier result.
- If destination storage aliases descriptor or coefficient storage, later reads can observe prior writes. No defensive snapshot of all such storage is made.

There is no explicit rollback. A dependency failure or invalid access can occur after earlier calls or destination writes have taken effect.

## Dependencies

| Opaque ID | Established call contract | Unknowns |
|---|---|---|
| F-d7be9cfb4000 | Receives the context field at offset 4 and 5; returns a 64-bit result used for the group bound and stream selections | Arithmetic meaning, errors, memory effects |
| F-c8a7bc15a71d | Receives the descriptor address; returns a 32-bit adjustment added to both destination bases | Adjustment meaning, descriptor mutation, ownership effects |
| F-4f672b3add3a | Receives a local record address, context address, 2, and a stream-selection value; called eight times | Record initialization rules, valid selection range, allocation behavior |
| F-91678318bed2 | Receives a stream record address; returns an address used for source-byte reads | Available byte extent, record mutation, returned-storage lifetime |
| F-870c4fcf7295 | Receives outer-counter × 6 and 10; returns a value used in destination addressing | Arithmetic meaning, failure behavior, other side effects |

No dependency is assumed to preserve global state, leave caller-visible objects unchanged, or acquire or release memory.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Zero outer bound.** If the low 32 bits returned by F-d7be9cfb4000 are zero, the adjustment call and all eight initialization calls still occur. No stream-fetch calls, row-selection calls, or destination-byte writes occur.

2. **Positive group count but no byte positions.** If the bound produces one group and W is zero, the routine still fetches all eight stream addresses, calls F-870c4fcf7295, and advances all eight records. It performs no source-byte reads or destination-byte writes.

3. **Byte-count truncation.** A signed 32-bit descriptor product of 15 gives W = 1. A product of −15 gives W = −1 and therefore no byte iterations. The latter is not rounded down to −2.

4. **Selection underflow.** If the extracted selection value is zero, initialization selections are 4,294,967,295, 0, 1, 2, 3, 4, 5, and 6. Acceptance requires the dependency to receive the wrapped first value, not a clamped zero.

5. **Copied output.** Given valid storage, one executed byte position, and no overlapping later store to the same address, the copied destination byte equals stream 1’s sample exactly. Coefficient values do not affect that copy.

6. **Fixed-point boundaries.** A wrapped signed accumulator below zero produces 0. An accumulator of 2²⁰ − 1 also produces 0. An accumulator of 2²⁰ produces 1. An accumulator at or above 267,386,880, while still signed-positive, produces 255.

7. **Zero samples.** If all eight samples at a position are zero and coefficient reads complete normally, all six written bytes are zero, regardless of coefficient values.

8. **Coefficient-width distinction.** If K₁ has the 32-bit value 65,536, its signed-16 contribution to the first filtered row is zero. Its contribution to the fifth filtered row is not narrowed and can be nonzero when x₆ is nonzero, subject to the remaining terms, wrapping, and saturation.

9. **Overlapping rows.** If all six calculated destination addresses for one position coincide, the final byte at that address is the fifth filtered result, assuming ordinary writable memory and no fault. The earlier copied and filtered values are overwritten in the established order.

## Uncertainties

The following remain **documented_only**:

- Coefficient values, initialization, mutability, and synchronization.
- Complete layouts of the context, descriptor, and stream records.
- Dependency success conditions and error signaling.
- Whether returned stream addresses remain valid across subsequent dependency calls.
- Intended handling of overflow, invalid addresses, negative dimensions, and negative strides.
- External requirements for source/destination non-overlap.
- Any concurrency guarantees.
- Whether the surrounding system ensures that the fourth word of every stream record is initialized before advancement.
- The semantic meaning of the extracted selection value and the row-selection result.

The likely interpolation purpose does not resolve these unknowns. In particular, no named interpolation kernel, image format, border policy, or coefficient normalization is established.

## Evidence

The evidence is the supplied static function representation and dependency-ID mapping.

Direct static evidence establishes the call order, descriptor accesses, eight-stream setup, byte-read order, six-write order, fixed-point formulas, saturation bounds, coefficient-width distinction, and stream-state advancement described above.

The interpretation as resampling or interpolation is inferred intent only. Dependency internals and runtime validity are not established.

No runtime tests have been performed.
