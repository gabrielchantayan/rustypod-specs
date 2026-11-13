<!-- Trusted provenance: F-a6f951a7b6ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a6f951a7b6ad

## Purpose

F-a6f951a7b6ad performs an ordered sequence of memory accesses through explicit arguments, incoming machine-state addresses, loaded address words, and arithmetic-derived addresses. It finishes with an unresolved indirect control transfer carrying three represented arguments.

The static evidence establishes memory mutation and data dependencies, not a higher-level operation. Descriptions such as “child,” “link,” and “branch” below distinguish reached addresses; they do not establish structure types, ownership, or intended data structures.

## Interface

### Inputs and output

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | Full-word and low-byte stores. |
| Second argument | 32 bits | No visible use. |
| Third argument | 32-bit address-valued quantity | Base of a word store at offset `0x54`, containing the argument itself. |
| Fourth argument | 32 bits | Full-word and low-byte stores. |

Two incoming machine-state values supply the **context base** and **working base**. Their association with a calling convention is unresolved.

Three statically referenced word locations supply the **first static value**, **second static value**, and **static payload**. Their contents are read at distinct points near the end, not captured at entry.

The indirect target is another unresolved machine-state dependency. No return value is represented. A return continuation appears after the transfer, but normal return is not established.

### Memory and arithmetic

Offsets are bytes. A **byte**, **halfword**, and **word** mean 8, 16, and 32 bits respectively. Unless explicitly identified as a byte or halfword, each access described below is a word access.

“Low byte” and “low halfword” mean truncation to 8 and 16 bits, not validation. Address-bearing words have a represented width of 32 bits. Byte order is unknown; overlapping accesses must use the target’s actual byte order.

The tail’s converted character quantity is unsigned 32-bit. Its offset additions use represented unsigned arithmetic. Other address sums, subtractions, and doublings involve represented signed integer/address expressions; their overflow behavior cannot safely be replaced with an assumed wraparound rule.

### Preconditions and ownership

Completion requires every reached address to permit the specified access width and direction. This includes addresses before a captured reference, doubled addresses, and sums of independently obtained values.

No checks for null pointers, bounds, alignment, ownership, or arithmetic overflow are visible. Regions need not be distinct. No allocation, release, or ownership transfer is established.

## Behavior

### Ordering rules

The visible body contains no conditional branch or loop before the indirect transfer.

The following phases are chronological. Captured values remain snapshots; a later access to the same offset is a new observation. For a store whose value requires a memory read, that read precedes the store.

Repeated stores and read/write-back pairs remain observable accesses even when ordinary memory would retain the same value. Aliasing can affect subsequent reads, destinations, and values.

### Entry and initial child

The first operations are:

1. Store the working-base address at context-base offset `0x44`.
2. Store the third argument at its own offset `0x54`.
3. Read context-base offset `0x14`.
4. Store the fourth argument’s low byte at working-base offset `0x11`.
5. Read working-base offset `0x14`, then store its low byte at offset `0x15` of the address obtained from the context.

Working-base offset `0x14` is then read again to select the **initial child**. Working-base offset `0x64` supplies the **early working snapshot**.

The initial child receives the first argument at `0x10`; working-base `0x44` receives the working-base address; the child receives the first argument’s low byte at `0x14`. Working-base `0x54` receives the fourth argument.

Next, working-base `0x74` supplies the **original working link**. The working-base address’s low byte is stored at working-base `0x09` **before** that captured link is copied to initial-child `0x54`.

Initial-child `0x10` receives word `0x6e`; working-base `0x44` again receives the working-base address; initial-child `0x20` receives word `0x6e`.

A fresh read of working-base `0x64` supplies the **late working snapshot**. Its low byte is stored at initial-child `0x09`. Initial-child `0x74` selects the **linked object**, whose `0x54` supplies the **linked payload**.

The next byte stores, in order, are:

- Initial-child `0x1c`: `0x79`.
- Working-base `0x11`: low late working snapshot.
- Working-base `0x09`: low early working snapshot.
- Initial-child `0x1c`: `0x68`.
- Working-base `0x11`: low late working snapshot again.
- Linked-object `0x11`: low working-base address.

### Initial relinking

Initial-child `0x50` supplies the **child payload**. The working base then receives the early working snapshot at `0x44` and the late working snapshot at `0x64` and `0x34`. Initial-child `0x11` receives the late snapshot’s low byte.

Working-base `0x74` is reread as the **refreshed working link**. Initial-child `0x10` receives the child payload; linked-object `0x54` receives its captured linked payload. Initial-child `0x00` receives the child payload’s low byte, followed by the working-base address’s low byte at child `0x09`.

A fresh working-base `0x14` read selects the **active base**. The next stores are specifically:

- Low initial-child address to linked-object `0x11`.
- Active-base address to initial-child `0x44`.
- Low child payload to initial-child `0x00` again.

Only afterward are linked-object `0x74` and `0x14` read, in that order. They supply the **interior reference** and **branch base**. Initial-child `0x54` then receives the initial-child address.

### First constant propagation

Branch-base `0x04` selects a destination. It receives the refreshed working link’s low byte at `0x11`, then byte `0x64` at `0x10`. Active-base `0x54` receives word `0x64`; the destination receives word `0x64` at `0x60`.

Active-base `0x74` is read before being replaced with the branch-base address. The captured old destination receives word `0x64` at `0x30`.

Branch-base `0x14` selects a **preliminary child**, which receives the refreshed working link’s low byte at `0x09`. Active-base `0x54` receives the active-base address.

A word containing the interior reference minus `0x74` is then stored at the interior reference minus `0x20`. This is an interior-relative access, not evidence of an allocation boundary.

### Branch-child mutation

Preliminary-child `0x14` supplies a nested destination. Branch-base `0x11` receives the original working link’s low byte. A fresh branch-base `0x14` word is captured, and its low byte is written to nested-destination `0x15`.

Branch-base `0x14` is read again to select the **branch child**. Active-base `0x64` supplies the **first active snapshot**, which is explicitly written back to the same location.

The branch child receives byte `0x73` at `0x0c`. Active-base `0x15` receives the first active snapshot’s low byte; branch-base `0x34` receives the previously captured branch-base `0x14` word.

Branch-child `0x74` supplies the **earlier branch link**. The branch child receives its own address at `0x54`. The branch base then receives:

- Low active-base address at `0x0d`.
- The captured branch-base `0x14` word at `0x54`.
- The branch-child address at `0x14`.

A new active-base `0x64` read supplies the **second active snapshot**. The branch child receives byte `0x74` at `0x10`; active-base `0x54` receives word `0x74`; branch-child `0x10` then receives word `0x74`.

Branch-child `0x74` is reread as the **later branch link**. Branch-base `0x54` receives the branch-base address, and branch-child `0x30` receives word `0x74`.

Active-base `0x01` receives the first active snapshot’s low byte. Treating that snapshot as an address, its offset `0x09` receives the second active snapshot’s low byte. Active-base `0x74` receives the second snapshot, and active-base `0x11` receives byte `0x74`.

After reading branch-child `0x60`, active-base `0x11` receives the first active snapshot’s low byte, active-base `0x34` receives the second snapshot, and branch-child `0x10` receives the low byte of its captured `0x60` word.

### The `0x66` region

Active-base `0x14` selects the **constant destination**. Branch-base `0x11` receives the earlier branch link’s low byte.

Active-base `0x44` receives the branch-base address. Active-base `0x11` receives the second active snapshot’s low byte; active-base `0x74` is captured; active-base `0x0d` receives the branch-base address’s low byte.

The constant destination receives word `0x66` at `0x10`. **Active-base `0x44` receives the branch-base address again**, before the constant destination receives byte `0x66` at `0x10`. Active-base `0x54` and constant-destination `0x60` then receive word `0x66`.

Active-base `0x74` is reread to select the **displaced destination**, then replaced with the branch-base address. The displaced destination receives word `0x66` at `0x40`. Active-base `0x0d` receives the earlier captured active-base `0x74` value’s low byte.

Displaced-destination `0x44` is read. Active-base `0x54` receives the active-base address; the later branch link receives its own address at `0x54`.

Displaced-destination `0x14` supplies another destination. Branch-base `0x11` receives the captured displaced-destination `0x44` word’s low byte. A new branch-base `0x14` read supplies a byte for that other destination’s `0x15`.

### Continuation traversal

Branch-base `0x14` is reread to select the **continuation child**. Active-base `0x64` supplies the **third active snapshot**. Continuation-child `0x20` receives word `0x73`.

A separate active-base `0x64` read supplies the **fourth active snapshot**. Its low byte is written to continuation-child `0x09`. Continuation-child `0x74` selects the **continuation object**.

That object receives the active-base address’s low byte at `0x0d`, then has its `0x54` word read and written back unchanged. Its `0x09` receives byte `0x74`.

**Active-base `0x44` now receives the third active snapshot**, before continuation-object `0x34` receives the active-base address. Thus later traversal through active-base `0x44` uses a field that has undergone this update, subject to intervening aliases.

Continuation-object `0x04` selects the **continuation destination**. Its `0x24` receives the fourth active snapshot. Active-base `0x19` receives the third snapshot’s low byte.

### Final graph updates

Continuation-destination `0x74` and `0x04` are read, in that order, supplying the **retained link** and **replacement word**. Active-base `0x74` selects the **current trunk**.

The address read from current-trunk `0x14` receives the retained link at `0x54`. Current-trunk `0x04` then selects a destination. Active-base `0x0d` receives the fourth active snapshot’s low byte twice consecutively. Active-base `0x64` receives the replacement word; the selected destination receives word `0x74` at `0x30`.

Current-trunk `0x04` is reread, and that result’s `0x74` selects the **late parent**. A fresh active-base `0x64` read supplies the **final active byte** by truncation. Active-base `0x1d` receives that byte.

A read through active-base `0x44`, followed by offset `0x14` of the resulting address, selects the **replacement child**. Late-parent `0x34` receives the retained link; active-base `0x14` receives the replacement-child address.

Active-base `0x54` is captured as the **saved active payload**. Replacement-child `0x09` receives the active-base address’s low byte.

Late-parent `0x04`, followed by that result’s `0x74`, selects the **late descendant**. Its `0x15` receives the retained link’s low byte. Active-base `0x54` receives the active-base address, active-base `0x11` receives the late-parent address’s low byte, and late-descendant `0x11` receives the saved active payload’s low byte.

Late-descendant `0x64` supplies the **late word**. **Active-base `0x09` receives the final active byte**, the same captured byte previously stored at `0x1d`. This store occurs before the next late-parent `0x04` read.

That read selects the **late child**. Its `0x09` receives the active-base address’s low byte. Its `0x50` supplies a payload; its `0x11` receives the late word’s low byte; its `0x09` again receives the active-base address’s low byte.

Active-base `0x14` receives the late word. Active-base `0x0d` receives the late-child address’s low byte, and late-child `0x00` receives its captured payload’s low byte.

Late-parent `0x74` and `0x14` are then read, supplying the **tail byte base** and **tail branch**. Late-child `0x54` receives its own address.

Tail-branch `0x04` selects the **tail destination**. The remaining stores before the arithmetic tail are, in order:

- Byte `0x64` to tail-destination `0x10`.
- Word `0x64` to active-base `0x54`.
- Word `0x64` to tail-destination `0x40`.
- Low late word to active-base `0x0d`.
- Low tail-destination address to tail-branch `0x09`.
- Late word to active-base `0x24`.
- Low active-base address to tail-branch `0x11`.

### Arithmetic tail and transfer

The ordering of static reads and computed-address accesses is part of the contract:

1. Read the first static value.
2. Read active-base `0x74` as the **tail link**.
3. Store the tail link’s low halfword at the sum of the tail link and active-base address.
4. Store the first static value’s low halfword at twice the tail-destination address.
5. Read a byte at the sum of the tail byte base and active-base address.
6. Store the tail link’s low halfword at the sum of the first static value and tail link.
7. Repeat the halfword store at the sum of the tail link and active-base address.

The byte read in step 5 passes through a plain-character interpretation and then becomes an unsigned 32-bit **converted character**. High-bit bytes may therefore produce either zero-extended or sign-extended-then-unsigned values; signedness is unresolved.

A word `0x45` is stored at converted-character offset `0x45`. After that store, the sequence is exactly:

- Read the second static value.
- Read the static payload.
- Read active-base `0x44` as the **final captured word**.
- Store byte `0x44` at converted-character offset `0x44`.
- Store the static payload as a word at twice the second static value.
- Store the second static value’s low halfword at the sum of the retained link and second static value.

Earlier computed-address writes can overlap these subsequently read locations. The static reads cannot be moved to entry or treated as interchangeable.

The indirect transfer receives three represented arguments: the converted character bitwise-ORed with `0x44`, the captured second static value, and the final captured word.

## Side effects

The established effects are incremental memory reads and writes, followed by the indirect transfer. No explicit allocation, release, I/O, synchronization, rollback, or error-reporting operation appears before that transfer.

Overlapping word and byte accesses can change later address words. For example, offsets `0x10` and `0x11`, or `0x14` and `0x15`, can overlap. Final values therefore require both an alias model and target byte order.

If an access faults, earlier writes may already have occurred. No normal error result is specified.

The indirect dependency’s effects are unknown. It cannot be assumed to preserve globals, retain memory contents, or refrain from releasing memory.

## Dependencies

- **F-a6f951a7b6ad** is the specified function.
- **Unresolved indirect dependency:** no separate opaque function ID is supplied. Identity, calling convention, return behavior, state mutation, and ownership effects are documented_only.
- **Static data dependencies:** the three word locations read in the tail are data sources, not established functions.

No other direct invocation is visible.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Truncation:** First argument `0x123456ab` produces an early word store containing all 32 bits and a separate byte store containing `0xab`. Neither is guaranteed to survive later writes.

2. **Initial alias-sensitive order:** The working-base `0x09` byte store follows capture of the original working link but precedes its copy to initial-child `0x54`. Reversing the stores is unacceptable even if a nonaliasing example has the same final contents.

3. **Relinking trace:** After selecting the active base, linked-object `0x11` is written before initial-child `0x44`. The repeated initial-child `0x00` store precedes both subsequent linked-object link reads.

4. **Working-state preservation:** Both branch-address stores to active-base `0x44` occur, with the first word-`0x66` store between them. Later, the third active snapshot is stored at active-base `0x44` before continuation-object `0x34` is written.

5. **Late byte update:** Active-base `0x09` receives the byte captured for `0x1d`, after reading the late descendant’s `0x64` and before selecting the late child. It is not derived from the newly read late word.

6. **High-bit character:** A tail byte `0x80` yields converted character `0x00000080` under zero-extension or `0xffffff80` under signed-character conversion. The first indirect argument is correspondingly `0x000000c4` or `0xffffffc4`.

7. **Static aliasing:** If an earlier computed write overlaps the second static location, its later read observes memory according to that ordering and the target memory model; an entry-time snapshot is not equivalent.

8. **Fault boundary:** An inaccessible doubled-address word destination need not undo earlier writes and need not permit execution of the indirect target.

## Uncertainties

The following remain documented_only:

- Executable accuracy of the represented function boundary, types, and access sequence.
- Actual calling convention and origins of implicit machine-state inputs.
- Byte order, plain-character signedness, alignment requirements, and signed/address arithmetic overflow behavior.
- Valid memory mappings, alias relationships, concurrency, and device-backed access semantics.
- Structure identities, meanings of constants, and intended operation.
- Static-location contents and mutability.
- Indirect-target identity, effects, argument interpretation, and return behavior.
- Environment-specific faults and any externally imposed recovery mechanism.

There is no established success status, failure status, or unconditional final-state postcondition.

## Evidence

Established evidence is the supplied static representation: access widths, offsets, constants, explicit and implicit inputs, captured-value dependencies, sequential accesses, arithmetic-derived destinations, and the unresolved indirect transfer.

The descriptive object roles and phase names are explanatory. Any higher-level interpretation would be inference, not established behavior.

No runtime tests, execution traces, architecture checks, or independent executable validation have been performed.
