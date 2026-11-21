<!-- Trusted provenance: F-fc56a5abc89a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fc56a5abc89a

## Purpose

F-fc56a5abc89a invokes an ordered series of indirect dependencies, performs mixed-width memory publications and global accesses, and reaches an unresolved terminal indirect transfer.

No application-level purpose is established. Numeric constants and address-like values do not establish parsing, formatting, allocation, initialization, or container semantics.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, captures, value relationships, and ordering.
- **[INFERENCE]:** consequences conditional on an explicitly stated execution model.
- **documented_only:** unresolved behavior for which no guarantee is established.

Every later effect depends on execution reaching it, earlier invocations returning, and required accesses succeeding.

## Interface

### Inputs and outputs

The represented interface has no declared arguments and no declared result. It exposes no explicit buffer, length, status, or ownership token.

Implicit inputs include incoming control-flow state, an incoming unsigned 32-bit register-derived value, incompletely defined secondary dependency-result state, dependency results, globals, and indirectly addressed memory.

Observable outputs comprise dependency invocations, memory reads and writes, global assignments, and the terminal transfer. Normal completion is not established.

### Widths and addressing

Explicit indirect accesses distinguish:

| Access | Width |
|---|---:|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word | 32 bits |
| Paired dependency-result capture | 64 bits, separated numerically into low and high 32-bit components |

All offsets in this contract are byte offsets. They identify accessed fields, not complete objects or allocation boundaries. Numeric separation of paired results does not determine memory byte order.

Named-global declarations are unavailable. Consequently:

- Reading global `0x05` into an unsigned 32-bit capture does not establish an eight-bit or 32-bit storage read.
- Assignments to globals `0x01`, `0x15`, and `0x67` explicitly narrow their operands to eight bits; destination access widths remain unresolved.
- Globals `0x54`, `0x60`, `0xa9`, `0x3a`, and `0x21` likewise have unresolved storage widths and destination conversions.
- A separate pointer-like global supplies a signed 32-bit capture, without establishing the width of its storage access.

Address arithmetic uses represented 32-bit quantities, including signed intermediates. Wrapping, signed overflow, negative-address interpretation, and invalid shift counts require an execution model. Arithmetic examples using bit-vector semantics are conditional, not unconditional language-level guarantees.

### Preconditions and failure behavior

Every reached target must support its represented invocation, and every reached location must support the applicable access.

No explicit bounds, nullness, alignment, lifetime, target-validity, or dependency-success checks are shown. No explicit error result, rollback, recovery, or compensating action is represented.

## Behavior

### Dependency phase and captured values

A long straight-line invocation phase precedes direct mutation. Its first target depends on incoming control-flow state plus `0xffe`; subsequent targets predominantly use fixed-relative calculations with that displacement.

Represented argument lists contain zero through four values. Calls without retained results remain ordered effects. No loop, retry, conditional branch, or explicit failure exit is shown in this phase. Dependencies themselves may branch, modify memory, fail, or not return.

Arguments include zero, secondary-result state, shifted captured words, paired-result components, and differently transformed copies of the same capture. Selections include:

- Incoming bits 2–18.
- Incoming low six bits shifted left by 11.
- Incoming low bit shifted left by 16.
- Result bits 8–23, 12–23, and 4–16.
- Bits 11–17 of a paired result’s high component.

Explicitly zeroed retained values remain zero through subsequent shifts and masks until reassignment. Under wrapping 32-bit semantics, cumulative shifts can also discard all original bits; this conclusion is conditional on that arithmetic model.

The following labels describe retained values, not object types:

| Label | Value relationship |
|---|---|
| Encoded base | An earlier paired result’s low component shifted left by 25 |
| Scaling seed | A later captured dependency result |
| Scaled base | Scaling seed multiplied by `0x2000000` |
| Compact index | Another result shifted left by 8, masked with `0x7fff`, then shifted right by 6 |
| Doubled index | Twice compact index |
| Shifted payload | A separate dependency result shifted left by 8 |
| Final pair | Last captured paired result before direct mutation |

A retained capture is not a later reread of its producer.

### Initial mutations and zero readback

The first publications occur in this order:

1. Encoded base as a word at final-pair-high plus `0x14`.
2. Final-pair-high’s low byte at encoded base plus 9.
3. Scaled base as a word at final-pair-low multiplied by `0x100`, plus `0x54`.
4. Encoded base’s low byte at encoded base plus 9, replacing the earlier byte.

Next, the routine clears the word at scaled base plus `0x14`, captures the byte at scaled base plus 5, clears the word at doubled index plus `0x14`, and publishes the captured byte as a zero-extended word at scaled base plus `0x34`. It then captures the word at scaled base plus `0x14`. Call this capture the **readback**.

**[INFERENCE]: Under ordinary sequential memory and conventional flat 32-bit addressing, the readback is zero.** Between the clear and capture:

- The doubled-index store writes only zero bytes, including any bytes overlapping the readback location.
- The zero-extended-byte publication begins `0x20` bytes beyond the cleared word. Its four-byte destination is disjoint from the readback’s four-byte source, including under conventional modulo-32-bit address wrapping.
- The intervening byte read does not change ordinary memory.

Ordinary aliasing therefore cannot make this readback nonzero. A nonzero readback requires additional unresolved behavior, such as device-memory semantics, external modification, or a nonstandard address model.

After capture, doubled index plus `0x14` is cleared again, and readback’s low byte is published twice at doubled index plus `0x15`. Both byte values are zero under the ordinary-memory conditions above.

Global `0x05` is then captured as an unsigned 32-bit **global-derived base**. Global `0x01` receives explicitly eight-bit-narrowed doubled index, followed by compact index’s low byte at global-derived base plus `0x15`.

Shifted payload’s low byte is written at scaled base plus `0x11`; shifted payload is then written as a word at readback plus `0x54`; finally, scaled base plus `0x11` is cleared. Under the zero-readback conditions, the payload word destination is address `0x54`. Its relationship to named-global storage depends on the memory model.

The word at global-derived base plus `0x54` is captured as the **anchor**, then replaced with readback—zero under those same conditions. Anchor is the memory value observed at that later capture, not necessarily an entry-state value.

### Anchor-derived publications

Compact index is published as a word at doubled index plus `0x34`. A word captured at scaled base plus `0x74`, the **captured link**, supplies a destination for scaling seed shifted left by 26 at offset `0x54`.

Another assignment performs two distinct word reads at doubled index plus `0x74`: one determines the destination base, and one supplies the value written at that base plus `0x54`. Their within-expression evaluation order is unresolved; both precede the destination write.

The following ordered publications occur:

- Anchor as a word at anchor plus `0x24`, then anchor’s low byte at captured link plus 9.
- Shifted payload as a word at captured link plus `0x54`.
- The anchor self-publication and captured-link byte publication again.
- Scaling seed shifted left by 26 at doubled index plus `0x14`.
- Anchor shifted left by one to global `0x54`.
- Anchor twice as a word at doubled index plus `0x14`.

Define **quadrupled anchor** as four times anchor and **octupled anchor** as eight times anchor. Quadrupled anchor’s low byte is published twice at doubled index plus `0x15`; global `0x15` then receives explicitly eight-bit-narrowed doubled index.

Next, shifted payload is published at quadrupled anchor plus `0x54`, and quadrupled anchor at doubled index plus `0x34`. A fresh word at doubled index plus `0x74` is captured and doubled to form the **doubled link**. Only afterward is quadrupled anchor published at its own `0x54`.

That capture must remain between the two quadrupled-anchor-plus-`0x54` publications: the destination may overlap its source.

Octupled anchor’s low byte is published at quadrupled anchor plus `0x15`, followed by doubled link as a word at anchor plus `0x44`.

A fresh byte at quadrupled anchor plus 5 selects a word at byte-value plus `0x74`. Its result is the **byte-selected link**. Doubled link is published at byte-selected link plus `0x44`.

Quadrupled anchor then receives, in order, its own low byte at `0x15`, byte-selected link’s low byte at `0x0d`, and its own word value at `0x14` and `0x54`. A fresh word at quadrupled anchor plus `0x64` supplies another base, which receives byte-selected link’s low byte at `0x0d` and its own word value at `0x14`.

### Distinct byte captures and dependent selection

A fresh word at quadrupled anchor plus 4 becomes the **captured source base**.

Before the first byte capture at captured source base plus 5, byte-selected link’s low byte is published at doubled link plus 9, and octupled anchor is published as a word at doubled link plus `0x44`. Either can affect the first byte through aliasing.

Between the first and second byte captures, the ordered effects are:

1. Doubled link’s low byte at quadrupled anchor plus 9.
2. First captured byte, zero-extended to a word, at byte-selected link plus `0x44`.
3. Octupled anchor as a word at quadrupled anchor plus `0x14`.
4. Captured source base’s low byte at doubled link plus `0x15`.
5. Quadrupled anchor as a word at doubled link plus `0x44`.

The second byte is then captured from captured source base plus 5. After capture, octupled anchor’s low byte is published at quadrupled anchor plus `0x15`. Only afterward is a word read at second-byte-value plus `0x74`, producing the **selected link**.

The selector is retained across the intervening byte publication; the selected word is read afterward. Neither byte capture can be substituted for the other.

Twice selected link becomes the **terminal base**. Before the constant stores, the routine publishes captured source base’s low byte at quadrupled anchor plus `0x15`, quadrupled anchor at its own `0x54`, terminal base’s low byte at quadrupled anchor plus 9, and octupled anchor at quadrupled anchor plus `0x14`.

### Constants and repeated publication groups

Constant `0x3a` is stored as:

- A byte at four times selected link.
- A halfword at eight times selected link.
- A word at four times selected link plus `0x20`.

Define **adjusted anchor** as quadrupled anchor minus `0x96`, and **publication base** as four times adjusted anchor.

Four successive word publications at publication base plus `0x20` contain `0x20`, `0x2f`, a freshly captured byte from publication base plus 4 zero-extended to a word, and `0x20`.

The pointer-like global is captured as the signed 32-bit **captured global base**. Global `0x60` subsequently receives `0x20`.

Each repeated publication group has four ordered effects:

| Order | Publication |
|---|---|
| First | Four times selected link as a word at scaled publication address plus `0x54` |
| Second | Publication base’s low byte at captured global base plus `0x19` |
| Third | Captured global base’s low byte at publication base plus `0x11` |
| Fourth | Scaled publication address as a word at captured global base plus `0x34` |

The first group uses octupled anchor as its scaled publication address; subsequent groups use eight times adjusted anchor. Captured global base remains retained throughout.

These are repeated straight-line effects, not an established runtime loop. This contract leaves their exact multiplicity **documented_only** rather than certifying a count.

### Late captures and global arithmetic

After those publications, global `0xa9` is captured into a 32-bit value. A word at four times selected link plus `0x74` is then captured as the **selected-field capture**.

Global `0x3a` receives adjusted anchor narrowed to a character, multiplied by eight, and increased by `0x30`. Character signedness and destination conversion remain unresolved. Multiplication and addition are not established as eight-bit arithmetic.

Only after that global assignment are words captured at terminal base plus `0x14` and publication base plus 4. The assignment can affect either capture if its storage overlaps the source.

The arithmetic result’s low eight bits match those of eight times adjusted anchor plus `0x30`; this does not establish an eight-bit destination write. This assignment is separate from the constant-`0x3a` stores.

### Final publications and fresh global reads

Publication base’s low byte is written at terminal base plus `0x0d`. Publication base then receives byte `0x6c` at 8, followed by captured global base’s low byte at `0x0d` and 9. Terminal base is published as a word at its own `0x54`.

Global `0x67` receives the eight-bit-narrowed `0xa9` capture. A word at terminal base plus `0x44` is captured before a fresh global `0x67` read supplies the assignment to global `0x21`.

The captured word supplies three ordered publications:

- Its low byte at publication base plus `0x10`.
- Its word value at captured global base plus `0x54`.
- A right-shifted byte at terminal base plus 1.

The signed right-shift count is publication base’s low eight bits. Negative-operand and oversized-count semantics remain unresolved.

Publication base then replaces the word at captured global base plus `0x54`. A fresh word at publication base plus `0x40` is retained while the earlier publication-base-plus-4 capture is written at captured global base plus `0x74`. The retained word subsequently overwrites publication base plus `0x10`, overlapping the earlier byte publication. Terminal base is published at captured global base plus `0x44`.

A second fresh `0x67` read supplies a byte at publication base plus 9. Next come publication base’s low byte at terminal base plus 9, the full `0xa9` capture at publication base plus `0x44`, and a fresh word capture at publication base plus `0x54`, called the **new base**. A third fresh `0x67` read supplies another byte at publication base plus 9.

### Completion and terminal transfer

Selected-field capture’s low byte is published at new base plus `0x0d`. The earlier terminal-base-plus-`0x14` capture is restored at terminal base plus `0x54`.

A fresh word at publication base plus `0x14` is retained. Selected-field capture’s low byte is published at terminal base plus `0x11`; new base’s low byte replaces publication base plus 9; new base is published as a word at publication base plus `0x44`.

A fresh word at terminal base plus 4 becomes the **further base**. Halfword `0x65` is written at twice further base before the word at further base plus `0x64` is captured.

New base receives the retained publication-base-plus-`0x14` word’s low byte at `0x11`, then word decimal 100 at `0x54`. A word is captured at further base plus `0x74`, followed by a fresh word at terminal base plus `0x64`, the **newest base**.

The last direct publications are further base’s low byte at terminal base plus 9, the further-base-plus-`0x64` capture at newest base plus `0x54`, and newest base as a word at twice that capture.

The terminal indirect transfer has four represented arguments: `0x72`, the further-base-plus-`0x64` capture, the earlier terminal-base-plus-`0x14` capture, and the further-base-plus-`0x74` capture.

An apparent return follows. The target, transfer semantics, return behavior, and return reachability remain unresolved.

## Side effects

Access widths, repeated publications, retained-versus-fresh values, and cross-statement ordering are behavioral distinctions. They must not be collapsed merely because destinations or values appear equal.

No general non-aliasing guarantee exists. Derived writes may affect later derived or global reads. Unknown global widths leave overlap extents unresolved. This general aliasing risk does **not** invalidate the ordinary-memory zero-readback conclusion: the specific intervening writes cannot introduce nonzero bytes into that source.

No direct allocation or release is established. Dependencies may allocate, retain, release, or mutate memory. Publishing an address-like value establishes neither ownership transfer nor lifetime extension.

No transaction, rollback, synchronization, atomicity, or reentrancy guarantee is established. Interrupted execution may leave earlier effects intact.

## Dependencies

F-fc56a5abc89a is the only supplied opaque function identity. No distinct opaque IDs are available for its callees, so none are invented.

Unidentified dependencies include the entry-state-derived invocation, subsequent indirect invocations, producers of incompletely represented secondary-result state, and terminal transfer.

Their identities, conventions, errors, memory effects, global preservation, ownership rules, and return behavior are **documented_only**. No callee is assumed pure or responsible for preserving globals or releasing memory.

## Acceptance examples

These examples are conditional static requirements, not executed tests.

### Zero readback despite ordinary aliasing

Assume successful accesses, ordinary sequential memory, conventional flat 32-bit addressing, and no external modification.

After the clear at scaled base plus `0x14`, an overlapping zero-word store at doubled index plus `0x14` cannot introduce nonzero bytes. Publishing even `0xff` as a zero-extended word at scaled base plus `0x34` cannot affect the readback: the two four-byte ranges are separated by `0x20` bytes.

The readback is therefore zero, both readback-derived byte publications are zero, the shifted-payload word destination is `0x54`, and zero replaces the anchor-source word after anchor capture. Earlier payload writes may affect what the anchor capture observes.

A nonzero device-register read or externally modified memory value falls outside these assumptions; ordinary aliasing alone is not an explanation.

### Width and arithmetic boundaries

For incoming word `0x0007ffff`, selecting bits 2–18 gives `0x0001ffff`. Under wrapping 32-bit shifts, shifting the incoming word left by 26 gives `0xfc000000`; selecting its low six bits and shifting left by 11 gives `0x0001f800`.

Under 32-bit bit-vector semantics, compact index ranges from 0 through 508 in multiples of four; doubled index ranges from 0 through 1016 in multiples of eight. These ranges do not establish valid memory.

If global `0x05` converts to `0x1234`, its byte-publication destination is `0x1249`, and the anchor capture address is `0x1288`. Truncation to `0x34` is unsupported without the declaration.

### Capture ordering

If quadrupled anchor plus `0x54` overlaps doubled index plus `0x74`, doubled-link capture occurs after the shifted-payload publication but before the self-publication. Moving the self-publication earlier can change the capture.

If the second source-byte capture is `0x20`, the selected-word address is `0x94`. The intervening octupled-anchor byte publication occurs before that word read. Overlap with the selector’s original source does not change the retained `0x20`; overlap with the selected word can change its contents.

### Global arithmetic and freshness

For narrowed character `0x12`, conventional signed and unsigned eight-bit interpretations both produce `0xc0` in the global-`0x3a` arithmetic.

For pattern `0x90`, unsigned interpretation produces decimal 1200, while signed eight-bit interpretation produces decimal −848. Both have low byte `0xb0`; wider stored values depend on the missing declaration.

If the three fresh `0x67` reads yield `0x22`, `0x33`, and `0x44`, global `0x21` receives the first subject to conversion, and the two explicit byte publications receive `0x33` and `0x44`. Reusing the original `0xa9` capture would not preserve these dependencies.

## Uncertainties

The following remain **documented_only**:

- Application intent, consumer, and function-boundary accuracy.
- Dependency identities, conventions, and secondary-result definitions.
- Terminal target and return reachability.
- Global storage widths, signedness, and conversions.
- Character signedness, signed overflow, oversized shifts, address wrapping, byte order, and alignment.
- Memory validity, external modification, device-memory behavior, and nonstandard addressing.
- Exact repeated-publication multiplicity.
- Allocation, retention, release, ownership, and lifetime effects.

Readback uncertainty is specifically conditional on exceptional memory or address-model behavior. Under the ordinary sequential flat-memory assumptions stated above, its value is established as zero.

## Evidence

This contract derives from static inspection of the supplied representation. Prior prose, comments, and reviewer feedback are not independent behavioral evidence.

Explicit access types support byte, halfword, and word distinctions. Capture widths and narrowed assignment operands do not establish missing global storage widths.

The zero-readback conclusion follows from the represented clear, the intervening zero-only store, and the disjoint publication `0x20` bytes beyond the cleared word. It does not rely on a general non-aliasing assumption.

The representation also establishes the doubled-link capture’s position between early publications, two distinct source-byte captures, the retained second selector across an intervening write, and three separate late global reads.

No runtime tests, smoke runs, execution traces, or memory-validity checks have been performed.
