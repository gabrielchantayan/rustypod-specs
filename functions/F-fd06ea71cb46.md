<!-- Trusted provenance: F-fd06ea71cb46; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd06ea71cb46

## Purpose

F-fd06ea71cb46 performs an ordered sequence of indirect transfers, transforms returned and ambient values, and then reads and modifies shared memory using computed addresses and mixed access widths. Its final visible operation is an unresolved indirect transfer with four displayed arguments.

The static evidence does not establish an application-level purpose. Recurring offsets, address-like values, and character-like constants do not prove that the function constructs objects, manipulates a particular collection, or formats text.

This is a partial behavioral contract. It specifies established widths, ordering constraints, arithmetic dependencies, and selected memory-access traces. It does not supply a complete dependency-call transcript or independently counted repeated-store multiplicity. Those limitations remain explicitly `documented_only`.

## Interface

### Inputs

The displayed interface has no explicit parameters. Execution nevertheless depends on:

- An incoming control-flow value used to select the first indirect target.
- An incoming 32-bit register-carried value used in early shifted arguments.
- Results from indirect transfers, displayed as 32-bit words or 64-bit quantities.
- Additional register-carried outputs whose production and calling convention are incompletely described.
- Existing bytes and words at computed addresses and fixed locations.

There is no established conventional caller signature beyond the absence of displayed parameters. The complete ambient input state and required memory image are `documented_only`.

### Outputs and completion

No scalar return value, success indicator, or error code is displayed. Observable behavior includes dependency transfers, their arguments, memory accesses, and dependency-generated effects.

A return is displayed after the final transfer. Normal return is conditional on that transfer and every preceding operation completing normally. The evidence does not guarantee that any unresolved dependency returns.

### Widths and memory layout

| Quantity or access | Width | Established interpretation |
|---|---:|---|
| Byte | 8 bits | Narrow stores retain the low eight bits. Unsigned byte reads widen without sign extension. |
| Halfword | 16 bits | Used for selected constant stores. |
| Word | 32 bits | Used for arithmetic, state, and address-like quantities. |
| Split transfer result | 64 bits | Low and high 32-bit portions are consumed separately. |

Offsets include 0x01, 0x04, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Offset 0x64 is also expressed as decimal 100.

These offsets do not establish one common record layout. Some bases originate from scaled integers or zero-extended bytes rather than identified allocations. Object boundaries, byte order, alignment requirements, and address-space mapping remain `documented_only`.

### Preconditions

Normal completion requires every reached indirect target to be executable under the actual calling convention and every memory access to be permitted at its actual width.

No displayed validation establishes those conditions. There are no visible null, bounds, allocation-success, or low-address checks. A computed address is not implicitly valid merely because it is subsequently dereferenced.

## Behavior

### Indirect transfers and retained values

The initial region contains a long, finite, straight-line sequence of indirect transfers. No explicit conditional branch or loop governs that region in the supplied representation.

The first target depends on incoming control-flow state. Many later targets are formed by adding 0xFFE to embedded continuation-like values. This target arithmetic does not resolve the dependencies’ identities or meanings.

Displayed arguments include shifted 32-bit results, low and high portions of split results, explicit zeros, masked fragments, repeated arguments, and values retained across intervening transfers. A later call can therefore depend on a result other than the immediately preceding one.

Transfers with no displayed arguments remain part of the observable sequence. Neither their ambient inputs nor their side effects are known. An unused displayed return value does not make a transfer removable.

Some intermediate values are explicitly zeroed and remain zero through subsequent visible shifts. That arithmetic fact does not justify eliminating intervening transfers or assuming that dependencies preserve memory or unspecified registers.

The exact call-by-call argument transcript is outside this partial contract and remains `documented_only`.

### Arithmetic widths and fragments

Visible constant left-shift counts range from **1 through 31**. Shifts by 1 and 2 occur, including in the explicit memory-mutation region. The lower bound is not 3.

Most intermediate arithmetic uses 32-bit words. Where a shifted result is retained in a 32-bit intermediate before another operation, that intermediate width matters. Under ordinary truncating machine-word semantics, discarded high bits cannot reappear in a later shift.

Established fragment operations include:

- Selecting bits 10–19 by retaining the low 20 bits and logically shifting right by 10.
- Retaining low 24-bit fragments before additional shifts.
- Other masks retaining low 22-, 21-, 20-, 19-, 18-, 17-, 16-, 15-, 14-, 13-, 12-, or 11-bit portions.
- Unsigned 64-bit right shifts by 34, 36, and 44, yielding respectively up to 30, 28, and 20 retained bits.
- Logical right shifts of unsigned 32-bit quantities.

Unsigned operations have the displayed unsigned interpretation. Signed left shifts, signed multiplication, and address arithmetic do not establish portable overflow semantics. Any modulo-2³² interpretation of those operations is conditional on the underlying machine using 32-bit truncation.

A late signed right shift uses a count taken from the low eight bits of a computed word. The displayed count can exceed 31. Count masking, oversized-count behavior, and negative-value right-shift behavior remain `documented_only`.

### Initial explicit memory mutations

“Terminal split result” below means the last displayed 64-bit transfer result before the explicit memory-mutation region. This is a descriptive role, not a dependency identity.

The opening mutation boundary has a definite order:

1. A retained word is stored at the terminal high-word address plus 0x14.
2. The low byte of that high word is stored at another retained base plus 0x09.
3. A retained word is stored at 256 times the terminal low word, plus 0x54.
4. The preceding offset-0x09 byte destination is overwritten with the low byte of a different retained word.
5. A separate computed base plus 0x14 is cleared.

Only afterward is a byte read from the separate base plus 0x05. That byte is retained, another computed word destination is cleared, and the retained byte is zero-extended into a 32-bit store at offset 0x34.

The cleared offset-0x14 word is subsequently read, but intervening stores exist. Its read value must not be assumed zero without establishing non-aliasing and appropriate memory semantics.

### Mixed-width updates and address chains

Subsequent operations combine narrow stores, repeated word stores, and address chains. Established ordering-sensitive behavior includes:

- Repeated zero stores to the same computed word destination.
- Two consecutive identical byte stores at an offset-0x15 destination.
- An offset-0x11 byte write, an intervening word store, and then clearing that byte.
- Capturing a word from offset 0x54 before replacing that location, then using the captured word for later address computations.
- Repeated self-address-like stores through offsets 0x14, 0x24, and 0x54.
- Reading offset-0x74 words and using their values as new bases.
- Reading an unsigned byte at offset 0x05 and using its value directly as the base for an offset-0x74 word read.

That last address lies between 0x74 and 0x173 before any platform-specific address interpretation. No separate table base is displayed.

Repeated reads from the same source are not generally interchangeable with one retained read. Likewise, intervening stores can change later pointer-like loads through aliasing.

### Early fixed-location ordering

Fixed locations participate in the same address space as computed accesses.

An unsigned byte at fixed location 0x05 is captured before a byte is written to fixed location 0x01. The captured value is then used as a base for an offset-0x15 byte store and later offset-0x54 word accesses. The word at that offset is captured before replacement.

Later, fixed word location 0x54 receives twice that captured word, using a displayed one-bit left shift. Computed offset-0x14 stores occur on either side of this fixed-location write. A later byte store to fixed location 0x15 occurs between repeated computed offset-0x15 byte writes and further computed word updates.

These are ordered accesses, not independent abstract globals. Earlier or later computed stores may overlap them.

### Constants and repeated replacement

A constant-write region stores 0x3A as a byte at a scaled base, as a halfword at another scaled base, and as a word at the first scaled base plus 0x20.

A different computed offset-0x20 word destination then receives four consecutive word stores:

- 0x20.
- 0x2F.
- The zero-extended byte read from its associated base plus 0x04.
- 0x20 again.

The byte source is read after the first two stores and before the third. This is not an entry-time memory snapshot.

After those four stores, a fixed-location word is loaded as a base. Fixed word location 0x60 is then assigned 0x20 before the repeated-store region begins.

The four-store sequence establishes a value at that sequence boundary, not at function exit. Subsequent computed stores and the final unresolved dependency may modify the destination.

### Repeated-store region

The function performs a long, finite series of store groups involving:

- A word at a scaled destination plus 0x54.
- A byte at the fixed-loaded base plus 0x19.
- A byte at another computed base plus 0x11.
- A word at the fixed-loaded base plus 0x34.

Within each displayed group, those four access classes occur in that order. The initial group uses a different scale source for its offset-0x54 destination and offset-0x34 stored value than the subsequent groups.

The repeated groups reuse retained bases and values; no advancing traversal is displayed. They are neither an unbounded loop nor a single abstract update. Individual stores can matter for overlapping memory or externally observable access behavior.

Exact group multiplicity has not been independently counted here and remains `documented_only`.

### Late captures and the fixed-location 0x67 trace

For this subsection only:

- **Working base** means the computed base used for late accesses at offsets 0x04, 0x09, 0x10, 0x40, 0x44, and 0x54.
- **Record base** means the retained base used for late accesses at offsets 0x01, 0x0D, 0x14, 0x44, 0x54, and 0x64.
- **Fixed-loaded base** means the retained base used throughout the repeated-store region.

These roles do not imply non-aliasing or ownership.

After the repeated stores, the function captures the word at fixed location 0xA9 and an offset-0x74 word through another retained base. It then writes a computed byte to fixed location 0x3A before capturing the record-base offset-0x14 word and working-base offset-0x04 word. The fixed-0x3A byte is the low-eight-bit result of eight times a retained adjusted base plus 0x30.

Several computed stores follow those captures, including the 0x6C byte at working base plus 0x08 and a self-address-like word at record base plus 0x54.

The fixed-location 0x67 dependency trace is established, rather than merely inventoried:

- **Publication and first read:** fixed byte 0x67 receives the low byte of the previously captured fixed-0xA9 word. A word is then loaded from record base plus 0x44. Only afterward is byte 0x67 read for the byte store to fixed location 0x21.
- **Second read:** after the 0x21 store, intervening computed writes include a byte at working base plus 0x10, two word writes at fixed-loaded base plus 0x54 with a shift-derived byte write between them, a word at fixed-loaded base plus 0x74, a word copy into working base plus 0x10, and a word at fixed-loaded base plus 0x44. Byte 0x67 is then read again for a store to working base plus 0x09.
- **Third read:** after that offset-0x09 store, a byte is written at record base plus 0x09 and the captured fixed-0xA9 word is written at working base plus 0x44. A word is then loaded from working base plus 0x54. Byte 0x67 is read again for another store to working base plus 0x09.

Each of those three reads obtains the byte present at its own access point. Intervening computed writes may overlap 0x67. Replacing the reads with the originally published byte is not justified without non-aliasing and suitable memory guarantees.

The word copy into working base plus 0x10 obtains its source from working base plus 0x40. It occurs after the earlier byte write at offset 0x10 and may overwrite that byte.

### Final updates and handoff

Following the third 0x67 read, further writes use the previously captured offset-0x74 word and record-base offset-0x14 word. The latter is written to record base plus 0x54 and retained for the final transfer.

The working-base offset-0x09 byte is subsequently overwritten with the low byte of the word previously loaded from working base plus 0x54. Thus neither 0x67-sourced store establishes a final value there.

A later word read from record base plus 0x04 supplies another base. A halfword 0x65 is stored at twice that loaded base. The function then reads its offset-0x64 word, performs further byte and word stores including a word value of 100 at offset 0x54, and reads its offset-0x74 word.

Additional accesses follow that offset-0x74 capture before the handoff: an offset-0x64 read through the record base, an offset-0x09 byte write, an offset-0x54 word write, and a word store through twice the captured offset-0x64 value.

The final transfer receives four displayed arguments:

1. 0x72.
2. The word captured from offset 0x64 of the later loaded base.
3. The earlier record-base offset-0x14 capture.
4. The later loaded base’s offset-0x74 capture.

The third argument is not reloaded after intervening stores. The fourth is captured later than the third, but is not necessarily read immediately adjacent to the transfer.

## Side effects

The function is not established pure or idempotent. Direct effects include byte, halfword, and word stores, repeated stores, fixed-location accesses, and computed-address reads that influence later addresses and arguments.

No visible allocation or release operation is identifiable. Dependencies may nevertheless allocate, retain, release, or modify memory. No ownership transfer or lifetime guarantee can be assigned to address-like arguments or stored values.

There is no displayed rollback, synchronization, transaction, or error-recovery mechanism. A fault or non-returning dependency may leave earlier effects in place. Error signaling, fault delivery, and partial-state visibility are `documented_only`.

## Dependencies

Only F-fd06ea71cb46 has a supplied resolved opaque identity.

| Opaque handle | Document-local scope |
|---|---|
| F-00000001 | The earlier indirect targets, collectively. |
| F-00000002 | The final unresolved transfer target. |

These handles are references, not resolved implementation identities. F-00000001 does not assert that all earlier transfers reach one function. F-00000002 does not assert that the final target differs from every earlier target.

Neither dependency handle carries a guarantee of normal return, global preservation, unspecified-register preservation, memory ownership, or resource release.

## Acceptance examples

These are conditional static criteria. No runtime tests have been performed.

### Narrowing and widening

A byte store derived from 0x123456AB writes 0xAB. An unsigned byte read of 0xFE widened into a word produces 254, not −2.

### Fragment selection

For selection of bits 10–19, 0x000FFC00 produces 1023 and 0x00100000 produces zero. Differences confined above bit 19 do not affect that fragment.

### Intermediate truncation

Under 32-bit truncating semantics, two successive left shifts by 28 with a 32-bit retained intermediate produce zero. Preserving an unbounded intermediate would not match.

### Ordered replacement boundary

If the four offset-0x20 stores complete to ordinary writable memory, the destination word is 0x20 **immediately after the fourth store**, absent concurrent modification. The trace still contains the preceding 0x20, 0x2F, and widened-byte stores.

A function-exit value of 0x20 additionally requires that no subsequent store or dependency modifies any byte of that word.

### Fixed-byte rereads

If the 0x67 publication writes 0xAB and no intervening operation changes that byte, its three subsequent reads obtain 0xAB under ordinary memory semantics.

If an intervening computed store changes 0x67 to 0xCD before the second read, that second read supplies 0xCD to working base plus 0x09. The third read depends on memory at its own later access point; it is not required to equal either earlier read.

### Captured final argument

If the earlier record-base offset-0x14 capture is 0x11223344, the final transfer’s third argument remains 0x11223344 even if that source location is later modified. Its first displayed argument is 0x72 whenever the transfer is reached.

## Uncertainties

The following remain `documented_only`:

- Application purpose, intended invariants, and record meanings.
- Complete calling convention and ambient input state.
- Dependency identities, bodies, side effects, and return guarantees.
- Whether split results are genuine 64-bit returns or register-pair representations.
- Executable control-flow fidelity of the supplied representation.
- Complete transfer transcript and exact repeated-store multiplicity.
- Memory mapping, byte order, alignment, concurrency, and device-memory behavior.
- Complete alias relationships and object boundaries.
- Signed overflow, address wrapping, and oversized-shift semantics.
- Ownership, allocation, release, error handling, and fault outcomes.

Record-like links and character-like constants are clues to possible intent only, not acceptance requirements.

## Evidence

Established static evidence consists of displayed access widths, arithmetic dependencies, constants, offsets, transfer order, repeated stores, captured-versus-reloaded values, and the final four-argument transfer.

In particular, the representation includes left shifts by 1 and 2, places later operations after the four offset-0x20 stores, and shows one write followed by three distinct reads of fixed byte 0x67 with intervening accesses.

No dependency implementations, caller contract, memory map, or runtime observations were available. No runtime tests have been performed. Conditional examples state consequences under explicit assumptions; they do not establish successful execution on any platform.
