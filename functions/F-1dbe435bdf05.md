<!-- Trusted provenance: F-1dbe435bdf05; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1dbe435bdf05

## Purpose

F-1dbe435bdf05 rescales two adjacent 32-bit integer representations in place using one common power-of-two shift. It returns a 32-bit unsigned encoding of the selected shift direction and count.

**Inferred intent:** bring a two-component quantity into a common magnitude range without independently scaling its components.

**Established behavior:** when neither input represents −2³¹, the larger original absolute magnitude determines the shift. For a nonzero pair, the larger resulting absolute magnitude satisfies

\[
2^{27}\leq M_{\mathrm{output}}\leq 2^{28}.
\]

The upper bound is strict when the selected operation is a left shift or a zero shift. After a right shift, the upper bound can be attained: arithmetic right shifting rounds negative values toward negative infinity, potentially increasing the magnitude relative to truncation toward zero.

If either input represents −2³¹, shift selection instead takes an exceptional path: both representations are shifted left by 27 bits, with 32-bit truncation. This path does not provide the ordinary magnitude guarantee.

The all-zero pair remains zero and returns 27.

## Interface

### Input and output storage

The function accepts a pointer to a pair of consecutive 32-bit unsigned elements.

| Element | Relative byte offset | Role |
|---|---:|---|
| First component | 0 | First input representation and first output destination |
| Second component | 4 | Second input representation and second output destination |

The caller must provide storage valid for reading and writing both elements. No null check, size check, alignment adjustment, or error-reporting path is visible.

The element type is unsigned, but sign tests and right shifts use a signed interpretation of the same 32-bit representations:

- `0x00000000` through `0x7fffffff` represent 0 through 2³¹−1.
- `0x80000000` through `0xffffffff` represent −2³¹ through −1.
- Unsigned negation and unsigned left shifts produce results modulo 2³².

These interpretations describe the represented 32-bit machine behavior. They do not establish portability across arbitrary language implementations.

The two results replace the original elements. No separately allocated result or additional output buffer is involved. The caller retains ownership of the supplied storage.

### Return value

The return value is unsigned 32-bit and describes the selected scale operation:

| Operation | Return encoding |
|---|---|
| Left shift by \(L\) | \(L\) |
| Zero shift | 0 |
| Right shift by \(R\) | \(2^{32}-R\) |

The complete set of possible return values is:

- 0 through 27;
- `0xffffffff`, denoting a right shift by 1;
- `0xfffffffe`, denoting a right shift by 2;
- `0xfffffffd`, denoting a right shift by 3.

The return value is not a success or failure indicator. In particular, zero inputs and signed-minimum inputs do not produce an error status.

## Behavior

### Observable access order

The source expresses the following order of accesses to caller-provided storage:

1. Read the first element.
2. Read the second element.
3. Write the first result.
4. Write the second result.
5. Return the shift encoding.

Both original elements are captured before either element is overwritten. Output calculations therefore use the original pair, not a mixture of original and updated components.

A store is expressed for each element even if its resulting representation equals its original representation.

This is the source-level access order. It is not an atomicity guarantee, a concurrency protocol, or a guarantee about externally observable device-memory transactions.

### Magnitudes used for selection

Each captured representation is classified by its signed interpretation. A nonnegative representation is used directly. A negative representation undergoes unsigned 32-bit negation.

For all signed values except −2³¹, this yields the ordinary absolute magnitude. For −2³¹, negation modulo 2³² produces `0x80000000` again. That representation remains negative when interpreted as signed.

The two magnitude representations are combined with bitwise OR. For ordinary inputs—those excluding −2³¹—the highest set bit of the combination is the highest set bit of the larger absolute magnitude. The combination need not equal either magnitude numerically.

Shift selection uses signed comparisons. Consequently, a combined representation with its high bit set behaves differently from an ordinary nonnegative magnitude.

### Ordinary shift selection

Assume neither component is −2³¹. Let \(M\) be the larger absolute input magnitude.

For \(M>0\), define

\[
k=\lfloor\log_2 M\rfloor.
\]

Thus, \(k\) is the zero-based position of the highest set bit. For the all-zero pair, selection behaves as though \(k=0\).

| Input magnitude class | Selected operation | Return |
|---|---|---|
| \(0\leq k\leq26\) | Left shift both components by \(27-k\) | \(27-k\) |
| \(k=27\) | Zero shift of both components | 0 |
| \(28\leq k\leq30\) | Arithmetic right shift both components by \(k-27\) | \(2^{32}-(k-27)\) |

For a nonzero ordinary pair, left-shift and zero-shift results satisfy

\[
2^{27}\leq M_{\mathrm{output}}<2^{28}.
\]

Right-shift results satisfy the slightly wider bound

\[
2^{27}\leq M_{\mathrm{output}}\leq2^{28}.
\]

The inclusive upper endpoint is necessary. For example, −536,870,911 shifted arithmetically right by 1 becomes −268,435,456, whose absolute magnitude is exactly 2²⁸.

A right-shift result with magnitude 2²⁸ must be negative. Positive results remain strictly below that magnitude. Thus, ordinary inputs can leave the half-open interval ending at 2²⁸ even without any signed-minimum input.

### Left-shift behavior

Left shifts operate on unsigned 32-bit representations. Any bits beyond the result width are discarded.

For ordinary inputs, the selected left shift keeps the corresponding signed mathematical results representable. Each component is multiplied exactly by the selected power of two; no information is lost.

The zero-shift case still writes both elements.

The all-zero pair selects a left shift by 27. Both stored results remain zero, but the return is 27 rather than zero. The return reports the selected operation, not whether the stored values visibly changed.

### Right-shift behavior

Right shifts use the signed interpretation of each original component. Under the represented arithmetic-right-shift semantics, shifting by \(R\) produces

\[
\left\lfloor\frac{x}{2^R}\right\rfloor.
\]

Consequences include:

- A positive component discards low-order bits.
- A negative component with discarded nonzero bits rounds toward negative infinity, not toward zero.
- A small positive component can become zero.
- A small negative component can become −1.
- A negative component’s resulting absolute magnitude can be one greater than the corresponding quotient truncated toward zero.

For a negative component, the output magnitude is the original absolute magnitude divided by \(2^R\), rounded upward. This is the reason the ordinary output bound permits equality at 2²⁸.

Both components receive the same shift. A much smaller component is not independently protected from precision loss. The return value does not generally permit exact reconstruction after a right shift.

### Signed-minimum behavior

If either original component is `0x80000000`, its magnitude representation retains the high bit. The combined magnitude representation is then negative under the signed comparisons used for shift selection.

All positive-threshold comparisons fail, leaving the selected position at zero. The resulting operation is a left shift by 27 for both components.

Required consequences:

- The return value is 27.
- Every component equal to −2³¹ becomes zero.
- The other component also undergoes an unsigned 32-bit left shift by 27.
- Each stored representation equals its original unsigned representation multiplied by 2²⁷, modulo 2³².
- Only the original low five bits of each component can affect its result.

This path neither rejects nor saturates the input. It can discard nearly all information and can produce outputs outside the ordinary magnitude range. No error is signaled.

### Widths and arithmetic bounds

The supplied operations use 32-bit representations throughout the data and return calculations.

Selected left-shift counts are between 0 and 27. Selected right-shift counts are between 1 and 3. Although the shift operations mask their counts to eight bits, every reachable count is already within these bounds. No selected count reaches the 32-bit operand width.

The negative return indicator for a right shift is formed by unsigned negation. Its wraparound encoding is intentional in the observable contract; it is not an overflow error report.

## Side effects

The established externally relevant effects are the two stores to caller-provided storage and the returned shift encoding.

No allocations, deallocations, ownership transfers, global-state accesses, I/O operations, or function calls appear in the supplied function.

The two-element update is not expressed as a single atomic transaction. The source establishes no locking, memory barriers, or synchronization with another observer. It does not guarantee that another observer sees the pair change together.

Invalid storage has no defined recovery behavior in the supplied material. If an access fails, no error return, rollback, or restoration is established. In particular, a failure associated with the second store is not accompanied by any visible mechanism for undoing the first store.

## Dependencies

No function dependencies are visible.

F-1dbe435bdf05 performs its work using direct accesses to the supplied pair and integer operations. There are no dependency calls whose ownership behavior or global-state preservation must be assumed.

The numerical interpretation relies on the represented 32-bit signed behavior and arithmetic right shifts described above. These are execution-semantics assumptions, not opaque function dependencies.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They have not been executed. Each assumes valid two-element storage and the stated 32-bit signed interpretation with arithmetic right shifts.

Except where a hexadecimal representation is given, components are shown as signed decimal values.

| Input pair | Required output pair | Required return |
|---|---|---|
| (0, 0) | (0, 0) | 27 |
| (1, −1) | (134,217,728, −134,217,728) | 27 |
| (3, 5) | (100,663,296, 167,772,160) | 25 |
| (134,217,727, 0) | (268,435,454, 0) | 1 |
| (134,217,728, 0) | (134,217,728, 0) | 0 |
| (268,435,456, 1) | (134,217,728, 0) | `0xffffffff` |
| (268,435,457, −3) | (134,217,728, −2) | `0xffffffff` |
| (−536,870,911, 0) | (−268,435,456, 0) | `0xffffffff` |
| (−1,073,741,823, 0) | (−268,435,456, 0) | `0xfffffffe` |
| (2,147,483,647, −1) | (268,435,455, −1) | `0xfffffffd` |
| (−2,147,483,647, 0) | (−268,435,456, 0) | `0xfffffffd` |
| (−2,147,483,648, 1) | (0, 134,217,728) | 27 |
| (−2,147,483,648, −1) | (0, `0xf8000000`) | 27 |
| (−2,147,483,648, −2,147,483,648) | (0, 0) | 27 |

The examples distinguish the following requirements:

- **Selected operation versus visible change:** the zero pair returns 27 despite remaining zero.
- **Common scaling:** the pair (3, 5) receives one shift determined by the larger magnitude.
- **Left-shift boundary:** magnitudes just below 2²⁷ shift left once; magnitude 2²⁷ selects no shift.
- **Precision loss:** the small positive component in (268,435,456, 1) becomes zero.
- **Negative rounding:** −3 shifted right once becomes −2, not −1.
- **Inclusive ordinary upper bound:** the negative examples with right shifts of 1, 2, and 3 attain output magnitude 2²⁸ without containing −2³¹.
- **Sign-dependent endpoint:** the largest positive signed input produces 268,435,455 after shifting right by 3, while its negative counterpart produces −268,435,456.
- **Signed-minimum exception:** a component equal to −2³¹ becomes zero while forcing the same left shift on its companion.

For the output representation `0xf8000000`, the signed interpretation is −134,217,728.

## Uncertainties

- **documented_only — Caller domain:** The supplied material does not establish whether callers exclude zero pairs, signed-minimum values, or any other inputs. No such restriction is assumed.
- **documented_only — Application meaning:** Common-scale normalization is inferred intent. Units, fixed-point conventions, component meanings, and downstream use of the return value are unknown.
- **documented_only — Exceptional-path intent:** It is unresolved whether the signed-minimum behavior is intentional, tolerated, or an unhandled boundary case.
- **documented_only — Language portability:** Unsigned-to-signed conversion for high-bit representations and right shifting negative signed values can depend on implementation rules. The numerical results here describe the represented machine semantics, not all possible compilation environments.
- **documented_only — Physical ABI:** Pointer width, calling convention, byte order within elements, and exact alignment requirements are not established by this material. The relative element offsets are established.
- **documented_only — Concurrent and device access:** No guarantees are established for data races, volatile storage, memory-mapped devices, or atomic observation of the pair.
- **documented_only — Access failure:** Fault delivery, partial completion, and recovery are not specified.
- **documented_only — Runtime confirmation:** No runtime tests, instrumentation, or observed executions have been performed.

## Evidence

**Established static evidence:** The function reads two adjacent 32-bit elements before writing either, forms magnitude representations using signed sign tests and unsigned negation, combines those representations with bitwise OR, and selects a shared shift using signed threshold comparisons. It applies unsigned left shifts or signed right shifts, writes the first component before the second, and returns a 32-bit unsigned shift encoding. No dependency calls or external-state accesses appear.

**Derived consequences:** For ordinary nonzero inputs, the highest set-bit position determines the shift. Exact left-shift and zero-shift results have maximum absolute magnitude in \([2^{27},2^{28})\). Arithmetic-right-shift results have maximum absolute magnitude in \([2^{27},2^{28}]\), with the upper endpoint attainable by negative results because of rounding toward negative infinity. The return-value set, bounded shift counts, zero-pair behavior, and signed-minimum exception follow from the same operations.

**Inferred intent:** Normalize a pair toward a shared magnitude range while retaining a common scale. No stronger application-level purpose or caller invariant is established.
