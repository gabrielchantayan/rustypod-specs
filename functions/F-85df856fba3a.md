<!-- Trusted provenance: F-85df856fba3a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-85df856fba3a

## Purpose

F-85df856fba3a performs an ordered sequence of unresolved indirect invocations, derives address-bearing values from ambient state and dependency results, mutates indirectly addressed and fixed-address storage, and ends with an unresolved four-argument transfer.

No application-level purpose is established. Character-like constants do not establish text construction, formatting, parsing, or an object model.

This contract distinguishes **established static evidence**, **conditional arithmetic deductions**, and **documented_only** unknowns. It describes the represented path, not proven runtime reachability. No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no meaningful return value. Effective inputs include:

- An ambient unsigned 32-bit value and incoming control-link state.
- Primary and secondary results of unresolved invocations.
- Values obtained from named fixed-address globals whose declarations are absent.
- Memory reached through returned, converted, loaded, shifted, and scaled values.

Outputs comprise invocation events, storage updates, and the final transfer. No success indicator, output length, error result, or caller-supplied output buffer is identified. An invocation with no explicit arguments may still consume implicit state.

### Widths and conversions

Explicitly typed pointer dereferences establish accesses of 8, 16, or 32 bits. Offsets below are byte offsets. An explicitly unsigned 8-bit pointer read produces a value between 0 and 255; converting that value to a 32-bit unsigned value preserves it.

Combined dependency results are represented as 64-bit values split into low and high 32-bit portions. This establishes neither an in-memory layout nor a complete return convention.

**Named fixed-address globals require a different rule.** Their declarations are unavailable. A conversion of a global’s value to 32 bits establishes the resulting retained value, not the width or signedness of the underlying read. Likewise, assigning an explicitly narrowed 8-bit payload to a global does not establish an 8-bit destination access. Further conversion to the global’s declared type remains unresolved.

In particular, the global associated with `0x00000005` supplies a value converted to unsigned 32 bits. It is not established as an unsigned byte read. A signed 8-bit value of −128, for example, could become `0xFFFFFF80`, rather than 128, upon that conversion.

Address-bearing arithmetic is represented with 32-bit values, including signed operands. Equivalences involving wrapping multiplication, shifts, or truncation are conditional on target bit semantics. Signed overflow, signed shifts, address wrapping, and oversized variable shift counts remain unresolved.

### Retained roles

These labels describe value dependencies, not validated objects:

| Role | Derivation |
|---|---|
| Shifted base | A retained dependency-result word shifted left by 25 |
| Scale seed | A later retained dependency-result word |
| Scaled base | Scale seed multiplied by `0x02000000` |
| Shifted payload | Another retained result shifted left by eight |
| Bounded quantity | A result shifted left by eight, masked with `0x7FFF`, then shifted right by six |
| Bounded base | Twice the bounded quantity |
| Final halves | Low and high 32-bit portions of the last combined pre-mutation result |
| Fixed-derived base | Value from the global associated with `0x00000005`, converted to unsigned 32 bits |
| Retained pointer | Word captured at fixed-derived-base offset `0x54` |
| Fourfold pointer / eightfold pointer | Four or eight times the retained pointer |
| Selected word | Result of the later byte-indexed offset-`0x74` lookup |
| Selected base | Twice the selected word |
| Adjusted seed | Fourfold pointer minus `0x96` |
| Publication base | Four times the adjusted seed |
| Publication pointer | Value read from the global associated with `0x087FBBBC`, retained in a signed 32-bit role |

Under 32-bit bit semantics, the bounded quantity is a multiple of four between 0 and 508 inclusive; the bounded base lies between 0 and 1016. **No corresponding byte-sized range is established for the fixed-derived base.**

Traversal requires returning intermediate dependencies, executable targets, compatible invocation conventions, valid storage lifetimes, accessible addresses, and supported alignment. No visible checks enforce these requirements.

## Behavior

### Invocation phase

The first indirect target depends on incoming control-link state plus `0xFFE`. Later targets use embedded continuation-like values with the same displacement.

The represented invocations are ordered. They must not be treated as pure, merged, or reordered merely because many lack explicit arguments. Dependencies may change memory or ambient state, invalidate storage, block, trap, or fail to return.

After the first six invocations, the represented computation derives a value by shifting the ambient unsigned word right by 15. Identified arguments derived from that value include:

- Its low 14 bits shifted left by three, at most `0x1FFF8`.
- Its low bit placed at bit 31.
- Its low bit placed at bit 16.

Other explicit arguments use zero values, primary results, secondary-result words, low and high result portions, masks, and shifts. Explicit argument counts range from zero through four. Several retained results remain in use across many intervening invocations.

Under 32-bit bit semantics, shifting a value left by 31 and subsequently by 30 loses all information; shifting left by 14 and subsequently by 23 does likewise. Widening an intermediate would change those deductions. Explicitly zeroed retained values remain zero-derived inputs without establishing anything about dependency side effects.

The complete invocation inventory and per-site argument correspondence are **documented_only** in this contract. This regional description is not a complete call-trace oracle.

### Initial mutations and fixed-derived base

The first mutation stores the shifted base as a word at final-high-half offset `0x14`. A byte store then writes the high half’s low eight bits at shifted-base offset nine. A word store places the scaled base at offset `0x54` from 256 times the final low half. A second, distinct byte store at shifted-base offset nine carries the shifted base’s low eight bits.

Under modular 32-bit semantics, that second byte is zero. Its equality with an earlier byte does not eliminate either represented store.

Scaled-base offset `0x14` is zeroed as a word. An unsigned byte read at scaled-base offset five is followed by zeroing bounded-base offset `0x14`; the byte’s zero-extended value is then stored as a word at scaled-base offset `0x34`.

The word at scaled-base offset `0x14` is captured. Another zero word store at bounded-base offset `0x14` precedes two distinct byte stores of the captured word’s low eight bits at bounded-base offset `0x15`. The captured word is not unconditionally zero: aliasing and nonordinary memory behavior remain unresolved.

The fixed-derived base is then captured from the global associated with `0x00000005`, with conversion to unsigned 32 bits. Afterward, the global associated with `0x00000001` is assigned an explicitly narrowed 8-bit payload from the bounded base. Neither operation’s underlying global access width is established.

Fixed-derived-base offset `0x15` receives the bounded quantity’s low eight bits. Scaled-base offset `0x11` receives the shifted payload’s low eight bits; the earlier captured word, used as an address base, receives the full shifted payload at offset `0x54`. Scaled-base offset `0x11` is then overwritten with zero.

The word at fixed-derived-base offset `0x54` is retained as the retained pointer and replaced with the earlier captured word. Later changes to the originating fixed global do not resample the fixed-derived base.

### Early indirect updates

Bounded-base offset `0x34` receives the bounded quantity as a word **after that replacement and before** the word read at scaled-base offset `0x74`. The newly loaded base receives the scale seed shifted left by 26 at offset `0x54`.

The next update contains **two distinct word-read expressions** at bounded-base offset `0x74`: one determines the destination base and the other the payload for a word store at destination offset `0x54`.

Both reads occur within the boundary after the shifted-seed store and before the following retained-pointer store. Their relative evaluation order is not established. Their values need not agree. Specifying one read followed by a self-valued store would lose represented behavior.

The retained pointer is written at its own offset `0x24`, and its low eight bits are written at offset nine of the base previously loaded from scaled-base offset `0x74`. That loaded base’s offset `0x54` receives the shifted payload. The retained-pointer word store and offset-nine byte store then occur again.

Bounded-base offset `0x14` receives the scale seed shifted left by 26. The global associated with `0x00000054` is next assigned the retained pointer shifted left by one; its storage width is unresolved. Two further word stores put the retained pointer at bounded-base offset `0x14`, followed by two byte stores of the fourfold pointer’s low eight bits at bounded-base offset `0x15`.

The global associated with `0x00000015` receives an explicitly narrowed 8-bit payload from the bounded base. Fourfold-pointer offset `0x54` then receives the shifted payload.

### Middle mutation and selection

Bounded-base offset `0x34` receives the fourfold pointer. A fresh word read at bounded-base offset `0x74`, doubled, supplies the **middle base**. This read is separate from both reads in the earlier compound update.

Fourfold-pointer offset `0x54` is overwritten with the fourfold pointer itself. Its offset `0x15` receives the eightfold pointer’s low eight bits, and retained-pointer offset `0x44` receives the middle base.

An explicitly unsigned byte read at fourfold-pointer offset five supplies the base for a word lookup at offset `0x74`. The resulting **first lookup word**, used as a base, receives the middle base at offset `0x44`. Fourfold-pointer offset `0x15` is then overwritten with the fourfold pointer’s low eight bits.

The first lookup word’s low eight bits are written at fourfold-pointer offset `0x0D`; fourfold-pointer offsets `0x14` and `0x54` receive the fourfold pointer as words.

A word read at fourfold-pointer offset `0x64` supplies another base. Its offset `0x0D` receives the first lookup word’s low eight bits, and its offset `0x14` receives that base itself. A subsequent word read at fourfold-pointer offset four supplies the **byte-source base**.

The middle base receives the first lookup word’s low eight bits at offset nine and the eightfold pointer at offset `0x44`. An unsigned byte read at byte-source-base offset five is retained across a byte store of the middle base at fourfold-pointer offset nine, then published as a zero-extended word at first-lookup-word offset `0x44`.

Fourfold-pointer offset `0x14` receives the eightfold pointer. Middle-base offset `0x15` receives the byte-source base’s low eight bits; middle-base offset `0x44` receives the fourfold pointer.

A **second, distinct unsigned byte read** at byte-source-base offset five occurs next. Fourfold-pointer offset `0x15` receives the eightfold pointer’s low eight bits before that newly read byte supplies a word lookup at offset `0x74`, producing the selected word.

Before constant writes, fourfold-pointer offset `0x15` receives the byte-source base’s low eight bits; offset `0x54` receives the fourfold pointer; offset nine receives the selected base’s low eight bits; and offset `0x14` receives the eightfold pointer.

The two byte-source reads cannot be merged: intervening writes may alias their storage.

### Constants and publication

Explicit pointer stores write:

- Byte `0x3A` at four times the selected word.
- Halfword `0x003A` at eight times the selected word.
- Word `0x0000003A` at offset `0x20` from four times the selected word.

Publication-base offset `0x20` receives successive word values `0x20`, `0x2F`, a freshly read unsigned byte from publication-base offset four, and `0x20`.

The publication pointer is then obtained from the global associated with `0x087FBBBC`. **Only afterward** is the global associated with `0x00000060` assigned `0x20`. Neither global’s physical access width follows from these uses.

Repeated publication begins after both events. Each group contains four ordered, explicitly typed stores:

1. Four times the selected word, as a word, at a derived base’s offset `0x54`.
2. The publication base’s low eight bits at publication-pointer offset `0x19`.
3. The publication pointer’s low eight bits at publication-base offset `0x11`.
4. The derived base, as a word, at publication-pointer offset `0x34`.

The first derived base is the eightfold pointer. Subsequent groups use eight times the adjusted seed. The publication pointer remains retained rather than being reread between groups.

Exact group cardinality is **documented_only** here. These groups describe repeated accesses, not merely four final field values.

### Final fixed-storage boundary

After publication, the global associated with `0x000000A9` is read into a retained 32-bit opaque value. This establishes the retained width, not the global read width or signedness. Next, a word is read at offset `0x74` from four times the selected word.

The global associated with `0x0000003A` is then assigned an expression formed from a one-byte character conversion of the adjusted seed, multiplied by eight, plus `0x30`. Character signedness and conversion to the undeclared destination type are unresolved. The assignment cannot unconditionally be described as an 8-bit store of that expression’s low eight bits.

After this assignment, the function captures words at selected-base offset `0x14` and publication-base offset four, in that order.

Selected-base offset `0x0D` receives the publication base’s low eight bits. Publication-base offset eight receives byte `0x6C`, followed by the publication pointer’s low eight bits at offsets `0x0D` and nine. Selected-base offset `0x54` receives the selected base as a word.

The global associated with `0x00000067` is assigned an explicitly narrowed 8-bit payload from the retained `0x000000A9`-derived value. A word read at selected-base offset `0x44` precedes a fresh read of the `0x00000067` global whose value is assigned to the global associated with `0x00000021`. Neither global assignment establishes an 8-bit destination access, and the latter assignment has no explicit intervening 8-bit narrowing.

The offset-`0x44` word supplies a byte at publication-base offset `0x10` and a temporary word at publication-pointer offset `0x54`. A right-shifted form supplies a byte at selected-base offset one; the shift count is the publication base’s low eight bits. Signed right-shift behavior and counts outside the supported range remain unresolved. Publication-pointer offset `0x54` is then overwritten with the publication base.

A word read at publication-base offset `0x40` precedes publication-pointer offset `0x74` receiving the retained publication-base-offset-four word. The newly read word is stored at publication-base offset `0x10`, overlapping the earlier byte destination. Publication-pointer offset `0x44` receives the selected base.

Two more distinct reads of the `0x00000067` global supply separate, explicitly 8-bit stores at publication-base offset nine. Between those reads, selected-base offset nine receives the publication base’s low eight bits, publication-base offset `0x44` receives the retained `0x000000A9`-derived value, and publication-base offset `0x54` is read as a word.

Thus three separate read expressions of the `0x00000067` global follow its initial assignment. Their storage widths remain unresolved; they cannot be replaced by retaining the initially assigned payload.

### Late destination and terminal transfer

The word captured at publication-base offset `0x54` becomes the **late destination base**. Its offset `0x0D` receives the low eight bits of the retained selected-word-derived offset-`0x74` word.

The saved selected-base-offset-`0x14` word is restored at selected-base offset `0x54`. Publication-base offset `0x14` is then read as a word and retained as the **late byte payload**.

Selected-base offset `0x11` receives the retained selected-word-derived byte. Publication-base offset nine receives the late destination base’s low eight bits, and publication-base offset `0x44` receives that base as a word.

A word read at selected-base offset four supplies the **terminal address**. Twice that address receives halfword `0x0065`; terminal-address offset `0x64` is then read and retained.

**After that halfword store and offset-`0x64` read**, late-destination-base offset `0x11` receives the late byte payload’s low eight bits. This is not a store at selected-base or publication-base offset `0x11`. It precedes both the word store of decimal 100 at late-destination-base offset `0x54` and the word read at terminal-address offset `0x74`.

A final word read at selected-base offset `0x64` supplies another base. Selected-base offset nine receives the terminal address’s low eight bits. The final loaded base’s offset `0x54` receives the retained terminal-address-offset-`0x64` word, and twice that retained word receives the final loaded base as a word.

The final unresolved transfer has four explicit word arguments:

1. `0x72`.
2. The retained terminal-address-offset-`0x64` word.
3. The saved selected-base-offset-`0x14` word.
4. The later terminal-address-offset-`0x74` word.

Its concrete target and return behavior are unresolved.

## Side effects

Effects are nontransactional. No visible bounds checks, null checks, locks, atomic updates, rollback, or recovery branches protect this path.

The named fixed-storage inventory is:

| Associated address | Established use; physical access type unresolved |
|---|---|
| `0x00000001`, `0x00000015` | Assignment of explicitly narrowed 8-bit payloads |
| `0x00000005` | Read followed by unsigned 32-bit conversion |
| `0x00000021` | Assignment from a fresh global read |
| `0x0000003A` | Assignment of the character-derived arithmetic expression |
| `0x00000054` | Assignment of a shifted retained value |
| `0x00000060` | Assignment of `0x20` |
| `0x00000067` | Assignment of an 8-bit payload, followed by three distinct read expressions |
| `0x000000A9` | Read into a retained 32-bit opaque value |
| `0x087FBBBC` | Read into the signed 32-bit publication-pointer role |

This inventory does not infer byte or word global accesses from names, receiving variables, or assigned payloads. Explicit pointer dereferences elsewhere retain their established widths.

Roles may alias. Endianness matters for overlapping accesses. Captured values remain retained despite later overwrites of originating storage.

No explicit allocation, release, or ownership transfer is established. Dependencies may nevertheless allocate, release, retain, or invalidate storage. Faults or nonreturning dependencies may prevent later effects; no clean error return or rollback is promised.

## Dependencies

These opaque IDs identify documentation-local boundaries, not resolved target identities:

| ID | Boundary |
|---|---|
| F-00000001 | Initial indirect invocation using incoming control-link state |
| F-00000002 | Remaining ordered pre-mutation invocation sites |
| F-00000003 | Terminal four-argument transfer |

F-00000002 does not imply a shared callee. For every boundary, implicit inputs, global preservation, ownership, exceptions, blocking, and nonreturn behavior remain unknown.

## Acceptance examples

All examples are conditional on reaching the relevant operations and successfully performing required accesses.

- **Ambient bits:** If the sampled ambient word has only bit 15 set, the identified derived arguments are 8, `0x80000000`, and `0x00010000`.
- **Bounded arithmetic:** Under 32-bit bit semantics, a bounded-quantity source with its low seven bits set yields 508 and bounded base 1016.
- **Typed unsigned byte:** An explicitly unsigned byte pointer read yielding `0xFF` supplies word value 255.
- **Undeclared fixed input:** If the `0x00000005` global supplies unsigned value 128, the fixed-derived base is 128. If its undeclared type is signed 8-bit and supplies −128, unsigned 32-bit conversion instead yields `0xFFFFFF80`. The contract does not choose between these declarations.
- **Global destination width:** Assigning an 8-bit payload at the global associated with `0x00000001` does not justify accepting only an 8-bit storage update. Destination width and further conversion remain unresolved.
- **Retained input:** Once the fixed-derived base is captured, later changes to its originating global do not cause a reread for its described uses.
- **Early ordering:** Replacement at fixed-derived-base offset `0x54` precedes the bounded-quantity store at bounded-base offset `0x34`, which precedes the scaled-base-offset-`0x74` read.
- **Two-read update:** Different results from the two bounded-base-offset-`0x74` reads may determine different destination and payload values. No unsupported relative evaluation order is required.
- **Publication boundary:** The publication-pointer global read precedes the `0x00000060` global assignment; both precede publication groups.
- **Fresh global reads:** Successive reads of the `0x00000067` global feed their respective consumers. The two explicitly byte-sized consuming stores narrow to eight bits; the assignment to the `0x00000021` global has unresolved destination conversion.
- **Late write:** The late byte payload reaches late-destination-base offset `0x11` after the terminal halfword store and offset-`0x64` read, but before the decimal-100 store and offset-`0x74` read.
- **Failure or completion:** An inaccessible address need not produce a clean error. Reaching the final transfer supplies the specified arguments, without guaranteeing a successful return.

## Uncertainties

The following remain **documented_only**:

- Application purpose and semantic interpretation of constants.
- Complete invocation inventory and exact publication-group cardinality.
- Declarations, widths, signedness, and destination conversions of every named fixed-address global.
- Character signedness and the final stored representation of the `0x0000003A` expression.
- Full ABI, implicit inputs, secondary-result conventions, and concrete targets.
- Target overflow, wrapping, signed-shift, and oversized-shift semantics.
- Endianness, alignment, mappings, device behavior, and concurrent interference.
- Object boundaries, lifetimes, ownership, and dependency effects.
- Terminal control-flow form, fault handling, interruption, and reentrancy.

The two reads in the early compound update have unresolved relative evaluation order. Described access relationships are static source-level evidence, not a guarantee of an identical machine-level trace under an unknown compilation and memory model.

## Evidence

Established evidence consists of explicit pointer-access types, value conversions, fixed-global references, retained-value dependencies, represented ordering boundaries, distinct read expressions, and the terminal argument shape.

Missing global declarations prevent deriving physical global access widths or signedness from their uses. In particular, an unsigned conversion does not prove an unsigned byte load, and an 8-bit payload does not prove an 8-bit global store.

Arithmetic ranges and information-loss examples are conditional deductions. Application intent is not established. Comments, prior prose, and reviewer claims are not independent behavioral evidence.

No runtime execution, dependency inspection, memory-map validation, or tests have been performed. Static evidence does not prove valid addresses, reachability, complete invocation conventions, or successful completion.
