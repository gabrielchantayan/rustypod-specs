<!-- Trusted provenance: F-f79d2cf69ee3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f79d2cf69ee3

## Purpose

F-f79d2cf69ee3 produces an unsigned integer word in caller-provided storage from the sign, exponent, and fraction fields of an input word.

Under the explicitly assumed 32-bit integer model, its established numerical behavior consists of:

- Zero for sufficiently small exponents.
- Truncating magnitude reduction for an intermediate exponent range.
- Magnitude expansion modulo \(2^{32}\) for larger, valid shift distances.
- Unsigned modular negation when the input sign is set.

For exponents requiring a leftward shift of 32 or more bits, neither the final result nor successful completion is established.

**Inferred intent:** the ordinary, non-overflowing results are consistent with converting a conventional binary32 encoding to an integer scaled by \(2^{16}\), truncating toward zero. This interpretation is not an additional input requirement or a promise of floating-point conversion semantics. In particular, the function does not provide established saturation, infinity handling, or NaN handling.

## Interface

### Arguments and result

| Element | Behavioral contract |
|---|---|
| Destination | An address identifying storage for one unsigned result word |
| Input | An unsigned word passed by value |
| Return | No returned value |
| Status | No separate success, overflow, or error indicator |

The input is interpreted as an integer bit pattern. No input buffer, floating-point object, or byte sequence is read through a separate address.

The destination’s contents on entry do not contribute to the ordinary result: the represented access sequence overwrites them before any destination read.

### Integer model and field layout

All concrete widths, ranges, and numerical examples in this contract are conditional on unsigned words and signed integers both being 32 bits. The supplied material does not independently establish the target ABI or native integer widths. Other integer models are **documented_only**.

Under the 32-bit model, the input fields are:

| Field | Bit positions | Range |
|---|---|---|
| Sign | 31 | Clear or set |
| Exponent | 30–23 | 0–255 |
| Fraction | 22–0 | 0–8,388,607 |

These are positions within an integer value. They do not specify byte order in memory.

For describing the result, define the **significand magnitude** as \(2^{23}\) plus the fraction. It therefore ranges from 8,388,608 through 16,777,215, inclusive.

The leading \(2^{23}\) contribution is unconditional. Neither exponent zero nor exponent 255 removes it. Although exponent bit 23 participates in the source expression forming this magnitude, its value cannot affect the magnitude because that position is forced to one.

### Storage requirements and ownership

Successful execution within the established numerical domain requires destination storage that is live, writable, sufficiently large for one word, and validly aligned for the execution environment. A set input sign also entails a destination read, so readable storage is required for that path.

There is no address-validation or recovery facility. Null, inaccessible, misaligned, undersized, or otherwise invalid storage has no established successful behavior or recoverable error result.

The caller retains ownership of the destination. The body contains no allocation, deallocation, external call, or mechanism that retains the address beyond the invocation.

## Behavior

### Numerical result before sign application

The following table specifies the magnitude for the established 32-bit domain. “Discard the remainder” means integer truncation of a nonnegative quantity.

| Exponent | Magnitude |
|---:|---|
| 0–110 | Zero |
| 111–133 | Significand magnitude divided by \(2^{134-\text{exponent}}\), discarding the remainder |
| 134 | Significand magnitude |
| 135–165 | Significand magnitude multiplied by \(2^{\text{exponent}-134}\), reduced modulo \(2^{32}\) |
| 166–255 | **documented_only**: no established result or completion guarantee |

This is a numerical contract, not a rounding-to-nearest conversion. For example, a positive magnitude of one and one-half output units becomes one.

The zero region has two distinct justifications in the static evidence:

- Exponents 0–102 imply rightward distances of at least 32. The function explicitly selects zero instead of performing such a right shift.
- Exponents 103–110 imply distances of 31 through 24. These are valid 32-bit distances, but they remove every potentially set bit of the 24-bit significand magnitude.

For exponents 111–134, the rightward distance ranges from 23 down to zero. The operand used for that reduction is positive and representable as a signed 32-bit integer, so the result does not depend on how a target handles right shifts of negative signed values.

For exponents 135–165, the leftward distance ranges from 1 through 31. The operand is unsigned; bits beyond the word width are discarded. This is modular arithmetic, not saturation or an error condition.

Consequences include:

- A nonzero significand magnitude can yield zero.
- A clear input sign can yield an output with bit 31 set.
- Increasing the exponent need not increase the resulting unsigned word.
- The output is not constrained to the positive signed 32-bit range.

### Sign and the final destination value

A clear input sign requires no destination read. Within the established numerical domain, the final word is the magnitude specified above.

A set input sign entails a destination read after the scaled magnitude has been written. The final write contains the additive inverse of the value obtained by that read, modulo \(2^{32}\).

For valid ordinary storage with no intervening modification, this means:

- Zero remains zero.
- A nonzero magnitude becomes \(2^{32}\) minus that magnitude.
- `0x80000000` remains `0x80000000`.

Negation is unsigned and modular. It is not checked signed negation. Once magnitude scaling has wrapped, the set input sign does not ensure that the final word would be negative under a later signed interpretation.

The read-dependent wording is material. The represented operation negates the destination value actually read, rather than a separately guaranteed retained magnitude. Whether a differing read value could arise in a valid execution is unresolved: the supplied material establishes no concurrency, device-memory, or external-interference contract.

### Oversized leftward scaling

Exponents 166–255 correspond to leftward distances of 32–121. Those distances are outside the valid range for a 32-bit operand.

The distance is restricted to eight bits in the supplied body. Every distance in this interval already fits in eight bits; that restriction therefore does not establish reduction modulo 32.

For this region, the contract does not require any particular result, including zero, saturation, or a result obtained from a hardware-masked distance. It also does not require a fault, normal return, or subsequent sign application.

The represented destination sequence includes a significand-magnitude write before the oversized operation. That source-level ordering is not a portable guarantee that an execution containing an invalid-width shift exposes or retains that intermediate value.

### Interpretation as binary32

The input field positions coincide with the conventional binary32 layout. The function nevertheless consumes an unsigned integer word and contains no floating-point classification operation.

Under the stated model:

- Both conventional signed-zero encodings produce zero.
- Every conventional subnormal encoding produces zero.
- Conventional infinity and NaN encodings have exponent 255 and lie in the unresolved oversized-leftward region.
- NaN payloads receive no distinct treatment.
- Where modular overflow is absent, magnitude truncation followed by sign application is consistent with truncation toward zero.

The body expresses no access to floating-point exception state. This does not establish a general exception or fault guarantee for invalid storage or unresolved operations.

## Side effects

### Destination accesses

For executions in the established numerical domain, the represented destination access order is:

| Input sign | Destination effects, in order |
|---|---|
| Clear | Write the significand magnitude; write the scaled magnitude |
| Set | Write the significand magnitude; write the scaled magnitude; read the destination; write the modular inverse of the value read |

There is no destination read before the initial write.

These relationships remain present even when consecutive values are equal. At exponent 134, both magnitude writes have the same value. With a set sign and zero scaled magnitude, the destination read and final zero write are still represented.

These are source-level access relationships, not a guarantee of distinct physical memory transactions. The destination is not established as volatile or atomic. Optimizations, device behavior, and externally observable transaction counts are outside the supplied evidence.

### State, synchronization, and failures

The destination word is the only memory object explicitly addressed by the body. No external calls, explicit global-object accesses, allocations, or releases appear.

There is no explicit synchronization, atomic update, memory barrier, rollback, or failure report. The function does not authorize concurrent accesses that would otherwise be invalid.

A missing status result must not be interpreted as a completion guarantee. The supplied material does not establish fault delivery, recoverability, or retained intermediate destination contents after invalid storage access or an oversized shift.

## Dependencies

There are no external function dependencies in the supplied body of F-f79d2cf69ee3. No additional opaque function ID is required.

The behavioral conclusions depend on environmental properties rather than callees:

- The assumed 32-bit unsigned and signed integer widths.
- Integer conversions and valid-width shift semantics.
- Unsigned arithmetic modulo \(2^{32}\).
- Destination accessibility, alignment, and memory semantics.
- Optimization rules governing ordinary memory accesses.

No unknown callee is assumed to preserve globals or to retain, own, or release memory; no callee appears.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence, not runtime observations. Unless identified as unresolved, they assume the 32-bit model, valid ordinary destination storage, no intervening modification, and successful execution.

### Final-word examples

| Input word | Condition illustrated | Required final word |
|---|---|---|
| `0x00000000` | Clear sign, exponent zero | `0x00000000` |
| `0x80000000` | Set sign, exponent zero | `0x00000000` |
| `0x007FFFFF` | Maximum fraction at exponent zero | `0x00000000` |
| `0x37000000` | Exponent 110, last zero-only exponent | `0x00000000` |
| `0x37800000` | Exponent 111, minimum fraction | `0x00000001` |
| `0x37FFFFFF` | Exponent 111, maximum fraction | `0x00000001` |
| `0xB7C00000` | Negative one-and-one-half output units before truncation | `0xFFFFFFFF` |
| `0x3F800000` | Conventional positive-one encoding | `0x00010000` |
| `0xBFC00000` | Conventional negative-one-and-one-half encoding | `0xFFFE8000` |
| `0x43000000` | Exponent 134, unchanged significand magnitude | `0x00800000` |
| `0x47000000` | Exponent 142, magnitude reaches bit 31 | `0x80000000` |
| `0xC7000000` | Negation of the high-bit-only word | `0x80000000` |
| `0x47800000` | Exponent 143, magnitude wraps to zero | `0x00000000` |
| `0x47800001` | Exponent 143, a fraction contribution survives | `0x00000200` |
| `0x52800001` | Exponent 165, leftward distance 31 | `0x80000000` |

The exponent-111 examples establish that the entire fraction range produces magnitude one at that boundary.

The negative fractional example requires negative one under a conventional signed interpretation, rather than negative two. It therefore distinguishes truncation toward zero from rounding toward negative infinity.

The two exponent-143 examples distinguish modular scaling from saturation: a one-unit fraction change changes the output from zero to 512.

### Access-order examples

For input `0x3F800000`, the represented destination writes contain `0x00800000` and then `0x00010000`. There is no explicit destination read.

For input `0xBFC00000`, the represented writes contain `0x00C00000` and then `0x00018000`. A destination read follows. If that read obtains `0x00018000`, the final write contains `0xFFFE8000`.

For input `0x80000000`, the represented writes contain `0x00800000` and then zero. The set sign still entails a read and a final write; if the read obtains zero, the final write is zero.

These examples specify represented ordering, not distinct bus transactions or permission for another execution context to observe intermediate writes.

### Unresolved cases

For input `0x53000000`, the exponent is 166 and the leftward distance is 32. Neither a final value nor normal completion is an acceptance requirement; both remain **documented_only**.

For inputs `0x7F800000` and `0x7FC00000`, the exponent is 255 and the distance is 121. No infinity-specific result, NaN-specific result, zero result, or wrapped-distance result is established.

For these unresolved inputs, the earlier represented write does not establish a recoverable partial result.

## Uncertainties

- **Native widths — documented_only.** The concrete field ranges, boundaries, and examples depend on the stated 32-bit model. Independent ABI evidence is absent.
- **Oversized left shifts — documented_only.** Exponents 166–255 have no established final value, completion behavior, or recoverable intermediate state.
- **Physical memory visibility — documented_only.** Source-level ordering does not prove distinct transactions or external observation of intermediate values.
- **Storage faults — documented_only.** Exact alignment requirements, fault types, delivery, and recovery are not established.
- **Externally changing storage — documented_only.** No valid interference or synchronization model is supplied. The ordinary numerical sign rule assumes the destination read obtains the preceding scaled magnitude.
- **Byte order — documented_only.** Integer bit positions are established conditionally; no memory byte layout is specified.
- **Application meaning — inferred intent.** Scaling by \(2^{16}\) is consistent with fixed-point conversion for suitable conventional binary32 inputs. Caller expectations, numeric type conventions, and overflow policy are not established.

## Evidence

The supplied function body is the sole static evidence. The previous draft and reviewer feedback are not independent evidence of behavior.

The body supports the conditional field layout, unconditional leading significand bit, scaling boundary at exponent 134, guarded rightward reduction, unsigned leftward scaling, destination access order, and sign-dependent destination read.

The numerical bounds follow from the represented operand widths and ranges:

- The significand magnitude occupies 24 bits, so rightward distances of at least 24 eliminate all set bits.
- The rightward path explicitly avoids distances of 32 or more under the stated model.
- Valid unsigned left shifts retain only the low 32 bits.
- The largest established leftward distance is 31.
- Restricting a distance to eight bits does not make distances 32–121 valid for a 32-bit operand.
- Final negation operates on an unsigned destination value and therefore has modular behavior.

These observations do not establish physical transaction visibility, target-specific oversized-shift behavior, or application-level floating-point semantics.

No runtime tests have been performed. No independent caller, ABI, target-instruction, or physical-memory evidence was supplied.
