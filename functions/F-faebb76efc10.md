<!-- Trusted provenance: F-faebb76efc10; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-faebb76efc10

## Purpose

F-faebb76efc10 performs an ordered series of mixed-width memory accesses, modifies a mutable network of address-bearing words, and transfers control to an unresolved indirect destination.

The supplied static evidence does not establish a higher-level purpose. It does not justify identifying the routine as a constructor, serializer, text processor, allocator, or destructor. Some constants correspond to printable characters, but their scattered byte and word stores do not establish a string or encoding.

The effective interface includes four explicit arguments, additional incoming machine state, global storage, and indirectly reachable memory. The explicit arguments alone are insufficient to determine behavior.

This contract distinguishes established operations from inferred interpretations and **documented_only** unknowns. No runtime tests have been performed.

## Interface

### Explicit arguments and result

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Not explicitly used in the supplied body. |
| Second argument | 32 bits | Doubled to form an early halfword-store address. |
| Third argument | 32 bits | Contributes to an entry-slot address; separately supplies a word read at offset `0x14`. |
| Fourth argument | 8 bits | Stored at offset `0x45` in a globally selected region. |

There is no explicit return value. The unresolved destination receives three visible arguments. Its result is unused; if execution returns as represented, this routine returns without a result.

There is no explicit length, output-buffer description, status result, or error channel.

### Additional incoming state

The body uses values not initialized within it:

- An address base for an early byte store.
- A 32-bit contribution used in address formation and as truncated store data.
- A signed 16-bit arithmetic input.
- A separate 16-bit store value.
- An indirect destination used in a comparison and in the terminal transfer.

Their calling-convention locations and provisioning mechanism are **documented_only**.

### Memory model and terminology

The representation uses 8-bit, 16-bit, and 32-bit accesses. Address-bearing words and the final visible arguments are represented at 32 bits. Two byte reads participate in signed arithmetic.

Offsets are byte offsets. Decimal `100` in the supplied representation means `0x64`.

For description only:

- **Entry slot:** third argument plus the incoming 32-bit contribution.
- **Selected region:** address supplied by the first global read.
- **Working record:** address read from selected-region offset `0x74`.
- **Companion record:** address obtained from working-record offset `0x14` during the companion phase.
- **Continuation record:** address obtained from a later, separate read of working-record offset `0x14`.

These names identify access relationships, not types, allocation boundaries, or ownership. Companion and continuation addresses need not remain equal under aliasing.

Execution requires every encountered access to be supported by the environment at its actual width and address. No checks establish those preconditions. Accessible positive-offset fields alone are insufficient: the routine also accesses doubled addresses, sums of addresses, and a negative displacement.

## Behavior

### Ordering and aliasing

The supplied body is a straight-line sequence ending in an indirect invocation. No explicit validation branch, loop, recovery branch, or earlier ordinary call is shown.

Memory ordering is part of the contract:

- A saved word retains the value read at its capture point.
- A later reload observes memory after intervening writes.
- Byte and halfword writes can change subsequent word reads.
- Repeated stores and read/write-back operations remain separate accesses.
- No disjointness is established among globals, arguments, hidden-state addresses, and reachable records.

The phase descriptions below do not authorize regrouping accesses across phase boundaries or eliminating intermediate stores.

### Entry mutation and global selection

Before reading the globals that select subsequent state, the routine performs three stores, in order: the incoming contribution’s low 16 bits to the entry slot; the separate incoming halfword to twice the second argument; and byte `0x45` to the incoming address base plus `0x45`.

It then reads the selected-region address and a second global word, narrowing the latter to 16 bits.

A word read from the already modified entry slot supplies an address for a byte `0x45` store at offset `0x45`. The original entry-slot halfword store is then repeated.

Selected-region offset `0x45` receives a halfword equal to the low 16 bits of the sum of:

- The incoming signed 16-bit arithmetic input.
- The second global’s narrowed 16-bit value.
- One if the incoming indirect destination compares below numeric address `0x46`, otherwise zero.

The precise interpretation of that destination comparison is **documented_only**.

The entry slot is read again as a word. That saved address receives word `0x00000045` at offset `0x45`. The fourth argument is subsequently stored as one byte at selected-region offset `0x45`, potentially overwriting part of the preceding halfword.

### Working-record setup and shifted-byte production

A word from third-argument offset `0x14` is captured and copied to offset `0x34` relative to the incoming contribution, not relative to the third argument.

The routine selects the working record through selected-region offset `0x74` and another record through offset `0x14` of the saved entry-derived address. It installs the entry-derived address at working offset `0x54`, installs a self-reference at the other record’s `0x54`, and places the captured third-argument-derived word at working offset `0x44`.

A byte containing the working address’s low eight bits is stored at entry-derived offset `0x09`. The routine follows working offset `0x64`, then the resulting record’s `0x14`. At offset `0x09` of the latter address it makes two consecutive byte stores: first the low byte of the earlier entry-linked record, then the low byte of the intermediate `0x64` target.

A further global word is captured. Following offset `0x14` from the third-argument-derived address produces a base whose address plus `0x20` is retained. The routine also follows the latest chain endpoint’s `0x64` link and writes the working address at its target’s `0x14`. The displaced address’s low byte is written at third-argument-derived offset `0x11`.

Two distinct word reads of working offset `0x44` follow:

- The first captures a word for a later write to working offset `0x74`.
- The second captures the destination base for a byte store at offset `0x08`.

The routine reads a signed byte at twice the earlier chain endpoint’s address, sign-extends it, shifts it right by that endpoint’s low eight bits, and retains the result’s low byte. It writes that byte at the second captured base plus `0x08`, then writes the **first captured word** to working offset `0x74`, then writes the result byte to working offset `0x11`.

Thus the `0x74` write is not a fresh copy of working offset `0x44`. The intervening byte store can overlap that field without changing the saved word.

The shift’s behavior for large counts is **documented_only**; the representation does not establish a portable masking rule.

Subsequent operations overwrite third-argument-derived offset `0x54` first with the displaced address and then with the captured global word, write the displaced address’s low byte at offset `0x19`, and install links at the second captured base’s `0x44` and the displaced address’s predecessor’s `0x74`.

### Companion state and snapshot restoration

Before selecting the companion, the routine follows the second captured base’s `0x14`, writes the captured global’s low byte at working offset `0x11`, and writes the low byte of a fresh working-`0x14` read at the reached record’s `0x15`. It then separately reads working offset `0x14` as the companion address.

An **earlier `0x64` snapshot** is captured from the working record. The companion receives word `0x73` at `0x10` and byte `0x73` at `0x14`; the working record receives a self-reference at `0x44` and the captured global word at `0x54`. A working-`0x74` snapshot is captured and copied to companion offset `0x54`.

The companion subsequently receives word `0x6E` at offsets `0x10` and `0x20`, with another working-`0x44` self-reference write between them. A **later `0x64` snapshot** is then captured. Its low byte is written to companion offset `0x09`.

The routine follows companion offset `0x74` and captures that target’s `0x54` word. The following stores have this exact order:

1. Byte `0x79` at companion offset `0x1C`.
2. Low byte of the later `0x64` snapshot at working offset `0x11`.
3. Low byte of the earlier `0x64` snapshot at working offset `0x09`.
4. Byte `0x68` at companion offset `0x1C`.
5. Low byte of the later `0x64` snapshot again at working offset `0x11`.
6. Low byte of the working address at the companion-linked target’s `0x11`.

The two companion-`0x1C` writes are therefore **not consecutive**. Intervening working-record writes may alias their destination.

A word captured from companion offset `0x50` later replaces companion offset `0x10`. Between capture and replacement, the routine restores the earlier `0x64` snapshot to working offset `0x44`, writes the later snapshot to working offsets `0x64` and `0x34`, writes its low byte at companion offset `0x11`, and captures working offset `0x74` again.

The companion-linked target’s saved `0x54` word is restored after intervening accesses. This is a snapshot restoration, not necessarily a redundant store.

Further byte writes affect companion offsets `0x00` and `0x09`. A fresh working-`0x14` read selects the continuation record. The companion receives that address at `0x44`, a repeated byte write at `0x00`, and a self-reference at `0x54`.

### Continuation mutations and constant stores

The continuation phase follows mutable `0x04`, `0x14`, and `0x74` links while interleaving constant stores with link rewrites and snapshot restoration.

Its principal constant-writing groups are:

| Constant | Established destinations |
|---|---|
| `0x64` | Byte at a reached record’s `0x10`; words at continuation `0x54`, reached `0x60`, and another reached record’s `0x30`. |
| `0x74` | Byte and word at a reached record’s `0x10`; words at continuation `0x54` and reached `0x30`. |
| `0x66` | Word followed by byte at a reached record’s `0x10`; words at continuation `0x54`, reached `0x60`, and another reached record’s `0x40`. |
| `0x73` | Byte at a reached `0x0C`; later, word at another reached `0x20`. |
| Later `0x64` | Byte at a reached `0x10`; words at continuation `0x54` and reached `0x40`. |

These are not contiguous text writes. Each group uses addresses selected at its own traversal point.

During this phase, a saved address is used for a word store at displacement `-0x20`, with that same saved address minus `0x74` as the value. This requires memory preceding the saved address.

Continuation offset `0x64` is read and written back unchanged, then separately read again. Those reads are not interchangeable under aliasing. The routine uses saved and newly read values for byte writes, including continuation offsets `0x01`, `0x11`, and `0x15`, and for word writes at `0x34`, `0x44`, and `0x74`.

The `0x74` constant group includes a byte store followed by a word store at the same reached offset `0x10`; a later store there uses the low byte of a word captured from that record’s `0x60`. The `0x66` group instead writes the word before the byte.

Further traversals perform a reached `0x54` read/write-back, write a continuation reference at reached `0x34`, and copy a captured word to reached `0x24`. Continuation offset `0x19` receives a snapshot’s low byte.

Near the end of this phase, continuation offset `0x0D` receives two consecutive identical byte stores. Its `0x64` word is then replaced using a separately captured chain value. Later traversals rewrite continuation `0x14` twice, affect offsets `0x09`, `0x0D`, `0x11`, and `0x1D`, establish another reached self-reference at `0x54`, and copy a captured word to continuation `0x24`.

No immutable initial graph can substitute for these later traversals.

### Final halfword stores, signed-byte read, and transfer

For this final sequence, distinguish:

- The **first late global integer**.
- The word freshly read from continuation offset `0x74`, called the **late link**.
- A **saved traversal link** used for the signed-byte address.
- The **last constant-target address**, which received the final byte `0x64` store.

After reading the first late global integer and then the late link, the routine performs these accesses in order:

1. Stores the late link’s low halfword at late-link-plus-continuation.
2. Stores the first late global integer’s low halfword at twice the last constant-target address.
3. Reads a signed byte at saved-traversal-link-plus-continuation and sign-extends it to a 32-bit address-bearing quantity.
4. Stores the late link’s low halfword at first-late-global-integer-plus-late-link.
5. Repeats the halfword store at late-link-plus-continuation.
6. Stores word `0x00000045` at signed-byte-derived quantity plus `0x45`.

The signed-byte read therefore occurs **between** the second and third halfword stores, not after all four. The later stores can overlap its address but cannot retroactively change the captured byte.

The routine next reads a second late global integer, reads a separate global store-data word, and captures continuation offset `0x44`. It then writes byte `0x44` at the signed-byte-derived quantity plus `0x44`, writes the global store-data word at twice the second late global integer, and writes that integer’s low halfword at its sum with another saved traversal address.

Finally, the incoming indirect destination is invoked with:

- The signed-byte-derived 32-bit quantity bitwise ORed with `0x44`.
- The second late global integer.
- The captured continuation-`0x44` word.

## Side effects

Memory mutation is the principal visible output. No explicit allocation, release, ownership transfer, I/O operation, diagnostic, or synchronization primitive appears before the terminal transfer.

This does not constrain the unresolved destination. It may modify globals, retain addresses, release memory, fail, or not return. No preservation or ownership guarantee can be assumed.

Invalid or unsupported accesses may prevent completion after earlier mutations. No rollback or transactional behavior is shown. Actual fault delivery is **documented_only**.

The described ordering is the ordering of the supplied representation. Atomicity, volatile behavior, and inter-thread synchronization are not established.

## Dependencies

| Dependency | Established contract |
|---|---|
| Unresolved indirect destination | One terminal invocation, three visible arguments, unused result. No separate opaque function ID is supplied. |
| External global storage | Supplies address selection, arithmetic data, saved store data, and late integers at distinct read points. |
| Execution environment | Supplies hidden entry values, address mappings, byte order, alignment behavior, arithmetic/address interpretation, and indirect-transfer semantics. |

F-faebb76efc10 is the only supplied opaque function ID. No standard-library identity, additional function ID, or recursive dependency is established.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Partial entry-slot update.** If the incoming contribution’s low halfword is `0x1234`, the initial store changes only that halfword. A subsequent word read must use the modified bytes, not an entry-time cached word.

2. **Saved `0x44` value.** Suppose both working-`0x44` reads initially yield the same word, but the shifted-byte destination overlaps working offset `0x44`. Working offset `0x74` still receives the first read’s saved word, not the potentially modified field.

3. **Separated companion writes.** If working offset `0x11` aliases companion offset `0x1C`, the later-snapshot byte store occurs after `0x79` and before `0x68`; it occurs again after `0x68`. An interpretation retaining only `0x79` followed immediately by `0x68` is incorrect.

4. **Late read before overwrite.** If first-late-global-integer-plus-late-link equals the signed-byte-read address, and earlier accesses leave that byte unchanged, the read captures its pre-store value. Moving that halfword store before the read can change the first indirect argument and is not equivalent.

5. **Width distinction.** Word `0x00000073` modifies four bytes in target byte order. Byte `0x73` modifies one. Neither substitutes for the other.

6. **Signed final quantity.** A late byte interpreted as `-1` produces the represented 32-bit value `0xFFFFFFFF`; ORing it with `0x44` leaves it unchanged. Completion of derived-address stores remains environment-dependent.

7. **Failure after mutation.** An inaccessible later link does not imply restoration of the opening stores. No recovery requirement is supported.

8. **Destination ownership effects.** A terminal destination that changes globals or releases reachable memory is not contradicted by this body. Such effects require that dependency’s own contract.

## Uncertainties

The following remain **documented_only**:

- High-level purpose and validity of the recovered function boundary.
- Complete calling convention and origins of hidden inputs.
- Target byte order, address representation, and alignment requirements.
- Exact arithmetic-overflow and address-wrap behavior.
- Right-shift behavior for large counts and target-specific signed-shift semantics.
- Exact interpretation of the indirect-destination comparison.
- Global meanings, ranges, and overlap relationships.
- Reachable memory extents, aliasing, ownership, and lifetime.
- Volatile, device-backed, or concurrent observation of accesses.
- Indirect destination identity, full signature, effects, and return behavior.
- Actual fault behavior and fidelity of the static representation to execution.

No inferred record schema, text interpretation, or ownership model is an acceptance requirement.

## Evidence

Established static evidence consists of the supplied representation’s access widths, expressions, ordered reads and writes, snapshot uses, constants, global reads, and terminal indirect transfer.

In particular, it shows two distinct working-`0x44` reads before shifted-byte storage; two intervening working-record stores between companion bytes `0x79` and `0x68`; and a signed-byte read separating the early and late final halfword stores.

The supplied annotations report overlapping global symbols and incomplete recovery of the final indirect transfer. They limit confidence in semantic identities and control-transfer interpretation; they do not establish additional behavior.

No executable inspection, independent type information, runtime trace, or test result is available. This is a static behavioral contract with explicitly retained unknowns, not a runtime-verified specification.
