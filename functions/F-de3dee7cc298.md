<!-- Trusted provenance: F-de3dee7cc298; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de3dee7cc298

## Purpose

F-de3dee7cc298 accepts an opaque 32-bit value, performs a sequence of indirect dependency interactions and bit-field transformations, modifies memory through computed and fixed addresses, and finishes with an unresolved four-argument dispatch.

The supplied representation establishes access widths, operand relationships, retained observations, and ordering between separately represented interactions. It does not establish a domain-level purpose. Allocation, rendering, serialization, and container maintenance are not established interpretations.

This contract distinguishes:

- **Established static evidence:** behavior directly represented in the supplied material.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** unresolved behavior for which this contract supplies no implementation assumption.

Completion requires every reached dependency to return and every reached memory access to succeed. No runtime tests have been performed.

## Interface

### Explicit input and result

The explicit input is one opaque **32-bit value**. The initial dependency receives all 32 bits unchanged. No preceding validation, narrowing, or input dereference is represented.

The declared result is **void**. There is no explicit success indicator, error result, or returned object. Observable outputs consist of dependency interactions, memory accesses, fixed-address publications, and the final dispatch.

### Implicit inputs and storage

Execution additionally depends on incoming control-flow state, calculated indirect targets, dependency results, existing memory, and global observations.

Explicit direct accesses have these widths:

| Access or value | Established width |
|---|---|
| Explicit input | 32 bits |
| Ordinary scalar intermediates | Generally 32 bits |
| Paired dependency results | 64 bits, including separately consumed 32-bit halves |
| Explicit byte accesses | 8 bits |
| Explicit halfword stores | 16 bits |
| Explicit word accesses | 32 bits |

A byte store retains the low eight bits; a halfword store retains the low sixteen. Explicit unsigned byte loads expand to values from 0 through 255.

The global associated with **0x05** is converted to unsigned 32 bits. Its declaration, underlying access width, and original signedness are **documented_only**. Its address does not justify treating its value as an unsigned byte.

### Arithmetic and preconditions

Unsigned masks and logical right shifts have direct bit-pattern meanings. Some additions, multiplications, and left shifts use signed-looking 32-bit quantities. Numerical examples below conditionally assume truncation to 32 bits at intermediate operations; they do not establish signed-overflow, address-wrapping, or invalid-shift semantics.

Intermediate values are not indefinitely wide integers. Under that truncating model, two successive left shifts by 28 bits of a retained 32-bit value produce zero.

Every reached indirect target must support its represented interaction. Every reached memory access must be supported by the execution environment. No visible check establishes mapping, executability, bounds, alignment, or lifetime.

No ownership transfer is established for the input, dependency results, or computed storage. Unknown dependencies are not assumed to preserve globals, retain memory, or allocate or release it.

## Behavior

### Dependency-driven phase

The initial interaction receives the unchanged input. Two further indirect interactions follow, the latter supplying a paired result. The next interaction receives three operands: the lower half shifted left eight bits, the upper half unchanged, and the upper half shifted left six bits.

The remaining dependency phase combines fresh scalar and paired results with retained earlier transformations. Established relationships include:

- Scalar results shifted left eight bits.
- Paired-result lower halves shifted eight bits alongside upper halves.
- Multiple differently shifted forms of one result.
- Explicit zero operands.
- Secondary machine-result values consumed without an ordinary assigned return value.
- Values retained across intervening calls.
- Masked fields subsequently shifted or reduced.

Low-field masks include widths of 11, 12, 13, 14, 17, 18, 19, 20, 21, 22, and 24 bits. Some extractions logically shift the complete 64-bit result right by 34, 36, or 44 bits. A complete-result shift by 36 corresponds to the upper half shifted right four bits, not the lower half shifted right four bits.

Several retained values are explicitly zeroed and later shifted without a replacement assignment. Secondary results, however, are not assumed zero or equivalent to unrelated results.

Calls remain part of the interaction trace even when their ordinary results are unused. Repeated argument shapes do not establish identical targets. The complete occurrence-by-occurrence dependency trace is **documented_only** here.

No explicit loop or conditional branch appears in the represented sequence. Dependency-internal control flow and termination are unknown.

### Opening direct updates

The following descriptive labels identify retained relationships, not object types:

- **Pair-low / Pair-high:** halves of the final paired result before direct updates.
- **Link-base:** a retained scalar shifted left 25 bits.
- **Scaled-base:** a different retained scalar multiplied by \(2^{25}\).
- **Shifted-value:** another retained scalar shifted left eight bits.
- **Selector:** a scalar shifted left eight bits, masked with 0x7fff, then logically shifted right six bits.
- **Selector-base:** twice Selector.

Under 32-bit truncation, Link-base and Scaled-base have zero low bytes. Selector ranges from 0 through 508 in multiples of four; Selector-base ranges from 0 through 1016 in multiples of eight.

The opening stores have this order: a word at Pair-high plus 0x14 receives Link-base; a byte at Link-base plus nine receives Pair-high’s low byte; a word at 256 times Pair-low plus 0x54 receives Scaled-base; and the same Link-base byte receives Link-base’s low byte. The last value is conditionally zero, but the earlier byte store remains a distinct represented effect.

Next, Scaled-base plus 0x14 is cleared as a word. An unsigned byte read at Scaled-base plus five precedes a word clear at Selector-base plus 0x14. The expanded byte is then stored as a word at Scaled-base plus 0x34.

A word read at Scaled-base plus 0x14 supplies **Cleared-field-observation**. It cannot universally be replaced with zero: intervening writes may overlap, and access semantics are unresolved. Another word clear at Selector-base plus 0x14 precedes two consecutive byte stores at Selector-base plus 0x15, both carrying this observation’s low byte.

### Low-address conversion and publications

The 0x05-associated global is read and converted to unsigned 32 bits, supplying **Low-global-base**.

Immediately afterward, fixed address **0x01** receives Selector-base’s low byte. The following byte store at Low-global-base plus 0x15 instead receives Selector’s low byte.

Scaled-base plus 0x11 next receives Shifted-value’s low byte. A word store at Cleared-field-observation plus 0x54 carries Shifted-value, after which Scaled-base plus 0x11 receives zero.

A word read at Low-global-base plus 0x54 supplies **Saved-link**. The same location is overwritten with Cleared-field-observation. Saved-link remains the retained earlier observation.

Selector-base plus 0x34 receives Selector as a word. A word read at Scaled-base plus 0x74 supplies a further base used for offset-0x54 word updates and offset-nine byte updates.

Two separate word reads at Selector-base plus 0x74 supply the destination base and data of one word store at the loaded base’s offset 0x54. Their relative evaluation order within that expression is not established. They must not be treated as necessarily equal observations.

Saved-link plus 0x24 receives Saved-link twice, with other computed updates interleaved.

The fixed-address word publication at **0x54** carries **twice Saved-link**, not a fresh observation of Low-global-base plus 0x54. Its boundary is ordered:

1. Selector-base plus 0x14 receives the earlier retained scalar multiplied by \(2^{26}\), as a word.
2. Fixed address 0x54 receives twice Saved-link, as a word.
3. Selector-base plus 0x14 receives Saved-link twice, as two word stores.

Two subsequent byte stores at Selector-base plus 0x15 carry the low byte of four times Saved-link. Only afterward does fixed address **0x15** receive Selector-base’s low byte.

Computed accesses are not assumed disjoint from these fixed addresses.

### Cross-linked and literal updates

The next region derives doubled, quadrupled, and eightfold values from Saved-link and fresh word observations. It also uses explicit unsigned byte observations as bases for later word reads.

Represented offsets include 0x04, 0x05, 0x09, 0x0d, 0x11, 0x14, 0x15, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74. Effects include self-valued stores, reciprocal byte updates, overwritten stores, and repeated reads. No uniform object layout follows from these offsets. The complete intermediate cross-linked trace is **documented_only** here.

A later loaded scalar supplies three literal destinations. In order, a byte at four times that scalar, a halfword at eight times that scalar, and a word at four times that scalar plus 0x20 each receive 0x3a.

For subsequent behavior:

- **Adjusted-scalar:** four times Saved-link minus 0x96.
- **Working-base:** four times Adjusted-scalar.
- **Anchor-base:** a separate global word loaded after the literal-update region.
- **Node-base:** twice the scalar supplying the three literal destinations.
- **Payload-base:** four times that same scalar.

Working-base plus 0x20 receives four word stores, in order: 0x20, 0x2f, an expanded unsigned byte observation from Working-base plus four, and 0x20. That byte is read after the 0x2f store and before the corresponding word store.

The constants do not establish text output or a string format.

### Repeated updates

After Anchor-base is loaded, fixed address **0x60** receives word 0x20.

The repeated region contains ordered four-store groups:

- A word at an eightfold base plus 0x54 receives Payload-base.
- A byte at Anchor-base plus 0x19 receives Working-base’s low byte.
- A byte at Working-base plus 0x11 receives Anchor-base’s low byte.
- A word at Anchor-base plus 0x34 receives the eightfold base.

The first group uses eight times Saved-link. Subsequent groups use eight times Adjusted-scalar. Anchor-base and the two byte values are retained, not reloaded for each group.

Exact repetition cardinality remains **documented_only** in this contract. Repeated effects are not authorized to collapse into one final-state update.

### Ordered 0x3a publication boundary

The transition out of the repeated region establishes this order:

1. Read and retain the word at fixed address **0xa9**.
2. Read and retain the word at Payload-base plus 0x74, called **Payload-observation**.
3. Publish a byte at fixed address **0x3a**, carrying the low byte of eight times Adjusted-scalar plus 0x30.
4. Read and retain the word at Node-base plus 0x14, called **Node-observation**.
5. Read and retain the word at Working-base plus four.

The 0x3a publication occurs between the first two and last two observations. Node-observation supplies the third final-dispatch operand. The Working-base-plus-four observation is later stored at Anchor-base plus 0x74.

### Final publications and fresh observations

Following this boundary, the ordered byte updates are: Node-base plus 0x0d receives Working-base’s retained low byte; Working-base plus eight receives 0x6c; Working-base plus 0x0d and then plus nine receive Anchor-base’s retained low byte. A word at Node-base plus 0x54 then receives Node-base.

The retained 0xa9 word’s low byte is published at **0x67**. A word read at Node-base plus 0x44 follows. The current 0x67-associated global is then read for the byte publication at **0x21**.

Working-base plus 0x10 receives the Node-base-plus-0x44 observation’s low byte. Anchor-base plus 0x54 receives that observation as a word. Next, Node-base plus one receives a byte derived by a signed-looking right shift of the observation, using Working-base’s low eight bits as the count. Anchor-base plus 0x54 then receives Working-base as a word.

A word read at Working-base plus 0x40 precedes these word stores: Anchor-base plus 0x74 receives the retained Working-base-plus-four observation; Working-base plus 0x10 receives the offset-0x40 observation; Anchor-base plus 0x44 receives Node-base.

A fresh read of the 0x67-associated global supplies a byte store at Working-base plus nine. Next, Node-base plus nine receives Working-base’s retained low byte, Working-base plus 0x44 receives the retained 0xa9 word, and a word read at Working-base plus 0x54 supplies **Forward-base**.

Another fresh read of the 0x67-associated global supplies another byte store at Working-base plus nine. Neither read is implicit reuse of the initial publication. Overlapping computed writes may change the observed global value.

### Final dispatch

After that second later 0x67-based store, the following access boundary is explicitly ordered:

1. Forward-base plus 0x0d receives Payload-observation’s low byte.
2. Node-base plus 0x54 receives Node-observation as a word.
3. Working-base plus 0x14 is read as a word and retained as **Working-field-observation**.
4. Node-base plus 0x11 receives Payload-observation’s low byte.
5. Working-base plus nine receives Forward-base’s low byte.
6. Working-base plus 0x44 receives Forward-base as a word.
7. Node-base plus four is read as a word, supplying **Later-base**.

The Working-base-plus-nine overwrite belongs between the Node-base-plus-0x11 byte store and the Working-base-plus-0x44 word store. It is not an unordered later effect.

These accesses cannot be regrouped by destination or payload. In particular, the Node-base-plus-0x54 word store precedes Working-field-observation, whereas the Node-base-plus-0x11 byte store follows that observation. Potentially overlapping addresses make this distinction material.

A halfword containing 0x65 is then written at twice Later-base. A word read at Later-base plus 0x64 supplies **Target-word**.

Forward-base plus 0x11 receives Working-field-observation’s low byte. Forward-base plus 0x54 receives decimal 100 as a word. A subsequent word read at Later-base plus 0x74 supplies the fourth dispatch operand.

A word read at Node-base plus 0x64 supplies another base. Node-base plus nine then receives Later-base’s low byte. The newly loaded base’s offset 0x54 receives Target-word as a word, and twice Target-word receives the newly loaded base as a word.

The final unresolved dispatch receives these four operands, in order:

1. 0x72.
2. Target-word.
3. Node-observation, retained after the 0x3a publication.
4. The later observation from Later-base plus 0x74.

A return is represented if the dispatch returns normally.

## Side effects

Established effects are indirect dependency interactions, mixed-width memory accesses, fixed-address and global observations, overwritten and repeated stores, and final dispatch.

No transaction, rollback, retry, cleanup handler, or recovery path is visible. A non-returning dependency prevents subsequent actions. A fault may leave earlier completed effects in place; precise fault behavior belongs to the environment.

Thread safety, reentrancy, atomicity, synchronization, and externally observable access semantics are not established. Computed storage may overlap other computed storage, fixed addresses, or globals. Matching final contents alone does not establish the required represented access order.

## Dependencies

These document-local opaque IDs identify interaction boundaries, not resolved implementation identities.

| ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-000000000001 | Initial indirect interaction receiving the unchanged input. | Target, argument meaning, effects, ownership, errors. |
| F-000000000002 | Collection of subsequent pre-memory interactions consuming or supplying scalar, paired, and secondary results. | Individual identities, full signatures, equivalence, preservation, lifetime effects. |
| F-000000000003 | Final dispatch receiving the four ordered operands specified above. | Target, calling convention, effects, termination, return behavior. |

The collection ID does not identify one interchangeable helper. No dependency is assumed to preserve globals or to own or release memory.

## Acceptance examples

These are conditional static obligations, not executed tests.

### Input and paired values

For input 0x89abcdef, F-000000000001 receives 0x89abcdef.

If the initial paired-producing interaction supplies lower half 0x12345678 and upper half 0x00000003, the next three operands are 0x34567800, 0x00000003, and 0x000000c0 under 32-bit truncation.

### Distinct selector publications

If the selector-producing scalar has its low seven bits set, Selector is 508 and Selector-base is 1016. Fixed addresses 0x01 and later 0x15 receive byte 0xf8. Low-global-base plus 0x15 instead receives byte 0xfc.

The 0x01 publication follows the global conversion and precedes the computed byte store. The 0x15 publication follows both computed stores carrying four times Saved-link’s low byte.

### Retained saved value

If Saved-link is 0x12345678, fixed address 0x54 receives word 0x2468acf0 under the truncating model. The earlier overwrite of Low-global-base plus 0x54 does not replace the retained value.

### Load widths

An explicit byte observation of 0xff at Scaled-base plus five supplies word 0x000000ff at Scaled-base plus 0x34.

If conversion of the 0x05-associated global produces 0x12345678, Low-global-base retains that entire value. Its offset-0x54 address is conditionally 0x123456cc, not 0x000000cc.

### Publication-sensitive observations

If the 0x3a publication overlaps the subsequent Node-base-plus-0x14 word read, ordinary memory semantics require Node-observation to reflect the post-publication storage. The third dispatch operand uses that retained observation.

If the Node-base-plus-0x54 store overlaps Working-base plus 0x14, Working-field-observation is taken after that store. Conversely, the following Node-base-plus-0x11 byte store cannot retroactively change the retained observation, even if it overlaps the same word.

### Final access boundary

For Payload-observation 0x123456ab, both final payload byte stores carry 0xab. Their equal data does not permit adjacent publication: the Node-base-plus-0x54 word store and Working-base-plus-0x14 word read intervene.

After the second payload byte store, Working-base plus nine receives Forward-base’s low byte, then Working-base plus 0x44 receives Forward-base as a word, and only then is Node-base plus four observed.

### Fresh globals and partial execution

If the initial 0x67 publication carries 0x12, while the two later reads supplying Working-base-plus-nine stores observe 0x34 and 0x56, those stores carry 0x34 and 0x56. Reusing 0x12 is insufficient.

If an earlier dependency does not return, later direct updates and final dispatch need not occur. No restoration of earlier effects is established.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and intended object layouts.
- Complete pre-memory dependency and intermediate cross-linked traces.
- Indirect identities, signatures, and secondary-result provenance.
- Exact repeated-update cardinality.
- Correspondence between the representation and executable control flow.
- Missing declaration-level global details, especially for the 0x05-associated global.
- Byte order, alignment, mapping, and externally observable access semantics.
- Signed overflow, signed left shifts, address wrapping, and variable right-shift behavior, including oversized counts.
- Dependency preservation, ownership, allocation, release, and lifetime effects.
- External fault handling and final-dispatch return behavior.

A structure-maintenance interpretation is an **inference** suggested by recurring offsets and link-like updates, not required behavior.

## Evidence

This contract derives from the supplied static representation. Embedded comments, the prior draft, and reviewer feedback are not independent behavioral evidence.

The final access boundary directly establishes the separation of the two payload byte stores by a word store and a retained word read. It also places the Working-base-plus-nine overwrite before the Working-base-plus-0x44 store and the Node-base-plus-four read.

Other established evidence includes explicit access widths, paired-result extraction, retained observations, repeated stores, the placement of the 0x3a publication between retained reads, distinct selector publications, the fixed 0x54 publication from twice Saved-link, fresh 0x67 observations, and final argument order.

No tools, additional implementation inspection, or runtime tests were used. Acceptance examples express derived conditions only.
