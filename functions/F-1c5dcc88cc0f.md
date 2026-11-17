<!-- Trusted provenance: F-1c5dcc88cc0f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1c5dcc88cc0f

## Purpose

F-1c5dcc88cc0f performs an ordered sequence of indirect invocations, transforms dependency results, reads and modifies memory through calculated addresses and absolute globals, and concludes with an unresolved four-argument invocation.

The supplied static representation establishes an access-and-state contract, not a higher-level application purpose. It does not establish parsing, formatting, allocation, device initialization, or a particular data structure. Repeated offsets do not prove object types; character-like constants do not prove text output.

This contract distinguishes:

- **Established static evidence:** represented accesses, value widths, conversions, dependencies between captured values, and operation order.
- **[INFERENCE]:** possible intent, excluded from acceptance requirements.
- **documented_only:** unresolved behavior requiring declarations, dependency contracts, or execution-environment information.

No runtime execution or tests have been performed.

## Interface

### Explicit inputs

The represented interface has four 32-bit input positions and no declared result.

| Position | Explicit use |
|---|---|
| First | No explicit body use. |
| Second | No explicit body use. |
| Third | A value shifted left by four bits is supplied to an early dependency. |
| Fourth | Values shifted left by seven and twenty-five bits are supplied to early dependencies. One invocation receives the seven-bit-shifted value twice. |

The third and fourth positions have represented signed interpretations. Valid input ranges and signed-shift overflow behavior are not established.

The first two positions are not proven behaviorally irrelevant. Incompletely recovered dependency interfaces may consume ambient argument state.

### Implicit inputs and completion

Execution also depends on incoming execution-link state, executable indirect targets, dependency-result channels, existing memory, globals, and environmental arithmetic and addressing rules.

Dependency outputs include represented 32-bit values, secondary result channels, and paired 64-bit values. Complete calling conventions and dependency signatures are unavailable.

There is no established return value, success indicator, or failure code. The final invocation receives:

1. `0x72`.
2. A captured 32-bit word from terminal-base relative `0x64`.
3. An earlier captured 32-bit word from doubled-linked-base relative `0x14`.
4. A later captured 32-bit word from terminal-base relative `0x74`.

These arguments are retained values, not requests to reload their source locations at dispatch. Whether the final target returns, terminates execution, faults, or transfers control elsewhere is documented_only.

### Preconditions and errors

Every reached indirect target must be executable, and each reached memory access must be permitted for its actual extent. Explicit byte, halfword, and word accesses require one, two, and four bytes respectively. Global access extents depend on unavailable declarations.

No explicit checks validate null addresses, alignment, bounds, arithmetic overflow, dependency results, or memory consistency. No error branch, rollback, or cleanup path is represented. A fault or nonreturning dependency can prevent subsequent operations while leaving earlier effects intact.

## Behavior

### Widths, arithmetic, and global storage

Explicitly typed accesses distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Paired dependency results are represented as 64-bit values. Explicit unsigned byte loads yield values from 0 through 255.

Receiving-value width and storage-access width are separate facts:

| Global associated with | Established value operation | Unresolved storage properties |
|---|---|---|
| `0x05` | Evaluated once and converted to unsigned 32 bits for an address base. | Source width and signedness; no established byte-range bound. |
| `0x01`, `0x15` | Assigned explicitly eight-bit-narrowed values. | Destination widths and assignment conversions. |
| `0x54` | Assigned a captured word shifted left once. | Destination width and signedness. |
| `0x60` | Assigned `0x20` after the shared-base capture. | Destination width and signedness. |
| `0xA9` | Evaluated once into a retained 32-bit value. | Source width, signedness, and conversion details. |
| `0x3A` | Assigned eight times a character-sized conversion of the adjusted index, plus `0x30`. | Character signedness and destination conversion and width. |
| `0x67` | Assigned an eight-bit-narrowed value, then separately evaluated three times. | Assignment and evaluation widths and signedness. |
| `0x21` | Assigned the first later evaluation of the global at `0x67`. | Destination width and conversion. |
| Shared-base global | Evaluated once into a represented signed 32-bit capture. | Source width, signedness, and conversion details. |

An eight-bit assignment expression does not establish a one-byte store. A 32-bit capture does not establish a four-byte source read.

Where examples assume truncating 32-bit arithmetic, left shifts retain the low 32 result bits and scaled addresses use the corresponding 32-bit results. Those assumptions do not establish portable validity for signed overflow or pointer arithmetic. Under truncating shifts, a shift by twenty-five retains contributions only from the original low seven bits; a shift by eight clears the low byte.

Unsigned 64-bit shifts by 34, 36, and 44 bits extract the upper word shifted right by two, four, and twelve bits respectively. Other masks select fields up to twenty-four bits wide. Explicitly zero-initialized values remain zero through represented masks and shifts; that does not make intervening invocations effect-free.

A late signed right shift uses the publication base’s low byte as its count. Negative operands and counts outside the ordinary 32-bit shift range remain documented_only.

### Indirect dependency phase

A long, fixed sequence of indirect invocations precedes the direct-memory mutation region. No conditional branch or loop controls this represented sequence. Every later boundary is contingent on earlier invocations completing.

Arguments include shifted inputs, literal zeroes, primary and secondary results, paired-result halves, extracted bit fields, and repeated copies of calculated values. Calls with unused results remain observable boundaries. A call showing no explicit arguments is not proven parameterless.

The first target depends on incoming execution-link state plus `0xFFE`. Subsequent targets use address-derived expressions. Their complete identities and signatures are unresolved. Dependencies cannot be assumed to preserve globals, memory contents, mappings, allocation lifetimes, or ownership.

### Initial direct-memory boundary

The following descriptive terms identify value relationships, not object layouts:

- **Shifted base:** the low word of an earlier paired result shifted left by twenty-five.
- **High-shift base:** a later 32-bit dependency result multiplied by `2²⁵`.
- **Small index:** bits 6 through 14 of a result already shifted left by eight, obtained through an unsigned mask and shift.
- **Boundary pair:** the final paired result before direct mutation.

The opening mutations publish the shifted base as a word at boundary-upper-word relative `0x14`, and that upper word’s low byte at shifted-base relative `0x09`. A word at boundary-lower-word times 256, relative `0x54`, receives the high-shift base. The byte at shifted-base relative `0x09` is then replaced with the shifted base’s low byte, which is zero under truncating 32-bit shifts.

The high-shift-base word at relative `0x14` is explicitly cleared. An unsigned byte from its relative `0x05` is captured. Before the cleared word is read back, the function clears a word at twice-small-index relative `0x14` and stores the captured byte, zero-extended, as a word at high-shift-base relative `0x34`.

**Under ordinary memory semantics, the relative-`0x14` readback is zero.** The potentially overlapping intervening word store writes zero. The potentially nonzero store at the same high-shift base plus `0x34` is disjoint from the four bytes beginning at plus `0x14`. Ordinary address aliasing therefore does not provide a reason for a nonzero readback.

Nonzero readback remains possible only under additional behavior not established here, such as concurrent modification, memory-mapped side effects, or nonordinary address mappings.

After the readback, the twice-small-index word clear is repeated. Two separate byte stores then publish the readback’s low byte at twice-small-index relative `0x15`. With ordinary memory semantics, both publish zero.

### Linked mutation

The global associated with `0x05` is evaluated and converted to an unsigned 32-bit destination base. This is not an established unsigned byte load.

The word at global-derived-base relative `0x54` is captured and replaced with the initial readback. Its old value becomes the retained **linked word**. With ordinary memory semantics, the replacement value is zero; under unresolved nonordinary behavior, it is the actual captured readback.

Assignments to globals associated with `0x01`, `0x15`, and `0x54` are interleaved with calculated-address writes. In particular, the `0x54` assignment receives twice the retained linked word, subject to the unresolved arithmetic and destination conversions.

Further accesses follow words at relative `0x74`, use explicitly unsigned bytes at relative `0x05` as additional address bases, and capture words at relative `0x04` and `0x64`. Those explicit byte-derived bases are in the range 0–255, unlike the global-derived base.

Updates include self-valued word stores, repeated stores at relative `0x24`, byte writes at relative `0x09`, `0x0D`, `0x11`, and `0x15`, and word replacements at relative `0x14`, `0x34`, `0x44`, and `0x54`. Repeated byte loads separated by writes remain separate evaluations.

A later word reached through an unsigned-byte-derived base supplies the **selected linked value**. Twice this value becomes the **doubled linked base**; four times it becomes the **payload base**.

No non-aliasing layout is established. For a shared base, a byte at relative `0x15` overlaps the word beginning at relative `0x14`. Earlier writes can therefore affect later pointer-like captures and address selection.

### Constants, publication, and repeated updates

The selected linked value supplies three explicit destinations for `0x3A`:

- A byte at four times the value.
- A halfword at eight times the value.
- A word at four times the value plus `0x20`.

These footprints are independently established.

The **adjusted index** is four times the retained linked word minus `0x96`. Four times that adjusted index is the **publication base**.

Publication-base relative `0x20` receives four word stores, in order: `0x20`, `0x2F`, a zero-extended unsigned byte captured from publication-base relative `0x04`, and `0x20`. The byte capture occurs after the first two stores and before the last two.

Only after all four stores does the function capture the shared-base global. The assignment of `0x20` to the global associated with `0x60` follows that capture. Repeated updates follow the assignment.

Each fixed update group preserves this four-store order:

1. Publish the payload base as a word at a calculated base’s relative `0x54`.
2. Publish the publication base’s low byte at shared-base relative `0x19`.
3. Publish the shared base’s low byte at publication-base relative `0x11`.
4. Publish the calculated base as a word at shared-base relative `0x34`.

The first group uses eight times the retained linked word as its calculated base. Subsequent groups use eight times the adjusted index. This is fixed repetition, not retry behavior. The shared-base value is retained throughout; writes overlapping its original global storage do not retroactively change that capture.

### Final captures and global publication

The final capture boundary has a significant interleaving:

- First, the global associated with `0xA9` is evaluated into a retained 32-bit value.
- Next, a word is captured from payload-base relative `0x74`.
- The character-based expression is then assigned to the global associated with `0x3A`.
- Only afterward are words captured from doubled-linked-base relative `0x14` and publication-base relative `0x04`, in that order.

This is not a simultaneous pre-mutation snapshot. The `0x3A` assignment can affect the last two captures if its actual storage extent overlaps them. Its expression includes character-sized conversion before multiplication; it is not unconditionally a one-byte store of eight times the full adjusted index plus `0x30`.

Subsequent byte stores publish the publication base’s low byte at doubled-linked-base relative `0x0D`, `0x6C` at publication-base relative `0x08`, and the shared base’s low byte at publication-base relatives `0x0D` and `0x09`. A word store publishes the doubled linked base at its own relative `0x54`.

The global associated with `0x67` then receives the low eight bits of the retained `0xA9` capture. Three distinct later evaluations of `0x67` occur:

- After a word capture from doubled-linked-base relative `0x44`, the first supplies the assignment to the global associated with `0x21`.
- After further mutations, the second supplies a byte store at publication-base relative `0x09`.
- After additional mutations and a word capture from publication-base relative `0x54`, the third supplies another byte store at publication-base relative `0x09`.

Before the second evaluation, publication-relative `0x10` receives a byte and later a word. Shared-base relatives `0x54`, `0x74`, and `0x44` are updated. Doubled-linked-base relative `0x01` receives the low byte of the signed-shift result described above.

Between the second and third evaluations, doubled-linked-base relative `0x09` receives a byte, the retained `0xA9` capture is stored explicitly as a word at publication-base relative `0x44`, and publication-base relative `0x54` is captured as a word.

The source-global evaluations need not be byte reads. Their values cannot be merged into one cached evaluation without additional justification.

### Remaining mutation and terminal dispatch

The captured publication-relative-`0x54` word becomes the **new destination base**. The following interleaving is established:

1. A byte store at new-destination-base relative `0x0D` publishes the low byte of the earlier payload-relative-`0x74` capture.
2. A word replacement at doubled-linked-base relative `0x54` publishes the earlier doubled-linked-relative-`0x14` capture.
3. A word is captured from publication-base relative `0x14`.
4. A byte store at doubled-linked-base relative `0x11` publishes the same payload-capture low byte used in the first event.
5. Publication-base relative `0x09` receives the new destination base’s low byte, followed by its full word value at relative `0x44`.

The doubled-linked-relative-`0x11` store does **not** precede the intervening word replacement or publication-relative-`0x14` capture. Aliasing can make this distinction observable. The capture can reflect the earlier new-destination-relative-`0x0D` store or doubled-linked-relative-`0x54` replacement, but cannot reflect the subsequent relative-`0x11` store.

A word captured from doubled-linked-base relative `0x04` supplies the **terminal base**. An explicit halfword store places `0x65` at twice that base before terminal-relative `0x64` is captured.

After that capture, new-destination-base relative `0x11` receives the low byte of the publication-relative-`0x14` capture. New-destination-base relative `0x54` receives the word value 100.

Terminal-relative `0x74` is then captured, followed by a word read from doubled-linked-base relative `0x64`. The terminal base’s low byte is written at doubled-linked-base relative `0x09`.

Finally, the terminal-relative-`0x64` capture is stored at relative `0x54` of the newly read base, and that newly read base is stored as a word at twice the terminal-relative-`0x64` capture. Terminal dispatch follows these mutations using the retained arguments specified in the interface.

## Side effects

Observable boundaries include indirect invocations, explicit typed memory reads and writes, global evaluations and assignments, and terminal dispatch.

Absolute globals associated with `0x01`, `0x05`, `0x15`, `0x21`, `0x3A`, `0x54`, `0x60`, `0x67`, and `0xA9` participate. Their expression-level conversions do not establish storage footprints.

Calculated accesses may overlap one another or global storage. Overlap analysis requires actual extents, byte order, and address semantics. No transaction, synchronization, atomic-publication guarantee, or rollback is established.

No explicit allocation or release operation is identified. Dependencies may allocate, release, retain, or modify memory. Ownership transfer, lifetime preservation, thread safety, reentrancy, and idempotence remain documented_only.

## Dependencies

F-1c5dcc88cc0f is the only supplied opaque function ID. No opaque IDs are supplied for its indirect dependencies; none are invented.

Unresolved dependency boundaries include the entry-link-derived target, later address-derived targets, primary and secondary result channels, paired-result producers, and the terminal four-argument target.

Complete signatures, target equivalence between invocation sites, error behavior, global preservation, and ownership effects are documented_only. No dependency is assumed to preserve memory merely because its result is unused.

## Acceptance examples

These are conditional requirements, not observed test results. Each assumes completion of preceding dependencies and permitted reached accesses.

### Truncating shifts

Under truncating 32-bit shifts, fourth input `1` supplies `0x80` at a seven-bit-shift boundary and `0x02000000` at a twenty-five-bit-shift boundary. Fourth input `0x80` supplies zero at the latter boundary.

### Initial zero readback despite ordinary overlap

Assume ordinary coherent memory, valid address calculations, and no concurrent modification. If the twice-small-index zero store partially or completely overlaps high-shift-base relative `0x14`, the readback remains zero: both writes publish zero.

A nonzero byte captured from high-shift-base relative `0x05` and subsequently stored as a word at relative `0x34` does not change this result. That word is disjoint from the word at relative `0x14`.

A nonzero readback requires additional nonordinary behavior; ordinary aliasing alone is insufficient.

### Global evaluation versus explicit byte load

If the global at `0x05` converts to unsigned 32-bit `0x1234`, the retained destination base is `0x1234`, not `0x34`.

An explicit unsigned byte load containing `0xFE`, by contrast, supplies 254.

### Ordered capture under aliasing

Assume ordinary little-endian memory and that publication-base relative `0x14` is the same address as doubled-linked-base relative `0x11`. If the captured word is `0x11223344` immediately before the latter byte store, that captured value remains `0x11223344` even if the subsequent store changes its first byte.

The later new-destination-relative-`0x11` byte store therefore receives `0x44`, not a byte recomputed from the modified source word. Earlier overlapping stores may already have affected the capture.

### Publication and global re-evaluation

All four publication-relative-`0x20` word stores precede the shared-global capture, which precedes the `0x60` assignment and repeated updates.

If the second and third evaluations of `0x67` yield low bytes `0x34` and `0x56`, the corresponding publication-relative-`0x09` stores publish those bytes separately.

### Widths and terminal arguments

The explicit `0x3A` byte, halfword, and word publications retain distinct footprints. Replacing all three with word stores is not equivalent.

The terminal invocation’s third argument remains the earlier doubled-linked-relative-`0x14` capture, taken after the `0x3A` global assignment. Later writes to its source location do not replace that retained argument.

## Uncertainties

The following remain documented_only:

- Higher-level purpose, object layouts, and complete control-flow recovery.
- Indirect target identities, complete invocation traces, calling conventions, and ambient argument consumption.
- Global storage widths, signedness, and assignment conversions.
- Character signedness and the final stored result of the `0x3A` assignment.
- Byte order, alignment requirements, low-memory accessibility, and memory-mapped behavior.
- Signed overflow, address wrapping, pointer validity, and exceptional shift-count semantics.
- Actual aliases, concurrency, nonordinary mappings, and memory lifetimes.
- Dependency ownership effects and final-transfer behavior.

[INFERENCE] Repeated offsets suggest structured state. They do not establish an object model or justify non-aliasing assumptions.

The initial zero-readback guarantee is conditional on ordinary memory semantics. Its unresolved exceptions do not weaken that guarantee by ordinary aliasing.

## Evidence

This contract derives from represented operations in the supplied static material. Embedded comments, prior prose, and reviewer feedback are not independent behavioral authority.

Explicit access types support byte, halfword, and word footprints. Explicit conversions support value widths without determining undeclared global storage extents.

The initial readback follows a clear of its own word, another zero-valued word store, and a potentially nonzero word store at the same base plus `0x34`. The latter is disjoint from the word at plus `0x14` under ordinary memory semantics.

The terminal mutation region places the doubled-linked-relative-`0x54` replacement and publication-relative-`0x14` capture between the new-destination-relative-`0x0D` and doubled-linked-relative-`0x11` byte stores.

No tools, external implementation inspection, dependency inspection, runtime execution, or runtime tests were used. No independently verified provenance or snapshot hash is asserted.
