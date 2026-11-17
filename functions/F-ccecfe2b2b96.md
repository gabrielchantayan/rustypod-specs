<!-- Trusted provenance: F-ccecfe2b2b96; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ccecfe2b2b96

## Purpose

F-ccecfe2b2b96 performs an ordered sequence of unresolved indirect invocations, then mixed-width memory mutations and pointer-like traversals, and finally a four-argument indirect dispatch.

No application-level purpose is established. Numeric constants resembling characters do not establish text output; repeated offsets do not establish object types; repeated stores do not establish collection insertion or iteration.

This contract distinguishes:

- **Established static evidence:** operations and ordering visible in the supplied representation.
- **Conditional deductions:** consequences of explicitly stated arithmetic and memory assumptions.
- **documented_only:** behavior not resolved sufficiently to make an executable guarantee.

No runtime tests have been performed.

## Interface

### Explicit inputs and output

| Position | Width and representation | Established use |
|---|---|---|
| First input | 32-bit integer | Forms early shifted dependency arguments. |
| Second input | Opaque 32-bit quantity | No explicit body use is represented. |
| Third input | 32-bit integer | Forms an early argument shifted left by four bits. |
| Output | No explicit return value | Observable results are dependency interactions and memory effects. |

The unused explicit second input does not prove independence from its incoming machine state. Implicit dependency inputs and calling conventions are unresolved.

### Implicit inputs

Execution also depends on an incoming return-link quantity, embedded target-forming quantities, dependency results, incompletely specified secondary register outputs, existing memory contents, and an unresolved final dispatch target.

Dependency results are represented as both 32-bit quantities and 64-bit quantities. Later operations select and transform their low or high 32-bit portions. Explicit function inputs alone therefore do not determine completion, addresses, written values, or dispatch arguments.

### Widths and arithmetic

A **byte**, **halfword**, and **word** mean 8, 16, and 32 bits respectively. Narrow stores retain the corresponding low bits. Byte loads widened into words are zero-extended, not sign-extended.

For conditional deductions below, the **32-bit machine interpretation** means arithmetic and left-shift results retain their low 32 bits, including address arithmetic. This is an interpretation of the represented machine operations, not a guarantee that every signed expression has independently established language-level semantics.

Unsigned right shifts are logical. A represented 64-bit result shifted right by 34, 36, or 44 selects its high word shifted logically by 2, 4, or 12 bits. These operations must not be interpreted as oversized shifts of a 32-bit operand.

Byte order, alignment enforcement, signed-overflow semantics, and oversized variable-shift behavior remain **documented_only**.

### Preconditions, errors, and ownership

Successful completion requires every reached indirect target to be callable and every reached memory location to support its access width. Loaded pointer-like words must support subsequent accesses. Dependencies must supply sufficiently defined result state.

The body does not visibly validate these conditions. It provides no established error code, rollback, null check, bounds check, or dependency-result check.

No allocation, deallocation, ownership transfer, reference counting, or caller-owned buffer contract is established. Unknown dependencies cannot be assumed to preserve globals or to acquire, retain, or release memory.

## Behavior

### Dependency-driven phase

Before the first explicit data-memory store, the function makes a long ordered series of indirect invocations.

The first invocation receives two equal arguments: the first input shifted left by seven bits. The second receives two zeros. After two additional invocations, a three-argument invocation receives:

1. The third input shifted left by four bits.
2. An unresolved secondary result.
3. The first input shifted left by 28 bits.

Later arguments combine earlier results through shifts, masks, duplicated argument positions, and high-/low-word selection. Calls with no explicit arguments remain part of the interaction sequence; their effects and implicit inputs are unknown. No visible branch checks whether an invocation succeeded before proceeding.

Under the 32-bit machine interpretation:

- Shifting the first input left by 28 and then by four yields zero. The subsequent extraction from bits 10–19 of that result also yields zero.
- Explicitly zero-initialized quantities remain zero through their shown shifts and masks until replaced.
- Discarded high bits are not recovered by later shifts.
- Masks `0x00ffffff`, `0x003fffff`, `0x000fffff`, and `0x0007ffff` retain 24, 22, 20, and 19 low bits respectively.
- Masking before shifting is not generally interchangeable with masking afterward.

Equal argument values do not make separate argument positions or separate calls redundant. Replacing this phase with pure arithmetic would discard possible dependency effects.

The complete reproducible invocation contract remains **documented_only**, because target identities, signatures, implicit inputs, and secondary outputs are unresolved.

### Address and value vocabulary

The following terms describe relationships, not validated objects or ownership:

| Term | Definition |
|---|---|
| Primary base | A late 32-bit dependency result multiplied by \(2^{25}\). |
| Selection quantity | A different late result shifted left by eight, masked to 15 low bits, then shifted logically right by six. |
| Small base | Twice the selection quantity. |
| Shifted base | The low word of another dependency result shifted left by 25. |
| Retained shifted word | A late 32-bit dependency result shifted left by eight and retained into the mutation phase. |
| Link value | The saved word read at offset `0x54` through a base obtained by reading fixed byte address `0x05`. |
| Reduced scale | Four times the link value, minus 150. |
| Adjusted base | Four times the reduced scale. |
| Anchor | A word read from a separate fixed global location. |
| Lookup result | The final pointer-like word obtained through a fresh byte-at-five lookup before the three `0x3a` stores. |
| Traversal base | Twice the lookup result. |
| Output base | Four times the lookup result. |

Under the 32-bit interpretation, the selection quantity is four times the low seven bits of its original result. The small base is therefore a multiple of eight from 0 through 1016. The primary and shifted bases have their low 25 bits clear. These alignments do not establish accessibility.

Offsets are byte offsets. Word accesses include `0x04`, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, and `0x74`. Nearby byte accesses can overlap word locations. No structure size or general non-aliasing guarantee follows.

### Initial mutation and the established zero read

The last result-producing invocation supplies a represented 64-bit result. The initial mutations are a word store at its high word plus `0x14`, a byte store at shifted base plus nine, a word store at the low word scaled by 256 plus `0x54`, and another byte store at shifted base plus nine.

The first byte uses the high result word’s low byte. The second uses the shifted base’s low byte, which is zero under the 32-bit interpretation. Both writes remain observable.

The following access group establishes an important conditional invariant:

- A zero word is written at primary base plus `0x14`.
- A byte is read at primary base plus five.
- A zero word is written at small base plus `0x14`.
- The zero-extended byte is written as a word at primary base plus `0x34`.
- The word at primary base plus `0x14` is read.
- Another zero word is written at small base plus `0x14`, followed by two byte writes at small base plus `0x15` using the saved read’s low byte.

**Under ordinary stable-memory semantics and the 32-bit interpretation, the saved word read is zero.** The small-base word store is either disjoint from the primary-base word or coincides with it and also writes zero. Both word addresses are congruent to four modulo eight, excluding partial overlap. The primary-base-plus-`0x34` word cannot overlap primary-base-plus-`0x14`.

Unknown device-backed behavior or concurrent modification can invalidate the stable-memory deduction. Unresolved aliasing among these preceding stores, by itself, does not.

The function next reads fixed byte `0x05`, then writes a byte at fixed address `0x01`. It writes through the byte-derived base plus `0x15`, writes the retained shifted word’s low byte at primary base plus `0x11`, writes the retained shifted word through the saved initial word plus `0x54`, and clears primary base plus `0x11`.

The link value is then read from the byte-derived base plus `0x54` before that word location is overwritten with the saved initial word. Later operations use the saved link value, not a reload after this overwrite.

### Traversal and scaled-address mutations

This region uses the link value, small base, retained quantities, and fresh memory reads to form further destinations.

Its observable features include:

- A primary-base-plus-`0x74` load followed by writes through that loaded address at offsets nine and `0x54`.
- Two represented word reads at small base plus `0x74` within a self-valued store through a loaded address plus `0x54`. Their relative evaluation order within that expression is not established.
- Repeated self-valued word writes at link value plus `0x24`.
- Successive small-base-plus-`0x14` stores of an intermediate shifted value and then the saved link value twice.
- A fixed word write at `0x54` containing twice the link value, and a fixed byte write at `0x15` containing the small base’s low byte.
- Computed stores based on four or eight times the link value.
- Fresh byte reads at offset five, widened into bases for word reads at offset `0x74`.
- Word mutations at offsets `0x14`, `0x34`, `0x44`, and `0x54`, interleaved with byte mutations at nine, `0x0d`, and `0x15`.

The repeated byte-at-five reads occur at distinct positions among writes. Their values must not be specified as one cached byte. Likewise, repeated pointer-like word reads cannot generally be merged across intervening mutations.

The last such lookup establishes the lookup result, traversal base, and output base. Three subsequent stores write numeric `0x3a`: a byte at the output base, a halfword at eight times the lookup result, and a word at output base plus `0x20`. Their widths are distinct; no string interpretation is established.

### Adjusted-base writes and anchor acquisition

The adjusted base is derived from the link value by scaling, subtraction of 150, and further scaling.

At adjusted base plus `0x20`, four word writes occur in this order:

1. Numeric `0x20`.
2. Numeric `0x2f`.
3. A zero-extended byte freshly read at adjusted base plus four.
4. Numeric `0x20` again.

**After the fourth write, the anchor is loaded from its fixed global location. Only then is numeric `0x20` written as a word at fixed address `0x60`.**

This ordering matters even if the final contents appear unchanged. The anchor is neither acquired before the four-write sequence nor reloaded after the fixed `0x60` write.

### Repeated anchor updates

The next region contains an explicitly repeated four-store pattern involving stable retained values:

- A word store at a scaled address plus `0x54`, containing the output base.
- A byte store at anchor plus `0x19`, containing the adjusted base’s low byte.
- A byte store at adjusted base plus `0x11`, containing the anchor’s low byte.
- A word store at anchor plus `0x34`, containing a scaled value.

The first group uses eight times the link value for the scaled destination base and final stored value. Subsequent groups use eight times the reduced scale instead.

No per-group pointer advance is represented. The region establishes repeated writes to the represented locations, not traversal of an array. Exact multiplicity is **documented_only** in this contract; no numerical count is claimed. Collapsing these writes to final memory contents is not established as equivalent.

### Final fixed accesses and saved values

After the repeated region, the following ordering is explicit:

1. Read the fixed word at `0xa9`.
2. Read the word at output base plus `0x74`.
3. Write fixed byte `0x3a`, using the low eight bits of eight times the reduced scale plus `0x30`.
4. Save the word at traversal base plus `0x14`.
5. Save the word at adjusted base plus four.

The latter two reads therefore occur **after** the fixed `0x3a` mutation. They cannot be moved before it merely because their values are retained.

Subsequent byte writes place the adjusted base’s low byte at traversal base plus `0x0d`, numeric `0x6c` at adjusted base plus eight, and the anchor’s low byte at adjusted base plus `0x0d` and nine. A self-valued word store then targets traversal base plus `0x54`.

### Fresh reads of fixed byte `0x67`

The function writes the saved fixed-`0xa9` word’s low byte at fixed address `0x67`. It then reads a word at traversal base plus `0x44`, retaining that word for later use. **After that word read**, a fresh read of byte `0x67` supplies the byte write at fixed address `0x21`.

Before the next `0x67` read, the function:

- Writes the retained word’s low byte at adjusted base plus `0x10`.
- Writes that retained word at anchor plus `0x54`.
- Writes a shifted byte at traversal base plus one.
- Writes the adjusted base at anchor plus `0x54`.
- Reads adjusted base plus `0x40`.
- Writes the earlier saved adjusted-base-plus-four word at anchor plus `0x74`.
- Transfers the saved `0x40` word to adjusted base plus `0x10`.
- Writes the traversal base at anchor plus `0x44`.

A **second fresh read** of fixed byte `0x67` then supplies a byte store at adjusted base plus nine.

Next, a byte containing the adjusted base’s low byte is written at traversal base plus nine; the saved fixed-`0xa9` word is written at adjusted base plus `0x44`; and a word is read from adjusted base plus `0x54`. A **third fresh read** of fixed byte `0x67` then supplies another byte store at adjusted base plus nine.

Each fixed-byte read observes memory at its own position. Intervening computed stores may alias `0x67`; the two later adjusted-base-plus-nine values are not guaranteed to equal the originally written byte or each other.

The two anchor-plus-`0x54` stores use **different value expressions**: first the saved word from traversal base plus `0x44`, then the adjusted base. Their numerical inequality is not established.

The intervening shift uses the saved word as a signed operand and the adjusted base’s low byte as its count. Results for unsupported counts, and the applicable signed-right-shift rule, remain **documented_only**.

### Final transfers and dispatch

The word loaded from adjusted base plus `0x54` becomes a later destination base. The function uses the saved output-base-plus-`0x74` word’s low byte for byte writes through that destination and through traversal base plus `0x11`. It restores the saved traversal-base-plus-`0x14` word at traversal base plus `0x54`.

A fresh adjusted-base-plus-`0x14` word is retained before further writes. Adjusted base plus nine and `0x44` are then overwritten using the later destination base.

A word read at traversal base plus four supplies a doubled address for a halfword store of `0x65`. Subsequent word reads use that loaded base plus `0x64` and `0x74`. Between them, byte and word writes through the later destination include its offset `0x11` and numeric 100 at its offset `0x54`.

The function then reads traversal base plus `0x64`, performs the remaining byte and word transfers, and writes the latter loaded address as a word through twice the saved `0x64` result.

The final indirect invocation receives four arguments, in order:

1. Numeric `0x72`.
2. The saved word loaded through offset `0x64` of the base obtained from traversal base plus four.
3. The word saved earlier from traversal base plus `0x14`.
4. The saved word loaded through offset `0x74` of that same loaded base.

A return is represented after the invocation. Whether its target returns normally is unknown.

## Side effects

Effects include dependency interactions, fixed-low-address accesses, computed-address mutations, duplicate writes, mixed-width overlap, and saved-value transfers.

There is no established confinement to input-owned storage. Earlier writes may remain visible if a later access fails. No rollback or transactional boundary is represented.

Dependencies may independently modify globals, memory, devices, ownership, or execution state. No explicit synchronization or exception handling is established. Thread safety, reentrancy, atomicity, and ordinary user-space suitability remain **documented_only**.

## Dependencies

**F-opaque-001** identifies the unresolved indirect-invocation family before mutation. It does not imply one common target. Represented interactions have zero through four explicit arguments, 32-bit and 64-bit results, and incompletely specified secondary outputs. Initial target formation depends on incoming execution state; later target formation commonly uses embedded quantities.

**F-opaque-002** identifies the final dispatch site. Its four explicit arguments are specified above. Target identity, calling convention, memory effects, ownership effects, failure behavior, and return behavior are unknown.

Neither identifier denotes a resolved implementation.

## Acceptance examples

These examples are conditional static deductions, not executed tests.

### Early arguments

With first input 1, third input 2, 32-bit arithmetic, and returning dependencies, the first invocation receives 128 twice. The early three-argument invocation receives 32 first and `0x10000000` third; its second argument remains unresolved.

### Small-base boundary

If the original result forming the selection quantity is 127, that quantity is 508 and the small base is 1016. If the result is 128, both are zero. This is bit selection, not saturation.

### Initial zero invariant

With stable ordinary memory and successful accesses, the initial primary-base-plus-`0x14` read returns zero even when the small-base zero store coincides with it. A nonzero result requires behavior outside those assumptions, such as concurrent modification or device semantics.

### Widening

A reached byte read yielding `0xfe` supplies word value 254 when zero-extended, not −2.

### Ordering

A distinguishing observer must see the four adjusted-base-plus-`0x20` writes in order, followed by anchor acquisition, then the fixed `0x60` word write. Moving anchor acquisition after the fixed write does not satisfy this ordering.

### Fresh fixed-byte values

If the three reads of byte `0x67` observe `0x12`, `0x34`, and `0x56`, the associated writes supply `0x12` to fixed `0x21`, then `0x34` and `0x56` to the two later adjusted-base-plus-nine stores. No cached-value requirement applies.

### Equal anchor-store values

If the saved traversal-base-plus-`0x44` word equals the adjusted base, both anchor-plus-`0x54` stores write the same numeric value. Both represented writes still occur.

### Inaccessible address

A reached inaccessible location does not imply a graceful error return, rollback, or final dispatch. Earlier effects may already have occurred.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid input domain.
- Dependency identities, signatures, implicit inputs, secondary outputs, and effects.
- Complete externally reproducible dependency trace.
- Signed arithmetic and variable-shift semantics not fixed by the supplied evidence.
- Byte order, alignment rules, device behavior, and concurrent-memory behavior.
- Object boundaries, general alias relationships, and ownership.
- Numerical multiplicity of the repeated anchor-update pattern in this contract.
- Executability of the low-address accesses in the intended environment.
- Final target identity and normal-return behavior.

These uncertainties do not negate the conditional initial-zero deduction or the explicitly established fixed-access ordering.

## Evidence

The contract is based solely on static inspection of the supplied representation: explicit input widths, indirect interactions, arithmetic dependencies, mixed-width accesses, saved reads, fixed-address effects, repeated mutations, and final argument formation.

The initial-zero conclusion follows from the preceding zero stores, base alignment, and disjoint primary offsets under ordinary stable-memory semantics. The anchor and final fixed-access orderings follow from their represented positions. The `0x67` behavior follows from three distinct reads separated by the described operations. No evidence establishes inequality between the two anchor-plus-`0x54` value expressions.

Semantic terms in this contract identify relationships only. No dependency implementation, external layout definition, runtime trace, or test result was available. No runtime tests have been performed.
