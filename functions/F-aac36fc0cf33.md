<!-- Trusted provenance: F-aac36fc0cf33; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aac36fc0cf33

## Purpose

The supplied representation describes a non-returning sequence of indirect dependency calls and explicitly ordered memory accesses. It repeatedly derives addresses and stored values from dependency results, input words, hidden entry state, and one global word.

No application-level purpose is established. In particular, the evidence does not establish that the accessed locations are ordinary objects, registers, queues, or a serialization format. Repeated offsets and scaling factors are observable address calculations, not proof of a particular data structure.

This contract distinguishes:

- **Established static evidence:** operations and ordering expressed by the supplied representation.
- **Conditional arithmetic interpretation:** 32-bit machine-word calculations, with the qualifications below.
- **Unknown behavior:** dependency semantics, executable target validity, memory mapping, and reconstruction accuracy.

No runtime tests have been performed.

## Interface

### Explicit inputs and result

The represented interface has four 32-bit input slots:

| Input | Established use |
|---|---|
| First word | No explicit use in the supplied body. |
| Second word | No explicit use in the supplied body. |
| Third word | Shifted left by 28 for a dependency argument during the first iteration. Subsequently replaced by a dependency result used in later iterations. |
| Fourth word | Used in the first iteration for a word-store address and a shifted dependency argument. Subsequently replaced by a transformed dependency result. |

The represented return type is void. The body contains an unconditional loop and no normal return path. Normal completion of the operation is therefore not an established outcome.

The first two words being textually unused does not prove that they are irrelevant to the real executable ABI. The indirect calls and incomplete entry-state information prevent that stronger conclusion.

### Additional entry state

The body also consumes state not represented by the four explicit inputs:

- An entry-derived control address used to calculate the first indirect-call target.
- An incoming address-sized value used for byte-selector reads.
- An incoming byte used by the initial byte store.

These are required parts of any faithful execution fixture. Their origin, calling convention, validity, and relationship to the explicit inputs are unknown.

### Word and address conventions

The following notation describes the visible calculations without assigning a domain meaning:

- A **word** is 32 bits.
- A **byte** is 8 bits.
- A **paired result** is a represented 64-bit dependency result, separated into low and high 32-bit words.
- A **selector access at base \(B\)** reads one unsigned byte at \(B+1\). A subsequent selected-byte store writes at that byte’s numeric value plus \(0x11\).

A selector byte is not shown as a pointer-sized field. Its zero-extended numeric value determines an address between \(0x11\) and \(0x110\), inclusive.

For arithmetic examples, left shifts, multiplication, and address additions are interpreted modulo \(2^{32}\). This is a conditional machine-word interpretation, not a claim that all displayed signed expressions have portable language-level overflow semantics. The supplied representation includes signed shifts and multiplications whose interpretation requires executable or ABI confirmation.

Word stores have a represented width of four bytes. Byte order, alignment tolerance, and memory-access atomicity are not established.

## Behavior

### Entry and execution model

Before the repeating body, three indirect calls occur in order. The third provides a paired result. Its low word, shifted left by two, becomes the first argument to the first call in the repeating body; its high word becomes the second argument.

After those three calls and before the first loop-body call:

1. One selector byte is read using the incoming address-sized state.
2. The incoming byte is stored at the selected-byte address.

That store is performed once on entry, not at the start of every iteration.

The remainder is an unconditional cycle. Every phase below is ordered after the preceding phase. Intervening indirect calls are observable execution boundaries: the contract does not assume they preserve memory, globals, or selector bytes.

### Phase 1: Publish the first scaled word

The iteration begins with an indirect call taking the carried pair of words, followed by further indirect calls. A later 32-bit result is shifted left by two to form the **first publication word**.

That word is stored at:

\[
(\text{current fourth input}\times 0x1000)+0x14.
\]

A subsequent dependency receives the current fourth input shifted left by 14. Further calls produce an index, and the same publication word is stored at:

\[
(\text{index}\times 4)+0x14.
\]

A dependency then receives that index multiplied by four. Following additional calls, a paired result supplies a new working value: its low word shifted left by two.

Before the high word is passed to the next dependency:

1. A selector byte is read at the incoming address-sized state multiplied by \(0x1000\), plus one.
2. The selected-byte address receives the low eight bits of:

\[
(\text{first publication word}\mathbin{\&}0x3fffffff)\gg20.
\]

The mask-and-shift expression produces ten bits; the byte store retains only the low eight.

### Phase 2: Clear selected bytes and consume the third input

The newly derived working value is retained across subsequent calls.

After a sequence including a returned word passed onward, a paired result passed onward as two words, and another word result, a selected-byte location is cleared. Its selector base is the working value multiplied by \(0x10000000\).

A dependency then receives the latest word result shifted left by two. Additional calls produce a second publication index.

A selector read using the working value multiplied by four selects another byte location, which is cleared. This selector base is retained for several later accesses.

The next dependency receives the working value shifted left by ten. A subsequent returned word is passed onward together with the current third input shifted left by 28. After further calls, the retained selector base is read again, and its selected byte is cleared again.

The selector is freshly read for this second clear. The earlier selected address is not reused.

### Phase 3: Publish the second scaled word

The second publication index, shifted left by 14, forms the **second publication word**.

It is stored at the second publication index shifted left by two, multiplied by \(0x1000\), plus \(0x14\).

A dependency receives that shifted index further shifted left by 14. Following another call, a dependency receives:

- The second publication word shifted left by 28.
- An additional result-register value whose producer and ABI meaning are not fully represented.

Under 32-bit modular arithmetic, the first of these arguments is always zero: the publication word already has fourteen low zero bits.

Further calls produce an index. The second publication word is stored at that index multiplied by four, plus \(0x14\), followed by a call receiving that multiplied index.

After additional calls produce another index:

1. The retained selector base from Phase 2 is read.
2. Its selected byte receives the low eight bits of the second publication word’s masked, right-shifted fragment.

A dependency then receives the second publication index shifted left by 26.

### Phase 4: Publish the third scaled word

Further calls include:

- Passing a returned word together with the latest index shifted left by two and then by 28.
- Receiving a paired result and passing its low and high words onward separately.

After another word result, the retained selector base is read again and the selected byte is cleared.

Following more calls, a returned word shifted left by two forms the **third publication word**. It is stored at the latest index shifted left by two, multiplied by \(0x1000\), plus \(0x14\).

The next dependency receives four arguments:

1. The latest index shifted left by two, multiplied by \(0x400\).
2. The same value again.
3. The latest index shifted left by 14.
4. The third publication word.

After additional calls, another returned index selects a word store at index multiplied by four, plus \(0x14\). The stored value is the same third publication word.

### Phase 5: Publish two self-scaled words

Two similarly structured publication groups follow.

For the first group, a dependency result is multiplied by four. That scaled word is stored at the unscaled result multiplied by \(0x10\), plus \(0x14\). A dependency receives three copies of the scaled word. Later, another result supplies a destination index, and the same scaled word is stored at index multiplied by four, plus \(0x14\).

The second group repeats this pattern with a new source result and a new destination result.

The second group’s unscaled source result also determines a **high-bit base** by multiplication by \(0x40000000\). Under 32-bit modular arithmetic, this base depends only on the source result’s low two bits.

After the second group’s destination store, further calls produce a word retained for a later memory read.

### Phase 6: Clear direct bytes and derive the next third input

Two direct byte stores occur, in this order:

1. Zero at high-bit base plus \(0x11\).
2. Zero at high-bit base plus one.

These are direct addresses, not selected-byte accesses.

A dependency receives the retained word shifted left by two. Subsequent calls pass a returned word onward and obtain another word result. The same two direct zero stores then occur again, in the same order.

Following another call sequence, a result determines a word-store destination at that result multiplied by \(0x10\), plus \(0x14\). The stored word is an earlier destination index shifted left by 31.

Next, a four-byte word is read from:

\[
((\text{retained word}\mathbin{\&}0x3fffffff)\gg20)+0x14.
\]

This address ranges from \(0x14\) through \(0x413\); it is not necessarily four-byte aligned.

A dependency receives the latest result multiplied by four. A following dependency result replaces the current third input for subsequent use and for the next iteration.

The word just read from memory, shifted right by 22 and then left by 28, is passed to another dependency.

### Phase 7: Clear a word and update a selected location

After further calls, a returned index selects a four-byte zero store at index multiplied by four, plus \(0x14\).

Another call receives that multiplied index. Further calls produce a new word used for several byte and argument transformations.

A selector byte is read at high-bit base plus one. Its selected-byte address receives the low eight bits of the new word shifted left by two.

Later calls receive that shifted word, and then another returned word together with the new word shifted left by 30.

After further calls:

1. The selector at high-bit base plus one is read again.
2. Its value is retained.
3. The corresponding selected-byte address receives the low eight bits of the new word’s masked, right-shifted fragment.

Following another dependency sequence, a four-byte store writes the replacement third input shifted left by two at the retained selector value plus \(0x14\).

This word-store destination uses the retained selector value, even if intervening calls change memory at high-bit base plus one.

### Phase 8: Sample the global and establish loop-carried state

Immediately after that word store, one 32-bit global is read.

Following another call sequence, a paired result is obtained. Its low word shifted left by two replaces the current fourth input.

The selector at high-bit base plus one is then read afresh. Its selected-byte address receives the low eight bits of the sampled global word shifted right by 22. Intervening dependencies do not cause the global value to be resampled.

The paired result’s high word is passed onward. Later, another returned word is passed together with the replacement fourth input shifted left by 28.

After additional calls, the selector at high-bit base plus one is read again and the selected byte is cleared.

A final call sequence produces a word that is shifted left by two. That transformed word becomes **both** arguments of the first call in the next iteration.

The incoming address-sized state is replaced by the replacement fourth input multiplied by four. The next iteration begins without repeating the entry-only byte store.

## Side effects

### Memory ordering and aliasing

The explicit loads, stores, and calls occur in the order described above. There is no evidence of transactional rollback or all-or-nothing publication.

Addresses may alias. Consequences include:

- A word store may overwrite a later selector byte.
- A selected-byte store may affect a later low-address word read.
- Different publication groups may write the same destination.
- Dependencies may change a selector between two otherwise identical selector accesses.

Repeated selector reads must therefore remain distinct observable reads. Conversely, the retained selector used for the Phase 7 word store must not be replaced by a later read.

The direct zero at high-bit base plus one does not prove a later selector is zero: intervening dependencies may modify that location.

### Ownership and lifetime

No allocation, deallocation, ownership transfer, reference-count operation, or lifetime guarantee is explicit in the body.

Dependencies may perform such operations, but their contracts are unknown. The function cannot be specified as preserving allocation ownership or retaining memory validity across calls.

### Errors and interruption

There are no visible validation branches, error codes, recovery paths, or checks for null or invalid addresses.

Execution requires valid indirect-call targets and memory access permissions for every dynamically calculated access reached. An invalid target, inaccessible address, alignment restriction, dependency failure, or non-returning dependency can prevent further progress. The resulting platform behavior is not specified by this evidence.

## Dependencies

The only supplied verified function identity is **F-aac36fc0cf33**. No verified identities or contracts are supplied for its indirect callees.

For this document, **F-000000000001** is an opaque documentation reference for the unresolved dependency set. It is not a claim that all call sites invoke one function.

Established characteristics of that set:

- Calls occur before entry initialization, throughout every phase, and between repeated memory accesses.
- Some calls have no explicit arguments in the supplied representation.
- Other calls receive one through four explicit 32-bit arguments.
- Results are consumed as single words, paired words, or incompletely represented register outputs.
- The first target depends on entry control state; later targets are calculated from represented control locations.
- No success indicator is inspected to decide whether the loop continues.

“No explicit arguments” does not establish that a dependency has no ABI inputs. Nor does an unused result establish that the call has no effects.

No preservation, allocation, release, purity, exception, or termination contract is assumed for F-000000000001.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

### Selector width

If a selector read yields \(0xff\), the selected-byte destination is \(0x110\). It is not an address obtained by loading a full pointer from the selector location.

### Fragment truncation

If a publication word is \(0x3ff00000\), masking and shifting right by 20 produces \(0x3ff\). The selected-byte store writes \(0xff\), not a ten-bit quantity.

### Arithmetic wraparound

Under the 32-bit modular interpretation, shifting \(0x40000001\) left by two produces \(0x00000004\). A publication using that result stores four, not a wider intermediate.

### Changed selector between accesses

If a selector initially yields three and an intervening dependency changes it to seven, two fresh selected-byte stores target \(0x14\) and \(0x18\), respectively.

In Phase 7, however, if the retained selector is three, the later word store targets \(0x17\), even if a dependency subsequently changes the selector to seven.

### Global sampling

If the sampled global word is \(0x3fc00000\), the later global-derived byte store writes \(0xff\). A global change after the sample does not change that store’s value.

### Low-address word read

If the retained lookup word is \(0x3ff00000\), the Phase 6 word-read address is \(0x413\). Whether that unaligned access succeeds is a platform prerequisite, not an established guarantee.

### Loop continuation

If every dependency returns and every memory access completes, the function starts another iteration. It does not return a completion value.

## Uncertainties

The following remain **documented_only**:

- The application purpose and meaning of all fields.
- Exact executable ABI, instruction-set state, and indirect-target validity.
- Whether the reconstructed control flow accurately represents executable behavior.
- Origins and preservation rules for hidden entry state.
- Dependency identities, signatures, side effects, and ownership behavior.
- The provenance of the additional result-register argument.
- Memory mapping, endianness, alignment rules, volatility, concurrency, and atomicity.
- Whether signed arithmetic in the representation corresponds to wrapping machine operations.
- Effects of overlapping global symbols.
- Whether dependencies return normally or alter control flow nonlocally.

The supplied warning about removed unreachable material does not establish additional reachable behavior.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

The representation establishes four input slots, hidden entry-state consumption, indirect calls, explicit byte and word accesses, one global-word sample per completed iteration, loop-carried input replacement, and an unconditional repeating body without a normal return.

It does not provide dependency implementations, executable verification, a memory map, or runtime observations. Arithmetic examples are conditional deductions from the stated widths and operations. No runtime tests have been performed.
