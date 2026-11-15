<!-- Trusted provenance: F-f65db2ae51ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f65db2ae51ce

## Purpose

F-f65db2ae51ce performs an ordered series of memory reads and writes, then invokes an unresolved indirect dependency. Its effects include 8-bit, 16-bit, and 32-bit stores through argument-derived addresses, incoming machine-state values, a global-selected address, and fixed numeric addresses.

No higher-level purpose is established. Character-like constants and repeated address publications do not establish text processing, object initialization, or ownership management.

This contract describes the supplied static representation. It does not establish that the represented interface is a complete executable function boundary.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No visible use |
| Context address | 32 bits | Word store at offset `0x34`; word read at offset `0x74` |
| Original record address | 32 bits | Byte and word accesses; source for later numeric address calculations |

The absence of a visible use of the first argument does not resolve its significance to the actual calling convention.

### Implicit inputs

Three incoming values have no visible definitions:

- **Seed address:** destination base for a word store at offset `0x44`.
- **Scale input:** doubled and quadrupled; the results serve as addresses and stored values.
- **Link input:** stored as a word and doubled to form a destination address.

Their actual calling-convention locations and validity requirements are unknown.

Further inputs are supplied by memory, a separate global pointer source, and the unresolved indirect target.

### Outputs and completion

There is no explicit result value. Outputs are the ordered memory effects and final indirect invocation.

A return is represented after the invocation. Whether the target returns normally, transfers control permanently, or terminates execution is unknown.

### Widths, arithmetic, and preconditions

All offsets are byte offsets. A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. Byte stores retain the low eight bits of their supplied values. Unsigned-byte reads produce values from `0` through `255`.

Address-bearing calculations use represented 32-bit quantities. Some calculations use signed values; the later scaled-record value is represented as unsigned. Wider arithmetic is not an established substitute. Overflow, signed left-shift behavior, address wrapping, alignment, and target endianness remain unresolved.

Successful completion requires every encountered read, write, and final invocation to be valid in the execution environment. The operation does not visibly enforce these requirements. Numerical examples below assume defined, representable calculations unless stated otherwise.

## Behavior

### Address relationships and captured values

The following descriptive names identify values, not object types:

| Name | Derivation |
|---|---|
| Doubled scale | Twice the scale input |
| Quadrupled scale | Four times the scale input |
| Context-derived address | Twice the word read from context offset `0x74` |
| Initial lookup address | Word read **at** `0x74` plus an unsigned-byte displacement obtained from doubled-scale offset `5` |
| Linked address | Word read from doubled-scale offset `0x64` |
| Record-linked address | Word read from original-record offset `4` |
| Selected address | Word read **at** `0x74` plus the second unsigned-byte displacement read from record-linked offset `5` |
| Doubled selection | Twice the selected address |
| Quadrupled selection | Four times the selected address |
| Adjusted record value | Original record address minus `0x96` |
| Scaled record address | Four times the adjusted record value |
| Extended record address | Eight times the adjusted record value |
| Global-selected address | Word captured from the separate global pointer source |

For both lookup addresses, the byte is added to `0x74` **before the word read**. Neither lookup reads a word at `0x74` and then adds the byte to that word. The displacement is not multiplied by four.

Captured values remain available independently of later changes to their source locations. Addresses are not assumed disjoint: preceding writes can affect subsequent reads and the addresses derived from them.

### Initial publications and first byte lookup

The first memory effect stores the link input as a word at context offset `0x34`. The subsequent word read from context offset `0x74` determines the context-derived address.

Doubled-scale offset `0x54` then receives the doubled scale as a word. Original-record offset `0x15` receives the low byte of the quadrupled scale, followed by a word store of the context-derived address at seed offset `0x44`.

The initial lookup reads an unsigned byte from doubled-scale offset `5`, then reads a word at the resulting address within `0x74…0x173`. Initial-lookup offset `0x44` receives the context-derived address as a word.

The ensuing publications occur in this order:

- Original-record offset `0x15` receives the low byte of the doubled scale.
- Doubled-scale offset `0x0D` receives the low byte of the initial lookup address.
- Doubled-scale offset `0x14` receives the doubled scale as a word.
- Original-record offset `0x54` receives the link input as a word.

Only after those publications is the linked address read from doubled-scale offset `0x64`. Linked offset `0x0D` receives the captured initial-lookup byte, and linked offset `0x14` receives the linked address as a word.

The record-linked address is then read from original-record offset `4`. Context-derived offset `9` receives the captured initial-lookup byte; context-derived offset `0x44` receives the quadrupled scale as a word.

### Two record-linked byte reads and selection

The first unsigned-byte read from record-linked offset `5` is captured before a byte store of the context-derived address’s low byte at doubled-scale offset `9`. That captured unsigned byte is then zero-extended and written as a word at initial-lookup offset `0x44`.

Before the second read from record-linked offset `5`, three further stores occur, in order:

- Quadrupled scale as a word at original-record offset `0x14`.
- Low byte of the record-linked address at context-derived offset `0x15`.
- Original record address as a word at context-derived offset `0x44`.

The second unsigned-byte read is distinct from the first. Its captured value supplies the displacement for the selected-address lookup, but an intervening store occurs before that lookup:

**After the second byte read, original-record offset `0x15` receives the low byte of the quadrupled scale. Only then is the selected-address word read at `0x74` plus the captured byte.**

The selected-address read is followed by these stores, in order:

- Original-record offset `0x15`: low byte of the record-linked address.
- Original-record offset `0x54`: link input, as a word.
- Doubled-scale offset `9`: low byte of the doubled selection.
- Original-record offset `0x14`: quadrupled scale, as a word.

Thus the pre-lookup byte store can affect the selected-address word, but cannot change the already captured lookup displacement. The record-offset-`0x54` store precedes the doubled-scale-offset-`9` store.

### Constants and scaled-record publications

The selected address determines three constant stores, in this order:

| Destination | Width | Value |
|---|---:|---:|
| Four times the selected address | 8 bits | `0x3A` |
| Eight times the selected address | 16 bits | `0x003A` |
| Quadrupled selection plus `0x20` | 32 bits | `0x0000003A` |

The working record value is reduced by `0x96` between the first and second stores. This is a change to the working numeric value, not an established write to caller-owned argument storage.

Scaled-record offset `0x20` subsequently receives four word stores: `0x20`, `0x2F`, the zero-extended byte read from scaled-record offset `4`, and `0x20`. The byte read occurs after the first two word stores and before the third.

The global-selected address is captured next. Absolute address `0x60` then receives word value `0x20`.

An initial publication group follows:

| Ordered destination | Width | Value |
|---|---:|---|
| Twice the link input, plus `0x54` | 32 bits | Quadrupled selection |
| Global-selected offset `0x19` | 8 bits | Low byte of scaled record address |
| Scaled-record offset `0x11` | 8 bits | Low byte of global-selected address |
| Global-selected offset `0x34` | 32 bits | Twice the link input |

Repeated groups then publish, in order, the quadrupled selection at extended-record offset `0x54`, the same two captured bytes at the same byte destinations, and the extended record address at global-selected offset `0x34`.

The global-selected address and both byte values are captured once; the repeated groups do not refresh them through memory reads. Exact repetition multiplicity remains **documented_only** in this contract. The group contents and ordering are established, but this is not a complete repeated-write event-count oracle.

### Late captures and fixed-address ordering

After the repeated publications, the operation captures a word from absolute address `0xA9`, then a word from quadrupled-selection offset `0x74`.

Absolute address `0x3A` next receives the low byte of eight times the adjusted record value plus `0x30`. This establishes the byte result, not a portable wider-arithmetic rule for overflowing calculations.

Two further words are captured, in order, from doubled-selection offset `0x14` and scaled-record offset `4`.

Before the next fixed-address access, the following stores occur in order:

- Doubled-selection offset `0x0D`: captured low byte of scaled record address.
- Scaled-record offset `8`: byte `0x6C`.
- Scaled-record offsets `0x0D`, then `9`: captured low byte of global-selected address.
- Doubled-selection offset `0x54`: doubled selection as a word.

The fixed-address sequence is specifically:

1. Absolute address `0x67` receives the low byte of the captured word from `0xA9`.
2. A word is read and captured from doubled-selection offset `0x44`.
3. Absolute byte `0x67` is read.
4. That byte is written to absolute address `0x21`.

The copy to `0x21` therefore occurs **after** the doubled-selection word read, not before it. A failure at that word read would precede the copy.

### Rearrangement and continuation capture

The word captured from doubled-selection offset `0x44` supplies a byte at scaled-record offset `0x10`, then a word at global-selected offset `0x54`.

Doubled-selection offset `1` next receives the low byte of that captured word after a represented signed right shift. The shift count is the low eight bits of the scaled record address. Negative operands and counts outside `0…31` lack a portable established result.

Global-selected offset `0x54` is then overwritten with the scaled record address as a word.

A word is captured from scaled-record offset `0x40`. The earlier captured scaled-record-offset-`4` word is written to global-selected offset `0x74`; the newly captured offset-`0x40` word is written at scaled-record offset `0x10`, overlapping the earlier byte store there.

The next effects, in order, are:

- A word store of doubled selection at global-selected offset `0x44`.
- A fresh read of absolute byte `0x67`, copied to scaled-record offset `9`.
- A byte store of the captured scaled-record-address low byte at doubled-selection offset `9`.
- A word store of the captured absolute-`0xA9` value at scaled-record offset `0x44`.

The word read next from scaled-record offset `0x54` is the **continuation address**. Another fresh read of absolute byte `0x67` is copied to scaled-record offset `9`.

Continuation offset `0x0D` receives the low byte of the earlier captured quadrupled-selection-offset-`0x74` word. Doubled-selection offset `0x54` receives the earlier captured doubled-selection-offset-`0x14` word.

A word is then captured from scaled-record offset `0x14`, before doubled-selection offset `0x11` receives that same quadrupled-selection-derived byte. Scaled-record offset `9` receives the continuation address’s low byte; scaled-record offset `0x44` receives the continuation address as a word.

### Terminal linkage and invocation

A word read from doubled-selection offset `4` becomes the **terminal base**. A halfword `0x0065` is stored at twice that base before the **terminal link** is read as a word from terminal-base offset `0x64`.

Continuation offset `0x11` receives the low byte of the captured scaled-record-offset-`0x14` word. Continuation offset `0x54` then receives word value `100`, or `0x64`.

A word is captured from terminal-base offset `0x74`. The **terminal peer** is then read as a word from doubled-selection offset `0x64`.

Before the indirect invocation:

- Doubled-selection offset `9` receives the terminal base’s low byte.
- Terminal-peer offset `0x54` receives the terminal link as a word.
- Twice the terminal link receives the terminal peer as a word.

The invocation occurs only after these effects.

## Side effects

Writes are not confined to the explicit argument regions. They reach incoming-state-derived memory, global-selected memory, scaled numeric addresses, and absolute locations.

| Absolute address | Visible access |
|---|---|
| `0x21` | Byte write |
| `0x3A` | Byte write |
| `0x60` | Word write |
| `0x67` | Byte write and subsequent byte reads |
| `0xA9` | Word read |

The two lookups additionally read words beginning at byte-displaced addresses in `0x74…0x173`.

No independence between global storage regions is established. Arbitrary aliasing among derived destinations and later read sources must also remain possible.

No allocation, deallocation, reference counting, ownership transfer, or rollback is established. Publishing an address-like value does not establish ownership.

There are no visible validation branches, error returns, bounds checks, or recovery paths. An invalid access can occur after earlier writes have taken effect. The representation does not specify fault delivery, atomicity, synchronization, or concurrent observation guarantees. The ordering described here is the represented access order, not a target memory-barrier guarantee.

## Dependencies

**D-01** is the opaque identifier used here for the unresolved indirect dependency. No mapped function identity is supplied for it.

Its four represented 32-bit arguments are:

| Position | Value |
|---|---|
| 1 | `0x72` |
| 2 | Terminal link |
| 3 | Earlier captured word from doubled-selection offset `0x14` |
| 4 | Late captured word from terminal-base offset `0x74` |

D-01’s identity, calling convention, effects, return behavior, global preservation, and memory ownership behavior are unknown. No preservation or release guarantee is implied.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Unsigned, byte-displaced lookup

If doubled-scale offset `5` contains `0xFE` when read, the initial lookup reads a word beginning at `0x172`. It does not use displacement `−2`, scale the displacement by four, or add `254` to a word first read at `0x74`.

### Second lookup ordering

If the second record-linked byte read captures `0x20`, the selected-address word is read at `0x94`. Before that word read, original-record offset `0x15` must receive the quadrupled scale’s low byte.

If that store overlaps the lookup word, the word read observes the intervening store under ordinary sequential memory assumptions. The lookup displacement remains the captured `0x20`.

Afterward, the record-offset-`0x54` word publication precedes the doubled-scale-offset-`9` byte publication.

### Byte truncation and access widths

If the quadrupled scale is `0x12345678`, a required byte store derived from it writes `0x78`, not a word.

If the selected address is `0x1000`, the three constant stores target byte address `0x4000`, halfword address `0x8000`, and word address `0x4020`, with values `0x3A`, `0x003A`, and `0x0000003A`. Multi-byte ordering is target-dependent.

### Record-derived addressing

For original record address `0x1000`, the adjusted value is `0x0F6A`, the scaled record address is `0x3DA8`, and the extended record address is `0x7B50`.

These are scaled numeric addresses, not offsets added back to the original record.

### Fixed-address copy ordering

If the captured absolute-`0xA9` word has low byte `0xAB`, absolute `0x67` receives `0xAB` before the doubled-selection-offset-`0x44` word read.

The read of byte `0x67` and write to `0x21` follow that word read. A fault at the word read must not be described as occurring after a completed copy to `0x21`. Without intervening changes to `0x67`, the copied byte is `0xAB`.

### Final invocation

If the terminal link is `0x2400`, the captured doubled-selection-offset-`0x14` word is `0x11223344`, and the terminal-base-offset-`0x74` word is `0x55667788`, D-01 receives those values after `0x72`, in that order, provided execution reaches the invocation.

All preceding represented stores remain required; matching only the final memory contents is insufficient to establish matching access behavior.

## Uncertainties

The following remain **documented_only**:

- Complete function boundary and calling convention.
- Origins and interpretation of the three implicit inputs.
- Business meaning of addresses, fields, and constants.
- Target endianness, alignment, and valid memory map.
- Overflow, address wrapping, and signed-shift behavior.
- Exact multiplicity of repeated publication groups.
- Hardware visibility and physical ordering of accesses.
- External or concurrent mutation of memory.
- D-01’s identity, effects, and return behavior.
- Reachability of the represented final return.

Linkage maintenance is a possible interpretation of the repeated address publications, but is inferred intent, not an established behavioral guarantee.

## Evidence

The evidence is limited to the supplied static representation. Embedded comments, symbolic names, the prior draft, and reviewer assertions were not treated as independent semantic authority.

Established observations include access widths, byte-displaced word lookups, captured-value dependencies, ordered overlapping stores, repeated publication structure, and the four indirect-invocation arguments.

The supplied representation establishes both critical ordering boundaries: the second record-linked byte capture precedes the record-offset-`0x15` store and selected-address word read; the doubled-selection-offset-`0x44` word read falls between the write to absolute `0x67` and the subsequent byte copy to `0x21`.

No runtime execution or runtime tests have been performed. Acceptance examples state conditional consequences of static evidence, not observed results.
