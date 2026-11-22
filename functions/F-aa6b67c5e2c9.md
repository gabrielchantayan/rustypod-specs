<!-- Trusted provenance: F-aa6b67c5e2c9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa6b67c5e2c9

## Purpose

F-aa6b67c5e2c9 performs a state-dependent sequence of dependency calls, selector-byte acquisitions, narrow memory mutations, word propagation, and bit-field transformations. Its application-level purpose is not established.

The explicit arguments do not determine the complete behavior. Execution also depends on incoming machine state, primary and secondary dependency results, computed addresses, and independently acquired shared-memory values.

**Inferred intent:** The extraction of bit ranges and repeated scaled addressing suggest processing of encoded state or references. This interpretation does not establish parsing, allocation, serialization, initialization, or any domain-specific acceptance requirement.

The supplied static representation reports invalid instruction data, omitted unreachable material, and overlapping global storage. Its visible endpoint is an invalid-instruction termination marker, not a demonstrated normal return. The following obligations describe represented operations conditionally: an operation is required only if execution reaches it after all preceding calls and accesses complete.

## Interface

### Explicit inputs

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Passed to the first indirect dependency. |
| Second | 32 bits | No explicit use is represented. |
| Third | 32 bits | No explicit use is represented. |
| Fourth | Signed 32 bits | Shifted left by three and passed in the fourth position of an early indirect call. |

No argument is established as a buffer, length, output object, or ownership token. The absence of explicit uses for the second and third arguments does not settle their machine-level relevance under the unresolved calling convention.

### Implicit inputs

Established implicit inputs include:

- An incoming return-link value used to derive the first indirect call target.
- Incoming machine words providing an early selector-source base and payload.
- Primary and secondary result words from dependencies.
- Results represented as low and high 32-bit halves.
- Bytes and words at computed addresses.
- The shared selector byte at address **1**.
- Three represented word acquisitions at address **40**.
- Two represented acquisitions of another shared word whose address is unspecified.

Address **124 is an output location**, not an established directly acquired shared input. The representation explicitly stores a dependency-derived word there; it does not explicitly read that location. A possible computed alias or an unknown dependency’s possible read does not establish an additional direct input acquisition.

### Outputs and continuation requirements

The represented interface has no return value. No success indicator, failure indicator, output length, or returned ownership is established. Observable outputs consist of memory mutations and dependency invocations.

Continuation requires reached call targets to be executable, preceding dependencies to return, and reached accesses to be valid under the actual memory map and alignment rules. Consumed secondary result words require an appropriate ABI interpretation. No validation or recovery mechanism for these conditions is established.

Actual normal return, trapping, or continuation beyond the represented endpoint remains **documented_only**.

## Behavior

### Widths, arithmetic, and access interpretation

Byte acquisitions produce unsigned values from 0 through 255. Byte stores retain eight payload bits; the halfword store retains sixteen. Word accesses are represented as 32 bits. Results spanning two words are divided into low and high 32-bit halves, without establishing that both halves jointly represent one application-level integer.

Unsigned right shifts are logical. Arithmetic performed in unsigned 32-bit expressions wraps modulo \(2^{32}\). This does not make signed expressions wrapping operations: signed shifts, multiplication, subtraction, and address calculations retain unresolved representability and machine-semantics conditions. Subsequent assignment to unsigned storage does not retroactively change a signed expression’s arithmetic.

Distinct represented acquisitions and stores are retained in this contract. Whether each corresponds to a separate physical access depends on executable fidelity, volatility, optimization, and memory semantics.

### Addressing layouts

These patterns establish access relationships, not object types or extents:

| Address pattern | Established access |
|---|---|
| Derived base plus 1 | Selector-byte acquisition. |
| Acquired selector plus 17 | Byte destination, ranging from address 17 through 272. |
| Four times an index plus 20 | Word source or destination. |
| Sixteen times an index plus 20 | A distinct word destination. |
| Derived base plus 52 | Word destination without an implied extra scale. |
| Derived base plus 4, 8, or 12 | Separately represented word acquisitions. |
| Derived base plus 28 | Byte destination or, in another phase, word source. |

Selector-source calculations also use factors of 2, 8, 16, 64, 1,024, 4,096, \(2^{25}\), and \(2^{28}\), as well as masked and right-shifted state. A shifted address must not be replaced with a four-byte-scaled address unless that scale is actually represented.

### Entry and initial propagation

The first indirect dependency receives the first explicit argument. Its target depends on incoming linkage state. A second indirect call precedes the first memory mutation.

That mutation acquires a selector at an incoming-state-derived base plus 1, then writes the low byte of another incoming word to selector plus 17. An early indirect call receives the same secondary result word in its first three positions and the fourth explicit argument shifted left by three in its fourth position.

Several calls follow, including a call receiving zero. The incoming-state-derived selector source is then acquired again. This second mutation uses the incoming unsigned payload shifted right by 22 and narrowed to a byte. Intervening dependencies prevent assuming that the two selector acquisitions agree.

Later calls receive results shifted left by two and an earlier retained result shifted left by 25.

The initial word-propagation stages, separated by calls, establish this order:

1. A zero word is stored at a secondary-result-indexed four-times-scaled address plus 20.
2. That secondary result is stored at another four-times-scaled address plus 20.
3. Four times a dependency-derived index is stored at sixteen times that index plus 20.
4. A word is acquired at four times that index plus 20.

The acquired unsigned word subsequently supplies a payload shifted right by 11. A separate word acquisition uses its value shifted right by 22, then increased by 20, **without an additional four-byte scale**.

### Selector transformations and retained acquisitions

Repeated phases alternate dependency calls, selector acquisitions, and byte stores at selector plus 17. Represented payloads include zero, narrowed scaled results, and extracted bit ranges.

Established extraction forms include:

- An unsigned word shifted right by 22.
- Its low 29 bits shifted right by 19.
- Its low 30 bits shifted right by 20.
- Its low 22 bits shifted right by 12.
- An unsigned two-word result shifted right by 54.

These forms can produce ten bits before byte narrowing. The byte destination receives only the low eight.

The byte at address 1 is acquired directly on multiple occasions. Computed selector-source addresses also equal 1 when their retained basis is zero. Equal addresses do not merge separately represented acquisitions.

Some selectors are used immediately. Others are retained as unsigned state, used in a byte mutation, and passed to a dependency after intervening calls. Those later arguments use the saved acquisition, not an established reload.

One late indirect call has two argument positions that independently acquire the byte at address 1. Both acquisitions precede invocation, but their relative evaluation order is unresolved. Under changing or access-sensitive memory, the two arguments can differ.

### Shared-word output and input sequences

A dependency-derived word is stored at address **124** before an indirect dependency receives that same saved word in two positions. This is a publication followed by invocation; the call arguments do not require a readback from address 124. The callee is not assumed to preserve the published word.

A different word, acquired at a dependency-derived base plus 20, is retained across calls and supplies later transformations.

The address-**40** sequence contains three represented acquisitions:

- An initial word acquisition is saved.
- Two argument positions each independently acquire the word at address 40 before a subsequent indirect invocation.
- After that invocation and another indirect call, the initial saved value is stored at a secondary-result-indexed four-times-scaled address plus 20.

The initial acquisition precedes both argument acquisitions. No relative order between the two argument acquisitions is established. Neither argument must equal the other argument or the saved value.

Later, a shared word at an unspecified address is acquired twice after a computed word store and before the next indirect call. The first acquisition becomes retained unsigned 32-bit state. The second independently supplies a value multiplied by four and assigned to a signed 32-bit intermediate.

The first value later contributes its low 30 bits shifted right by 10 to a call. The second supplies selector-source bases and later signed transformations. No dependency call separates these two acquisitions, but their equality is not guaranteed under an access-sensitive memory model. The shared word’s exact declaration and conversion behavior remain **documented_only**.

### Dependency-derived structured accesses

A two-word result supplies a high-half base for an ordered word acquisition at offset 8, followed by one at offset 12. Both precede the next dependency call. The offset-8 word is retained for a later byte payload. The offset-12 word supplies a later selector-source basis.

Another word acquisition uses a previously retained low-half base plus 4 and supplies a saved byte payload. In a separate phase, a retained result is adjusted by 12, passed to a dependency, and later used as the base for a word acquisition at offset 4.

These relationships do not establish allocation origin, object size, lifetime, or ownership.

During late signed scaling, two argument positions independently acquire a word at the shifted signed base plus 28. Both acquisitions precede invocation; their relative order and equality are unresolved.

A later call likewise independently acquires a word twice at a four-times-scaled address plus 20. One acquisition contributes an unsigned right-shifted argument; the other contributes a left-shifted argument. This is not one retained acquisition used twice.

### Narrow outputs, overlap, and repeated stores

A retained unsigned state is explicitly zeroed. Before its relevant reassignment:

- A dependency-derived word is stored at zero times eight, hence address zero.
- Later, a dependency-derived halfword is stored at the address obtained by masking that zero state and shifting right by 11, also address zero.

Calls and other accesses intervene. The word spans addresses 0–3 and the halfword spans addresses 0–1. Both overlap the selector at address 1. Later selector acquisitions therefore cannot be assumed to return its pre-store value. Endianness, dependency effects, and memory semantics determine the actual value.

A later byte destination is derived from the low half of a two-word result, masked to 30 bits and shifted right by seven. Its additional four-times-state contribution is zero in that phase. The payload comes from a later result’s low byte.

Elsewhere, the high half of a two-word result supplies a byte destination at offset 28; the low half supplies the payload.

Two consecutive-store patterns are established:

- A retained unsigned selector is stored as a word at a computed destination plus 52, immediately followed by that selector shifted left by one at the same destination.
- A signed-result-derived shifted payload is stored twice consecutively at a secondary-result-derived destination plus 52.

No call separates either pair. Ordinary final-memory contents do not describe both represented accesses.

### Late signed scaling and final propagation

The signed basis derived from the second acquisition of the unlocated shared word is later shifted left by 26 and then multiplied by 1,024.

These are not established unsigned operations. Under an explicitly assumed wrapping 32-bit interpretation, the combined factor \(2^{36}\) makes the product zero modulo \(2^{32}\). That conclusion is conditional; defined signed overflow and negative-input shift behavior are not established.

The intervening calls and the independent word acquisitions at the shifted base plus 28 remain part of the represented sequence even under wrapping semantics. A zero final product does not eliminate earlier accesses.

Subsequent phases propagate results through four-times-index and sixteen-times-index destinations. They include explicitly zero-derived payloads, unsigned large-factor stores, retained word acquisitions, and additional selector mutations.

One unsigned stage stores a retained low result half multiplied by \(2^{26}\) at that same half multiplied by \(2^{28}\), plus 20. Later propagation includes a masked low-29-bit extraction shifted right by nine.

Near the end, a word acquired after a sixteen-times-index store is retained across calls. Its value shifted right by 12 is later stored at another four-times-scaled destination plus 20. Further call arguments include a low-six-bit extraction shifted left by 14 and a separate signed result shifted left by 30, retaining the signed-arithmetic uncertainty.

The final visible phase performs selector acquisitions, byte mutations, and indirect calls. No cleanup or successful return is demonstrated.

## Side effects

Established direct outputs include:

- Bytes at selector plus 17.
- Words at scaled-index destinations plus 20.
- A 32-bit word at address **124**.
- Overlapping 32-bit and 16-bit stores at address zero.
- A byte at a dependency-derived base plus 28.
- Words at computed destinations plus 52.
- A byte at a masked-result-derived destination.

Established direct shared inputs include the selector byte at address 1, the three word acquisitions at address 40, and the two acquisitions of the unlocated shared word. Address 124 is not included as a directly acquired input.

Dependencies can introduce additional reads, writes, failures, or nonreturning behavior. They are not assumed to preserve globals, selector sources, computed destinations, or aliased memory.

No explicit allocation, release, ownership transfer, or resource-retention contract is established. Repeatedly passing equal values does not establish borrowing or resource management.

No rollback, transaction, synchronization, atomicity, thread-safety, or reentrancy guarantee is visible. Earlier effects are not visibly undone when a subsequent call or access prevents continuation.

## Dependencies

The mapped direct dependencies occur in this relative order, with other calls and accesses between them:

| Opaque function ID | Established invocation shape |
|---|---|
| F-23325cd5e50c | One 32-bit argument. |
| F-bbb54b5f036d | The same saved word in two positions. |
| F-f80b3ac065fc | Low result half, high result half, then two further copies of the low half. |
| F-de15789688a8 | The same saved word in two positions. |
| F-ea83f9a17404 | The same saved word in two positions. |
| F-674e75a953e8 | The same saved word in two positions. |

Two further directly referenced dependencies have no supplied opaque IDs. One receives low and high result halves followed by two further copies of the low half. The other receives the same saved word twice. Their identities remain **documented_only**; neither is equated with a mapped dependency.

Numerous indirect dependencies use targets derived from incoming linkage state or represented internal address calculations. Calls consume single-word results, two-word results, and secondary result words. Complete target identities and calling conventions remain unresolved.

No dependency is established as pure, memory-preserving, guaranteed to return, or responsible for acquiring or releasing memory.

## Acceptance examples

These are conditional static obligations, not executed tests. They assume the relevant operations are reached and their accesses are permitted.

1. **Selector narrowing.** A selector acquisition of 7 and payload 0x1234 produce a byte store at address 24 containing 0x34.

2. **Ten-bit extraction.** Retaining the low 30 bits of 0x3FF00000 and shifting right by 20 produces 0x3FF. A subsequent byte store retains 0xFF.

3. **Explicit signed argument.** A fourth argument of 5 supplies 40 in the early fourth argument position. This does not establish negative-input or signed-overflow behavior.

4. **Different scales.** An index of 3 selects address 32 for four-times-index plus 20, but address 68 for sixteen-times-index plus 20.

5. **Address-124 publication.** If the published saved word is 0x12345678, that value is stored at address 124 before the following call receives two copies of it. Those arguments use the saved word, not an established readback. No preservation after invocation is guaranteed.

6. **Three address-40 acquisitions.** If the initial acquisition yields 0x11111111 and the acquisitions assigned to the two argument positions yield 0x22222222 and 0x33333333, those latter values occupy their respective argument positions. The later computed store uses 0x11111111. This does not specify which argument acquisition occurred first.

7. **Independent unlocated-word values.** If the first acquisition yields 0x00000400 and the second yields 3, the retained unsigned state is 0x00000400 and the scaled signed basis is 12, assuming the represented conversion. The former contributes 1 to the masked-right-shift call; selector acquisitions based on the latter use address 13.

8. **Independent address-1 arguments.** If the acquisitions assigned to the first and second argument positions yield 4 and 9, the call receives 4 and 9. Replacing them with two copies of one saved acquisition loses represented behavior.

9. **Address-zero overlap.** Both the word store and later halfword store at address zero include address 1. A later selector acquisition is fresh; its value requires additional endianness and memory-effect assumptions.

10. **Consecutive writes.** A retained selector of 9 produces consecutive word stores of 9 and 18 at the same offset-52 destination.

11. **Conditional signed wrapping.** Under wrapping 32-bit semantics, shifting a signed basis of 1 left by 26 produces bit pattern 0x04000000; multiplying that pattern by 1,024 wraps to zero. Without that assumption, the zero result is not guaranteed.

12. **Completion boundary.** Reaching the final visible call establishes neither successful return nor cleanup.

## Uncertainties

The following remain **documented_only**:

- Application purpose, intended input domain, and executable fidelity.
- Recovered boundary and control-flow accuracy.
- Actual return, trap, or continuation behavior.
- Complete ABI, hidden incoming state, and secondary-result semantics.
- Unresolved dependency identities, effects, preservation rules, and errors.
- The unlocated shared word’s address, declaration, and conversion rules.
- Signed overflow, signed shifts, and out-of-range conversions.
- Address-space width, alignment, endianness, mapping, volatility, and device behavior.
- Physical execution of each represented acquisition or store.
- Aliasing, object extents, allocation origins, lifetimes, and ownership.
- Whether low-address accesses reflect executable behavior.
- Concurrency and reentrancy safety.

No explicit input-validation branches, dependency-error checks, or recovery paths are established. This does not exclude faults or errors inside dependencies.

## Evidence

**Established static evidence:** Argument widths and represented uses; hidden-state consumption; access widths; retained versus independently acquired values; ordered structured reads; three address-40 acquisitions; two unlocated-word acquisitions; direct and computed address-1 acquisitions; the address-124 word store without an explicit read there; overlapping address-zero stores; consecutive stores; signed scaling; mapped dependency invocation shapes; and the invalid-instruction endpoint.

**Inferred intent:** Encoded-state or reference processing suggested by bit extraction and scaled addressing. No acceptance condition depends on this interpretation.

**Unknown:** Runtime fidelity, full ABI, memory semantics, dependency contracts, ownership, and normal completion.

This contract relies on the supplied static representation and opaque dependency mapping. Prior prose and review feedback are not independent execution evidence. No runtime tests have been performed.
