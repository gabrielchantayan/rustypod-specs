<!-- Trusted provenance: F-dc4f2265cb7d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc4f2265cb7d

## Purpose

F-dc4f2265cb7d processes two multiword operands, a modulus-like object, and a context value. Its main path constructs a local accumulator through multiplication-and-reduction passes, performs additional reduction passes when required by the selected counts, and invokes opaque dependencies for correction and destination publication.

A shortcut directly clears the destination’s first 32-bit word when the product of the scaled operand headers compares equal to zero.

**Inferred intent:** the arithmetic resembles Montgomery multiplication and reduction. This interpretation does not establish a mathematical result, canonical range, or external representation. Those properties depend on unavailable conversion-factor values, arithmetic semantics, and dependency contracts.

**Established boundary:** the function directly initializes and updates the local accumulator. It does not directly initialize the separate working buffer or directly publish the main-path accumulator into the destination.

## Interface

The function accepts five arguments, in order:

1. **Destination address.** Used for a direct 32-bit zero store on the shortcut; otherwise supplied to the final publication dependency.
2. **Context address.** Represented by a 32-bit integer in the supplied interface. Main-path arithmetic reads a 32-bit word at byte offset 8.
3. **First operand address.**
4. **Second operand address.**
5. **Modulus object address.**

There is no explicit return value or status result.

### Object layout

The arithmetic inputs are accessed as 32-bit words:

| Byte offset | Observed role |
|---|---|
| 0 | Header, used in signed count calculations |
| 4 | Payload word 0 |
| 8 | Payload word 1 |
| 4 + 4k | Payload word k |

This establishes addressing and word width, not byte order, allocation size, serialization, or ownership.

Four global factors participate in header decoding, payload decoding, header encoding, and payload encoding. These names describe their uses, not established inverse relationships. Their values, declared arithmetic types, and stability are unavailable. The function does not snapshot all factors at entry.

The local accumulator comprises one header followed by 130 32-bit payload slots. A separate 524-byte local buffer is passed to dependencies.

### Arithmetic-width contract

The representation explicitly uses unsigned 32-bit intermediate values, unsigned 64-bit multiplication operands, high-word extraction, low-word extraction, and 32-bit unsigned carry tests. Where an operation is explicitly unsigned, its unsigned width governs its arithmetic. Extracting a low word from an explicitly widened product retains its low 32 bits.

**Payload scaling is not uniformly established as unsigned arithmetic.** Several factor-times-payload expressions are evaluated before an explicit unsigned conversion. Others have no explicit unsigned conversion at the scaling operation, including expressions assigned afterward to unsigned temporaries, expressions used in carry tests, and the high-word trimming predicate.

Consequently:

- An unsigned conversion after scaling establishes the converted result’s interpretation, not the scaling multiplication’s original signedness or overflow behavior.
- Assignment to an unsigned temporary does not prove that the preceding multiplication was unsigned.
- A surrounding unsigned sum or carry operation does not, by itself, resolve the arithmetic of a nested scaling expression.
- Terms eventually multiplied by zero still contain preceding scaling operations whose reads and intermediate arithmetic cannot be dismissed.

The reduction-only coefficient also includes a chain of multiplications without an explicit unsigned conversion before the chain begins. Its final unsigned storage does not establish wrapping semantics for every preceding multiplication.

Header scaling, header products, signed absolute values, count arithmetic, and result-header construction likewise require care concerning signedness and overflow. No blanket low-32-bit wrapping contract applies to all header, payload-decoding, or payload-encoding expressions.

### Preconditions for ordinary-path interpretation

A sufficient storage envelope for the directly shown main-path accesses, subject to valid arithmetic, includes:

- Modulus-derived count N is between 1 and 129.
- Both operand magnitudes are positive.
- The greater magnitude L does not exceed N; the smaller is S.
- Each operand provides its declared payload.
- The modulus provides at least N payload words.
- Context offset 8 is readable.
- Every addressed input word is readable.

Additionally, all header, count, scaling, coefficient, and encoding expressions must have defined arithmetic under their actual types. Without those types, avoiding overflow in every potentially signed intermediate is a conservative condition, not a complete representation contract. In particular, validity must cover repeated scaling expressions and scaling that precedes multiplication by zero.

These conditions are not checked. They do not establish dependency buffer capacity, destination capacity, mathematical validity, or termination.

## Behavior

### Header reads and shortcut selection

The function first reads and scales the modulus header, takes its signed absolute value, and derives N − 1. It then reads and scales the second operand header, followed by the first operand header.

The shortcut condition compares the product of the two scaled operand headers with zero. It is not expressed as two independent header-zero tests.

When that comparison succeeds:

- The destination’s first 32-bit word is directly set to zero.
- No destination payload word is directly changed.
- No dependency is called.
- No operand payload, modulus payload, or context word is read.
- All three input headers have already been read.

This path still requires valid preceding header and product arithmetic. A zero scaled operand header does not excuse an earlier invalid modulus-header calculation.

If the header-product operation were independently established to wrap at 32 bits, nonzero operands could also produce a zero low-word product. That behavior is conditional, not established here.

### Operand-role selection and initialization

On the main path, safely computed absolute scaled operand headers determine the traversal counts. The greater magnitude becomes L and supplies the long operand. The other becomes S and supplies the short operand.

Equal magnitudes retain the first operand as long and the second as short. No later operation explicitly reapplies either input header’s sign.

Accumulator indices 0 through N are directly cleared, in ascending order. These are raw zero stores, not stores made through the payload-encoding factor.

The function then reads and scales long-operand payload word 0 and retains the resulting unsigned word for subsequent multiplication passes. The separate 524-byte buffer receives no direct initialization.

### Multiplication-and-reduction passes

There are S passes under ordinary positive counts. Short-operand positions are visited in ascending order.

At each pass’s beginning:

- The current short-operand word is read and scaled into a retained unsigned value.
- The same stored word is read and scaled again for the initial explicitly widened multiplication.
- That multiplication combines the converted short value with the retained first long-operand value.
- A separately scaled and unsigned-converted accumulator word 0 is added.
- The context word at byte offset 8 is read and used with the low word of that sum to form a reduction coefficient.
- The coefficient is explicitly widened and multiplied by a scaled, unsigned-converted modulus word 0.
- High-word extraction and a low-word carry test establish the initial reduction carry.
- The high word of the operand-product sum establishes the initial product carry.

The scaling expressions feeding those explicit unsigned operations remain subject to the arithmetic qualifications above.

For positions 1 through L − 1, the pass combines the current accumulator contribution, incoming product carry, long-operand word multiplied by the retained short value, incoming reduction carry, and coefficient multiplied by the corresponding modulus word.

Each such position writes an encoded result to the preceding accumulator slot. Product and reduction carry contributions are tracked separately.

For positions L through N − 1, the long-operand product is absent. The pass continues with accumulator contributions, propagated product carry, and modulus products.

Each pass finishes by writing:

- Index N − 1, from the final product carry, reduction carry, and scaled old accumulator word N.
- Index N, from the two carry indicators associated with those final additions.

The direct stores use the payload-encoding factor. Their precise arithmetic must not be generalized from the explicitly unsigned products elsewhere.

Structurally, a pass moves contributions one payload position downward. The function does not test whether the discarded low-word contribution is zero.

Some compound expressions contain repeated accumulator, operand, or modulus reads. Some repeated scaling operations occur in terms subsequently multiplied by a value set to zero. Such terms do not contribute numerically when their arithmetic is valid, but neither their reads nor their intermediate scaling semantics may be omitted from the contract. The representation does not establish a total evaluation order among all operands of compound expressions.

### Reduction-only passes

After the multiplication passes, additional passes execute while the outer count remains below N. For valid S ≤ N, their number is N − S.

Each pass forms a reduction coefficient from scaled accumulator word 0 and context offset 8. Unlike an explicitly widened product, this coefficient expression does not establish unsigned semantics for every multiplication preceding its unsigned assignment.

The pass then:

- Combines the coefficient’s explicitly widened product with scaled, unsigned-converted modulus word 0 and accumulator word 0.
- Visits positions 1 through N − 1.
- Combines the incoming carry, scaled accumulator word, and modulus product at each position.
- Writes the encoded low result one accumulator position lower.
- Writes indices N − 1 and N using the final carry and old high accumulator word.

There is no operand-product contribution in these passes. If S equals N, none execute. If S exceeds N, they are skipped rather than rejected; earlier accesses may already exceed the supported storage envelope.

### Local header construction

The trimming scan begins with candidate index N. It tests a scaled accumulator word for zero only while the candidate index is positive. A zero result decreases the candidate index.

The scan does not test index 0. It never trims below that index. Thus, if all tested high words scale to zero, the locally constructed object still encodes one payload word, regardless of the contents of index 0.

The zero test itself uses a scaling expression without an explicit unsigned conversion. Its result is meaningful only under valid scaling arithmetic.

The header immediately preceding the accumulator is written using the header-encoding factor and one more than the retained index. No sign adjustment is added. At N = 129, an untrimmed object contains 130 payload words plus its header.

### Dependency-controlled correction

F-bcd78fb452c3 is first called with the separate buffer address and original modulus address. Its return is ignored. This call alone does not establish initialization, copying, or any buffer write.

Correction direction starts at +1. The subsequent control flow is:

- F-08a778c50842 receives the accumulator object and original modulus.
- A negative signed return exits correction.
- Otherwise, F-08a778c50842 receives the accumulator and separate buffer.
- A negative return from this second call changes direction to −1 and skips F-aa5c6722c7c4.
- A nonnegative return invokes F-aa5c6722c7c4 with accumulator and buffer.
- If that call occurred while direction remained +1, F-796113959089 receives the buffer and +1, without an intervening buffer-versus-modulus comparison.
- In every other case, F-08a778c50842 receives buffer and original modulus. F-796113959089 receives buffer and the current direction only if this comparison returns strictly positive.

Once direction becomes −1, it never returns to +1.

**Inferred intent:** these dependencies may compare, subtract, and adjust a working modulus. Those meanings are not established contracts.

There is no iteration limit or independent progress check. Negative dependency returns are used as branch outcomes, not separately classified as errors.

### Publication request

After correction exits, F-bcd78fb452c3 receives the caller’s destination and the local accumulator object. Its return is ignored.

This is the only explicit main-path use of the destination. The call does not prove that destination bytes are written, that publication succeeds, or that failure is reported.

## Side effects

Established direct effects include input-header reads, main-path payload reads, repeated context-offset-8 reads, global-factor reads, local accumulator and header stores, and the shortcut’s destination-header zero store.

No direct write to the separate buffer is shown. No direct write to an input object, context, or global factor is shown.

Dependencies may mutate supplied objects or globals. Their preservation must not be assumed. Most arithmetic precedes the dependency calls, but later correction and publication behavior can still depend on their mutations.

No explicit heap allocation or release is present. Dependency allocation, release, address retention, and ownership behavior remain unknown.

Both local object addresses cease to designate live local storage after return. Safe dependency use requires compatible lifetime behavior.

Aliasing is unchecked. On the shortcut, the three headers are read before the destination store. General aliasing support, including effects of dependency mutation through aliased arguments, is `documented_only`.

## Dependencies

| Opaque ID | Established use | Unresolved contract |
|---|---|---|
| F-bcd78fb452c3 | Called with buffer and modulus; later with destination and accumulator. Returns ignored. | Writes, copying, initialization, capacity, ownership, allocation, release, failure. |
| F-08a778c50842 | Receives two object addresses; signed return controls negative, nonnegative, and strictly positive branches. | Ordering meaning, purity, mutation, error encoding, required object validity. |
| F-aa5c6722c7c4 | Receives accumulator and buffer after their comparison returns nonnegative. Return ignored. | Arithmetic meaning, mutation targets, memory extent, failure. |
| F-796113959089 | Receives buffer and exactly +1 or −1 on the specified branches. Return ignored. | Adjustment meaning, memory effects, capacity, ownership, failure. |

No dependency is assumed to preserve globals, preserve arguments, or own or release supplied storage.

## Acceptance examples

These are conditional static cases, not executed tests.

1. **Zero scaled operand header:** if all preceding header arithmetic and the shortcut product are defined, a zero scaled operand header causes a direct destination-header zero store. No dependency or payload access occurs.

2. **Equal operand magnitudes:** with valid positive magnitudes, the first operand supplies the long traversal and the second supplies the short traversal.

3. **Safely negatable negative header:** its magnitude controls traversal. No independent result-sign adjustment follows.

4. **Single-word operation:** for N = L = S = 1, with valid scaling and storage, one multiplication-and-reduction pass executes. Inner traversals beginning at position 1 are skipped. Accumulator indices 0 and 1 are written; no reduction-only pass follows.

5. **Additional reductions:** for N = 3, L = 2, S = 1, and valid arithmetic, one multiplication pass and two reduction-only passes execute.

6. **Unsigned conversion after scaling:** if payload scaling would overflow under its actual signed type, a later unsigned conversion does not establish a valid wrapped decoded word. This case is `documented_only`, not an accepted modulo-arithmetic example.

7. **Zero-multiplied contribution:** a scaling expression that precedes multiplication by zero still requires valid intermediate arithmetic. Its payload read is not contractually eliminated.

8. **Immediate correction exit:** if the first accumulator-versus-modulus return is negative, both F-bcd78fb452c3 calls still occur, but neither adjustment dependency executes. Buffer initialization and destination publication remain unproven.

9. **Direction change:** a negative accumulator-versus-buffer return selects −1 and skips F-aa5c6722c7c4. Adjustment then requires a strictly positive buffer-versus-modulus return.

10. **All high words scale to zero:** trimming reaches index 0 without testing that slot. The header encodes one payload word before correction.

11. **Untouched working buffer:** if the initial dependency leaves the buffer unchanged, the function supplies no direct fallback initialization.

12. **Zero modulus count:** this is not a validated rejection. Main-path arithmetic includes an index N − 1 store, which becomes index −1. No safe error result is established.

## Uncertainties

The following remain `documented_only`:

- Global-factor values, types, widths relevant to expression evaluation, and stability.
- Signedness and overflow behavior of payload scaling before unsigned conversion or assignment.
- Arithmetic of uncast coefficient chains, encoding expressions, and scaling-based zero tests.
- Header arithmetic, absolute-value overflow, count overflow, and shortcut-product overflow.
- Representation inverses, normalization, modulus restrictions, and operand ranges.
- Mathematical correctness, low-word cancellation, and canonical output bounds.
- Buffer initialization, destination publication, dependency ownership, capacity, mutation, failure, and termination.
- Invalid pointers, insufficient storage, concurrency, externally changing values, and unsupported aliasing.

There is no explicit validation or error-reporting path. This does not establish that execution cannot fault or that dependencies cannot fail or diverge.

## Evidence

Established observations come solely from the supplied static representation: argument accesses, explicit conversions, word extraction, carry operations, local storage extents, traversal bounds, direct stores, and dependency call order.

Explicit unsigned widening establishes particular product operations. It does not establish the signedness or overflow semantics of scaling expressions evaluated before those conversions. Assignments to unsigned storage likewise do not resolve every preceding intermediate operation.

The separate buffer has no direct store. Initialization and copy-like effects are therefore unresolved rather than inferred from argument positions.

Arithmetic intent and dependency meanings are distinguished from direct observations. No dependency implementations, definitive global-factor type contracts, or runtime traces were supplied.

No runtime tests have been performed.
