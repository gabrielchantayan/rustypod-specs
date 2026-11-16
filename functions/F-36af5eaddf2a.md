<!-- Trusted provenance: F-36af5eaddf2a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-36af5eaddf2a

## Purpose

F-36af5eaddf2a performs an ordered sequence of indirect dependency calls, transforms dependency-derived values, and then reads and modifies memory through calculated addresses and fixed global locations. It finishes with an unresolved indirect control transfer represented as a call.

**Established static evidence:** the supplied body establishes explicit argument uses, arithmetic forms, mixed-width accesses, saved-value dependencies, and represented operation order. It does not establish a recognizable application-level operation.

**Inferred intent:** masking and shifting suggest packed-field manipulation; repeated offsets and address-valued writes suggest structured state. Neither observation establishes a device protocol, collection type, text operation, or ownership model.

This contract describes the represented behavior without assigning those interpretations normative meaning.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Forwarded unchanged to the initial indirect dependency. |
| Second argument | 32 bits | No explicit use in the supplied body. |
| Third argument | Signed 32 bits | Left-shifted by five positions for two early dependency calls. |

The first and second arguments have no established pointer, scalar, or handle interpretation. The second argument’s lack of explicit use does not establish its irrelevance under an unresolved calling convention.

### Implicit inputs

Execution also depends on:

- Entry control-link state, from which the first call target is calculated using displacement 0xFFE.
- Primary and secondary machine-result locations populated by dependencies.
- Other indirect targets calculated from embedded control-location values.
- Existing contents of calculated memory locations.
- Fixed globals, including those associated with addresses 0x05, 0x67, 0xA9, and 0x087FBBBC.
- The unresolved final transfer target.

The explicit arguments alone therefore do not determine the function’s effects.

### Outputs and completion

The represented interface has no explicit return value. Outputs are dependency interactions and memory changes.

The final transfer receives four represented arguments:

1. Constant 0x72.
2. A word captured late in the closing phase.
3. A word saved near the beginning of that phase.
4. Another late closing-phase word.

Their precise read dependencies appear below. The constant is an argument, not an established success status.

Normal completion requires every preceding call and access to complete and the final target to return. Neither termination nor return from that target is guaranteed.

### Preconditions and ownership

Reached targets must support the actual calling convention. Reached memory locations must support the required widths, address calculations, and access permissions. No visible validation establishes those conditions.

No allocation, release, retention, ownership transfer, or lifetime extension is established. A dependency result used as an address must not be assumed to identify newly allocated memory or remain valid because it was returned by a call.

## Behavior

### Ordering model

The visible body contains a long dependency-call phase, a direct-memory phase, and a closing rewrite phase. There is no explicit conditional branch or loop. Repeated memory operations are expanded in the representation.

This says nothing about control flow inside dependencies.

Represented call and memory-access order is significant. Calls without displayed arguments still occur. Repeated stores and later reloads must not be replaced merely by equivalent final ordinary-memory contents: aliasing or externally observable memory may distinguish them.

Where an individual expression contains multiple reads, the body does not necessarily establish an independent ordering among those reads. The ordering claims below concern the represented sequence and explicit saved-value dependencies, not an invented processor-level trace.

### Dependency interactions

The initial dependency receives the first argument unchanged. Early calls forward primary results shifted left by eight, extract upper 32-bit portions of paired 64-bit results, and sometimes forward those upper portions unchanged or shifted left by six.

The third argument’s five-position shift is supplied to two early calls without a visible range check.

Later interactions include:

- Explicit zero arguments.
- The same transformed value in several argument positions.
- Values saved across intervening calls.
- Shifts extending through count 31.
- Unsigned masks and logical right shifts selecting smaller fields.
- Secondary machine results used without an independently established high-level return signature.

An intervening dependency may change globals or memory consulted later. Conversely, the body explicitly retains some values; subsequent memory changes do not automatically replace those saved values.

Individual target identities and the complete dependency-specific semantic contracts remain **documented_only**. Similar argument patterns do not make targets interchangeable or calls removable.

### Widths and arithmetic

The representation distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and paired 64-bit results.

Numerical extraction of the lower and upper halves of a 64-bit result does not establish memory byte order.

Unsigned masks select fields including the lowest 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Their ranges follow the selected widths. For example:

- Lowest 20 bits, shifted right by ten: 0 through 1,023.
- Lowest 24 bits, shifted right by twenty: 0 through 15.
- Lowest 18 bits, shifted right by eleven: 0 through 127.

Unsigned right shifts are logical. Explicit unsigned byte dereferences yield values from 0 through 255 when widened. Byte and halfword stores retain the low eight and sixteen bits respectively.

Many shifts, products, additions, and address calculations use represented signed 32-bit quantities. Negative left operands, signed overflow, address wraparound, and conversion of out-of-range unsigned values to signed values are not independently resolved. Mathematical descriptions below identify dependencies; they do not assert arbitrary-precision evaluation or unconditional modulo-32-bit machine behavior.

One closing operation right-shifts a signed word by a count selected from a retained value’s lowest eight bits. Counts can exceed 31. Shift-count handling and negative signed-right-shift behavior remain **documented_only**.

### Initial direct-memory changes

The final paired dependency result supplies both an upper-word address base and a lower-word value scaled by 256. Earlier dependency-derived shifts supply other bases and store values.

The first direct effects, in represented order, include:

- A word store at upper-result base plus 0x14.
- A byte store at a retained shifted base plus 9.
- A word store at the 256-scaled lower-result base plus 0x54.
- A second byte store at that same offset-9 destination.
- A word clear at another retained base plus 0x14.
- An explicit unsigned byte read from that base plus 5.

After a further derived word clear, the byte read is stored, zero-extended, as a word at offset 0x34. The cleared offset-0x14 word is then reloaded. Another clear precedes two consecutive identical byte stores at a doubled selector base plus 0x15.

The reload is a real dependency. Aliasing between intervening accesses may invalidate substitution of the earlier zero, and the environment may give reads observable behavior.

### Fixed globals during the middle phase

The following descriptions use neutral value roles, not object types:

- **Selector:** the late call-derived value selected by masking a shifted word to 15 bits and shifting right by six.
- **Doubled selector base:** twice that selector.
- **Converted global value:** the value associated with address 0x05 after conversion to unsigned 32 bits.
- **Saved link word:** the word read at converted global value plus 0x54, before that destination is overwritten.

The fixed-location interactions have these ordering constraints:

| Fixed location | Established behavior and position |
|---|---|
| 0x05 | Consulted after the first pair of identical offset-0x15 byte stores. Its value is converted to unsigned 32 bits and retained for address formation. |
| 0x01 | Receives the low byte of the doubled selector base immediately after that consultation. |
| 0x54 | Receives a represented word equal to the saved link word shifted left by one, after intervening derived updates. |
| 0x15 | Receives the low byte of the doubled selector base, after another pair of identical derived offset-0x15 byte stores. |

**The source access width at 0x05 is unresolved.** The body supplies a conversion to unsigned 32 bits but neither its declaration nor an explicit byte dereference. It does not establish that this input is one byte, lies in 0–255, or is zero-extended from eight bits. This differs from the explicit offset-5 byte dereferences elsewhere.

After the 0x01 store, the converted global value supplies an offset-0x15 byte destination and the offset-0x54 read/write destination that produces the saved link word. Derived operations intervene before the fixed 0x54 and 0x15 stores. Thus those fixed writes are not an adjacent initialization block.

### Derived layouts and aliasing

The body repeatedly uses calculated bases with byte offsets 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, and 0x19, and word offsets including 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74.

These offsets do not establish one common record layout. There are no established object extents or disjointness guarantees.

Middle-phase operations include self-address-like word stores, repeated writes at offsets 0x24 and 0x54, and loads at offset 0x74 whose results become later address bases. Explicit unsigned byte reads at offset 5 also supply small numerical bases for offset-0x74 word reads.

One offset-0x74 expression consults that word both as a destination-base source and as stored data. Equality of those separately consulted values is not guaranteed by a saved snapshot.

Byte and word destinations may overlap each other or fixed globals. Neither alignment nor endianness is established. A replacement must not impose aligned-access or disjoint-object assumptions unsupported by external preconditions.

### Constants and the retained global base

For the remaining phases, define these descriptive quantities:

- **Link value:** the later offset-0x74 word obtained through an explicit unsigned-byte-derived base.
- **Doubled link base:** twice the link value.
- **Quadrupled link base:** four times the link value.
- **Adjusted base:** four times the saved link word, minus 0x96.
- **Retained data base:** four times the adjusted base, represented as unsigned 32 bits.
- **Retained global base:** the represented word loaded from fixed global address 0x087FBBBC.

The constant 0x3A is written at three widths: byte at the quadrupled link base, halfword at eight times the link value, and word at quadrupled link base plus 0x20.

A separate word destination, retained data base plus 0x20, receives, in order:

1. 0x20.
2. 0x2F.
3. The zero-extended byte read from retained data base plus 4.
4. 0x20 again.

That byte is read after the first two stores.

**Only after all four stores** is the retained global base loaded from 0x087FBBBC. The fixed word at 0x60 is then written with 0x20. The base load therefore precedes the 0x60 write, even if that write or a later derived store could alias its source.

The global base is retained throughout the repeated region. It is not reloaded for each repetition.

### Repeated region

Each represented group has the same four-width pattern:

- Word store at an eightfold-scaled base plus 0x54, carrying the quadrupled link base.
- Byte store at retained global base plus 0x19, carrying the retained data base’s low byte.
- Byte store at retained data base plus 0x11, carrying the retained global base’s low byte.
- Word store at retained global base plus 0x34, carrying the corresponding eightfold-scaled base.

The first group uses eight times the saved link word. Subsequent groups use eight times the adjusted base.

There is no visible cursor advancement, fresh global-base load, or additional input consumption between groups. Stores remain separate even when their destinations and values repeat. Exact multiplicity is **documented_only** in this contract; no single bulk-update equivalence is asserted.

### Closing snapshots and calculated fixed byte

After the repeated region, the function captures the represented word at fixed address 0xA9. It next saves the word at quadrupled link base plus 0x74.

It then writes a calculated byte at fixed address 0x3A: the low-byte result of eight times the adjusted base plus 0x30. That is an arithmetic byte value, not established text output.

Only after the 0x3A write are two further words captured:

- Doubled link base plus 0x14.
- Retained data base plus 4.

Consequently, a 0x3A write that aliases either later source can affect those later snapshots. It cannot retroactively change the already captured 0xA9 word.

The closing phase subsequently stores the captured 0xA9 word at retained data base plus 0x44, irrespective of later reads of the global byte at 0x67.

### Closing global byte dependencies

Before consulting 0x67, the body writes several derived bytes, including 0x6C at retained data base plus 8, and writes the doubled link base to its own offset 0x54. It then stores the captured 0xA9 word’s low byte at fixed address 0x67.

The subsequent global-byte uses are distinct:

- A derived word read at doubled link base plus 0x44 occurs before the read of 0x67 that populates fixed byte 0x21.
- Later, 0x67 is read for a byte store at retained data base plus 9.
- After additional derived writes and a word read at retained data base plus 0x54, 0x67 is read again for another byte store at retained data base plus 9.

Thus the supplied body explicitly shows **three subsequent read uses of 0x67: one fixed-byte destination and two derived-byte destinations**. It does not establish three derived-destination stores from that global.

These reads must not be collapsed into unconditional copies of the original captured low byte. Intervening derived writes may alias 0x67; even the first derived offset-9 destination may itself be 0x67.

### Remaining closing rewrites and final arguments

Closing operations also:

- Store the signed, variable-right-shift result as a byte at doubled link base plus 1.
- Rewrite retained global base offsets 0x44, 0x54, and 0x74.
- Read retained data base plus 0x40 and copy that word to its offset 0x10, after an earlier byte store at offset 0x10.
- Restore doubled link base plus 0x54 from its earlier saved offset-0x14 word.
- Propagate the low byte of the saved quadrupled-link offset-0x74 word to derived offsets 0x0D and 0x11.

A later word read at doubled link base plus 4 supplies another base. A halfword 0x65 is stored at twice that base. Its offset-0x64 word is saved for the final transfer’s second argument.

After further byte and word writes, including word value 100 at a derived offset 0x54, that base’s offset-0x74 word is saved for the fourth argument. A fresh doubled-link offset-0x64 read supplies a destination base and the value for the final doubled-address word store.

The final transfer’s third argument remains the earlier saved doubled-link offset-0x14 word. Later overwrites do not replace it.

## Side effects

Direct effects include mixed-width calculated-address writes, fixed-global writes, repeated identical stores, and address-like state rewrites. Dependencies may additionally affect any state allowed by their unresolved contracts.

No transaction, rollback, explicit error result, diagnostic, retry, allocation check, null check, or local fault handler is visible. If execution fails partway through, earlier effects may remain; the final transfer need not occur.

Atomicity, synchronization, reentrancy, thread safety, interrupt safety, and memory-mapped-device behavior remain **documented_only**.

## Dependencies

Only the subject function has an established published opaque identity. These additional opaque IDs are documentation-local labels:

| Opaque ID | Role | Established constraint |
|---|---|---|
| F-unknown-001 | Initial indirect dependency | Receives the first argument unchanged; target depends on entry control-link state. |
| F-unknown-002 | Remaining indirect dependency set | Ordered interactions consume transformed values and supply primary or secondary results. |
| F-unknown-003 | Final unresolved transfer | Receives 0x72 and the three specified saved words after closing writes. |

F-unknown-002 denotes unresolved targets, not one interchangeable callee. No dependency is assumed to preserve globals, preserve reachable memory, allocate, release, or retain memory.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Argument forwarding:** for first argument 0x12345678, F-unknown-001 receives that exact word. No direct dereference of that argument is established.
- **Small signed shift:** third argument 3 supplies 96 at both identified early shifted uses. Overflowing or negative-input shift behavior is not specified.
- **Explicit byte widening:** an explicit offset-5 byte read yielding 0xE2 supplies word value 226 where widened. This criterion does not apply to the unresolved-width global at 0x05.
- **Global conversion without byte narrowing:** if the missing declaration permits the 0x05-associated input to convert to 0x1234, subsequent address formation uses that converted value, not automatically 0x34.
- **Retained base:** if 0x087FBBBC yields a base and a subsequent write changes that global’s storage, repeated-region destinations still use the captured base.
- **Observable repetition:** the four offset-0x20 stores remain 0x20, 0x2F, loaded byte, 0x20. Emitting only the last store is insufficient for an observable-access environment.
- **Stable 0x67:** if the 0xA9 snapshot is 0x123456E2 and no intervening effect changes 0x67, its later reads yield 0xE2.
- **Aliased 0x67:** if an intervening derived write changes 0x67 to 0x7A, a later read uses 0x7A. The saved 0xA9 word remains 0x123456E2.
- **Snapshot order:** if the 0x3A store overlaps a subsequently captured source, that snapshot reflects the state after the store, subject to memory semantics.
- **Partial failure:** a fault after the 0x6C byte store does not imply rollback or execution of F-unknown-003.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and successful-state invariant.
- Individual dependency identities, signatures, preservation rules, and side effects.
- Completeness of represented control flow and secondary-result interpretation.
- Source access width and original type of the global at 0x05.
- Address validity, alignment, byte order, object extents, and global overlap.
- Signed arithmetic, address overflow, and oversized-shift semantics.
- Exact repeated-region multiplicity as an enumerated trace.
- Ownership, retention, release, and lifetime rules.
- Final target identity, purpose, termination, and return behavior.

Unusual low-address accesses do not justify replacing visible behavior with a more plausible algorithm.

## Evidence

This contract is based solely on static examination of the supplied body and opaque identity map. Prior prose, comments, and reviewer claims are not independent behavioral authority.

Explicit dereferences support access-width claims; the unsigned conversion of the undeclared 0x05-associated global does not independently establish its source width. Closing expressions establish the distinction between the saved 0xA9 word and later consultations of 0x67.

Packed-field and structured-state interpretations are inferences only. No implementation access, runtime execution, smoke run, or tests have been performed.
