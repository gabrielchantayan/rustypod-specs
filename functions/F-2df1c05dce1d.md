<!-- Trusted provenance: F-2df1c05dce1d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2df1c05dce1d

## Purpose

F-2df1c05dce1d conditionally transforms a byte range described by a two-word descriptor and writes the transformed bytes to a caller-supplied address.

The established behavior combines:

- Validation of a descriptor pointer, a destination-address condition, and an unsigned length relationship.
- Optional publication of the length used for validation.
- A fresh calculation of the number of bytes to transform.
- Direction-sensitive copying with a per-byte transformation.
- A 32-bit return value: zero after the transformation path, or a value read from a global failure-value location after rejection.

The apparent intent is an overlap-aware transformation of the portion remaining after an offset into a descriptor-backed region. That intent is inferred. In particular, the validation length and the transformation length are calculated separately and need not agree when the optional output aliases relevant state.

## Interface

### Inputs

The interface has six arguments, in this order:

| Argument | Width and interpretation |
|---|---|
| Descriptor | Pointer to a descriptor containing at least two consecutive 32-bit words. A null pointer is explicitly rejected. |
| Requested length | Unsigned 32-bit value used in validation. It does not directly limit the transformation loop. |
| Length mode | Signed 32-bit value tested only for zero versus nonzero. |
| Offset | Signed 32-bit value participating in length arithmetic and source-address formation. Negative values are not explicitly rejected. |
| Destination address | Unsigned 32-bit address value. Zero receives special treatment during initial validation. |
| Reported-length output | Optional pointer to writable 32-bit storage. Null suppresses the report. |

The descriptor layout is:

| Byte offset | Meaning |
|---|---|
| 0 | A 32-bit word used as the source base address. |
| 4 | A 32-bit signed quantity multiplied by a global multiplier when calculating lengths. |

The source does not establish the meaning of the descriptor’s second word beyond its arithmetic use. It need not be a byte count by itself.

### Global inputs

Two global storage locations affect behavior:

- A multiplier used in the length calculations.
- A failure value returned when validation rejects the request.

The multiplier participates in signed-word arithmetic in the supplied representation. The return value and the lengths used in comparisons are 32-bit quantities.

The function samples the multiplier at entry and uses that captured value later when calculating the transformation length. The validation expression also refers to the global multiplier. These are distinct uses in the supplied representation; the source does not establish synchronization or a guarantee that both observe the same value.

### Result

The function returns a 32-bit value:

- Zero when validation passes and execution reaches the end of the transformation path.
- The current global failure value when validation fails.

The failure value is not established to be nonzero. Consequently, a return value of zero alone does not prove that validation passed unless an external contract constrains that global.

There is no explicit result for invalid memory access, arithmetic exceptions imposed by the execution environment, or other faults.

### Memory preconditions

For a defined successful execution, all memory actually accessed must be valid for its use:

- Descriptor words that are read must be readable.
- A nonnull reported-length output must permit a 32-bit write.
- Every source byte visited by the transformation must be readable.
- Every destination byte visited must be writable.

No explicit checks establish these properties. Neither the descriptor nor the destination is validated against an allocation size.

The required pointer alignment, address-space rules, and behavior of unaligned 32-bit accesses are not established.

## Behavior

### 1. Initial global sampling and eligibility

The function captures the global multiplier before deciding whether the descriptor or destination conditions permit further work.

The initial conditions require both:

1. A nonnull descriptor.
2. Either a requested length of zero or a nonzero destination address.

Thus, a zero destination address is rejected when the requested length is nonzero. A zero destination address is not, by itself, rejected when the requested length is zero.

If these conditions fail, the function does not access descriptor fields or write the reported-length output. It returns the global failure value.

### 2. Validation-length calculation

When initially eligible, the function reads the descriptor’s second word and calculates a validation length from:

- The global multiplier used by the validation expression.
- The descriptor’s second word.
- The offset.

The arithmetic relationship is:

**validation length = multiplier × descriptor quantity − offset**

The resulting value is held and compared as an unsigned 32-bit quantity.

At the 32-bit machine-arithmetic interpretation, the stored result is the low 32 bits of that expression. However, the supplied representation contains signed intermediate arithmetic. Behavior at signed-overflow boundaries is not established as a portable language-level contract; see **Uncertainties**.

### 3. Length acceptance

Validation succeeds exactly when both of the following unsigned comparisons hold:

- The validation length is no greater than the requested length.
- Either the length mode is nonzero, or the requested length is no greater than the validation length.

Consequently:

- **Zero length mode:** the validation length must equal the requested length.
- **Nonzero length mode:** the validation length may be any unsigned value no greater than the requested length.

Every nonzero mode value receives the same treatment, including negative values.

When this validation fails, the function returns the global failure value without writing the reported-length output or any destination bytes.

### 4. Optional length report

After validation succeeds, a nonnull reported-length output receives the validation length as one 32-bit write.

This write occurs before the function reads the source-base descriptor word for transformation and before it rereads the descriptor’s second word for the transformation-length calculation.

The reported length is therefore not necessarily the number of bytes subsequently transformed.

This ordering is observable when the output aliases:

- Either descriptor word.
- The multiplier’s storage.
- Source bytes that will later be read.
- Destination storage.
- Other writable state relevant to the call.

No non-aliasing requirement is present in the supplied body.

### 5. Transformation setup

After the optional report, the function:

1. Reads the descriptor’s first word as the source base.
2. Rereads the descriptor’s second word.
3. Calculates the transformation length using the multiplier captured at entry and the offset.

The arithmetic relationship is:

**transformation length = captured multiplier × reread descriptor quantity − offset**

The source starts at the captured source base plus the offset.

Both the source base and the transformation length are established after the optional output write. Subsequent byte writes do not cause these setup values to be reloaded.

Under stable memory, equal multiplier observations, and no relevant aliasing, the validation and transformation lengths agree. This equality is not unconditional.

### 6. Traversal direction

The function compares the destination address with the source start as unsigned 32-bit address values.

- If the destination address is strictly lower than the source start, bytes are processed in increasing index order.
- Otherwise, bytes are processed in decreasing index order.

Equality selects decreasing order.

For a transformation length of \(N\):

- Increasing order visits indices \(0\) through \(N-1\).
- Decreasing order visits indices \(N-1\) through \(0\).

If \(N\) is zero, no source or destination byte accesses occur. The earlier descriptor reads and optional length report still occur.

The direction rule resembles overlap-safe movement for ordinary, nonwrapping address ranges. It does not establish safety for address wrapping, invalid ranges, concurrent modification, or memory-mapped devices with unusual access semantics.

### 7. Per-byte transformation

At each visited index, the function reads one source byte and writes one destination byte at the matching relative index.

For an input byte value \(b\), the stored output byte is:

\[
(95b - 7) \bmod 256
\]

This describes the low-byte result. Whether the intermediate source byte is treated as signed or unsigned does not change that low-byte result.

Examples:

| Source byte | Destination byte |
|---|---|
| `00` | `F9` |
| `01` | `58` |
| `02` | `B7` |
| `80` | `79` |
| `FF` | `9A` |

Each source-byte read precedes its corresponding destination-byte write. The traversal direction determines the order of these read/write pairs.

When source and destination starts are equal, the transformation is in place, visited from the last byte to the first.

### 8. Completion

After all selected bytes are transformed, the function returns zero.

There is no explicit post-transformation validation, terminator write, length adjustment, or restoration of overwritten memory.

## Side effects

### Explicit writes

The function may perform:

1. One 32-bit reported-length write after validation succeeds.
2. One destination-byte write for each byte in the separately calculated transformation length.

No explicit writes occur on a rejected request.

A successful zero-length transformation can still perform the reported-length write.

### Aliasing and state changes

Aliasing is behaviorally significant rather than normalized away:

- Reporting into the descriptor’s first word changes the source base subsequently read.
- Reporting into the descriptor’s second word changes the quantity subsequently used to calculate the transformation length.
- Reporting into source storage can change bytes before transformation begins.
- Reporting into multiplier storage does not replace the multiplier already captured at entry for the transformation calculation.
- Destination writes can modify descriptor or global storage if their addresses overlap it, but the already captured transformation parameters are not recalculated.

The supplied body does not provide transactionality. If an access faults after the report or after some byte writes, earlier writes are not explicitly undone.

### Ownership

The body contains no allocation, deallocation, reference-count update, or explicit ownership transfer.

All accessed storage is supplied indirectly through arguments or global addresses. No release obligation is established by this function body.

## Dependencies

No calls to other functions are present in the supplied body. There are therefore no opaque callee IDs to list.

Observable behavior depends on:

- The two global storage locations.
- Descriptor memory.
- Optional reported-length storage.
- Source and destination memory.
- The execution environment’s integer, address, and memory-access semantics.

No unstated callee behavior is needed to explain the supplied body. Memory validity, synchronization, and ownership remain external preconditions rather than services provided here.

## Acceptance examples

These are conditional static examples, not runtime test results. Unless stated otherwise, assume valid accessible memory, stable globals and descriptor fields, no arithmetic overflow, no address wrapping, and a reported-length output that does not alias relevant state.

### Exact-length success

Suppose the multiplier is 4, the descriptor quantity is 3, and the offset is 2.

The calculated length is 10.

With requested length 10 and zero length mode:

- Validation passes.
- A nonnull report receives 10.
- Ten bytes beginning two bytes after the descriptor’s source base are transformed.
- The function returns zero after those writes.

### Nonzero mode accepts a larger request

Using the same calculated length of 10, requested length 12 and any nonzero mode pass validation.

The function transforms 10 bytes, not 12. The requested length is a validation bound, not the loop count.

### Zero mode rejects a larger request

With calculated length 10, requested length 12 and zero mode fail validation.

No report or destination write occurs. The return is the global failure value.

### Insufficient requested length

With calculated length 10 and requested length 9, validation fails for either mode.

### Zero destination address

A zero destination address with a nonzero requested length is rejected before descriptor fields are accessed.

With requested length zero, a valid descriptor, and calculated length zero, a zero destination address is accepted. Under the stability assumptions, no destination bytes are accessed.

This is not a general size-query contract: the function still reads the descriptor after validation, and aliasing can change the subsequent transformation length.

### Overlapping ranges

Suppose the destination starts one byte above the source start and the transformation length is four.

The function visits indices 3, 2, 1, and 0. Under ordinary contiguous-memory semantics, this avoids overwriting the next source byte before reading it.

If the destination starts one byte below the source start, indices are visited in increasing order instead.

### Report aliases the descriptor quantity

Suppose the entry multiplier and validation multiplier are both 2, the initial descriptor quantity is 3, and the offset is zero. Let the reported-length output point to the descriptor’s second word.

With requested length 6 and zero mode:

- Validation uses length 6.
- The report replaces the descriptor quantity with 6.
- The transformation calculation rereads that quantity and produces length 12.
- Twelve bytes are transformed if the relevant memory is valid.

This illustrates why a successful length comparison does not unconditionally bound destination writes by the requested length.

## Uncertainties

The following are **documented_only**:

- **Integer-overflow semantics.** The supplied representation mixes signed arithmetic with unsigned storage and comparison. Exact behavior when signed multiplication, subtraction, or source-base addition overflows requires execution-level evidence. Low-32-bit arithmetic describes the apparent machine interpretation, not a proven portable signed-arithmetic guarantee.
- **Address wrapping.** No range checks prevent source or destination address arithmetic from crossing the 32-bit address boundary. Validity and behavior of such accesses are not established.
- **Multiplier stability.** The captured multiplier and the validation expression’s multiplier are not guaranteed equal by the supplied body. Physical read timing and concurrent changes require additional evidence.
- **Descriptor stability.** Repeated descriptor reads need not observe identical values if memory changes. No synchronization mechanism is shown.
- **Fault behavior.** Invalid pointers, read-only destinations, unaligned words, and inaccessible address zero are not assigned explicit status returns.
- **Global failure-value meaning.** Its numeric value, stability, and relationship to other status conventions are unknown.
- **Descriptor semantics.** The multiplier’s units and the descriptor quantity’s intended interpretation are not established.
- **Overlap limits.** Direction selection supports the inferred overlap-aware intent only under suitable memory and nonwrapping-range assumptions.

## Evidence

**Established static evidence:** the supplied function body contains the initial multiplier capture; null and destination gating; unsigned length comparisons; an optional 32-bit report before source setup; repeated descriptor access; unsigned address-based direction selection; the byte transformation; and zero-versus-global return paths.

**Inferred intent:** the function appears to transform a remaining descriptor-backed byte range while selecting a direction suitable for overlapping ordinary memory ranges.

**Unknowns:** execution-level overflow behavior, actual global values, allocation boundaries, synchronization guarantees, and broader caller contracts are not supplied.

No runtime tests have been performed. The acceptance examples are deductions from the supplied static representation under their stated conditions.
