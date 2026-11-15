<!-- Trusted provenance: F-224b106ddacc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-224b106ddacc

## Purpose

F-224b106ddacc is a six-input routine whose execution is directed by repeatedly observed global values. It maintains distinct arithmetic states, obtains an object address, conditionally accesses memory and invokes dependencies, retains an address for publication, and returns a 32-bit word.

**Established static evidence:** explicit operations include a request to F-d33f6d597e5d with argument 52, object-relative writes, queries through opaque dependencies, individual zero-byte writes, output publication, and a call to F-a8db34e89b44.

**Inferred intent:** allocation, initialization, copying, submission, termination, and release are plausible interpretations. They are not established ownership, lifetime, or success guarantees.

No unconditional successful sequence is established. Dispatch can repeat operations, reset arithmetic state, recover, return, or consume transition state that has not been initialized.

## Interface

| Input position | Representation | Established use |
|---|---|---|
| 1: output location | 32-bit address | Saved on entry; separately retained during preparation; later checked and used for a four-byte publication |
| 2: first integer | 32-bit word | Supplies two distinct products; one retained transformation is classified and written at object offset 12 |
| 3: second integer | 32-bit word | Initializes mutable arithmetic context during preparation |
| 4: opaque word | 32-bit word | Saved on entry; no subsequent use is visible |
| 5: input record | 32-bit address | Retained during preparation; subsequently checked, read, and supplied to dependencies |
| 6: third integer | 32-bit word | Supplies a retained product used for the object-offset-8 write |

The result is an unsigned 32-bit word. Retained return state begins at zero. No Boolean, status-code, length, or handle interpretation is established.

Memory requirements depend on operations actually reached:

- Publication requires four writable bytes at the retained output address.
- Object accesses require the particular fields accessed, reaching bytes 48–51 for the highest explicit field.
- The input-record query reads its first four bytes.
- Each explicit byte write requires its independently calculated destination to be writable.
- Dependencies may impose additional, unknown requirements.

A request argument of 52 does not establish that the resulting address denotes 52 accessible bytes. Similarly, a dependency argument of 24 does not establish its access extent. Null checks are separate operations, not universal guards.

Arithmetic below describes represented 32-bit results and address displacements, including wraparound. Complements affect 32 bits. Explicit signed right shifts are arithmetic. Signedness of some shifts on incompletely typed globals remains **documented_only**. The supplied representation does not settle original-language signed-overflow rules, alignment, or endianness.

## Behavior

### State and observation model

A, B, C, D, and E denote five distinct global locations used repeatedly below. Other globals supply adjustments, selectors, successor values, and return words. These labels do not imply stable values or non-aliasing with supplied memory.

Initially zeroed state includes the arithmetic accumulator, working value, running product, query accumulator, mutable second-derived value, saved product, return state, current object, publication candidate, retained addresses, and displacement-related context.

Scratch storage contains eight 32-bit words. Only words 0–2 are explicitly initialized. Three additional scalar locations, whose addresses are supplied to dependencies, initially contain zero.

The current object, publication candidate, intermediate address, and byte-write base are separate address roles. Likewise, updates to working value, running product, query accumulator, arithmetic accumulator, and return state do not implicitly synchronize them.

### Admission, reset, recovery, and common transitions

Initial dispatch state comes from a global. Each admission attempt observes B and then A. It admits an unsigned 32-bit dispatch-state-plus-adjustment result in the inclusive range 0–49.

A failed admission reads a recovery successor, clears the arithmetic accumulator, and retries. Admitted operation **48** enters that same recovery behavior.

Admitted operation **0** is a different reset operation: it first clears the arithmetic accumulator, then reads its own successor and redispatches. Its successor location differs from recovery’s. Thus:

- Operation 0: accumulator clear **before** successor observation.
- Recovery: successor observation **before** accumulator clear.

Neither operation explicitly clears the other arithmetic states or return state.

After admission, another A observation is shifted right by 29. Operation selection independently rereads the admission adjustment; therefore admission and selection can disagree. Reads within compound expressions or dependency argument lists do not generally establish an evaluation order.

Many operations use a common transition. They retain an arithmetic result and a selector, read two successor operands, select a successor, and then assign the retained result to return state. Selection uses 32-bit arithmetic equivalent to first operand multiplied by selector XOR 1, plus second operand multiplied by selector.

For selectors zero and one, this chooses the first and second operands respectively. Ordinary sign-based operations use the result’s top bit. Half-E and C-product processing reverse that polarity.

An unmatched selected value executes no operation body but still reaches the common transition. Its successor operands are admission-time A and incoming dispatch state. The shared selector and arithmetic result are not initialized on entry:

- A first unmatched selection has no determinate continuation established here.
- A later miss can reuse stale selector/result state, including after a direct operation changed return state.

Several operations bypass the common transition. No termination bound is established.

### Preparation

Preparation retains the input-record address and initializes the mutable second-derived value from a fresh D observation multiplied by the second integer.

It separately retains:

- Fresh B multiplied by the third integer.
- Fresh D multiplied by the first integer and fresh A, forming the first transformation.
- The saved output address.

It then reads a successor before replacing working value with another fresh-D-times-first-integer product. This final product does not include A. The separate D observations need not agree.

### Object request, layout, and submission

F-d33f6d597e5d receives 52. Its result replaces current object, not publication candidate. Return state is cleared.

The returned address is compared with null. A successor location is observed twice: the first observation is reduced by 18; the second is unadjusted. Non-null selects the adjusted first observation; null selects the second.

Explicit object accesses are limited to:

| Byte offset | Established access |
|---:|---|
| 0 | Four-byte zero write; four-byte reads for dependency arguments |
| 4 | Four-byte retained field-value write |
| 8 | Four-byte captured-A-times-third-product write |
| 12 | Four-byte first-transformation write |
| 16 | Four-byte zero write; address supplied to an unidentified dependency |
| 40 | Four-byte zero write |
| 44 | Four-byte zero write |
| 48 | Four-byte write of one |

One initialization operation writes zero at offset 0, then at offset 16. It selects object plus 20 as intermediate address, reads its successor, and then installs that address as byte-write base.

Another writes one at offset 48, zero at offset 40, then zero at offset 44. A fresh-B-times-prior-working-value-times-fresh-A product becomes saved product before the successor read, and return state and working value afterward.

Submission proceeds as follows:

1. Write the first transformation at offset 12.
2. Invoke an unidentified dependency with object plus 16, retained input, and 24.
3. After normal return, capture A.
4. Write the retained field value at offset 4.
5. Read a submission selector.
6. Write captured A multiplied by the retained third product at offset 8; also assign that product to working value.
7. Invoke F-359c673cd26d with the selector, 1, and current object.

The latter dependency’s result multiplied by fresh post-call B becomes running product. The predicate ORs two multiples of that running product: one scaled by captured A, the other by captured A’s arithmetic-right-shift-by-29 minus captured A. It also becomes the common-transition result.

Captured A remains available for arithmetic; it does not preserve object memory. F-359c673cd26d may modify the object, so the post-call offset-8 contents are unknown.

No path-independent complete object initialization is established.

### Query forms and separate destinations

F-6d7cdf4ae0e7 has five eight-argument forms. Positions below specify argument placement, not evaluation order. “Object word” means the four-byte word at offset 0.

| Form | Arguments in order |
|---|---|
| Initial scalar | 1; 0; 0; object word; selector; 0; 0; first scalar address |
| Scalar-derived, scratch 0 | 1; 0; fresh A × fresh B × first scalar; object word; selector; 0; 0; scratch-word-0 address |
| Scalar-derived, scratch 2 | Same shape, with a distinct selector and scratch-word-2 address |
| Literal 20 | 1; 20; fresh A × fresh B × scratch word 2; object word; selector; 0; scratch-word-3 address; second scalar address |
| Input record | 1; fresh A × fresh B × first input word; independently observed A × B × scratch word 0; object word; selector; 0; input plus 4; scratch-word-1 address |

The scalar-derived forms retain their pre-call product as working value. The returned word, scaled by admission-time B, replaces running product only. Their predicate ORs admission-time-A and shifted-A-minus-admission-time-A multiples of running product.

The literal-20 form selects scratch word 3 as intermediate address and retains its pre-call product as residual state. The scaled return replaces working value, running product, and query accumulator. Its predicate uses the same admission-time-A construction.

The input-record form selects input plus 4 as intermediate address. The input-derived product becomes residual state; the scratch-derived product becomes query accumulator. Its scaled return replaces working value, not running product. The predicate ORs admission-time-A and fresh post-call-C multiples of working value.

The initial-scalar form scales its result by fresh post-call B into working value and running product. Its predicate ORs:

- Working value multiplied by fresh C XOR a separately observed B shifted right by 30.
- Working value multiplied by a separate C observation.

F-7f98eb34143f receives seven arguments: selector, zero, object word, another selector, working value, zero, and the third scalar’s address. Its result multiplied by fresh post-call B replaces running product, not explicitly working value. Its predicate uses an A-scaled product and a second product scaled by an independently shifted-A-minus-another-A expression.

Selecting scratch word 3 as intermediate address is also possible without a query and without changing byte-write base.

### Arithmetic and displacement families

These operations establish arithmetic transformations, not validated lengths or bounds.

**Expansion.** Fresh C multiplied by the mutable second-derived value supplies a base. The operation retains base-minus-one, boundaries base plus 0x60000 and base plus 0x70000, adjusted displacements, and fresh A multiplied by the second-derived value for the later offset-4 write.

The prior query accumulator plus one participates in distinct adjustment-dependent addition, OR, XOR, and complement expressions. A residual mask uses the complement of an independently reread adjustment shifted left by one; that observation must not be merged with the earlier displacement adjustment.

A correction involving 0xFFF9FFFE and the predecessor is combined with the new query accumulator and a complemented, adjustment-masked 0x60000 boundary. The result replaces working value and the common-transition result, with ordinary sign selection.

**Boundary continuation.** This refreshes an adjusted base and sets the second byte displacement to base plus 0x60001. Its residual subtracts an OR-combined adjusted-address expression from the retained 0x60000 boundary. The predicate combines that residual, a correction involving 0xFFF9FFFD, and a complemented mask of the new displacement.

A separate fresh-B-times-fresh-A-times-running-product value becomes query accumulator, saved product, and common-transition result. Working value is not explicitly replaced. Consequently, predicate and returned arithmetic state differ.

**Half-E processing.** Independent E observations provide half-E-minus-E and half-E multipliers for the arithmetic accumulator. The operation retains a half-E displacement and another displacement 18 above the first product. Corrections use 0xFFFFFFEB, 21, and adjacent boundaries 18 and 19 above that product.

The predicate becomes working value and common-transition result. Mutable second-derived value is refreshed from a retained increment and adjustment mask; query accumulator becomes arithmetic accumulator plus fresh B. Nonnegative selects the second successor; negative selects the adjusted first.

**C-product processing.** Fresh C times arithmetic accumulator supplies the first displacement; fresh A times that accumulator supplies the second. Intermediate address becomes byte-write base before residual calculation.

Correction and residual calculations use 0xFFFFFFEB, the C-product predecessor, boundaries 18 and 19 above that product, and an independently E-derived increment mask. The result becomes working value and common-transition result. Query accumulator becomes arithmetic accumulator plus fresh B. Branch polarity matches Half-E processing.

### Classification and complemented-boundary transition

Classification examines the retained first transformation, not the raw first integer.

The first classification successor is read before comparison with 1. Only if unequal is a second successor read, before comparison with 2. Equality with 1 or 2 directly redispatches using the corresponding observation.

For transformation 10, fresh A scales mutable second-derived value. Independent adjustments then produce new accumulator, displacement, correction, residual, and query-accumulator states. Distinct boundaries subtract 0x70001 and 0x70004; separately complemented masks of the latter contribute to the calculation.

Return state becomes query accumulator minus the updated second-derived value. A first successor observation is increased by 18; an independent unadjusted observation is also read. Nonnegative selects the adjusted first; negative selects the second. Only afterward is return state copied to working value.

Other transformation values use retained arithmetic context. An OR-combined displacement/residual expression is subtracted from a retained auxiliary value. A correction involving 0xFFF8FFFE and the retained predecessor is combined with a complemented accumulator mask. The result replaces working value and common-transition result under ordinary sign selection.

The **complemented-boundary operation**, selected by **40**, has a separate direct transition:

- Return state becomes the 32-bit complement of the OR of retained correction and retained base plus 0x70001.
- The shared classification-10 transition location is read; this first observation is reduced by 2.
- That location is independently reread during sign-based successor selection.
- Nonnegative return state selects the adjusted first observation; negative return state selects the unadjusted second.
- Only after successor selection is the result copied to working value.

Both successor observations occur regardless of the sign. This operation does not refresh the shared selector/result retained by common-transition operations.

### Other transitions and operations

Paired common transitions use distinct locations per row below. Each pair observes the first value, applies the listed adjustment, and independently observes the unadjusted second value.

| Operation | First adjustment |
|---|---:|
| Scalar-derived query, scratch 2 / scratch 0 | −41 / −1 |
| Boundary continuation / literal-20 query | +21 / +30 |
| Half-E processing / submission | −20 / −2 |
| OR transformation of old return / address capture | −18 / −15 |
| F-7f98eb34143f / initial-scalar query | +5 / −9 |
| Expansion / input-record query | −2 / −20 |
| Other-value classification / C-product processing | +3 / +24 |

The output-null operation tests retained output, reads a successor and adds 34, observes E and assigns E times mutable second-derived value to query accumulator, then rereads the successor location. Null selects the adjusted first observation; non-null selects the second. Return state is not explicitly changed.

The input-null operation similarly uses two successor observations, first adjusted by −20, without that arithmetic update.

Each byte-write operation writes exactly one zero byte at byte-write base plus one retained displacement. Fresh B times fresh A times query accumulator then becomes working value. Its successor is read before the used displacement becomes return state and the product becomes arithmetic accumulator. The two operations use different displacements and successor locations.

Other direct operations:

- Load return state from one of three globals, after reading a successor.
- Multiply old return state by fresh A, read a successor, then copy the result to working value.
- Scale working value, running product, query accumulator, or saved product by fresh A and then fresh B. Running product is updated before the successor read; return state and working value afterward.
- Preserve old return as working value while using the OR of fresh-A and fresh-C multiples of old return as the common-transition result.
- Read a successor and redispatch without another explicit state update.

Address capture calculates a predicate from three C observations, including an explicit arithmetic right shift by 28. After both successor reads, current object is copied to publication candidate. The predicate becomes common-transition return state.

### Publication, cleanup, and return

Publication first writes publication candidate through retained output. Fresh B times admission-time A times running product then becomes return state. A successor is read before that result is copied to working value. Running product is unchanged. Publication does not return.

F-a8db34e89b44 receives current object. After normal return, publication candidate is cleared and a successor is read. Current object is not explicitly cleared.

The return operation returns retained return state without further visible cleanup.

## Side effects

Explicit effects include four-byte object writes, one-byte zero writes, caller-output publication, dependency calls, and repeated global observations.

There is no explicit assignment naming a global. However, supplied addresses and object-relative accesses are not established as disjoint from globals.

Dependencies may modify supplied storage or globals, retain addresses, or affect storage lifetime. No preservation or ownership rule is assumed merely because a dependency returns normally.

Publication is not established as transactional, unique, non-null, or subsequent to complete initialization. Guaranteed cleanup, safe terminator placement, leak freedom, and post-publication lifetime are unresolved.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-d33f6d597e5d | Receives 52; result becomes current object | Storage validity, alignment, failure, ownership |
| F-6d7cdf4ae0e7 | Five eight-argument forms | Access extents, outputs, mutation, retention |
| F-7f98eb34143f | Seven arguments including working value and scalar address | Memory/global effects, result meaning |
| F-359c673cd26d | Selector, 1, current object | Mutation, retention, ownership, result meaning |
| F-a8db34e89b44 | Current object | Release or other lifetime effects |

The object-plus-16/input/24 dependency has no supplied opaque ID. Its identity, access extent, copying semantics, overlap rules, result meaning, and return behavior remain **documented_only**.

Scratch and scalar addresses are supplied to dependencies. Safe retention beyond their storage lifetime and ownership transfer are not established.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Distinct reset ordering.** If admitted selection is 0, arithmetic accumulator is cleared before its successor is observed. If admission fails, or selection is 48, recovery observes its different successor before clearing the accumulator. Neither path requires return state or query accumulator to become zero.

2. **Complemented-boundary, negative result.** With correction zero and retained base zero, operation 40 sets return state to 0xFFF8FFFE. Successor observations 100 and 200 produce adjusted first value 98, but the negative result selects 200. Working value receives 0xFFF8FFFE only after selection.

3. **Complemented-boundary, nonnegative result.** With correction zero and retained base 0x7FF8FFFF, the boundary is 0x80000000 and its complement is 0x7FFFFFFF. Successor observations 100 and 200 select 98. The second observation remains required despite not being selected.

4. **Classification reads.** Transformation 1 requires only the first classification-successor observation. Transformations 2, 10, and 7 require both, in order.

5. **Output-null ordering.** First successor observation 100, intervening E observation 3, second-derived value 5, and second successor observation 200 yield query accumulator 15. Null selects 134; non-null selects 200.

6. **Selection mismatch.** Admission may accept one value while a changed adjustment causes an unmatched selected value. On the first such miss, initialized selector/result state cannot be assumed. A later miss can reuse stale common-transition state.

7. **Query destinations.** A scalar-derived pre-call product of 6 and scaled return of 9 leave working value 6 and running product 9. Literal-20 instead installs its scaled return into working value, running product, and query accumulator. Input-record does not replace running product.

8. **Submission mutation boundary.** Captured A of 2 and third product of 3 require the explicit offset-8 write of 6. Later predicate arithmetic retains captured A of 2, but F-359c673cd26d may change that field.

9. **Separate address roles.** A new current object does not replace an earlier publication candidate. Selecting scratch word 3 as intermediate address does not itself redirect byte writes.

10. **No inferred copy or release.** The address/address/24 call does not prove equality between object bytes 16–39 and input bytes. Calling F-a8db34e89b44 does not, without its contract, prove release.

## Uncertainties

The following remain **documented_only**:

- Global values, stability, aliasing, concrete successors, reachability, and termination.
- Consequences beyond indeterminate-value consumption on a first unmatched selection.
- Application-level result meanings and error policy.
- Dependency access extents, mutation, retention, exceptional exits, and ownership.
- Address validity, alignment, endianness, and storage lifetime.
- Actual publication ordering relative to initialization and cleanup.
- Original signed-overflow rules and unsettled global-shift signedness.

Null comparisons establish local decisions, not a complete failure-handling contract. No uniform error return is established.

## Evidence

**Established static evidence:** the supplied representation supports six inputs, represented 32-bit words and addresses, explicit initializations, separate arithmetic and address states, dependency argument positions, object offsets, byte writes, repeated global observations, and dispatch transfers.

It specifically establishes operation 0’s clear-before-successor ordering, recovery’s successor-before-clear ordering, and operation 40’s return assignment, adjusted first successor observation, independent second observation, sign-based selection, and subsequent working-value assignment.

**Inferred intent:** allocation, copying, registration, termination or padding, and resource release remain interpretations only.

**Limitations:** dependency implementations, global contents, aliasing guarantees, and settled type information are absent. No runtime tests have been performed.
