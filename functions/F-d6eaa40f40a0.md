<!-- Trusted provenance: F-d6eaa40f40a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6eaa40f40a0

## Purpose

F-d6eaa40f40a0 performs an ordered sequence of indirect dependency invocations, transforms dependency-derived machine values, updates shared memory through mixed-width accesses, and makes a final unresolved indirect transfer.

The established output is an effect trace, not a returned calculation. No explicit arguments or returned value are represented. Execution nevertheless depends on incoming machine state, dependency results, shared-state observations, and the validity of derived addresses.

No application-level purpose is established. Numeric constants resembling characters do not establish text output, serialization, or a particular data model.

This contract distinguishes:

- **Established static evidence:** represented invocations, conversions, access widths, dependencies between values, and ordering.
- **Conditional consequences:** behavior requiring explicitly stated arithmetic, memory, or symbol-type assumptions.
- **documented_only:** unresolved behavior for which the supplied evidence does not support a definite acceptance rule.

No runtime tests have been performed.

## Interface

### Inputs and outputs

| Property | Established interface |
|---|---|
| Identity | F-d6eaa40f40a0 |
| Explicit parameters | None represented |
| Declared result | No value |
| Implicit inputs | Incoming return-link state, dependency results, secondary machine-result state, and shared memory |
| Observable outputs | Indirect invocations, shared-state accesses, memory updates, and the final indirect transfer |
| Explicit error result | None represented |
| Completion | A return is represented after the final transfer, conditional on that transfer returning |

The first indirect destination is derived from incoming return-link state plus 0xFFE. Subsequent destinations are derived from embedded locations plus the same offset. An ordinary standalone invocation with arbitrary incoming machine state is therefore not an established supported interface.

Secondary machine-result values feed later arguments without supplied defining interfaces. They must not be assumed zero or equated with ordinary primary results.

### Widths and representations

**Byte**, **halfword**, and **word** mean 8, 16, and 32 bits respectively. The representation also contains 64-bit dependency results whose low and high words are consumed separately.

Explicit byte accesses retain or obtain eight bits. Explicit unsigned byte reads promoted to words are zero-extended: 0xE1 becomes 225. This rule does **not** independently establish the type of an undeclared global merely because its value is converted to an unsigned word.

In particular, the global associated with absolute address **0x05** is converted to a 32-bit unsigned value, but its declaration is absent. Its source width, signedness, physical read width, and any extension preceding the conversion remain unresolved. The converted value is called the **global-derived base** below; it is not unconditionally a zero-extended byte or restricted to 0–255.

Offsets are numeric address displacements, not recovered structure fields. Byte order, alignment behavior, object boundaries, and the accessibility of low memory are unspecified.

### Preconditions and ownership

Every reached indirect destination must support the represented invocation, and every reached memory access must be supported at its actual address and width. No validation is represented.

Derived accesses can involve absolute low-memory addresses, unaligned words, scaled dependency results, and values loaded from memory. The global-derived base may lie outside the byte range because its source type is unknown.

No allocation, release, reference counting, or ownership transfer is established. Address-valued results are not proven live, owned, or stable. Unknown dependencies may impose lifetime requirements or change ownership; those requirements remain documented_only.

## Behavior

### Execution order and arithmetic

The represented body consists of an invocation-heavy region, a direct-memory region, and a final indirect transfer. No explicit conditional branch or runtime loop appears in the supplied body. Dependencies may branch, fault, or fail to return.

Invocation order matters. Calls with unused results, zero-valued arguments, or no represented arguments remain observable interactions.

For conditional arithmetic examples, a **32-bit bit-vector model** means that word arithmetic and derived addresses retain their low 32 bits. This model does not establish the behavior of every represented signed expression on the actual target. Signed overflow, signed left shifts, signed conversions, and exceptional shift counts require target semantics.

Relevant boundaries include:

- Byte truncation retains eight bits rather than converting to a truth value.
- A word shifted left eight bits and then logically right eight bits retains only its original low 24 bits.
- Successive word left shifts totaling at least 32 discard all original bits under the bit-vector model.
- Wide-result extraction by 34, 36, or 44 bits must operate on the wide value, not merely its low word.
- Subtraction of 0x96 occurs before the later fourfold and eightfold scaling.
- A late right shift uses the low eight bits of a derived address as its count. Counts of 32 or greater, and right shifts of negative signed values, remain target-dependent.

### Dependency-derived values

Dependency results are shifted, masked, split into high and low words, retained across intervening calls, and duplicated into multiple argument positions. Explicit zero arguments are interspersed with result-derived arguments.

The following descriptive labels identify values used in the direct-memory region; they are not source identifiers or recovered object names.

| Label | Definition |
|---|---|
| **Token base** | Low word of an earlier wide dependency result shifted left 25 bits |
| **Aligned base** | A later word result multiplied by 2²⁵ |
| **Extracted index** | A result shifted left eight bits, masked with 0x7FFF, then logically shifted right six bits |
| **Doubled index** | Twice the extracted index |
| **Shifted token** | A separate retained word result shifted left eight bits |
| **Final high address** | High word of the last wide dependency result before direct accesses |
| **Final low address** | Low word of that result multiplied by 256 |

Under the bit-vector model, the token base and aligned base have their low 25 bits clear. The extracted index is a multiple of four from 0 through 508; the doubled index is a multiple of eight from 0 through 1016. The shifted token has its low eight bits clear.

These values have distinct origins even when their numeric values coincide.

### Initial memory updates

The initial ordered effects include:

1. A word at final high address plus 0x14 receives the token base.
2. A byte at token base plus 9 receives the low byte of the final high address.
3. A word at final low address plus 0x54 receives the aligned base.
4. The byte at token base plus 9 receives the low byte of the **token base itself**.

The fourth effect is not sourced from the shifted token. Under the bit-vector model its stored byte is zero.

Next, the word at aligned base plus 0x14 is cleared before an explicit byte read at aligned base plus 5. The word at doubled index plus 0x14 is cleared, and the sampled byte is written, zero-extended, as a word at aligned base plus 0x34.

The word at aligned base plus 0x14 is then sampled. After another clear at doubled index plus 0x14, its low byte is stored twice at doubled index plus 0x15.

These statements describe ordered observations and updates, not immutable field values. Address overlap, external modification, and memory-specific behavior can affect later observations.

### Global-derived traversal and shared-state assignment

After that first pair of identical byte stores, the global associated with absolute address 0x05 is evaluated and converted to an unsigned word. That converted value becomes the global-derived base.

The following uses all retain the source-type qualification:

- A byte destination is formed at global-derived base plus 0x15.
- A word is read at global-derived base plus 0x54.
- That same word destination is subsequently overwritten.
- The word obtained there supplies later traversal and scaling values.

No unconditional byte-sized read, zero-extension, or 0–255 address range is established for this global evaluation.

After obtaining the global-derived base, absolute address 0x01 receives a byte-valued assignment containing the low byte of the doubled index. An explicit byte store at global-derived base plus 0x15 receives the low byte of the extracted index.

The shifted token supplies a byte store at aligned base plus 0x11 and a word store through the previously sampled aligned-base word plus 0x54. The byte at aligned base plus 0x11 is then cleared. Both byte stores remain represented even though the shifted token’s low byte is already zero under the bit-vector model.

The word at global-derived base plus 0x54 is read before being replaced by the earlier aligned-base sample. Its original value is retained as the **traversed value** for subsequent operations.

### Traversal updates and the assignment at absolute address 0x15

The extracted index is stored as a word at doubled index plus 0x34. A word read at aligned base plus 0x74 supplies another destination base.

Subsequent effects include:

- Successive word updates at that destination base plus 0x54.
- An update whose destination and stored value each require a separate word read at doubled index plus 0x74. The relative order of those two reads is not established.
- Two word stores of the traversed value to traversed value plus 0x24.
- Two byte stores of its low byte at the other traversed base plus 9.
- A word assignment at absolute address 0x54 containing the traversed value shifted left one bit.
- Successive word stores at doubled index plus 0x14, first of an aligned-result-derived value and then twice of the retained traversed value.

The traversed value is then scaled by four. Its low byte is stored **twice** at doubled index plus 0x15. This is the second pair of identical stores at that particular derived destination; its source differs from the earlier aligned-base sample.

**Immediately after this second pair, absolute address 0x15 receives a byte-valued assignment containing the low byte of the doubled index.** The assigned expression is not the low byte of the fourfold-scaled traversed value.

This absolute assignment precedes the subsequent scaled-address word updates, including the word store through the fourfold-scaled traversed value plus 0x54 and the fourfold-value store at doubled index plus 0x34. It cannot be omitted merely because absolute address 0x15 might alias another destination.

### Scaled-address traversals

The next region uses fourfold and eightfold scaling of the retained traversed value, plus further values loaded through offsets 4, 0x14, 0x44, 0x64, and 0x74.

Observable behavior includes self-address word stores, scaled-address word stores, low-byte stores at offsets 9, 0x0D, and 0x15, and word stores of explicitly sampled unsigned bytes.

An explicit byte read at offset 5 can become the numeric base for a word read at offset 0x74. Unlike the undeclared global at 0x05, these explicit byte reads establish eight-bit unsigned values. They do not establish a conventional pointer encoding.

Repeated byte reads at the same represented offset remain separate observations. Intervening writes, aliasing, or memory behavior can make their values differ.

### Constant stores and repeated updates

A further traversed word supplies destinations for numeric constant 0x3A:

- A byte store at its fourfold-scaled address.
- A halfword store at its eightfold-scaled address.
- A word store at its fourfold-scaled address plus 0x20.

The earlier fourfold-scaled traversed value is reduced by 0x96. This is the **adjusted value**. Its fourfold-scaled address is the **adjusted fourfold base**.

At adjusted fourfold base plus 0x20, the ordered word values are:

**0x20, then 0x2F, then a zero-extended byte sampled at adjusted fourfold base plus 4, then 0x20.**

The byte sample occurs after the 0x2F store and before the word store carrying the sampled value.

A separate shared word is then sampled and retained as the **shared base**, followed by a word-valued assignment of 0x20 at absolute address 0x60.

An extended region repeatedly interleaves four effects in this order:

- A word store at an eightfold-scaled destination plus 0x54.
- A byte store at shared base plus 0x19, containing the low byte of the adjusted fourfold base.
- A byte store at adjusted fourfold base plus 0x11, containing the low byte of the shared base.
- A word store at shared base plus 0x34, containing the corresponding eightfold-scaled value.

The first group uses the pre-subtraction eightfold traversed value. Later groups use eight times the adjusted value. The word supplied to the first destination remains a separate, further-traversed value scaled by four.

The groups are explicitly repeated without a represented runtime loop. Exact multiplicity is not enumerated in this contract and remains documented_only here; no equivalence based solely on their final memory contents is established.

### Final updates and transfer

After the repeated region, a word-valued global at absolute address 0xA9 is sampled. A word is also loaded at offset 0x74 through the retained further-traversed fourfold address.

Absolute address 0x3A receives a byte-valued expression equivalent, under the bit-vector model, to the low byte of eight times the adjusted value plus 0x30.

The remaining region includes:

- A retained word read at offset 0x14 of a doubled traversed base.
- A word read at adjusted fourfold base plus 4.
- A byte store of 0x6C at adjusted fourfold base plus 8.
- Low-byte stores at offsets 9, 0x0D, 0x10, and 0x11.
- A byte-valued assignment at absolute address 0x67 from the sampled 0xA9 word, followed by a copy through absolute address 0x21.
- A byte store of a variable-right-shift result at doubled traversed base plus 1.
- Successive word updates at offsets 0x44, 0x54, and 0x74.
- A word store at adjusted fourfold base plus 0x10 after an earlier byte store at that offset.
- A halfword store of 0x65 through twice a loaded address.
- A word store of 100 at a traversed base plus 0x54.
- A final word store through twice a loaded word.

Later uses of the global at 0x67 occur after intervening writes. They must not be replaced automatically with the earlier sampled low byte: aliasing may affect those observations.

The final indirect transfer has four represented arguments:

1. 0x72.
2. A word loaded near the end at offset 0x64 of a traversed base.
3. The retained word sampled earlier at offset 0x14 of the doubled traversed base.
4. A word loaded near the end at offset 0x74.

Its destination and effects are unresolved. The represented return is conditional on this transfer returning.

## Side effects

Direct memory effects include explicit byte, halfword, and word accesses. Shared-state effects also include:

| Absolute address | Established operation |
|---|---|
| 0x01 | Assignment of the doubled index’s low byte |
| 0x05 | Evaluation of an undeclared global, followed by unsigned-word conversion; source width and signedness unresolved |
| 0x15 | Assignment of the doubled index’s low byte after the second pair of stores at doubled index plus 0x15 |
| 0x21 | Assignment from the global at 0x67 |
| 0x3A | Byte-valued arithmetic assignment |
| 0x54 | Word-valued assignment from the shifted traversed value |
| 0x60 | Word-valued assignment of 0x20 |
| 0x67 | Low-byte assignment and later observations |
| 0xA9 | Word-valued observation |

Global declarations are not supplied. Assignment-expression width must not be used indiscriminately to infer a global’s physical access width or its relationship to overlapping symbols.

Unknown dependencies may modify shared memory and machine state. They are not assumed pure or globally preserving.

No atomicity, locking, rollback, transaction boundary, or recovery path is established. A fault or non-returning dependency may leave a completed prefix of effects visible. Thread safety, reentrancy, and idempotence are unestablished.

## Dependencies

F-d6eaa40f40a0 is the only supplied opaque function identity. No opaque identities are available for the indirect dependencies; none are invented here.

| Interaction | Established | documented_only |
|---|---|---|
| Initial indirect invocation | Destination derived from incoming return-link state plus 0xFFE | Identity, complete inputs, effects, completion |
| Subsequent invocations | Ordered indirect calls using embedded-location-derived destinations | Signatures, target equality, errors, ownership |
| Word and wide results | Feed arithmetic, retained values, and later arguments | Application meanings and actual return convention |
| Secondary machine results | Supply additional later arguments | Defining interface and lifetime |
| Final transfer | Four arguments in the stated order | Destination, mechanism, effects, return behavior |

No dependency is assumed to preserve globals or to own, retain, allocate, or release passed memory.

## Acceptance examples

These are conditional static consequences, not executed tests.

### Token-base replacement

Assume supported memory and 32-bit bit-vector arithmetic. For token base 0x02000000 and a final high address ending in 0xAB, acceptance requires a byte store of 0xAB at 0x02000009, the intervening word store through the final low address, and then a byte store of zero at 0x02000009.

The replacement source is the token base, not the shifted token.

### Qualified global conversion

If symbol metadata establishes that the global at 0x05 is unsigned eight-bit and its value is 0xE1, the global-derived base is 225.

If instead metadata establishes an unsigned 16-bit global with value 0x01E1, the converted base is 481. Its plus-0x15 byte destination is then 0x01F6, not 0x00F6. Both interpretations remain conditional until the declaration is supplied.

### Absolute 0x15 assignment

Assume extracted index 4, doubled index 8, and retained traversed value 0x123. Under bit-vector arithmetic, the fourfold value is 0x48C.

The second pair of byte stores at doubled index plus 0x15 therefore writes 0x8C twice at 0x1D. The next shared-state assignment writes the doubled index’s low byte, **0x08**, at absolute address **0x15**, before the subsequent scaled-address word updates. Writing 0x8C there would use the wrong source.

### Index and explicit byte boundaries

A relevant result with low seven bits 0x7F yields extracted index 508 and doubled index 1016 under the bit-vector model.

An explicitly sampled unsigned byte 0xE1 becomes word value 225. This example does not establish the undeclared global at 0x05 as a byte.

### Transient effects and completion

Observable memory must retain all four constant-region word stores: 0x20, 0x2F, the sampled byte promoted to a word, and 0x20. Matching only the final value is insufficient.

If preceding operations complete, the final transfer receives 0x72 and the three specified words. No successful return, explicit failure code, or rollback is established if that destination does not return.

## Uncertainties

The following remain **documented_only**:

- Application purpose and data meanings.
- Dependency identities, signatures, ownership, and global effects.
- Incoming-state requirements and actual calling convention.
- Completeness of the represented control flow.
- Source width and signedness of the global at 0x05, including its physical read width and extension behavior.
- Other absent global declarations and overlapping-symbol relationships.
- Byte order, alignment, low-memory availability, and device-memory semantics.
- Target signed-arithmetic and variable-shift behavior.
- Allocation, lifetime, address validity, faults, and exception handling.
- Exact repeated-group multiplicity not enumerated here.
- Final transfer destination and mechanism.

These unknowns do not make the absolute 0x15 assignment’s source ambiguous: its assigned expression is the doubled index truncated to eight bits.

## Evidence

This contract uses static inspection of the supplied representation. Prior prose, comments, and reviewer feedback are not independent behavioral evidence.

The representation establishes an unsigned-word conversion of the global associated with 0x05, but supplies no declaration establishing an unsigned byte read. Consequently, every downstream use is described through a qualified global-derived base.

It also explicitly places the low-byte assignment of the doubled index at absolute address 0x15 after the second pair of identical stores sourced from the fourfold-scaled traversed value and before the subsequent scaled-address updates.

Other established evidence includes mixed-width accesses, transient and duplicate stores, retained-value provenance, wide-result extraction, indirect invocation ordering, and the final four-argument transfer.

No dependency bodies, symbol metadata resolving the global at 0x05, external calling-convention definition, memory map, or runtime traces were supplied. No runtime tests or implementation verification have been performed.
