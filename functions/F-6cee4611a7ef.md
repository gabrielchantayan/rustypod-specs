<!-- Trusted provenance: F-6cee4611a7ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6cee4611a7ef

## Purpose

F-6cee4611a7ef destructively rewrites memory reached through two entry-state roots and subsequently through mutable links. It stores markers, truncated values, captured links, and self-references. Its final phase performs computed-address writes and invokes an unresolved indirect destination.

The established behavior is an ordered sequence of accesses, not merely a transformation from an initial graph to a final graph. Earlier writes can change later traversal, including when a byte write overlaps a link word.

No application-level purpose, object model, ownership scheme, or data format is established. No runtime tests have been performed.

## Interface

### Explicit inputs and additional state

The represented interface has four positional inputs:

| Input | Represented width | Use |
|---|---:|---|
| First | 32 bits | No visible use. |
| Second | 32 bits | Address receiving a word at offset `0x14`. |
| Third | 32 bits | Stored as a word, truncated for byte stores, and used as an address. |
| Fourth | 32 bits | Stored as a word and truncated for a byte store. |

The body also consumes entry state whose initialization is absent:

- A **primary root** address.
- A **secondary root** address.
- An unsigned 32-bit **index state**.

Three external 32-bit values are read near completion. Their storage classification and initialization are unknown.

### Results and completion

There is no represented explicit return value. Outputs are memory mutations and an indirect invocation with three arguments:

1. A byte-derived unsigned 32-bit value, bitwise ORed with `0x44`.
2. The second of the three late external values.
3. A late snapshot of the active record’s word at `0x44`.

The representation returns after the indirect invocation. Whether the underlying transfer actually returns, and its ABI, remain unresolved.

### Required access conditions

Every reached address must support the specified access width and direction for normal completion. This includes low addresses derived from a byte, doubled addresses, sums of address-like values, and a negative displacement.

There are no visible validation checks or error-return branches. Records need not be distinct. Aliasing is permitted by the representation but may change every subsequent target and may make an otherwise valid traversal invalid.

## Behavior

### Access conventions and interpretation

All offsets are byte offsets. **Byte**, **halfword**, and **word** mean one, two, and four bytes, respectively. Unless a width is explicitly stated otherwise below, field reads and writes are words. “Low byte” and “low halfword” retain eight and sixteen bits.

The named records and snapshots below are descriptive identities introduced for this contract, not structure types. A record name denotes an address already captured; it does not imply that a later read of the field that selected it returns the same address.

Fields overlap. For example, a byte at `0x11` overlaps a word at `0x10`, and a byte at `0x15` overlaps a word at `0x14`. A word marker therefore cannot be replaced by a byte marker, and an apparently redundant store cannot be discarded from the represented access sequence.

Byte order, alignment rules, and pointer width are not established independently of the recovered representation. Address-like fields are accessed as four-byte values. Their use does not prove a portable structure layout.

### Entry linking and first working record

The primary-root address is first stored at the second input’s `0x14`. The third input’s low byte is then stored at secondary-root `0x11`. Two successive reads of primary-root `0x44` capture separate values: an **initial link snapshot** and an **initial selected record**.

A byte is read at twice the unsigned index state. It is promoted through the represented character and integer types, right-shifted by the index state’s low eight bits, and reduced to a byte. This **shifted byte** is written to the initial selected record’s `0x08`. The initial link snapshot is then stored at primary-root `0x74`, followed by the shifted byte at primary-root `0x11`.

Secondary-root `0x54` receives the third input and then the fourth input as separate word writes. Secondary-root `0x19` receives the third input’s low byte. The initial selected record’s `0x44` receives the primary-root address. The third input is then stored at the address represented by that same input plus `0x54`.

The initial selected record’s `0x14` selects a byte-write target. Primary-root `0x11` receives the fourth input’s low byte. A fresh primary-root `0x14` read supplies the low byte written to that target’s `0x15`.

Another primary-root `0x14` read selects the **first working record**. Primary-root `0x64` is captured as the **early root value**.

### First working-record rewrite

The first working record receives word `0x73` at `0x10`; primary-root `0x44` becomes a self-reference; and the working record receives byte `0x73` at `0x14`. Primary-root `0x54` receives the fourth input.

A fresh primary-root `0x74` read produces the **early link value**. Primary-root `0x09` receives the primary address’s low byte, and working-record `0x54` receives the early link value. Working-record `0x10` becomes word `0x6e`; primary-root `0x44` is written as a self-reference again; working-record `0x20` becomes word `0x6e`.

A new primary-root `0x64` snapshot, the **later root value**, supplies a byte at working-record `0x09`. Working-record `0x74` selects a **first linked record**, whose `0x54` is captured.

Working-record `0x1c` receives byte `0x79`, primary-root `0x11` receives the later root value’s low byte, and primary-root `0x09` receives the early root value’s low byte. Working-record `0x1c` then receives `0x68`. Primary-root `0x11` again receives the later root value’s low byte, followed by the primary address’s low byte at first-linked-record `0x11`.

Working-record `0x50` is captured as its **payload**. Primary-root `0x44` receives the early root value; primary-root `0x64` and `0x34` receive the later root value, in that order. Working-record `0x11` receives the later root value’s low byte.

A fresh primary-root `0x74` read captures the **marker link value**. Working-record `0x10` receives the payload. The captured first-linked-record `0x54` value is written back there. Working-record `0x00` receives the payload’s low byte, then its `0x09` receives the primary address’s low byte.

A fresh primary-root `0x14` read selects the **active record** used throughout the remaining description. First-linked-record `0x11` receives the working address’s low byte; working-record `0x44` receives the active address; working-record `0x00` receives the payload’s low byte again.

First-linked-record `0x74` is captured as the **negative-displacement target**, and its `0x14` selects the **branch record**. Working-record `0x54` then becomes a self-reference.

### First decimal-100 group and intervening branch writes

Branch-record `0x04` selects a **marker child**. That child first receives the marker link value’s low byte at `0x11`, then byte `100` at `0x10`. Active-record `0x54` receives word `100`, followed by word `100` at child `0x60`.

The active record’s current `0x74` target is captured. Active-record `0x74` is replaced with the branch address **before** word `100` is written at the captured target’s `0x30`.

Branch-record `0x14` selects a **descendant**. Its `0x09` receives the marker link value’s low byte. Active-record `0x54` becomes a self-reference. A word equal to the negative-displacement target minus `0x74` is stored at that target minus `0x20`.

The descendant’s `0x14` selects another byte-write target. Branch-record `0x11` receives the early link value’s low byte. A fresh branch-record `0x14` read produces a **branch-link snapshot**, whose low byte is written at the byte-write target’s `0x15`.

Branch-record `0x14` is read again to select the **current descendant**. Active-record `0x64` is captured as the **first active value**, then written back unchanged. Current-descendant `0x0c` receives byte `0x73`; active-record `0x15` receives the first active value’s low byte.

Branch-record `0x34` now receives the branch-link snapshot. Current-descendant `0x74` is captured as the **early descendant link**, then current-descendant `0x54` becomes a self-reference. Branch-record `0x0d` receives the active address’s low byte. Branch-record `0x54` receives the branch-link snapshot, and branch-record `0x14` receives the separately reloaded current-descendant address.

Only after those writes is active-record `0x64` read again, producing the **second active value**. These intermediate branch writes remain observable even though branch-record `0x54` is later replaced with a self-reference.

### The `0x74` group

Current-descendant `0x10` receives byte `0x74`; active-record `0x54` receives word `0x74`; current-descendant `0x10` then receives word `0x74`.

Current-descendant `0x74` is read again, capturing the **saved descendant target**. Branch-record `0x54` becomes a self-reference. Current-descendant `0x30` receives word `0x74`.

Active-record `0x01` receives the first active value’s low byte. At the address represented by the first active value, byte offset `0x09` receives the second active value’s low byte. Active-record `0x74` receives the second active value, and active-record `0x11` receives byte `0x74`.

Current-descendant `0x60` is captured. Active-record `0x11` is replaced by the first active value’s low byte; active-record `0x34` receives the second active value; current-descendant `0x10` receives the captured `0x60` value’s low byte.

### The `0x66` group and following self-reference

A fresh active-record `0x14` read selects the **next marker record**. Branch-record `0x11` receives the early descendant link’s low byte. Active-record `0x44` receives the branch address; active-record `0x11` receives the second active value’s low byte.

Active-record `0x74` is captured as the **pre-marker link**. Active-record `0x0d` receives the branch address’s low byte. Next-marker-record `0x10` receives word `0x66`; active-record `0x44` again receives the branch address; next-marker-record `0x10` receives byte `0x66`. Active-record `0x54` and next-marker-record `0x60` then receive word `0x66`.

The active record’s current `0x74` target is captured as the **old marker target**. Active-record `0x74` is replaced with the branch address **before** old-marker-target `0x40` receives word `0x66`.

Active-record `0x0d` receives the pre-marker link’s low byte. Old-marker-target `0x44` is captured. Active-record `0x54` becomes a self-reference, followed by a self-reference at saved-descendant-target `0x54`.

Old-marker-target `0x14` selects a byte-write target. Branch-record `0x11` receives the captured old-marker-target `0x44` value’s low byte. A fresh branch-record `0x14` read supplies the low byte written at the byte-write target’s `0x15`.

### Late traversal and saved-value propagation

Another branch-record `0x14` read selects the **late descendant**. Active-record `0x64` is captured as the **first late value**. Late-descendant `0x20` receives word `0x73`. Active-record `0x64` is read again as the **second late value**, whose low byte is written at late-descendant `0x09`.

Late-descendant `0x74` selects the **late linked record**. Its `0x0d` receives the active address’s low byte. Its `0x54` is read and written back unchanged, followed by byte `0x74` at its `0x09`.

Active-record `0x44` receives the first late value. Late-linked-record `0x34` receives the active address. Late-linked-record `0x04` selects a **carrier**, whose `0x24` receives the second late value. Active-record `0x19` receives the first late value’s low byte.

Carrier `0x74` is captured as the **saved traversal value**; carrier `0x04` is captured as the **replacement active value**. Active-record `0x74` is captured as the **traversal anchor**. Reading anchor `0x14` selects a record whose `0x54` receives the saved traversal value.

Anchor `0x04` selects a **marker carrier**. Active-record `0x0d` receives the second late value’s low byte twice consecutively. Active-record `0x64` receives the replacement active value. Marker-carrier `0x30` receives word `0x74`.

A fresh anchor `0x04` read, followed by that target’s `0x74`, selects the **propagation record**. A fresh active-record `0x64` read supplies a **retained byte**, written at active-record `0x1d`.

Active-record `0x44` is read, then its target’s `0x14` selects a **replacement link**. Propagation-record `0x34` receives the saved traversal value. Active-record `0x14` receives the replacement link.

### Final record preparation

Active-record `0x54` is captured as the **prior active field**. Replacement-link `0x09` receives the active address’s low byte. Propagation-record `0x04`, followed by that target’s `0x74`, selects a **follower**.

Follower `0x15` receives the saved traversal value’s low byte. Active-record `0x54` becomes a self-reference; active-record `0x11` receives the propagation address’s low byte; follower `0x11` receives the prior active field’s low byte.

Follower `0x64` is captured as the **final record value**. Active-record `0x09` receives the retained byte. A fresh propagation-record `0x04` read selects the **final working record**.

Final-working-record `0x09` receives the active address’s low byte. Its `0x50` is captured as the **final payload**. Its `0x11` receives the final record value’s low byte, and its `0x09` again receives the active address’s low byte.

Active-record `0x14` receives the final record value. Active-record `0x0d` receives the final working address’s low byte. Final-working-record `0x00` receives the final payload’s low byte.

Propagation-record `0x74` is captured as the **final byte-read base**; propagation-record `0x14` selects the **final branch**. Final-working-record `0x54` becomes a self-reference. Final-branch `0x04` selects the **final child**.

Final-child `0x10` receives byte `100`; active-record `0x54` receives word `100`; final-child `0x40` receives word `100`. Active-record `0x0d` receives the final record value’s low byte. Final-branch `0x09` receives the final child address’s low byte. Active-record `0x24` receives the final record value. Final-branch `0x11` receives the active address’s low byte.

### Computed-address writes and dispatch

The first external word is loaded. Active-record `0x74` is then captured as the **final active link**. Its low halfword is stored at the sum of that link and the active address.

The first external word’s low halfword is stored at twice the final child address. A character byte is read at the sum of the final byte-read base and the active address. Character promotion followed by conversion to unsigned 32 bits produces the **dispatch value**.

The final active link’s low halfword is stored at the sum of the first external word and that link. The halfword store at the sum of the final active link and active address is repeated.

Word `0x45` is stored at dispatch value plus `0x45`. The second external word is loaded, then the third external word. Active-record `0x44` is captured for dispatch.

Byte `0x44` is stored at dispatch value plus `0x44`. The third external word is stored as a word at twice the second external word. The second external word’s low halfword is stored at its sum with the saved traversal value.

The indirect invocation then receives the three arguments specified in the interface.

## Side effects

The function directly modifies caller-reachable and pointer-reached storage. No allocation, deallocation, reference counting, locking, or explicit I/O operation is visible before dispatch.

Self-references and backlinks establish stored values, not ownership transfer or retention. The indirect dependency’s memory effects, ownership behavior, and changes to external state are unknown.

No rollback or recovery path is represented. If an access fails, preceding effects are not shown being undone. Exact fault behavior is outside the available evidence. No synchronization or whole-operation atomicity is established.

## Dependencies

- **F-6cee4611a7ef:** the supplied function identity; no recursive invocation is visible.
- **Unresolved indirect dependency — documented_only:** no opaque identity is available. Its target set, ABI, return behavior, ownership contract, and external-state preservation are unknown.
- **External state — documented_only:** two roots, the unsigned index state, and three late word reads. Their producers and lifetime guarantees are unavailable.

No additional function identity can be assigned from the supplied evidence.

## Acceptance examples

These are conditional static consequences, not executed tests.

- **Link replacement precedes marker write.** In the first decimal-100 group, suppose the captured old active `0x74` target plus `0x30` equals active `0x74`. The branch address is stored first, then word `100` overwrites it. For the `0x66` group, the corresponding condition uses old target plus `0x40`; the final overlapping value is `0x66`, not the branch address.

- **Branch snapshots remain distinct.** If an intervening write changes branch `0x14` between its snapshot and reload, branch `0x34` and its intermediate `0x54` receive the earlier snapshot, while the later write to branch `0x14` uses the reloaded descendant address. One cached value does not satisfy this behavior.

- **Saved traversal value has two destinations.** With stable, valid links, the saved traversal value reaches both the anchor’s `0x14` target at `0x54` and the propagation record at `0x34`. The latter target comes from a fresh anchor `0x04` traversal; it cannot be assumed identical to an earlier target.

- **Byte writes affect later links.** If a byte write at `0x15` overlaps a subsequently read word at `0x14`, the later read consumes the changed word. Entry-time graph snapshots are insufficient.

- **Widths remain observable.** A word store of `0x66` modifies four bytes. The following byte store at the same offset is a separate one-byte effect. A value `0x123456ab` truncates to byte `0xab` or halfword `0x56ab`; physical byte order remains unspecified.

- **Low-address dispatch writes.** If the final character read yields nonnegative `0x20`, the word write uses address `0x65`, the byte write uses `0x64`, and the first indirect argument is `0x64`. Completion requires those addresses to be writable.

## Uncertainties

- **documented_only — representation fidelity:** recovered types, pointer treatment, and indirect-transfer form are not independently verified.
- **documented_only — character semantics:** plain-character signedness is unspecified. High-bit bytes may promote differently, affecting both the initial shift and final dispatch-derived addresses.
- **documented_only — shifts:** the count can exceed a 32-bit operand’s valid shift range. No masking rule or defined result is established for such cases; right shift of a negative promoted character also requires target confirmation.
- **documented_only — arithmetic:** unsigned index doubling is represented as 32-bit unsigned arithmetic. Signed address additions, subtractions, and doublings are not established as wrapping operations. Overflowing cases cannot safely be assigned unbounded mathematical or modular results.
- **documented_only — execution environment:** byte order, alignment, memory mapping, concurrency semantics, and exact fault behavior are unavailable.
- **Inferred intent only:** repeated marker groups suggest structured memory rewriting, but no domain interpretation is established.

## Evidence

Established static evidence is the supplied representation of F-6cee4611a7ef: four explicit inputs, additional entry state, ordered byte/halfword/word accesses, mutable-link traversal, three late external reads, and an unresolved indirect invocation.

No executable, target specification, dependency implementation, ownership contract, or runtime trace was supplied. No runtime tests have been performed. The descriptive record names organize captured addresses without asserting disjoint storage or stable links.
