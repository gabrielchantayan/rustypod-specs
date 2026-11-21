<!-- Trusted provenance: F-b0a56bbf3c57; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b0a56bbf3c57

## Purpose

F-b0a56bbf3c57 performs an ordered sequence of dependency invocations, memory captures, integer transformations, and memory writes. Its explicit effects include propagating the second input, extracting portions of retained results, writing selector-chosen byte destinations, and publishing words and a halfword to low-address storage.

No domain-level purpose is established. The evidence does not justify identifying the function as an allocator, serializer, decoder, initializer, or release operation.

This contract distinguishes three evidence levels:

- **Established static evidence:** accesses, widths, argument relationships, and ordering expressed in the supplied decompiled representation.
- **Inferred intent:** interpretations that are not behavioral guarantees.
- **documented_only:** unresolved properties for which no replacement behavior is assumed.

The representation ends at an invalid-instruction termination marker. Each described effect is conditional on execution reaching it. A complete executable trace and normal completion are not established.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit read is represented. |
| Second input | 32 bits | Supplied to the first indirect invocation and subsequently written unchanged as a word. |

Signedness, pointer interpretation, validation requirements, and ownership roles are not established. The lack of an explicit first-input read does not prove that its incoming machine location is irrelevant to execution.

### Additional execution inputs

Behavior also depends on incoming link-related state, an incoming 32-bit register value without a represented initialization, dependency results, existing memory, and a late global word whose address is unspecified.

Some invocations produce represented 64-bit containers. Their lower and upper 32-bit halves have distinct uses. Other invocations expose secondary results whose calling convention is unresolved. These are execution dependencies, not reconstructed public parameters.

### Outputs and preconditions

No ordinary return value or normal return is represented. Explicit outputs are dependency invocations and memory effects.

Every reached access requires accessible storage of the indicated width. This includes addresses zero, 1, 40, and 124, computed addresses, and the unspecified-address global. Every reached invocation requires a usable target and a compatible execution environment.

No explicit validation establishes accessibility, alignment, object lifetime, or target validity. Address zero is used as storage rather than rejected as null. The terminal marker does not establish a particular hardware exception, process exit status, or recovery mechanism.

## Behavior

### Widths, layouts, and arithmetic

Explicit memory operations include unsigned 8-bit reads, 8-bit writes, 16-bit writes, and 32-bit reads and writes. Byte reads produce values from 0 through 255. Narrow writes retain the low eight or sixteen bits; they do not saturate.

Explicit unsigned transformations use 32-bit arithmetic, discarding overflowing high bits. Unsigned right shifts are logical. Some address calculations pass through signed-looking intermediates; their overflow and pointer-conversion semantics are **documented_only**. Simplifications involving those intermediates are conditional on 32-bit bit-vector evaluation.

Important access relationships include:

| Address relationship | Access |
|---|---|
| Four times a result, plus 20 | Word read or write |
| Sixteen times a result, plus 20 | Word write |
| Computed base plus 1 | Unsigned byte selector read |
| Selector plus 17 | Byte write |
| Result-derived base plus 52 | Word write |
| Result-derived base plus 28 | Byte write or, separately, word read |
| Result-derived base plus 4, 8, or 12 | Word capture |

These relationships establish neither object layouts nor nonaliasing. Selector-plus-17 destinations range from 17 through 272 inclusive; selector source addresses need not lie in that range.

For represented 64-bit results, extracting a half and shifting the entire container are different operations. One late byte comes from shifting the complete unsigned 64-bit result right by 54 and truncating to eight bits.

### Initial propagation and incoming state

The first indirect invocation receives the second explicit input before any explicit store. Additional invocations precede a word store of that input at four times a dependency result plus 20.

A subsequent result determines a destination at sixteen times that result plus 20. The stored word is four times the same result. A word is then captured from four times that result plus 20.

The capture later supplies:

- A logical right shift by eleven, published through another result-selected word destination.
- A logical right shift by twenty-two, plus 20, used directly as a word-read address.

The second address has no additional factor of four.

The incoming unsigned register value supplies a selector-read address through its logical right shift by eleven, plus 1. Its logical right shift by twenty-two instead supplies an invocation argument; no selector read at that shifted value plus 1 is established.

An early selector-selected store writes the low byte of four times a retained result. After intervening invocations, the same selector-address expression is read again. This second observation selects a destination for the low byte of the retained result masked by `0x1fffffff` and shifted right by nineteen.

Further early selector stores publish zero and a retained result shifted right by twenty-two. Some selector-source expressions evaluate to address 1 because their contributing values are explicitly zero. Repeated address expressions remain separate reads.

### First mapped dependency and intermediate publication

A call-produced word is supplied to **F-23325cd5e50c** and retained. Subsequent operations include a selector-selected zero-byte store, a selector capture from sixteen times the retained word plus 1, and a byte store containing the retained word masked by `0x3fffffff` and shifted right by twenty.

A later invocation receives the captured selector minus the retained word in two argument positions. This subtraction is unsigned 32-bit arithmetic.

Further invocations consume secondary results and explicit zero-derived arguments. During this phase, a direct byte read at address 1 selects a destination receiving zero. This direct read occurs after F-23325cd5e50c and before the word publication at address 124. It is distinct from computed selector reads, including those that also evaluate to address 1.

A later result is written as a word at address 124 before the next invocation receives that result twice.

Subsequently, a word is captured at a secondary-result-derived base plus 20. It is retained across invocations, reused in arguments, and later published after a left shift by eleven.

### Address-40 observations and paired-result fields

The address-40 sequence contains three separately represented word reads:

1. An initial capture retained for later publication.
2. Two additional reads supplying separate argument positions of one invocation.

Both argument reads follow the initial capture and precede their invocation. Their mutual evaluation order is unresolved. Another invocation occurs before the initial capture is written at four times a secondary result plus 20. The stored value is the original capture, not either argument observation or a post-call reread.

A later paired result supplies its upper 32-bit half as a base for word captures at offsets 8 and 12, in that order. After intervening invocations, a selector read at the offset-12 capture plus 1 chooses the destination for the low byte of the offset-8 capture.

Another phase captures a word at offset 4 from a retained result-derived base. Its low byte is later written through a selector-selected destination.

**F-bbb54b5f036d** is then invoked with one word repeated twice. Later, an unresolved dependency receives the lower half, upper half, lower half, and lower half of a paired result. **F-f80b3ac065fc** receives the same four-position relationship from another paired result.

A subsequent paired result supplies its upper half as a destination base plus 28. The byte written there is the low byte of that result’s lower half.

### Direct address-1 reads and overlapping publications

Direct reads at address 1 are explicit caller-side dependencies. They must not be attributed solely to callees or replaced by previously retained selectors.

The sequence around the address-zero publications is:

| Stage | Established access and use |
|---|---|
| Before word publication | Two distinct direct byte reads at address 1 supply two argument positions of one unresolved invocation. Their mutual evaluation order is unknown. |
| Before the word store | Additional invocations and computed selector accesses occur. |
| Word publication | A result is written as a 32-bit word at address zero, covering addresses 0–3. |
| After word publication | An invocation receives the published result twice; another follows. A fresh direct address-1 read then selects a zero-byte destination. |
| Before halfword publication | After further invocations, another direct address-1 read is captured. Its selected destination receives the low byte of a retained secondary result shifted right by twenty-two. The captured selector is later passed to an invocation without rereading address 1. |
| Halfword publication | The low sixteen bits of another result are written at address zero, covering addresses 0–1. |

Address 1 lies within both publication ranges. Reads after either store can therefore depend on this function’s own writes, as well as intervening dependency effects. Endianness is unspecified, so a published numeric value alone does not determine the byte subsequently observed at address 1.

Four further direct address-1 reads follow the halfword publication:

1. An invocation receives the full halfword-producing result twice, and another produces a paired result. A computed selector based on the earlier word-publication value selects a zero-byte store. Following more invocations, a direct address-1 capture selects a destination for the low byte of the paired result’s upper half shifted right by twenty-two.

2. Further invocations consume that retained selector. A later paired result supplies a byte for a direct computed-address store. Its destination derives from the earlier paired result’s lower half masked by `0x3fffffff` and shifted right by seven. After invocations and another computed-selector zero store, a fresh direct address-1 capture selects another zero-byte destination and is subsequently passed as an argument.

3. After additional paired-result handling, a word is captured at offset 4 from an address previously formed as an earlier paired result’s lower half plus 12. That captured word supplies a selector-read base for a zero-byte store. Following further invocations, a new direct address-1 read selects a destination for a retained secondary result shifted right by twenty-two.

4. Later invocations produce another paired result. A computed selector based on the retained secondary result shifted right by twenty-one selects a zero-byte store. After further invocations, another direct address-1 capture selects a destination for the low byte of the complete paired result shifted right by 54. The selector is subsequently passed as an argument before **F-de15789688a8**.

Together with the earlier read before address-124 publication, there are nine explicitly direct address-1 read occurrences: one early read, two argument reads, two reads between the overlapping publications, and four afterward. Computed selector reads that evaluate to address 1 are additional occurrences.

### Remaining mapped dependencies and repeated stores

The remaining mapped dependencies occur in this relative order:

1. **F-de15789688a8**, with one word repeated twice.
2. **F-ea83f9a17404**, with one word repeated twice.
3. **F-674e75a953e8**, with one word repeated twice.

An additional unresolved dependency is invoked between the first two. Numerous invocations and accesses separate these mapped dependencies. No explicit failure check tests their results.

Before F-674e75a953e8, two consecutive word stores target a retained value shifted right by nine, plus 52. They write a captured unsigned selector widened to 32 bits, then twice that selector.

A later secondary-result-derived destination plus 52 receives the same shifted word expression twice consecutively. Both stores remain represented occurrences despite identical values.

Further word stores propagate retained bases, scaled results, and values shifted right by twelve. These accesses do not establish object-field meanings or validity checks.

### Late global captures and final propagation

After a secondary-result-selected word store, the unspecified-address global word is read twice, with no represented invocation between the reads.

The first capture survives intervening invocations and later supplies an argument masked by `0x3fffffff` and shifted right by ten. Four times the second capture supplies a retained selector-read base.

Two selector reads use that retained base plus 1, separated by invocations. Their destinations receive:

- The low byte of four times a retained dependency result.
- The low byte of the same result masked by `0x3fffffff` and shifted right by twenty.

Neither selector read rereads the global. Conversely, the two original global captures must not be collapsed into one.

The four-scaled second capture also feeds a left shift by twenty-six, followed by multiplication by 1024. Under 32-bit bit-vector evaluation, the final product is zero; the intermediate shifted value need not be zero.

Two separate word reads at the intermediate shifted base plus 28 supply invocation arguments. Their mutual evaluation order is unspecified. Later selector reads use the multiplication-derived base plus 1, conditionally address 1.

After F-674e75a953e8, its retained argument is published through secondary-result-selected word storage. Further writes propagate paired-result components, explicit zero-derived values, and scaled results.

A late invocation receives two distinct word reads from the same four-scaled base plus 20. One observation is shifted right by twelve; the other is shifted left by two. They are not one shared capture.

Still later, another word is explicitly captured from a four-scaled address plus 20. After intervening invocations, that capture shifted right by twelve is stored elsewhere.

The final selector-write pair publishes the low byte of four times a retained result, then its masked high portion. These writes use separate selector reads from the same retained computed address. Further invocations precede the invalid-instruction termination marker. No cleanup, rollback, consistency check, or normal return is established.

## Side effects

### Memory and ordering

Dependencies can observe preceding stores and modify storage read afterward. No unknown dependency is assumed to preserve globals, memory contents, or storage lifetime.

Retained captures differ from later observations. This distinction applies especially to address 40, the twice-read unspecified-address global, direct address-1 reads, and selector addresses reread after invocations.

The address-zero stores overlap but have different widths. The halfword store covers only addresses 0–1 and does not itself replace addresses 2–3. Intervening dependencies may independently modify any of those bytes.

Selector-plus-17 writes cannot directly target address 1. Other computed writes are not established as disjoint from it. In particular, the masked-and-shifted direct byte-store destination can equal 1.

Atomicity, volatility, synchronization, reentrancy, and thread safety are **documented_only**. Preserving represented access occurrences does not establish that every occurrence has separately observable hardware behavior.

### Ownership, errors, and partial effects

No allocation, release, ownership transfer, or borrowing discipline is established. Retaining an address across a dependency invocation does not prove that the referenced storage remains alive. Passing a value to a dependency does not establish transfer or release.

No explicit validation, retry, error-status branch, or compensation is represented. An inaccessible address, unusable target, or nonreturning dependency can prevent later operations after earlier effects have occurred. No transactional guarantee follows.

## Dependencies

| Opaque function ID | Established invocation |
|---|---|
| F-23325cd5e50c | Once, with one represented 32-bit argument. |
| F-bbb54b5f036d | Once, with one word repeated twice. |
| F-f80b3ac065fc | Once, with lower half, upper half, lower half, lower half of a paired result. |
| F-de15789688a8 | Once, with one word repeated twice. |
| F-ea83f9a17404 | Once, with one word repeated twice. |
| F-674e75a953e8 | Once, with one word repeated twice. |

The table establishes relative order, not adjacency.

Additional indirect and unresolved dependencies are **documented_only**. Their complete interfaces, identities, secondary-result conventions, memory effects, ownership rules, and termination behavior are unavailable. Similar invocation patterns do not establish a common helper or interchangeable behavior.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Initial payload and incoming state

If the initial destination-selecting result is 3, the second explicit input is written unchanged as a word at address 32.

If the incoming unsigned register value is `0x00400000`, its eleven-bit-shift selector address is 2049. Its twenty-two-bit shift supplies argument value 1, not a selector read at address 2.

### Selector range and truncation

A selector of 255 chooses destination 272. Writing the low byte of four times 65 writes 4.

### Address-40 observations

If the initial capture yields 11 and the two argument-position observations yield 22 and 33, those argument positions receive 22 and 33. Their evaluation order remains unresolved. The later retained-value store writes 11.

### Direct address-1 observations

If the two pre-word-publication argument reads observe 5 and 9, their respective argument positions receive 5 and 9. No equality requirement follows.

Conditionally assuming little-endian storage and no intervening modification of address 1, publishing word `0x11223344` makes the subsequent direct selector observation `0x33`, choosing address 68 for the zero-byte store.

Under the same assumptions, a later halfword publication of `0xa1b2` makes address 1 contain `0xa1`. A subsequent direct selector observation chooses address 178, not 68. Actual intervening dependencies may change either observation.

### Complete paired-result shift

If the retained 64-bit result is `0x4000000000000000`, shifting the complete container right by 54 and truncating writes byte value 0. Its lower half is zero, but the full shifted value is 256 before byte truncation; a lower-half-only calculation does not describe the represented operation.

### Separate global captures

If the unspecified-address global’s first capture is 1024 and its second is 3, the later masked-and-shifted argument is 1. Both selectors based on the second capture read retained address 13, although the observations may differ.

Under 32-bit bit-vector arithmetic, the subsequent large-shift-and-multiply result is zero.

### Repeated stores

For a captured selector of 7, the consecutive offset-52 stores write word values 7 and then 14. The separate identical-value pair likewise retains both represented store occurrences.

## Uncertainties

- **documented_only — executable fidelity:** bad instruction data, removed unreachable material, and truncated control-flow recovery limit executable interpretation.
- **documented_only — ABI:** incoming machine state, secondary results, and paired-result conventions lack authoritative definitions.
- **documented_only — memory model:** endianness, alignment, mappings, volatility, and concurrent mutation rules are unavailable.
- **documented_only — global location:** the late global word’s address and alias relationships are unspecified.
- **documented_only — argument evaluation:** mutual ordering of separate reads supplying one invocation is unresolved, including the two direct address-1 argument reads.
- **documented_only — arithmetic:** portable semantics of signed-looking overflow and address conversion are unresolved.
- **documented_only — dependencies and completion:** preservation guarantees, lifetime effects, normal return, and behavior beyond the terminal marker are unknown.
- **Inferred intent only:** repeated selector and propagation patterns suggest regular encoding or state publication, but establish no protocol or object model.

## Evidence

The evidence consists of the supplied decompiled implementation representation. It supports the stated access widths, dataflow relationships, retained captures, mapped dependency order, and relative placement of explicit reads and writes, subject to its recovery limitations.

Explicit unsigned-byte references at address 1 establish caller-side reads before, between, and after the overlapping address-zero publications. Their placement and retained uses differ from computed selector reads. The word store covers addresses 0–3; the halfword store covers addresses 0–1. This overlap does not require an endianness assumption.

The representation includes notices about bad instruction data, removed unreachable material, and overlapping global symbols. These limit executable interpretation without establishing a domain purpose or a dependency contract.

No runtime execution, smoke checks, or tests have been performed. The acceptance examples are conditional static expectations, not observed execution results.
