<!-- Trusted provenance: F-f13a3e11d9cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f13a3e11d9cc

## Purpose

The supplied static representation establishes a stateful operation that writes interconnected records, derives memory addresses from narrow integer values, executes two software-breakpoint operations, and finally attempts an indirect transfer to an address obtained from one unsigned byte.

No application-level purpose is established. Record-like layouts, self-links, repeated constants, and stored data addresses do not establish allocation, serialization, parsing, dispatch, or valid object initialization.

This contract describes represented behavior, conditional on execution reaching each operation. It does not establish that the recovered interface or control flow is complete. No runtime tests have been performed.

## Interface

### Explicit inputs and result

| Position | Represented type or width | Use |
|---|---|---|
| First | Opaque 32-bit value | Stored as the first word of the fourth input’s destination. |
| Second | 32-bit address-like integer | Used as the base of a word write at byte offset `0x54`. |
| Third | Opaque 32-bit value | No visible use. |
| Fourth | Address of writable 32-bit words | Receives five consecutive words near entry. |

No return value is specified. The representation ends with an indirect call followed by a return; actual return depends on reaching that point and on the unknown target’s behavior.

### Implicit entry dependencies

The operation also consumes incoming state outside those four inputs. The following are descriptive roles, not recovered formal parameters:

- **Anchor:** address used repeatedly for fields at offsets including `0x14`, `0x34`, `0x44`, `0x54`, and `0x74`.
- **Initial record:** writable record address whose numeric value is doubled and subsequently used as another memory address.
- **Initial node:** address initially installed into several links.
- **Lookup owner:** address supplying the shared record through a word read at `0x64`.
- **Stack destination:** address receiving four consecutive words.
- **Stack-linked record:** address used in subsequent record rewriting.
- **External holder:** address receiving a word at `0x14`.
- **Frame-local address:** address within the current execution frame, published into several records.

The **shared record** is the address loaded from the lookup owner. Other record roles below are obtained by reads during execution; they are not necessarily distinct allocations.

### Widths and interpretation

All offsets are byte offsets. A sequence of record words means ordered 32-bit stores at offsets `0`, `4`, `8`, and so forth.

The representation includes 32-bit word accesses, 8-bit accesses, and one signed 16-bit read. Loaded unsigned bytes used as addresses are zero-extended, giving bases in `0x00`–`0xFF`. Address-to-byte stores retain only the low eight bits.

Address arithmetic is represented in a 32-bit-oriented environment. Overflow behavior, pointer conversion rules, alignment requirements, and byte order are not independently established. In particular, doubled addresses must not be interpreted as allocation sizes or as guaranteed overflow-free mathematical results.

Two fixed data-address values are called **token A** and **token B**. Their contents are not read here, and neither is established as a function.

## Behavior

### Preconditions and failure model

There are no visible checks for null values, alignment, accessible extents, arithmetic overflow, or executable transfer addresses. Every reached access therefore requires the corresponding address to support that access at that time.

This includes accesses through doubled addresses, unsigned-byte-derived low addresses, signed-halfword-derived addresses, and the absolute location `0xD9`.

No error result, rollback, recovery branch, or atomicity guarantee is shown. An access failure or noncontinuing breakpoint can leave earlier writes visible. The following order is significant: aliasing can change subsequent reads, and repeated stores are not interchangeable with a single presumed final store.

### 1. Entry publication and initial records

Execution first writes the anchor as a word at second-input base plus `0x54`. It then reads the shared-record address from lookup-owner offset `0x64`.

The fourth-input destination receives these five words, in order:

1. First explicit input.
2. Shared-record address.
3. Fourth-input destination’s own address.
4. Initial-record address.
5. Anchor.

A word read from anchor offset `0x34` selects the **initial destination**. Before writing its leading words, execution performs these stores:

1. Initial node into initial-record offset `0x14`.
2. Initial node into its own offset `0x14`.
3. Initial node into anchor offset `0x54`.

The initial record’s numeric address is doubled. Its first three words then receive token A, the doubled address, and the anchor. A byte is read from initial-record offset `0x16` and retained as the **initial retained byte**. This read follows the preceding link and word stores; its value must not be assumed to be an untouched entry byte.

The initial destination receives five words: token A, shared record, initial destination itself, initial record advanced by `0x0C`, and anchor.

### 2. Doubled-address and stack-destination updates

Anchor offset `0x54` is reread to select a **selected node**. The operation writes, in order:

1. Doubled initial-record address to selected-node offset `0x34`.
2. Low byte of the stack-destination address to selected-node offset `0x15`.
3. Shared-record address to selected-node offset `0x54`.
4. Byte `0x64` to doubled-address offset `0x18`.
5. Twice the selected node’s numeric address, as a word, to initial-retained-byte base plus `0x24`.

The stack destination receives four words: doubled selected-node address, doubled initial-record address, selected node, and anchor.

Next, the doubled initial-record address is written to selected-node offset `0x14`, and to its own offset `0x24`. Byte `0x61` is written at doubled-address offset `0x0C`. The word store at selected-node offset `0x14` overlaps the earlier byte store at `0x15`.

Execution reads anchor offset `0x14` as the **anchor-linked object**, then anchor offset `0x34` as the **working destination**. It writes the anchor’s low byte to selected-node offset `0x15`.

The working destination receives four words: shared record, anchor-linked object, selected node, and anchor. Its address advanced by `0x10` is the **extension record**.

The anchor-linked object receives word `0x61` at `0x10`, followed by its own low address byte at `0x09`. The first software breakpoint, immediate `0xC5`, follows.

### 3. Rewriting after the first breakpoint

Continuation depends on the execution environment. No preservation guarantee is established for breakpoint handling.

If execution continues, the extension-record address’s low byte is written to anchor offset `0x11`, followed by the frame-local address as a word at anchor-linked-object offset `0x44`.

The extension record receives five words: stack-linked record, shared record, extension record itself, frame-local address, and anchor.

A word read from anchor offset `0x44` selects the **auxiliary record**. Its first three words become `0x6F`, stack-linked record, and anchor. The extension record’s first four words are then rewritten as `0x6F`, shared record, extension record itself, and the zero-extended initial retained byte. Its fifth word is not explicitly rewritten at this point.

The stack-linked record receives these stores in order:

1. Its own low address byte at `0x12`.
2. Extension-record address at `0x14`.
3. Three leading words: auxiliary record advanced by `0x0C`, stack-linked record itself, and anchor.

A word is read from working-destination offset `0x28` and retained as the **payload**. The extension record’s first three words are then overwritten with shared record, stack-linked record, and anchor.

Finally, the payload’s low byte is written to auxiliary-record offset `0x1E`.

### 4. Byte-derived and signed-halfword-derived addresses

A word read from auxiliary-record offset `0x80` supplies a **byte-source object**. That object receives byte `0x65` at `0x0C`, then supplies a byte from `0x09`.

The loaded unsigned byte becomes a low address base. At that base, execution writes the anchor as a word at `0x14`, then writes the byte itself at `0x15`. These accesses overlap.

The next reads occur in this order:

1. A 32-bit value from absolute address `0xD9`, retained as the **first global-derived base**.
2. The shared record’s second word, retained as the **secondary record**.
3. A signed 16-bit value from secondary-record address plus the unsigned byte-derived offset.

The signed halfword is promoted with its sign for subsequent address calculations. A negative value is not treated as an unsigned 16-bit address.

The following effects then occur in order:

1. Shared-record byte `0x11` receives the low-eight-bit result of advancing the working-destination address by `0x1C`.
2. Signed-halfword-derived base plus `0x0D` receives the payload’s low byte.
3. A word read from that base plus `0x64` selects the **halfword-selected destination**.
4. Anchor byte `0x11` receives the halfword value’s low byte.
5. The shared record receives three words: `0x6D`, frame-local address, and anchor.
6. Secondary-record byte `0x02` receives `0x6D`.
7. First-global-derived-base offset `0x58` receives the anchor as a word.
8. The halfword-selected destination receives five words: payload, shared record advanced by `0x0C`, destination itself, frame-local address, and anchor.
9. Secondary-record byte `0x10` receives `0x6D`.
10. External-holder offset `0x14` receives the secondary-record address.

The secondary-record address remains the value captured before the shared record’s second word was overwritten; it is not recovered by rereading that field.

### 5. Final linking and transfer-byte selection

Anchor offset `0x34` is reread to obtain the **current target**.

The current target receives byte `0x61` at `0x0C`, then supplies a byte from `0x09`. Using that unsigned byte as a base, execution writes the anchor as a word at `0x14`, followed by the byte itself at `0x15`.

The current target then receives another byte `0x61` at `0x0C`, the payload as a word at `0x44`, and the zero-extended byte as a word at `0x24`.

A word read from anchor offset `0x54` selects the **linking record**. The next operations are specifically:

1. Write the current-target address as a word to linking-record offset `0x34`.
2. Write the anchor as a word to linking-record offset `0x54`.
3. Read the current target’s word at `0x74`, retaining it as the **first target link**.

Both linking-record stores precede that first `0x74` read. If either store aliases the field being read, the read observes the resulting memory rather than an assumed prior value.

The current target then receives word `0x69` at `0x10`. The linking record receives the current-target address at `0x24`, followed by three leading words: current target, linking record itself, and anchor.

The secondary record receives three leading words: first target link, linking record, and anchor. A word read from secondary-record offset `0x18` supplies the low byte written to current-target offset `0x12`.

The current target’s field at `0x74` is read again. This second result selects the **transfer-byte object**; it is not unconditionally interchangeable with the first target link because intervening writes may alias the field.

The transfer-byte object receives byte `0x65` at `0x0C`. Its byte at `0x09` is then read and zero-extended to form the **transfer address**, in `0x00`–`0xFF`. The captured byte is also retained for a later store.

### 6. Final preparation and indirect transfer

At the transfer-address base, execution writes the anchor as a word at `0x14`, then the captured transfer byte at `0x15`.

Absolute address `0xD9` is reread as the **second global-derived base**. A word read from first-target-link offset `0x04` supplies the **final linked object**.

The subsequent operations are ordered as follows:

1. Write byte `0x65` to final-linked-object offset `0x18`.
2. Write the transfer address as a word to that object’s offset `0x34`.
3. Reread anchor offset `0x34` as the **final destination**.
4. Overwrite transfer-address offset `0x14` with the final-linked-object address as a word.
5. Write the final-linked-object address into its own offset `0x24`.
6. Write the anchor into anchor offset `0x54`.
7. Overwrite final-linked-object byte `0x18` with `0x79`.
8. Write token B as a word to second-global-derived-base offset `0x24`.
9. Write the first target link as a word to anchor offset `0x14`.
10. Write five words to the final destination: token B, first target link, final destination itself, transfer address, and anchor.
11. Read anchor offset `0x74` as a 32-bit signed **index**.
12. Execute the second software breakpoint with immediate `0xC5`.

If continuation occurs, a word read from transfer-address offset `0x74` supplies another destination base. Its offset `0x54` receives the final-destination address as a word.

The captured transfer byte is then written at transfer address plus the index. This is byte indexing, without four-byte scaling. The index is subsequently written as a word at transfer-address offset `0x54`.

Finally, execution attempts an indirect call to the transfer address. No explicit call arguments are shown. The represented return is reached only if that call returns.

## Side effects

- Writes affect explicit destinations, implicit records, frame-address-bearing records, global-derived memory, and narrow-integer-derived addresses.
- Byte stores overlap word stores, including fields at `0x14`/`0x15` and `0x10`/`0x12`.
- Repeated reads may obtain different values because of aliasing, breakpoint handling, or other environmental changes.
- Retained values remain distinct from later rereads; notably the first target link and transfer-byte-object address need not match.
- Frame-local addresses escape. Their usable lifetime after return is unknown.
- Breakpoint handling and the indirect target may have additional effects; preservation of globals or memory is not established.
- No allocation, release, ownership transfer, or retention policy is established.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| Breakpoint mechanism | Two occurrences with immediate `0xC5`. | Handling, continuation, and state preservation are documented_only. |
| Indirect executable dependency | Called at the zero-extended address of a loaded byte. | Identity, arguments, effects, ownership behavior, and return behavior are documented_only. |
| Tokens A and B | Stored as opaque data-address words. | Contents and meaning are documented_only. |
| Absolute location `0xD9` | Read twice as a 32-bit address-bearing value. | Stability, memory classification, and ownership are documented_only. |
| Incoming implicit state | Supplies addresses required by the represented accesses. | Complete calling contract is documented_only. |

Only F-f13a3e11d9cc has a supplied opaque function identity. No opaque function ID is available for the indirect dependency; assigning one would invent evidence.

## Acceptance examples

These are conditional criteria, not executed tests.

1. **Entry order:** With accessible memory, the second-input `0x54` store precedes the lookup-owner `0x64` read. If they overlap, the read uses the memory resulting from the store.

2. **Unused input:** Changing only the third explicit input introduces no visible use in this representation. Hidden calling-convention effects remain unresolved.

3. **Doubling without overflow:** For initial-record numeric address `0x1000`, the doubled base is `0x2000`, and accesses use it directly. This does not request additional storage.

4. **Overlapping byte-derived stores:** For a loaded base byte `0x20`, the word store begins at `0x34`, followed by a byte store at `0x35`. Acceptance preserves both accesses and their order; the resulting word value depends on byte order.

5. **Signed halfword:** A halfword pattern `0xFFFF` contributes signed value `−1`, not `65535`, to later addressing. Its conversion to a physical address remains environment-dependent.

6. **Linking-store values and order:** Given noninterfering current-target and linking-record addresses `0x2000` and `0x3000`, and anchor `0x4000`, execution writes word `0x2000` at `0x3034`, then word `0x4000` at `0x3054`, before reading the word at `0x2074`.

7. **First versus second target link:** If the two current-target `0x74` reads differ, the second result supplies the transfer-byte object. The first result still supplies the later `0x04` read and is stored into the final records.

8. **Breakpoint termination:** If the first breakpoint prevents continuation, preceding effects remain possible, but subsequent record rewrites and the indirect transfer are not reached.

9. **Final address and indexing:** If the captured transfer byte is `0x7A` and the captured index is `3`, the post-breakpoint byte store targets `0x7D`; the index word store targets `0xCE`; the attempted call targets `0x7A`. All preceding accesses must also succeed.

## Uncertainties

- **documented_only — interface:** Implicit entry dependencies may reflect an incomplete function boundary or calling convention.
- **documented_only — arithmetic:** Overflow, signed left-shift behavior, address wrapping, and integer-to-address conversions are not independently established.
- **documented_only — layout:** Byte order, alignment, address validity, and mapped low-address memory are unknown.
- **documented_only — execution:** Breakpoint semantics, executable permissions, and instruction-set address conventions are unknown.
- **documented_only — ownership:** No evidence establishes who owns or releases any record, or whether escaped frame addresses are consumed within their lifetime.
- **Inferred intent only:** Repeated constants and self-links suggest structured state manipulation, but no application meaning is established.

## Evidence

The supplied static representation supports the stated access widths, constants, offsets, ordered writes and reads, byte truncation and widening, signed halfword interpretation, breakpoint placement, and byte-derived indirect call.

In particular, it establishes the current-target store at linking-record offset `0x34`, followed by the anchor store at `0x54`, before the first current-target `0x74` read.

Diagnostic annotations flag overlapping global symbols and unresolved indirect control flow. These limit confidence in the recovered representation; they do not supply application semantics.

No executable observations, caller contracts, dependency contracts, memory image, or ABI documentation were supplied. No runtime tests have been performed.
