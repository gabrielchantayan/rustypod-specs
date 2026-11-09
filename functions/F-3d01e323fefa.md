<!-- Trusted provenance: F-3d01e323fefa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3d01e323fefa

## Purpose

F-3d01e323fefa operates on a destination byte region through three ordered stages:

1. An initial selector chooses between an ascending byte-transfer loop and a dependency call.
2. A descending sequence of indexed processing calls is followed by one unconditional processing call.
3. A separate selector may enable an in-place dependency call followed by table-based byte replacement.

Normal completion returns the 32-bit value zero. Dependency return values do not affect that result.

The evidence establishes the access patterns and call ordering, but not a standard algorithm, fixed buffer length, or fixed number of iterations. Descriptions such as “transfer” and “replacement” refer to visible operations, not to an established higher-level purpose.

This contract separates independently reduced behavior from unresolved arithmetic. In particular, the three loop-admission decisions remain documented-only limitations; their source expressions are not reproduced here.

## Interface

### Arguments

The function accepts twelve arguments in this order:

| Position | Meaning established by visible use |
|---|---|
| 1 | Source address for the initial byte-transfer branch; otherwise passed to F-d2332752ccd0. |
| 2 | Destination address, used for direct byte accesses and passed to every dependency. |
| 3 | A 32-bit position input used to initialize the descending processing stage. It is not established to be a length. |
| 4 | Address of the 32-bit initial selector. |
| 5 | Address of the 32-bit final selector. |
| 6 | Opaque 32-bit value passed to F-381f168d3859. |
| 7 | Opaque 32-bit value used by the initial F-d2332752ccd0 call. |
| 8 | Opaque 32-bit value passed to F-381f168d3859 and F-a2ac4e171e52. |
| 9 | Opaque 32-bit value passed to F-381f168d3859. |
| 10 | Opaque 32-bit value passed to F-381f168d3859. |
| 11 | Opaque 32-bit value used by the final F-d2332752ccd0 call. |
| 12 | Base address of the final byte-lookup region. |

Addresses and arithmetic operands are represented as 32-bit words. Direct payload accesses are one byte wide. Selector and global-control accesses are 32 bits wide.

No visible validation checks nullness, alignment, accessible extent, overlap, or suitable selector values.

### Arithmetic conventions

The behavioral reductions below use 32-bit wrapping arithmetic. Signed comparisons interpret the resulting word as two’s-complement. Arithmetic right shifts extend the sign bit.

For the lookup description:

- \(W(x)\) means reduction modulo \(2^{32}\).
- \(\operatorname{low}_3(x)\) means the unsigned value of the lowest three bits, from 0 through 7.
- Address addition describes the represented 32-bit computation, not a guarantee that a wrapped address is accessible.

These conventions describe the supplied representation. They do not establish the overflow or pointer rules of any implementation language.

### Global controls and retained state

The function reads eleven global control words. They have these distinct roles:

- An offset control, sampled at entry and sampled again before final selection.
- A shared boundary control, used live by some loops and captured for the middle stage.
- Two displacement controls used by the initial transfer decision.
- An initial-loop mixing control.
- A boundary control shared by the initial and final loops.
- A live middle-stage displacement control.
- A middle-stage mixing control captured before that stage.
- A live middle-stage boundary control.
- A live displacement added to positions passed to F-381f168d3859.
- A final-loop mixing control.

Their values are not supplied.

One retained boundary key is formed after the initial stage: it combines a read of the shared boundary control with an arithmetic-right-shift-by-29 of another read of that control, using bitwise exclusive-or. The two reads are not established to be an atomic snapshot. This key remains retained through the final stage.

### Result, ownership, and preconditions

If control reaches the return, the result is zero. There is no visible allocation, release, or ownership-transfer operation.

Dependency ownership effects are unknown. This function does not establish that arguments remain allocated, are not retained, or remain valid after a dependency returns.

Successful execution requires every reached memory access to be valid, every reached dependency to return normally, and every reached loop to terminate. The function does not enforce these conditions.

## Behavior

### Initial selection

The offset control is captured before evaluating the initial selector. Call this captured value the **entry anchor**.

Under coherent reads of the selector and relevant control words, selection is exactly:

- Selector zero: enter the direct byte-transfer branch.
- Any nonzero 32-bit selector: invoke the initial dependency branch.

This includes nonzero selectors whose signed interpretation is negative.

The selector appears twice in the selection expression. The simple zero/nonzero rule therefore does not specify behavior when those reads observe different values.

### Initial byte-transfer branch

The first candidate position is initialized using a fresh read of the offset control, not necessarily the entry anchor. Each transferred byte uses a common offset relative to source and destination.

Consequently:

- With an unchanged offset control between entry capture and position initialization, the first candidate offset is zero.
- If those reads differ, the first candidate offset is the entry anchor minus the later offset-control value, with 32-bit wrapping.

Before each byte transfer, an arithmetic admission decision is evaluated from the candidate position and the initial-loop control words. A negative signed result stops this loop without a source-byte or destination-byte access for that candidate. A nonnegative result admits one transfer.

The exact numeric admission rule has not been independently reduced into a usable behavioral constraint. It is documented-only, rather than replaced by an assumed length or range.

For each admitted candidate:

1. Read one byte from the source at the candidate offset.
2. Write that byte to the destination at the same offset.
3. Advance the candidate offset by one modulo \(2^{32}\).

This is an ordered byte transfer, not a source snapshot. If source and destination overlap, an earlier destination write can affect a later source read.

### Initial dependency branch

For a coherent nonzero initial selector, the function instead calls F-d2332752ccd0 with:

1. Argument 1: source address.
2. Argument 7.
3. Argument 2: destination address.

The direct transfer loop does not run in this branch. The dependency is not established to perform an equivalent transfer or process any particular number of bytes.

### Descending processing stage

After the selected initial branch finishes, the function captures the middle-stage boundary reference, mixing reference, and retained boundary key. Separate reads of the shared boundary control supply its captured uses; consistency between them is not guaranteed.

The initial processing position is argument 3 minus the entry anchor, with 32-bit wrapping.

Before each possible F-381f168d3859 call, a second arithmetic admission decision is evaluated. A nonnegative signed result ends the stage. A negative result admits one call.

Its precise numeric admission set is documented-only. Established dependencies include the current position, the captured middle-stage values, and live middle-stage displacement and boundary controls.

Each admitted call to F-381f168d3859 receives:

1. Destination address.
2. Current processing position plus the current call-displacement control, wrapping to 32 bits.
3. Argument 6.
4. Argument 10.
5. Argument 8.
6. Argument 9.

After the call returns, the processing position decreases by one modulo \(2^{32}\).

Captured values remain captured. Live controls are used again on later candidates. A dependency that changes a live control can therefore change subsequent decisions or call positions; preservation is not assumed.

After this loop terminates, F-a2ac4e171e52 is called exactly once with:

1. Destination address.
2. Argument 8.

This call occurs even when there were no indexed processing calls.

### Final selection

Only after F-a2ac4e171e52 returns does the function sample the offset control for final-stage use. Call the retained value the **final anchor**. A related preceding-word value is obtained separately for the later lookup calculation.

The following reduction assumes coherent reads of the final selector and offset control throughout selection, including these captures. Let the final selector be \(s\). The final anchor’s top two bits completely determine which selectors enable the final stage:

| Final anchor, as an unsigned word | Arithmetic shift right by 30 | Enabled selectors, exhaustively |
|---|---:|---|
| `0x00000000`–`0x3FFFFFFF` | 0 | None |
| `0x40000000`–`0x7FFFFFFF` | 1 | `0x00000001` |
| `0x80000000`–`0xBFFFFFFF` | −2 | `0x7FFFFFFF`, `0x80000000` |
| `0xC0000000`–`0xFFFFFFFF` | −1 | `0x80000000` |

Every selector not listed for its row skips the final stage.

These are wrapped-arithmetic boundary cases, not a positive-selector test. In particular, both listed selectors in the third row are enabled.

If selection fails, the function returns zero without the final dependency call or lookup loop.

### Final dependency and replacement loop

When enabled, F-d2332752ccd0 is called with:

1. Destination address.
2. Argument 11.
3. The same destination address.

Its first and third arguments alias exactly.

The replacement position is initialized from the retained final anchor after the call. Regardless of changes the dependency makes to the live offset control, the first candidate destination offset is:

- Zero if the retained final anchor is nonnegative.
- `0xFFFFFFFF` if it is negative.

The latter is a wrapped offset one byte before the destination base; accessible storage there is not guaranteed.

Before each replacement, a third arithmetic admission decision is evaluated. A negative signed result ends the loop without a destination-byte or lookup-byte access for that candidate. A nonnegative result admits a replacement.

Its exact numeric admission rule remains documented-only. It uses the retained boundary key and live final-stage controls, so dependency effects and aliases can affect it.

For an admitted candidate, the direct observable order is:

1. Read the current destination byte.
2. Read one lookup byte selected using that byte and the candidate’s lookup page.
3. Write the lookup byte back to the same destination location.

Successive candidate offsets increase by one modulo \(2^{32}\).

### Independently reduced lookup behavior

Under coherent final-anchor captures, the complicated byte arithmetic reduces to a paged lookup: the original destination byte is the column index, without a net byte-value rotation.

Let:

- \(\alpha\) be the retained final anchor.
- \(\kappa\) be the retained boundary key.
- \(\delta\) be the current destination offset.
- \(b\) be the destination byte just read, interpreted from 0 through 255.
- \(\theta=W(\alpha-\kappa)\).

The selected page number is:

\[
p=W\!\left(\theta+8-\operatorname{low}_3(\theta-\delta)\right).
\]

The lookup displacement is:

\[
W(256p+b).
\]

This is a behavioral reduction, not a transcription of the original lookup calculation. Its consequences are:

- The lookup displacement’s low eight bits equal the original destination byte.
- The selected page depends on the destination offset modulo eight, not on the byte value.
- Eight consecutive candidates traverse eight page numbers and then repeat that page pattern.
- Only the low 24 bits of the page number affect the 32-bit displacement.
- All possible lookup offsets for that invocation lie in eight 256-byte pages, interpreted modulo the represented address width.

This does not establish that the caller supplied those pages or that a single 256-byte lookup region is sufficient.

If the separately captured final-anchor-related values are inconsistent, this reduction is not guaranteed. That case remains documented-only.

## Side effects

Direct effects consist of selector and global-control reads, initial source reads and destination writes, final destination and lookup reads, final destination writes, and dependency calls.

The visible body does not directly assign selector or global-control storage. Nevertheless, writes through supplied addresses may alias that storage. Dependencies may also modify it.

No disjointness requirement is checked. Relevant consequences include:

- Initial destination writes can alter subsequent source reads.
- Destination writes can change later selector or control reads.
- Replacement writes can alter later lookup bytes when lookup and destination storage overlap.

There is no rollback. A fault or nonreturning dependency leaves earlier completed effects in place.

No finite termination bound is established. Candidate positions wrap, while admission depends on unknown controls.

## Dependencies

| Opaque ID | Established invocation contract |
|---|---|
| F-d2332752ccd0 | Optional initial call with source, argument 7, destination; independently optional final call with destination, argument 11, destination. Return value ignored. |
| F-381f168d3859 | One call per admitted descending-stage candidate, with the six arguments specified above. Return value ignored. |
| F-a2ac4e171e52 | Exactly one call after the descending loop terminates and before final selection. Receives destination and argument 8. Return value ignored. |

No dependency is established to preserve globals, preserve buffers, avoid retaining pointers, or preserve ownership.

## Acceptance examples

These are conditional obligations derived statically, not executed tests.

1. **Initial zero selector.** With coherent reads, selector zero chooses the direct transfer branch and omits the initial F-d2332752ccd0 call.

2. **Initial negative selector.** With coherent reads, selector `0x80000000` chooses F-d2332752ccd0 with arguments 1, 7, and 2.

3. **Immediate transfer rejection.** If the first transfer admission result is negative, that branch performs no direct source-byte read or destination-byte write.

4. **Overlapping transfer.** If two candidates are admitted, the second source read occurs after the first destination write and can observe its effect.

5. **Empty descending stage.** If its first admission result is nonnegative, no F-381f168d3859 call occurs. F-a2ac4e171e52 still occurs once.

6. **Changed call displacement.** If an indexed dependency call changes the live call-displacement control, a later admitted call uses the subsequently read value.

7. **Final zero selector.** Under coherent reads, final selector zero never enables the final stage.

8. **Positive final-anchor case.** Final anchor `0x40000000` and selector one enable the final stage.

9. **First omitted boundary.** Final anchor `0x80000000` and selector `0x7FFFFFFF` enable it. The two reduced signed components are `0x7FFFFFFE` and `0x7FFFFFFF`, both nonnegative.

10. **Second omitted boundary.** Final anchor `0xC0000000` and selector `0x80000000` enable it. Both reduced components equal `0x7FFFFFFF`.

11. **Adjacent disabled boundary.** Final anchor `0xC0000000` and selector `0x7FFFFFFF` do not enable it.

12. **Immediate replacement rejection.** If enabled selection is followed by a negative first replacement admission result, the final F-d2332752ccd0 call still occurs, but no direct replacement access follows.

13. **Lookup column boundaries.** Under the lookup reduction’s assumptions, original bytes 12, 13, and 255 select columns 12, 13, and 255 respectively within the selected page.

## Uncertainties

### Established static evidence

Established behavior includes argument order, access widths, dependency ordering, candidate directions, retained versus live controls, coherent-read selector cases, the reduced lookup behavior under its stated assumptions, and zero return on normal completion.

### Inferred intent

The initial direct branch resembles a byte copy. The final loop resembles page-dependent substitution. Neither observation establishes a named algorithm or intended application.

### Documented-only unknowns

- Exact numeric admission sets for the three loops; this contract does not fully specify iteration counts for arbitrary global configurations.
- Actual control values and intended buffer extents.
- Behavior with inconsistent selector or control reads.
- Lookup reduction when final-anchor-related captures disagree.
- Exact ordering among reads inside compound arithmetic expressions.
- Atomicity, concurrency, alignment, faults, and platform address-wrap behavior.
- Dependency semantics, ownership effects, and memory-validity guarantees.
- Whether intended inputs always terminate.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. No implementation access, dependency inspection, runtime tests, or execution traces were available.

The exhaustive final-selector table and paged-lookup description are mathematical deductions under their stated 32-bit and coherent-read assumptions. They are not runtime observations. Unresolved loop decisions are explicitly retained as limitations rather than reconstructed as source-shaped equations or guessed buffer lengths.
