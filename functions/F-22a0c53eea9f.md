<!-- Trusted provenance: F-22a0c53eea9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-22a0c53eea9f

## Purpose

F-22a0c53eea9f performs an ordered transformation of externally supplied memory, follows links obtained from that memory, and invokes an unresolved indirect target.

The established behavior is an access-sensitive memory contract. The evidence does not establish a conventional object model, collection operation, text construction, or protocol. Interpreting the small constants as characters would lose material distinctions between byte and word writes, address calculations, and intervening link changes.

Successful execution depends on incoming machine state and a memory graph that remain largely undocumented. The function does not validate that environment before modifying it.

## Interface

### Inputs and result

The displayed interface contains four explicit 32-bit arguments:

| Argument | Visible use |
|---|---|
| First | No direct use in the visible body. |
| Second | No direct use in the visible body. |
| Third | Stored as a word, narrowed to a byte, and used as an address base. |
| Fourth | Stored as a word and narrowed to a byte. |

There is no established explicit return value. The representation ends with an indirect invocation followed by a return without a result. Whether that invocation returns is unknown.

Additional incoming state supplies:

- **Index:** a 32-bit unsigned value.
- **Root:** the initial primary address.
- **Peer:** an address used by early stores.
- An unresolved indirect control-transfer target.

Three ambient word-sized locations are read near the end. Their contents are called **Terminal A**, **Terminal B**, and **Payload**. These names identify values captured at particular reads, not immutable global state.

The calling-convention origins of the additional inputs are **documented_only**.

### Widths and memory interpretation

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. All offsets below are byte displacements and are hexadecimal unless explicitly stated otherwise. Decimal 100 is the value 0x64.

“Low byte” and “low halfword” mean the least significant 8 and 16 bits of a captured value. They do not mean the first bytes at its storage address.

Addresses and stored address values are represented through the supplied 32-bit operations. Exact pointer representation, alignment requirements, and byte order are unknown. No structure layout, allocation boundary, or non-aliasing guarantee is established.

Names such as **First node**, **Working node**, and **Companion** are contract labels for captured addresses. They do not imply types or ownership.

### Preconditions and failures

Reaching the terminal invocation requires every executed access to have a usable destination or source of the stated width. This includes:

- Storage reached through words loaded from memory.
- Storage addressed using the third argument.
- Negative displacements, doubled addresses, and sums of address-valued words.
- The ambient terminal locations and indirect target.

No null, bounds, alignment, or arithmetic checks are visible. No recoverable error result, rollback, or cleanup path is established. Failure can occur after substantial partial mutation.

## Behavior

### Access-order contract

The following stages preserve the visible ordering of reads and writes. Within each paragraph, described operations occur in the stated order.

A retained value is not interchangeable with a later reload of the same location. Arbitrary aliasing can make the values differ. Repeated stores and read-and-write-back operations remain observable accesses even when ordinary memory would end with unchanged bytes.

The contract does not authorize collapsing separate link reads or moving reads across intervening stores.

### Initial capture and argument-dependent writes

Root +0x44 is read twice. The first word is retained; the second supplies **Initial link**.

A byte is read at twice Index. The unsigned 32-bit multiplication determines this initial address value. The displayed signed-character interpretation is widened to 32 bits, shifted right by Index’s low eight bits, and narrowed to obtain **Derived byte**. Exact negative-value shifting and out-of-range shift behavior remain platform-dependent.

Derived byte is written to Initial link +0x08. The first Root +0x44 capture is stored at Root +0x74, followed by Derived byte at Root +0x11.

Peer +0x54 receives the third argument and then the fourth argument as separate word writes. Peer +0x19 receives the third argument’s low byte. Initial link +0x44 receives Root, and third argument +0x54 receives the third argument itself.

Initial link +0x14 is then read as an address. Root +0x11 receives the fourth argument’s low byte. A fresh Root +0x14 word supplies the byte written to that address +0x15. Root +0x14 is read again to select **First node**.

### Root snapshots and First-node mutation

An initial Root +0x64 word is retained as **Root snapshot A**. First node +0x10 receives word 0x73; Root +0x44 receives Root; First node +0x14 receives byte 0x73; and Root +0x54 receives the fourth argument.

Root +0x74 is captured as **Root link A**. Root +0x09 receives Root’s low byte, and First node +0x54 receives Root link A. First node +0x10 then receives word 0x6E, Root +0x44 receives Root again, and First node +0x20 receives word 0x6E.

A new Root +0x64 capture, **Root snapshot B**, supplies the byte written to First node +0x09. First node +0x74 selects **Branch**, whose +0x54 word is retained.

The next writes, in order, are:

- Byte 0x79 to First node +0x1C.
- Root snapshot B’s low byte to Root +0x11.
- Root snapshot A’s low byte to Root +0x09.
- Byte 0x68 to First node +0x1C.
- Root snapshot B’s low byte to Root +0x11 again.
- Root’s low byte to Branch +0x11.

First node +0x50 is captured. Root +0x44 receives Root snapshot A; Root +0x64 and +0x34 receive Root snapshot B; First node +0x11 receives Root snapshot B’s low byte.

A fresh Root +0x74 capture becomes **Root link B**. First node +0x10 receives the captured +0x50 word, and Branch +0x54 receives its retained word. First node’s byte zero receives the +0x50 low byte, followed by Root’s low byte at First node +0x09.

### Working-node selection and first restructuring

Root +0x14 is reloaded to select **Working node**. Branch +0x11 receives First node’s low address byte; First node +0x44 receives Working node; and First node’s byte zero receives the captured +0x50 low byte again.

Branch +0x74 supplies **Outer link**, then Branch +0x14 supplies **Companion**. First node +0x54 receives First node.

An address read from Companion +0x04 receives Root link B’s low byte at +0x11 and byte 100 at +0x10. Working node +0x54 then receives word 100, before that same +0x04 target receives word 100 at +0x60.

Working node +0x74 is captured before being replaced with Companion. The captured target receives word 100 at +0x30.

Companion +0x14 selects an address whose +0x09 receives Root link B’s low byte. Working node +0x54 receives Working node. Outer link −0x20 receives the word-valued address Outer link −0x74.

These negative displacements require storage preceding Outer link; they are not ordinary nonnegative field accesses.

### Separate Working-node snapshots and 0x74 mutations

The address just selected through Companion +0x14 is followed through its +0x14. Companion +0x11 receives Root link A’s low byte. A fresh Companion +0x14 word is captured and its low byte written to the followed target +0x15. Companion +0x14 is then reloaded to select **Selected descendant**.

Working node +0x64 is captured as **Working snapshot A** and written back unchanged. Selected descendant +0x0C receives byte 0x73. Working node +0x15 receives Working snapshot A’s low byte; Companion +0x34 receives the earlier Companion +0x14 word.

Selected descendant +0x74 is captured as **Descendant link A**. Selected descendant +0x54 receives its own address. Companion +0x0D receives Working node’s low byte; Companion +0x54 receives the earlier Companion +0x14 word; Companion +0x14 receives Selected descendant.

A later Working node +0x64 read supplies **Working snapshot B**. The following accesses remain ordered:

- Selected descendant +0x10 receives byte 0x74.
- Working node +0x54 receives word 0x74.
- Selected descendant +0x10 receives word 0x74.
- Selected descendant +0x74 is freshly captured as **Descendant link B**.
- Companion +0x54 receives Companion.
- Selected descendant +0x30 receives word 0x74.
- Working node +0x01 receives Working snapshot A’s low byte.
- **Working snapshot A +0x09 receives Working snapshot B’s low byte.**
- Working node +0x74 receives Working snapshot B.
- Working node +0x11 receives byte 0x74.

Selected descendant +0x60 is then read. Working node +0x11 receives Working snapshot A’s low byte, Working node +0x34 receives Working snapshot B, and Selected descendant +0x10 receives the freshly read +0x60 word’s low byte.

The destination based on Working snapshot A and the byte from Working snapshot B must remain distinct. Neither their full values nor their low bytes are established equal.

### Two +0x74 captures during the 0x66 mutations

Working node +0x14 is reloaded as **Mutation target**. Companion +0x11 receives Descendant link A’s low byte; Working node +0x44 receives Companion; Working node +0x11 receives Working snapshot B’s low byte.

Working node +0x74 is now captured as **Early mutation link**, before Working node +0x0D receives Companion’s low byte and before the 0x66 writes.

Mutation target +0x10 receives word 0x66. Working node +0x44 receives Companion again. Mutation target +0x10 receives byte 0x66, Working node +0x54 receives word 0x66, and Mutation target +0x60 receives word 0x66.

Only then is Working node +0x74 captured again as **Late mutation link**. Working node +0x74 receives Companion, and Late mutation link +0x40 receives word 0x66. Working node +0x0D receives **Early mutation link’s** low byte, not necessarily Late mutation link’s low byte.

Late mutation link +0x44 is captured. Working node +0x54 receives Working node; Descendant link B +0x54 receives Descendant link B. Late mutation link +0x14 is read, Companion +0x11 receives the captured +0x44 low byte, and a fresh Companion +0x14 word supplies the byte written to the newly selected address +0x15.

### Later navigation and the +0x64 replacement

Companion +0x14 is reloaded as **Later descendant**. Working node +0x64 is captured as **Later snapshot A**. Later descendant +0x20 receives word 0x73, followed by another Working node +0x64 capture, **Later snapshot B**. Later descendant +0x09 receives Later snapshot B’s low byte.

Later descendant +0x74 selects **Retained descendant**. Its +0x0D receives Working node’s low byte; its +0x54 word is read and written back; its +0x09 receives byte 0x74. **Working node +0x44 receives Later snapshot A before Retained descendant +0x34 receives Working node.**

Retained descendant +0x04 selects **Intermediate target**. Intermediate target +0x24 receives Later snapshot B, and Working node +0x19 receives Later snapshot A’s low byte.

Intermediate target +0x74 is captured as **Retained address**, followed by its +0x04 word as **Replacement word**. Working node +0x74 selects **Current link**. Current link +0x14 selects a target whose +0x54 receives Retained address. Current link +0x04 is then captured as **Constant target**.

Working node +0x0D receives Later snapshot B’s low byte twice consecutively. Working node +0x64 then receives Replacement word. Only afterward does Constant target +0x30 receive word 0x74.

Current link +0x04 is reloaded, and that newly obtained address’s +0x74 selects **Navigation node**. Working node +0x64 is freshly read as **Fresh replacement snapshot**; its low byte is written to Working node +0x1D.

Working node +0x44 is read and followed through +0x14 to select **Replacement node**. Navigation node +0x34 then receives Retained address, before Working node +0x14 receives Replacement node.

Working node +0x54 is captured, then Replacement node +0x09 receives Working node’s low byte. Navigation node +0x04 is followed through +0x74 to select **Final selected node**. Its +0x15 receives Retained address’s low byte.

Working node +0x54 receives Working node, Working node +0x11 receives Navigation node’s low byte, and Final selected node +0x11 receives the previously captured Working node +0x54 low byte.

### Final graph preparation

Final selected node +0x64 is captured as **Selected snapshot**. Working node +0x09 receives Fresh replacement snapshot’s low byte.

A fresh Navigation node +0x04 read selects **Final byte target**. That target +0x09 receives Working node’s low byte, its +0x50 word is captured, its +0x11 receives Selected snapshot’s low byte, and its +0x09 receives Working node’s low byte again.

Working node +0x14 receives Selected snapshot. Working node +0x0D receives Final byte target’s low address byte. Final byte target’s byte zero receives its captured +0x50 low byte.

**After that byte-zero write, Navigation node +0x74 is captured as Terminal-read link, then Navigation node +0x14 is captured as Final neighbor. Only after both reads does Final byte target +0x54 receive its own address.**

Final neighbor +0x04 selects **Final constant target**. This target receives byte 100 at +0x10; Working node +0x54 then receives word 100; afterward Final constant target +0x40 receives word 100.

Working node +0x0D receives Selected snapshot’s low byte. Final neighbor +0x09 receives Final constant target’s low address byte. Working node +0x24 receives Selected snapshot, and Final neighbor +0x11 receives Working node’s low byte.

### Terminal arithmetic and invocation

The terminal accesses occur in this order:

1. Read Terminal A.
2. Read Working node +0x74 as **Terminal link**.
3. Store Terminal link’s low halfword at Terminal link + Working node.
4. Store Terminal A’s low halfword at twice Final constant target.
5. Read a signed byte at Terminal-read link + Working node; retain its sign-extended 32-bit unsigned representation as **Terminal byte value**.
6. Store Terminal link’s low halfword at Terminal A + Terminal link.
7. Repeat the halfword store at Terminal link + Working node.
8. Store word 0x45 at Terminal byte value +0x45.
9. Read Terminal B, then Payload, then Working node +0x44.
10. Store byte 0x44 at Terminal byte value +0x44.
11. Store Payload as a word at twice Terminal B.
12. Store Terminal B’s low halfword at Retained address + Terminal B.

The unresolved target is then invoked with three visible arguments:

- Terminal byte value bitwise OR 0x44.
- Terminal B.
- The captured Working node +0x44 word.

The first argument uses OR, not addition. The third argument remains the captured word even if subsequent writes overlap its source.

Apart from the initial unsigned Index multiplication, the exact overflow and address-wrap behavior of the represented address arithmetic is **documented_only**; portable mathematical-address semantics are not established.

## Side effects

Visible side effects comprise reads, writes, and the indirect invocation. Stores of addresses establish neither ownership transfer nor object lifetime.

No allocation, deallocation, reference counting, synchronization, logging, or explicit error reporting is visible before the invocation. Their absence within the unresolved target is not established.

All accessed storage is externally supplied from the visible body’s perspective. There is no visible restoration after failure. Atomicity, concurrency behavior, volatile semantics, and memory-order guarantees are undocumented.

## Dependencies

- **F-22a0c53eea9f** is the supplied function identity.
- The terminal indirect dependency’s opaque function ID is unavailable: **documented_only**.
- Incoming Index, Root, Peer, target state, ambient words, and the reachable memory graph are environmental dependencies.

No identified direct callee appears. The indirect dependency is not assumed to preserve globals, preserve captured storage, retain or release memory, or return normally.

## Acceptance examples

These examples are conditional consequences of the static representation, not executed tests.

1. **Separate snapshots:** If Working snapshot A is 0x1000 and Working snapshot B is 0x2000AB, the relevant byte store targets 0x1009 and writes 0xAB. It does not write 0x00 merely because the destination base has low byte zero.

2. **Two mutation links:** If Early mutation link and Late mutation link differ because an intervening store aliases Working node +0x74, the word-0x66 destination uses Late mutation link +0x40. The subsequent Working node +0x0D byte still comes from Early mutation link.

3. **Interleaved link overwrite:** If Working node +0x44 overlaps Retained descendant +0x04, writing Later snapshot A there precedes the read selecting Intermediate target. Moving that read earlier can select a different target.

4. **Replacement ordering:** If Constant target +0x30 overlaps Working node +0x64, the word-0x74 store occurs after Replacement word is installed. The later fresh +0x64 read must reflect the memory state after both stores.

5. **Final captures precede self-store:** If Final byte target +0x54 overlaps Navigation node +0x74 or +0x14, Terminal-read link and Final neighbor still come from reads preceding that self-address store.

6. **Constant-store interleaving:** If Working node +0x54 overlaps Final constant target +0x40, the latter’s word-100 store follows the Working-node store. An access trace grouping both target writes before the Working-node write is not equivalent.

7. **Signed terminal byte:** Under the represented signed-byte interpretation, a read value of −1 produces Terminal byte value 0xFFFFFFFF and first invocation argument 0xFFFFFFFF. Usability of the associated computed addresses is a separate prerequisite.

8. **Captured invocation argument:** If the final Payload store overlaps Working node +0x44, the third invocation argument remains its earlier captured value.

9. **Partial failure:** An invalid terminal halfword destination does not imply reversal of earlier graph mutations or delivery of a conventional error result.

## Uncertainties

### Established static evidence

The representation establishes the displayed access widths, constants, capture dependencies, ordered overwrites, narrowing operations, and terminal argument construction.

### Inferred intent

Repeated self-address stores and link replacements suggest interconnected mutable state. No particular data structure, invariant, or application purpose follows from that inference.

### Documented-only unknowns

- Actual calling convention, function boundary, and origins of additional incoming state.
- Pointer representation, byte order, alignment, and address-wrap rules.
- Signed-character interpretation and right-shift behavior on the underlying target, especially for negative operands or counts of 32 or more.
- Ambient-word provenance and stability.
- Storage validity, allocation extents, aliasing relationships, and device-memory semantics.
- Indirect target identity, complete signature, effects, ownership rules, and return behavior.
- Whether the supplied representation fully captures the underlying control flow.

These uncertainties do not justify equating separate snapshots, omitting overwritten stores, or replacing the access sequence with abstract field initialization.

## Evidence

The sole evidence is the supplied static function representation. The prior draft and reviewer feedback are not independent evidence; the capture distinctions and ordering constraints above follow from the supplied representation.

No runtime tests, execution traces, independent binary inspection, or implementation access were used. Acceptance examples are conditional specifications, not empirical verification.
