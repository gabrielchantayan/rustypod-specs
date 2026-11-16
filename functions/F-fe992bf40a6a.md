<!-- Trusted provenance: F-fe992bf40a6a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe992bf40a6a

## Purpose

F-fe992bf40a6a invokes a sequence of indirect dependencies, retains selected results, performs ordered mixed-width memory updates through computed and fixed addresses, and finishes with another indirect invocation.

**Established static evidence:** two 32-bit inputs, no explicit return value, dependency-result transformations, byte/halfword/word accesses, repeated stores, retained memory observations, and a terminal invocation.

**Inferred intent:** recurring offsets and self-referential stores resemble interconnected-record updates. They do not establish a particular data structure, allocation policy, text format, or output operation.

This contract describes the supplied static representation, conditional on execution reaching each operation. It does not establish executable control-flow fidelity. Missing dependency implementations, global declarations, and platform semantics remain `documented_only`.

## Interface

### Inputs and completion

| Input | Representation | Established use |
|---|---|---|
| First input | Opaque 32-bit value | Forwarded unchanged to F-U001. |
| Second input | Signed 32-bit integer | Forwarded unchanged to F-U001 together with its value shifted left six bits. |

Neither input is directly dereferenced before F-U001. Neither is established to represent a pointer, size, handle, or ownership-bearing resource.

F-U001’s target depends on incoming return context plus displacement 0xFFE. The explicit inputs alone therefore do not determine execution.

No explicit return value is represented. Observable outputs consist of dependency invocations, memory accesses, and their ordering. A return follows the terminal invocation, F-U003, if that dependency returns normally.

### Widths and execution assumptions

Explicit memory operations distinguish:

- **Byte:** 8 bits.
- **Halfword:** 16 bits.
- **Word:** 32 bits.

Byte and halfword stores retain the low eight and sixteen bits, respectively. Explicit unsigned-byte observations used numerically yield values from 0 through 255.

Some dependency results are represented as 64-bit quantities separated into lower and upper 32-bit portions. This establishes represented data dependencies, not a verified wide-return ABI.

Arithmetic descriptions below use a **conditional 32-bit machine-bit interpretation**: addition, subtraction, multiplication, and left shifts retain the low 32 bits. This is not a claim about portable signed-overflow semantics. Addresses are interpreted modulo 2³² under that model.

The observation at fixed address **0x05** has no supplied global declaration. Its conversion to unsigned 32 bits is established; its underlying load width, signedness, extension, and storage extent are not. Its converted value is not established to lie within 0–255.

Normal completion requires usable indirect targets, returning dependencies, accessible addresses, and supported access widths and alignment. No validation or recovery mechanism is visible.

## Behavior

### Dependency phase

F-U001 receives, in order, the first input, the second input, and the second input shifted left six.

Two subsequent invocations show no explicit arguments. The latter supplies a represented 64-bit result. The next invocation receives its lower portion shifted left eight, its upper portion unchanged, its lower portion shifted left fourteen, and its lower portion shifted left eight again.

The remaining pre-update invocation sequence is collectively designated F-U002. It consumes retained results, upper/lower portions, secondary machine-result values, explicit zeros, and shifted or masked fields. No explicit input-dependent branch or iteration is represented in this phase.

An invocation with no explicit arguments shown does not establish a zero-argument signature. Similar argument patterns do not establish common target identity.

Visible transformations include selecting bits 10–19, applying 24-bit and 22-bit masks, extracting upper-result fields, and positioning retained bits at bit 16 or bit 31. Under the conditional 32-bit interpretation, successive left shifts totaling at least 32 discard all original bits. Locally zeroed quantities remain zero through subsequent local shifts and masks. These simplifications do not remove dependency invocations or their unknown effects.

### Retained descriptive quantities

These names identify values, not source identifiers or object types.

| Quantity | Definition |
|---|---|
| **Shift base** | An earlier dependency result shifted left 25. |
| **Scaled base** | Another earlier result multiplied by 0x02000000. |
| **Field** | A result shifted left eight, masked to fifteen bits, then shifted right six. |
| **Small base** | Twice Field. |
| **Payload** | An earlier result shifted left eight and retained across later calls. |
| **Final lower / upper** | Lower and upper portions of the last represented 64-bit result before direct updates. |

Field equals four times its originating result’s low seven bits: 0–508 in multiples of four. Small base is 0–1016 in multiples of eight. Shift base, Scaled base, and Payload have zero low bytes under the stated arithmetic model.

### Opening updates and the zero reload

Opening stores place Shift base at Final upper plus 0x14, Final upper’s low byte at Shift base plus nine, and Scaled base at 256 times Final lower plus 0x54. A second byte store at Shift base plus nine supplies Shift base’s low byte, which is zero under the conditional interpretation.

The next memory-order interval is significant:

1. Scaled base plus 0x14 is cleared as a word.
2. An unsigned byte is observed at Scaled base plus five.
3. Small base plus 0x14 is cleared as a word.
4. The captured byte is stored as a zero-extended word at Scaled base plus 0x34.
5. The word at Scaled base plus 0x14 is reloaded as **Reloaded base**.

**Under ordinary byte-addressable memory semantics, with no external modification or access-triggered memory effects, Reloaded base is zero.** The two four-byte ranges at Scaled base plus 0x14 and Scaled base plus 0x34 are disjoint: their starting addresses differ by 0x20, including under modular 32-bit addressing. The intervening Small base store writes only zero bits; even partial overlap cannot introduce nonzero data. There is no intervening dependency invocation.

This conclusion does not depend on endianness. Nonordinary memory behavior could invalidate ordinary store/read reasoning, but aliasing alone does not make this reload unknown.

Another word clear at Small base plus 0x14 precedes two separate byte stores at Small base plus 0x15. Both supply Reloaded base’s low byte, hence both write zero under ordinary memory semantics. These remain represented accesses despite their redundant values.

### Capture at 0x05 and immediate consequences

After both duplicate byte stores, the global at **0x05** is observed and converted to unsigned 32 bits. The retained conversion is **Captured base**. Only afterward does fixed byte address **0x01** receive Small base’s low byte. That later store cannot retroactively change Captured base, even if it overlaps the global’s unresolved storage extent.

Subsequent updates place:

- Field’s low byte at Captured base plus 0x15.
- Payload’s low byte at Scaled base plus 0x11.
- Payload as a word at Reloaded base plus 0x54.
- Zero as a byte at Scaled base plus 0x11.

Thus, under ordinary memory semantics, **the Payload word store targets fixed address 0x54**, because Reloaded base is zero.

The word at Captured base plus 0x54 is then captured as **Retained word**, before that same word is replaced with Reloaded base. Under ordinary semantics, this replacement writes zero. The captured word remains available independently of the replacement.

Captured base uses the complete unsigned 32-bit conversion, not an established eight-bit extraction. Earlier overlapping updates can affect Retained word.

### Computed updates and fixed-address milestones

Small base plus 0x34 receives Field. A word observed at Scaled base plus 0x74 supplies **First loaded base**. Its plus-0x54 word receives twice Scaled base, then later Payload. Between those stores, the word reached through Small base plus 0x74 participates in a self-referential plus-0x54 update.

Retained word is stored at Retained word plus 0x24 twice. Two byte stores of its low byte at First loaded base plus nine are interleaved with these updates. Repetitions are separate accesses.

The following milestone order is established:

- Small base plus 0x14 receives twice Scaled base.
- Fixed word address **0x54** receives twice Retained word.
- Small base plus 0x14 receives Retained word twice.
- Two byte stores at Small base plus 0x15 each supply the low byte of four times Retained word.
- Fixed byte address **0x15** receives Small base’s low byte.

The twice-Retained-word store at 0x54 is distinct from the earlier Payload store there. Neither establishes the location’s final value: intervening or later computed writes can overlap it.

### Later computed transformations

Define **Quadrupled word** as four times Retained word and **Octupled word** as eight times Retained word.

After the fixed 0x15 store, Quadrupled word plus 0x54 receives Payload, and Small base plus 0x34 receives Quadrupled word. A fresh word observation at Small base plus 0x74, doubled, supplies **Doubled loaded base**.

Subsequent updates use offsets 0x09, 0x0D, 0x14, 0x15, 0x44, 0x54, and 0x64. Important dependencies include:

- Quadrupled word plus 0x54 is overwritten with Quadrupled word.
- Its plus-0x15 byte receives Octupled word’s low byte, then Quadrupled word’s low byte.
- Retained word plus 0x44 receives Doubled loaded base.
- An unsigned byte at Quadrupled word plus five selects a word observation at that byte value plus 0x74. Its result supplies another base for plus-0x44 updates.
- A word at Quadrupled word plus 0x64 supplies a base whose plus-0x0D byte and plus-0x14 word are updated; the latter becomes self-referential.
- A word at Quadrupled word plus four supplies a base whose plus-five byte is observed twice, separated by computed writes.
- Doubled loaded base receives byte updates and plus-0x44 word updates using Octupled word and Quadrupled word.

The second plus-five byte observation selects a later word observation at that byte value plus 0x74. Call the result **Late word**. Define **Late doubled base** and **Late quadrupled base** as twice and four times Late word.

Before the constant-writing region, further stores restore Quadrupled word at Quadrupled word plus 0x54, place Late doubled base’s low byte at Quadrupled word plus nine, and place Octupled word at Quadrupled word plus 0x14.

Separate byte observations must not be consolidated merely because their addresses derive from the same retained base. Intervening writes may overlap them.

### Constants and repeated updates

Three stores supply 0x3A at different widths and destinations:

| Width | Destination |
|---|---|
| Byte | Late quadrupled base |
| Halfword | Eight times Late word |
| Word | Late quadrupled base plus 0x20 |

Define **Adjusted value** as Quadrupled word minus decimal 150 and **Arithmetic base** as four times Adjusted value.

Arithmetic base plus 0x20 receives successive words 0x20, 0x2F, a zero-extended unsigned byte observed at Arithmetic base plus four, and 0x20 again. The byte observation occurs after the 0x2F store and before its replacement.

The word at fixed address **0x087FBBBC** is then captured as **Repeated base**. Fixed word address **0x60** receives 0x20 afterward, before the repeated-update region.

Each repeated group has four ordered store positions:

- Late quadrupled base as a word at an arithmetic-derived destination plus 0x54.
- Arithmetic base’s retained low byte at Repeated base plus 0x19.
- Repeated base’s retained low byte at Arithmetic base plus 0x11.
- An arithmetic-derived word at Repeated base plus 0x34.

The first group uses Octupled word for its arithmetic-derived destination and final word. Later groups use eight times Adjusted value. Addresses and byte values are retained rather than reloaded between groups. These are explicit repeated updates, not established collection traversal. Exact group cardinality is `documented_only` in this contract.

### Finalization and fresh fixed-byte observations

Finalization captures the word at **0xA9** as **Captured global word**, then the word at Late quadrupled base plus 0x74 as **Captured field word**.

Fixed byte address **0x3A** next receives eight times Adjusted value’s low byte plus 0x30, truncated to eight bits. No digit-range validation is represented.

The word at Late doubled base plus 0x14 is captured as **Saved field word**, followed by the word at Arithmetic base plus four as **Saved link word**.

Computed stores then place Arithmetic base’s retained low byte at Late doubled base plus 0x0D, 0x6C at Arithmetic base plus eight, and Repeated base’s retained low byte at Arithmetic base plus 0x0D and nine. Late doubled base plus 0x54 becomes self-referential.

Only then does fixed byte address **0x67** receive Captured global word’s low byte.

A word at Late doubled base plus 0x44 supplies **Loaded field**. A fresh observation at 0x67 supplies the byte store at **0x21**.

Loaded field’s low byte is stored at Arithmetic base plus 0x10, and its full word at Repeated base plus 0x54. Late doubled base plus one receives the low byte of Loaded field shifted right by a count from Arithmetic base’s low byte. Signed right-shift semantics and counts outside 0–31 remain unresolved. Repeated base plus 0x54 is then overwritten with Arithmetic base.

A fresh word observation at Arithmetic base plus 0x40 supplies a later word store at Arithmetic base plus 0x10. Between them, Repeated base plus 0x74 receives Saved link word. Repeated base plus 0x44 then receives Late doubled base.

Another fresh byte observation at 0x67 supplies a store at Arithmetic base plus nine. Late doubled base plus nine next receives Arithmetic base’s retained low byte, and Arithmetic base plus 0x44 receives the **full Captured global word**.

The word at Arithmetic base plus 0x54 is captured as **Loaded destination**. A further fresh observation at 0x67 then supplies another byte store at Arithmetic base plus nine. Computed writes between observations may overlap 0x67, so the observed bytes need not agree.

### Final retained fields and terminal invocation

Captured field word’s low byte is stored at Loaded destination plus 0x0D. Saved field word is restored to Late doubled base plus 0x54.

A word observed at Arithmetic base plus 0x14 is retained for a later byte store at Loaded destination plus 0x11. Before that store, Captured field word’s low byte goes to Late doubled base plus 0x11, Loaded destination’s low byte goes to Arithmetic base plus nine, and Loaded destination’s full word goes to Arithmetic base plus 0x44.

A word at Late doubled base plus four supplies **Final base**. A halfword at twice Final base receives 0x65. The word at Final base plus 0x64 is captured as **Final link**.

Loaded destination plus 0x11 receives the low byte retained from Arithmetic base plus 0x14, followed by a word store of decimal 100 at Loaded destination plus 0x54.

The word at Final base plus 0x74 is captured as **Final field**. A subsequent word observation at Late doubled base plus 0x64 supplies the final destination. Late doubled base plus nine receives Final base’s low byte; final destination plus 0x54 receives Final link; and twice Final link receives the final destination as a word.

F-U003 receives four represented 32-bit arguments, in order: **0x72, Final link, Saved field word, Final field**.

## Side effects

Writes are not confined to a caller-provided buffer. Computed addresses may overlap one another and fixed locations. Recurring offsets establish neither independent fields nor object boundaries.

Fixed byte stores include 0x01, 0x15, 0x21, 0x3A, and 0x67. Fixed word accesses include 0x54, 0x60, 0xA9, and 0x087FBBBC. The ordinary-memory zero-reload conclusion additionally establishes the earlier Payload word store at 0x54.

The observation at 0x05 is not an established byte access. Its overlap footprint remains unresolved.

Duplicate stores and fresh observations belong to the represented order. Their separate hardware visibility is platform-dependent. No atomicity, synchronization, or thread-safety guarantee is established.

No explicit allocation, release, reference counting, ownership transfer, locking, rollback, error result, or recovery branch is visible. Dependencies are not assumed to preserve globals or own/release memory. Faults or nonreturning dependencies may prevent completion without undoing earlier effects.

## Dependencies

The IDs below denote opaque document-local boundaries, not resolved implementation identities.

| ID | Boundary | Unknown properties |
|---|---|---|
| F-U001 | Initial context-derived invocation | Identity, actual signature, effects, completion. |
| F-U002 | Remaining pre-update invocation sequence | Individual identities, signatures, secondary results, effects. |
| F-U003 | Terminal invocation | Identity, calling convention, effects, return behavior. |

F-U002 groups multiple unresolved boundaries without implying a shared target. Secondary machine results cannot be replaced with zero or assigned guessed API semantics. Unresolved dependency properties are `documented_only`.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Input forwarding:** second-input pattern 0x00000003 supplies 0x00000003 and 0x000000C0 to F-U001. Under 32-bit truncation, 0x04000000 shifted left six supplies zero.

- **Field boundary:** originating low-seven-bit pattern 0x7F produces Field 508 and Small base 1016. Higher bits do not affect that derivation.

- **Disjoint nonzero write:** if the captured opening byte is 0xA5, its word store at Scaled base plus 0x34 cannot alter the cleared word at Scaled base plus 0x14. Under ordinary memory semantics, Reloaded base remains zero.

- **Overlapping zero write:** even if the Small base clear overlaps some or all bytes of the cleared Scaled base word, it cannot introduce nonzero bits. Both following duplicate byte stores at Small base plus 0x15 therefore supply zero.

- **Zero-reload propagation:** with ordinary memory semantics and Payload 0x12345600, the Reloaded-base-dependent Payload store writes that word at 0x54. The later replacement at Captured base plus 0x54 writes zero, not an unconstrained reloaded value.

- **Conditional unsigned-byte global:** if the missing declaration makes the 0x05 global unsigned eight-bit with observed value 0x80, Captured base is 128. Its plus-0x15 store uses 0x95, and its plus-0x54 word capture uses 0xD4.

- **Conditional signed-byte global:** if instead it is signed eight-bit with observed value −128, conversion produces 0xFFFFFF80. The corresponding modular addresses are 0xFFFFFF95 and 0xFFFFFFD4. The supplied declaration evidence cannot select between these cases or exclude wider storage.

- **Capture ordering:** the 0x05 observation follows both duplicate opening byte stores and precedes the 0x01 store. Later modification does not change its retained conversion.

- **Later fixed-word source:** Retained word 0x80000003 produces 0x00000006 at the later fixed 0x54 milestone under 32-bit truncation.

- **Adjusted byte:** Adjusted value with low byte 0x1F produces 0x28 at 0x3A after multiplication, addition, and byte truncation.

- **Full-word retention and freshness:** Captured global word 0x123456AB initially supplies 0xAB at 0x67 and later supplies its full word at Arithmetic base plus 0x44. Intervening aliases may make the three subsequent observations of 0x67 differ.

- **Partial execution:** if a dependency does not return, later effects are not required. Earlier completed effects are not visibly rolled back.

## Uncertainties

The following remain `documented_only`:

- Application-level purpose, valid input domain, and executable control-flow fidelity.
- Dependency identities, actual signatures, secondary-result provenance, and wide-return ABI.
- The 0x05 global’s datatype, load width, signedness, extension, and storage footprint.
- Endianness, alignment tolerance, memory map, fault behavior, signed arithmetic, and variable-shift semantics.
- Nonordinary memory behavior, including device-backed reads/writes or external modification.
- Allocation boundaries, ownership, lifetime, concurrency, and exact repeated-group cardinality.

**Ordinary aliasing is not an uncertainty about Reloaded base:** the intervening nonzero word store is disjoint, and the potentially overlapping store writes zero. A nonzero reload would require departure from the stated ordinary-memory assumptions, not merely overlapping computed addresses.

## Evidence

The supplied static representation is the sole behavioral evidence. Prior prose, comments, reviewer feedback, and descriptive labels are not independent evidence.

Explicit access types establish byte, halfword, and word operations. The undeclared 0x05 observation establishes conversion to unsigned 32 bits but not an eight-bit load.

The zero-reload conclusion follows from the complete intervening access interval: one byte observation, one zero word store, and one nonzero-capable word store at a disjoint offset. Static dependency propagation establishes the two zero byte stores, the Payload store at 0x54, and the zero replacement at Captured base plus 0x54 under ordinary memory semantics.

Static ordering also supports retained captures, repeated accesses, full-word retention, and terminal arguments. No dependency implementations, validated execution trace, platform ABI, or runtime memory map were supplied. **No runtime tests have been performed.**
