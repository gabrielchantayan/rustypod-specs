<!-- Trusted provenance: F-fdea2c4a66f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdea2c4a66f2

## Purpose

F-fdea2c4a66f2 is an externally state-dependent memory transformation followed by an unresolved indirect invocation. The supplied static representation establishes numerous byte, halfword, and word accesses, including overlapping writes and accesses to low absolute addresses. It does not establish a reliable application-level purpose.

The supported contract is therefore an ordered memory-effects contract, not a claim that the function formats text, initializes an object, or manipulates a particular data structure. Several written constants resemble character values, but that resemblance is insufficient to establish encoding or intent.

All behavior below is conditional on execution reaching the relevant operation, the computed addresses being accessible, and the recovered operations accurately representing execution. No runtime tests have been performed.

## Interface

### Explicit inputs

The visible interface has four inputs and no declared return value.

| Contract label | Width | Established use |
|---|---:|---|
| First address | 32 bits | Base for byte writes at offset 9. |
| First payload | 32 bits | Stored as an uninterpreted word through an ambient base. |
| Second address | 32 bits | Initial destination base; subsequently reduced by 150 for address calculations. |
| Second payload | 32 bits | Stored as a word and, separately, narrowed to one byte. |

The address inputs appear as signed 32-bit quantities in the supplied representation. That does not establish a portable pointer interface or valid signed-overflow behavior.

### Ambient inputs

Four additional entry-state quantities are consumed without initialization in the visible body:

- **Ambient selector base:** supplies both an address for a byte read and low-byte values for writes.
- **Ambient payload base:** supplies the destination for the first payload.
- **Ambient link base:** supplies destinations and a low-byte value.
- **Ambient link value:** is stored directly and used in doubled-address calculations.

These are required execution-state inputs even though the visible four-argument interface does not expose them. Their calling convention and initialization requirements are **documented_only**.

A separately stored 32-bit global value is read partway through execution and then used as another address base. This is called the **loaded base** below. Its value is determined at that read, not necessarily at function entry.

### Derived quantities and widths

For describing the accesses:

- **Selector** is the unsigned byte read from ambient selector base plus 5.
- **Seed** is the 32-bit value read at numeric address Selector plus `0x74`.
- **Double-seed** and **quadruple-seed** are the respective products of Seed by 2 and 4.
- **Adjusted address** is the original second address minus 150.
- **Quadruple-adjusted** and **octuple-adjusted** are the respective products of Adjusted address by 4 and 8.

These labels describe values, not objects or ownership relationships. In particular, Selector is used directly as a small numeric address component. There is no visible table-base addition or word-index scaling before the read at Selector plus `0x74`.

Byte stores retain the low eight bits. Halfword stores cover two bytes; word stores cover four. Endianness, alignment tolerance, address wrapping, and overflow semantics require target-specific confirmation.

### Preconditions and outputs

Successful completion of the visible body requires all reached reads and writes to be supported by the execution environment, including low absolute addresses and computed addresses that may be unaligned.

There is no visible null check, range check, capacity check, or error result. The observable outputs are memory effects and the final indirect invocation. A normal return is represented after that invocation, but whether the target returns is unknown.

## Behavior

### Initial destination updates and seed acquisition

Before reading Selector, the function performs the following ordered updates:

1. The byte at First address plus 9 receives the low byte of Ambient link base.
2. The word at Ambient payload base plus `0x44` receives First payload.
3. The word at Second address plus `0x14` receives Second payload.
4. The byte at Ambient link base plus `0x15` receives the low byte of Ambient selector base.
5. The word at Ambient link base plus `0x44` receives the original Second address.

Selector is then read from Ambient selector base plus 5.

Next, the byte at Second address plus `0x15` receives the low byte of Second payload. Seed is read at Selector plus `0x74`, after that byte store.

The following updates complete this phase:

- The byte at Second address plus `0x15` receives the low byte of Ambient selector base.
- The word at Second address plus `0x54` receives Ambient link value.
- The byte at First address plus 9 receives the low byte of Double-seed.
- The word at Second address plus `0x14` receives Second payload again.

The two locations at Second address plus `0x14` and plus `0x15` overlap. The last word store replaces the intervening byte changes within its four-byte span. Those byte changes still belong to the ordered trace and can matter for device memory, concurrent observation, or aliasing with subsequent reads.

### Constant writes and adjusted-address preparation

The function writes:

- One byte containing `0x3A` at Quadruple-seed.
- One halfword containing `0x003A` at eight times Seed.
- One word containing `0x0000003A` at Quadruple-seed plus `0x20`.

The original Second address is reduced by 150 for the remaining adjusted-address calculations. This adjustment does not retroactively change earlier destinations.

At Quadruple-adjusted plus `0x20`, four word writes occur in this order:

1. `0x00000020`.
2. `0x0000002F`.
3. The zero-extended byte read from Quadruple-adjusted plus 4.
4. `0x00000020`.

The byte read occurs after the first two word writes and before the third. Consequently, an overlapping source address can observe an earlier write.

The loaded base is then read from its global storage location. After that read, the function stores the 32-bit value `0x00000020` at absolute address `0x60`.

### Repeated link-style updates

First, a word at twice Ambient link value plus `0x54` receives Quadruple-seed.

A three-write linkage update follows:

- Loaded base plus `0x19` receives the low byte of Quadruple-adjusted.
- Quadruple-adjusted plus `0x11` receives the low byte of Loaded base.
- Loaded base plus `0x34` receives twice Ambient link value as a word.

The body then contains a long, finite, explicitly repeated sequence. Each repetition has the same ordered effects:

- A word at Octuple-adjusted plus `0x54` receives Quadruple-seed.
- Loaded base plus `0x19` receives the low byte of Quadruple-adjusted.
- Quadruple-adjusted plus `0x11` receives the low byte of Loaded base.
- Loaded base plus `0x34` receives Octuple-adjusted as a word.

No visible condition selects among these repetitions, and no source value is reread between them. The repetitions must not be treated as a single write for an observer that distinguishes repeated accesses.

The exact repetition count is **documented_only** in this contract. It is not asserted as a verified numeric trace property. This limits acceptance of implementations or observers that depend on the precise number of accesses; it does not justify inventing a count or replacing the sequence with an unbounded loop.

### Late snapshots and overlapping updates

After the repeated sequence, the function captures a word from absolute address `0xA9`, followed by a word from Quadruple-seed plus `0x74`.

It then writes a byte at absolute address `0x3A`. The value is the low eight bits of eight times Adjusted address plus `0x30`.

Two more words are captured, in order:

- Double-seed plus `0x14`.
- Quadruple-adjusted plus 4.

The function then:

- Writes the low byte of Quadruple-adjusted at Double-seed plus `0x0D`.
- Writes byte `0x6C` at Quadruple-adjusted plus 8.
- Writes the low byte of Loaded base at Quadruple-adjusted plus `0x0D`, then at plus 9.
- Stores Double-seed as a word at Double-seed plus `0x54`.
- Writes the low byte of the earlier absolute-address snapshot at absolute address `0x67`.

A word is next read from Double-seed plus `0x44`; call its captured value **First late link**. After that read, the byte currently at absolute address `0x67` is copied to absolute address `0x21`.

The low byte of First late link is written at Quadruple-adjusted plus `0x10`, and First late link is stored as a word at Loaded base plus `0x54`.

A byte is written at Double-seed plus 1 using a right shift of First late link. The shift count is the low eight bits of Quadruple-adjusted. Signed right-shift behavior and counts outside the ordinary 32-bit range are **documented_only**; a portable shift result is not established.

The word at Loaded base plus `0x54` is then overwritten with Quadruple-adjusted.

Next, a word is read from Quadruple-adjusted plus `0x40`. The earlier snapshot from Quadruple-adjusted plus 4 is stored at Loaded base plus `0x74`, and the newly read word is stored at Quadruple-adjusted plus `0x10`. This word store overlaps and replaces the earlier byte store at the same starting address.

The remaining updates before the next link read are:

- Loaded base plus `0x44` receives Double-seed as a word.
- Quadruple-adjusted plus 9 receives the current byte at absolute address `0x67`.
- Double-seed plus 9 receives the low byte of Quadruple-adjusted.
- Quadruple-adjusted plus `0x44` receives the earlier word snapshot from absolute address `0xA9`.

### Final link traversal and invocation preparation

A word is read from Quadruple-adjusted plus `0x54`; call it **Second late link**. The byte at Quadruple-adjusted plus 9 is again written from the current byte at absolute address `0x67`.

The low byte of the captured word from Quadruple-seed plus `0x74` is written at Second late link plus `0x0D`. The earlier snapshot from Double-seed plus `0x14` is restored as a word at Double-seed plus `0x54`.

A word is then captured from Quadruple-adjusted plus `0x14`. Subsequently:

- Double-seed plus `0x11` receives the low byte of the captured word from Quadruple-seed plus `0x74`.
- Quadruple-adjusted plus 9 receives the low byte of Second late link.
- Quadruple-adjusted plus `0x44` receives Second late link as a word.

A word read from Double-seed plus 4 supplies the **Traversal base**. A halfword containing `0x0065` is written at twice Traversal base.

The function reads a word at Traversal base plus 100, yielding the **Traversal payload**. It then writes the low byte of the snapshot from Quadruple-adjusted plus `0x14` at Second late link plus `0x11`, and stores the word value 100 at Second late link plus `0x54`.

The last two visible reads capture:

- A word at Traversal base plus `0x74`, the **Final snapshot**.
- A word at Double-seed plus 100, the **Final destination**.

Finally:

- Double-seed plus 9 receives the low byte of Traversal base.
- Final destination plus `0x54` receives Traversal payload as a word.
- Twice Traversal payload receives Final destination as a word.

The unresolved indirect endpoint is then invoked with four represented arguments, in order:

1. The value `0x72`.
2. Traversal payload.
3. The earlier snapshot from Double-seed plus `0x14`.
4. Final snapshot.

No additional function-level memory operation is visible after that invocation.

## Side effects

The function directly modifies memory reachable through explicit inputs, ambient state, loaded values, arithmetic-derived addresses, and absolute addresses `0x21`, `0x3A`, `0x60`, and `0x67`.

No disjointness assumption is established. An earlier write may affect a later read through overlap, including the global read that supplies Loaded base. Accordingly, final-state descriptions are valid only when their aliasing assumptions are stated. Captured values remain snapshots even if their original storage is later overwritten.

The representation does not establish atomicity, synchronization, hardware access widths, or memory barriers. Its access order is static evidence, not proof of a concurrent-memory guarantee.

No explicit allocation, release, ownership transfer, rollback, or error reporting appears in the visible body. The indirect endpoint may have additional effects; none are constrained here.

A failure during execution may leave earlier writes in place. No cleanup path is visible.

## Dependencies

The only visible invocation is an unresolved indirect endpoint. No verified opaque dependency ID is supplied for that endpoint, so this contract does not invent one.

Established dependency information is limited to the four represented arguments and invocation position described above. The following are unknown:

- Target selection and validity.
- Calling convention and return behavior.
- Additional ambient-state inputs.
- Global-memory modifications.
- Ownership, allocation, or release behavior.
- Exception, fault, or nonlocal control-flow behavior.

No preservation of globals or caller-owned memory is assumed.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

### Selector addressing

If the Selector read returns `0x7F`, the Seed read is attempted at numeric address `0xF3`. It is not a read from an implied object base or a scaled table entry. Whether that address permits a word read is an environmental precondition.

### Overlapping payload writes

Assume little-endian storage, accessible ordinary memory, Second payload equal to `0x11223344`, and no unrelated aliases into the destination.

The initial word store at Second address plus `0x14` places bytes `44 33 22 11`. The two byte stores at plus `0x15` temporarily change the second byte. The later repeated word store restores `44 33 22 11` at that stage. Acceptance based only on those restored bytes would miss the required intermediate accesses.

### Byte-to-word conversion

If the read at Quadruple-adjusted plus 4 returns byte `0xE1`, the third word write at Quadruple-adjusted plus `0x20` is `0x000000E1`, not a sign-extended value. The next word write is `0x00000020`.

### Early state survives a later fault

If the first five writes succeed but the Selector read faults, those writes have already been attempted. No successful error return, rollback, or final invocation is established.

### Final invocation arguments

If the relevant snapshots yield Traversal payload `0x2000`, the saved word from Double-seed plus `0x14` equal to `0xAABBCCDD`, and Final snapshot `0x12345678`, the represented invocation arguments are `0x72`, `0x2000`, `0xAABBCCDD`, and `0x12345678`.

This criterion is conditional on all preceding accesses and the unresolved transfer succeeding. It makes no claim about the endpoint’s effects.

## Uncertainties

The following are **documented_only**:

- The true application-level purpose and object layouts.
- The complete entry-state calling convention.
- The identity and storage layout of the global supplying Loaded base.
- The exact numeric count of repeated linkage groups.
- Signed-overflow, address-wrap, signed-shift, and excessive-shift-count behavior.
- Endianness and alignment requirements.
- Whether low addresses denote ordinary memory, device registers, or invalid mappings.
- Whether recovered access widths correspond directly to hardware transactions.
- Whether the unresolved transfer is accurately characterized as a returning call.
- The validity of the recovered function boundary and unusual ambient inputs.

Repeated stores and pointer-like offsets suggest state linkage, but that is inferred intent only. They do not establish lists, ownership relations, text generation, or any other higher-level abstraction.

## Evidence

This contract is based solely on the supplied static representation of F-fdea2c4a66f2.

Established static evidence includes the four visible inputs, additional ambient-state dependencies, access widths and ordering, overlapping stores, low-address accesses, captured values, repeated update pattern, and represented final invocation arguments.

The supplied representation also signals overlapping global storage and an unresolved control transfer. Those limitations prevent stronger claims about layout and control flow.

No implementation access, runtime execution, memory trace, dependency analysis, or runtime tests have been performed. Acceptance examples state conditional expectations rather than observed results.
