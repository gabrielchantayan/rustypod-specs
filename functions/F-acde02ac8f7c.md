<!-- Trusted provenance: F-acde02ac8f7c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-acde02ac8f7c

## Purpose

F-acde02ac8f7c performs an ordered sequence of unresolved indirect calls, transforms call-derived values, modifies byte-addressed memory through computed and global addresses, and makes a final unresolved indirect transfer.

Its established behavior is stateful. Relevant observations include dependency interactions, argument transformations, access widths, intermediate stores, global reloads, and the distinction between saved values and later memory contents.

No higher-level purpose is established. Numeric constants resembling character codes do not establish formatting, text output, serialization, or a device protocol.

This contract describes the supplied static representation. It does not establish that the entire represented sequence is reachable or executable in a particular environment.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit dataflow use is visible. |
| Second argument | 32 bits | No explicit dataflow use is visible. |
| Third argument | Signed 32 bits | Supplies early shifted arguments to indirect calls. |

The third argument initially contributes a three-bit left shift. That transformed value subsequently contributes left shifts by 10 and 26 bits.

Conditionally interpreting these as 32-bit bit-vector operations, the latter transformations retain the low 32 bits of the original argument shifted by 13 and 29 bits. They are not unbounded integer products.

The absence of explicit uses of the first two arguments does not establish that unresolved dependencies cannot observe their associated entry machine state.

### Implicit inputs

Execution also depends on:

- Entry state used to form the first indirect target.
- Results represented as 32-bit quantities or 64-bit containers.
- Secondary call-result state not resolved into a reliable interface.
- Existing computed-address memory contents.
- Byte and word globals, including overlapping storage.
- The destination and calling convention of the final indirect transfer.

The explicit arguments alone do not determine the memory effects or completion behavior.

### Outputs and completion

No explicit scalar result is represented. Outputs are dependency interactions and memory changes.

The final indirect transfer receives four represented 32-bit arguments:

1. Constant 0x72.
2. A word captured from offset 0x64 of a memory-derived base.
3. A saved word captured earlier from offset 0x14.
4. A word captured near the end from offset 0x74 of the same base used for the second argument.

A return is represented after the transfer. Whether that destination returns normally, exits nonlocally, or never returns is documented_only.

### Preconditions and ownership

All executed call targets must be executable, and all executed memory accesses must satisfy the environment’s address, access-permission, alignment, and lifetime requirements.

No visible validation establishes these conditions. In particular, small integers and zero-extended bytes are used as address bases without null or range checks.

No direct allocation, release, ownership transfer, or lifetime extension is established. Dependency ownership effects remain documented_only.

## Behavior

### Evidence and arithmetic model

**Established static evidence** means operations and relative ordering visible in the supplied representation. **Inferred intent** is explicitly distinguished from those observations. **documented_only** marks unresolved behavior; it is not permission to choose a convenient implementation.

Accesses distinguish:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.
- Call-result container: 64 bits where represented.

Narrow stores retain the destination-width low bits. Unsigned byte loads are zero-extended when subsequently stored as words. Endianness is not established.

Unless otherwise qualified, numerical examples involving overflow are conditional on 32-bit bit-vector arithmetic. The representation contains signed shifts, signed multiplication, and signed address arithmetic whose overflow behavior is not independently established as a portable execution contract.

Unsigned right shifts are logical. Shifting a 64-bit result right by 32 extracts its upper word; shifts by 34, 36, and 44 select progressively smaller portions of that upper word. These operations must not be confused with shifts of a previously narrowed low word.

### Indirect-call phase

A long, straight-line sequence of indirect calls precedes the first explicit direct-memory store. No visible conditional branch selects a shorter sequence.

The first target depends on entry machine state. Later targets use code-associated target expressions with a recurring displacement. Their identities and bodies are unresolved.

Established interaction properties include:

- Calls with no explicit arguments and calls with one through four represented arguments.
- Repeated use of call results shifted left by eight bits.
- Identical transformed values supplied in multiple argument positions.
- Separate use of low and high words of represented 64-bit results.
- Retention of values across intervening calls.
- Explicit zero arguments and retained zero-valued transformation chains.
- Arithmetic involving secondary call-result state.

A call without explicit arguments does not establish a zero-argument calling convention. An unused result does not establish absence of side effects.

Masking includes retained fields of 24, 22, 20, 19, 17, and 13 bits. Examples of established unsigned bounds are:

| Transformation | Result bound |
|---|---:|
| Retain 20 bits, then shift right by 10 | 0–1023 |
| Retain 19 bits, then shift left by two | 0–0x1ffffc |
| Retain 17 bits, then shift right by four | 0–8191 |

Left shifts followed by right shifts can permanently discard bits. Unknown callee effects do not justify changing explicitly retained values; retained values likewise do not imply preservation of globals.

### Memory model and descriptive values

The direct-memory region contains no further visible helper calls before the final transfer.

Offsets describe accesses, not verified structures. Word offsets include 0x04, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Byte offsets include 0x01, 0x05, 0x08, 0x09, 0x0d, 0x10, 0x11, 0x15, and 0x19.

No disjointness between bases or globals is established. A byte at offset 0x15 can overlap a word at offset 0x14. A global write can affect a subsequent computed-address read.

For the later regions, this contract uses descriptive mathematical values:

- **Retained word:** the word read through the early byte-global-derived base at offset 0x54, before that field is overwritten.
- **Adjusted scale:** four times the retained word, minus 150.
- **Derived base:** four times the adjusted scale.
- **Publication base:** a word captured from a separate global before publication begins.
- **Late link:** the later word read at offset 0x74 through a zero-extended byte base.
- **Doubled link:** twice the late link.

These are explanatory names, not object types or ownership claims.

### Initial memory effects

The first store writes a retained call-derived value as a word at offset 0x14 from the upper word of the final call result.

An offset-0x09 byte store is followed by an offset-0x54 word store through a base formed from the final result’s low word multiplied by 256. The offset-0x09 byte is then overwritten with another retained value.

A different scaled base has its offset-0x14 word cleared, followed by an unsigned byte read at offset 0x05. A word at offset 0x14 from a small doubled base is cleared before the loaded byte is stored, zero-extended, at offset 0x34 from the other scaled base.

The previously cleared word is then read back. The small-base clear is repeated, followed by two byte stores at offset 0x15 of that small doubled base, using the read-back word’s low byte.

The small value is derived from a call result shifted left by eight, masked to 15 bits, and shifted right by six. Under bit-vector arithmetic it is a multiple of four between 0 and 508. Its doubled base is a multiple of eight between 0 and 1016. The corresponding offset-0x14 word address lies numerically between 20 and 1036; validity of that range is not established.

### Early globals and linked updates

The byte global at address 0x05 is read and zero-extended as an address base. **After that read**, the low byte of the small doubled base is written to the byte global at address 0x01.

Computed-address byte and word updates follow. They include overwriting a byte at offset 0x11 with zero and reading the byte-global-derived base’s offset-0x54 word before replacing that word. The pre-overwrite value becomes the retained word.

Subsequent accesses use offset-0x74 links, repeated self-value stores at offset 0x24, and repeated byte stores at offset 0x09. One link-derived offset-0x54 update itself depends on represented reads of the offset-0x74 field; no immutable structure snapshot is established.

During repeated offset-0x14 updates through the small doubled base:

- Twice the retained word is written to the word global beginning at address 0x54.
- Further computed-address writes occur.
- The low byte of the small doubled base is subsequently written to the byte global at address 0x15.

Thus, the byte-global read, first byte-global write, word-global write, and later byte-global write are separate ordered events, not one grouped global-state update.

Further accesses derive doubled, quadrupled, and eightfold values from the retained word. They populate offset-0x34, 0x44, and 0x54 fields, follow an offset-0x64 link, and use a byte read at offset 0x05 as another address base.

An offset-0x05 byte is explicitly read again after intervening writes. The late link comes from the resulting byte-based offset-0x74 read. Its doubled value supplies the doubled link used in the final region.

### Constants and repeated publication

The constant 0x3a is stored at three widths:

| Width | Destination |
|---|---|
| 8 bits | Four times the late link |
| 16 bits | Eight times the late link |
| 32 bits | Offset 0x20 from four times the late link |

At offset 0x20 from the derived base, word stores occur in this order: 0x20, 0x2f, a freshly loaded unsigned byte from derived-base offset 0x04, and 0x20.

**The publication-base global is read before the word-global store of 0x20 at address 0x60.** The captured publication base is used afterward. If these locations overlap, publication still uses the value captured before the store, not an assumed post-store reload.

The initial publication group writes:

- Four times the late link to offset 0x54 from eight times the retained word.
- The derived base’s low byte to publication-base offset 0x19.
- The publication base’s low byte to derived-base offset 0x11.
- Eight times the retained word to publication-base offset 0x34.

Subsequent repeated groups preserve that relative store order but use eight times the adjusted scale for both the offset-0x54 destination base and the value published at offset 0x34.

These are repeated accesses, not merely a final-state assignment. An exact repetition count is not asserted by this contract.

### Final captures and intervening global mutation

The final region does **not** capture all saved values before further mutations.

Its initial ordering is:

1. Read the 32-bit global beginning at address 0xa9.
2. Read the word at offset 0x74 from four times the late link.
3. Write the byte global at address 0x3a.
4. Capture the word at doubled-link offset 0x14.
5. Capture the word at derived-base offset 0x04.

The intervening byte-global write retains the low byte of **eight times the adjusted scale plus 0x30**.

The two later saved reads must reflect that preceding write if their storage overlaps address 0x3a. Neither the offset-0x14 nor offset-0x04 capture is an entry snapshot or a pre-mutation snapshot.

### Final byte-global reloads and remaining updates

After those captures, computed-address updates write the derived base’s low byte, the publication base’s low byte, byte constant 0x6c, and a self-base word at doubled-link offset 0x54.

The low byte of the captured 0xa9 global word is then assigned to the byte global at address 0x67.

That assignment does not create an immutable source for later byte stores:

- A word is read at doubled-link offset 0x44.
- The byte global at 0x67 is read to populate the byte global at 0x21.
- Further computed-address writes occur before the next reload.
- The byte at 0x67 is reloaded for a store at derived-base offset 0x09.
- Intervening computed-address writes and a derived-base offset-0x54 word read occur.
- The byte at 0x67 is reloaded again for another store at derived-base offset 0x09.

The supplied representation therefore establishes **three reloads total: one global-to-global copy and two computed-address byte stores**. It does not show a third computed-address store sourced from that byte global.

Intervening writes may change the reloaded byte through aliasing. In particular, the doubled-link offset-0x09 byte store between the two computed-destination reloads cannot be ignored.

Other final effects include a byte derived from a variable-count signed right shift, word copies from offsets 0x40 and 0x04, replacing publication-base offset 0x54 with the derived base, and using a freshly loaded derived-base offset-0x54 word as another base.

Near completion, a word loaded at doubled-link offset 0x04 supplies a further base. The function writes halfword 0x65 at twice that base, reads its offset-0x64 word, writes constant 100 at another offset-0x54 destination, and captures its offset-0x74 word.

After another offset-0x64 read through the doubled link, final byte and word writes occur, including a word store through twice the previously captured offset-0x64 value. The final transfer uses the saved values described under Interface; intervening stores do not automatically refresh them.

## Side effects

Established side effects are ordered indirect calls, mixed-width direct memory accesses, global reads and writes, and the final indirect transfer.

There is no visible rollback, transaction boundary, lock, synchronization primitive, bounds validation, or structured error-reporting path.

Repeated stores and overwritten intermediate values remain part of the represented access sequence. Whether every access is externally distinguishable depends on the execution model, mapping, concurrency, and memory-mapped behavior.

The function is not established as idempotent. Identical explicit inputs may encounter different dependency state or memory contents.

## Dependencies

F-acde02ac8f7c is the only authoritative opaque function ID supplied.

All other indirect dependencies, including the final destination, lack mapped opaque IDs. Their identities, signatures, implicit inputs, effects, failure modes, and ownership behavior are documented_only. No substitute IDs are invented.

Repeated target patterns do not prove identical dependency identity. Different target expressions do not prove distinct identity. No dependency is assumed to preserve globals or memory lifetimes.

The final destination is not established as F-acde02ac8f7c itself.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input transformation.** With third argument 1 and 32-bit bit-vector shifts, the initial transformed value is 8; its later shifts by 10 and 26 yield 0x00002000 and 0x20000000.

2. **Discarded bits.** With only original bit 29 set, the initial three-bit shift yields zero under that model. Later transformations must not recover the discarded bit.

3. **Narrowing.** A byte store supplied with 0x123456ab writes 0xab. It does not write four bytes.

4. **Clear and read-back.** With ordinary stable storage and no interfering alias, reading back the cleared word yields zero. Otherwise, the actual ordered read remains necessary.

5. **Early global ordering.** If a computed write overlaps an early global, acceptance must preserve the byte-base read before the 0x01 write, and the word-global 0x54 write before the later 0x15 write.

6. **Publication capture.** If the 0x60 store overlaps publication-base storage, publication uses the word read before that store.

7. **Intervening capture mutation.** If address 0x3a overlaps either later saved word, that capture includes the effect of the intervening byte write. The earlier 0xa9 and offset-0x74 captures remain saved values.

8. **Reload-sensitive publication.** If a computed-address write between the two 0x67 reloads changes that byte global, the second derived-base offset-0x09 store uses the newly read byte, not necessarily the original captured global’s low byte.

9. **Intermediate constants.** An observer able to distinguish derived-base offset-0x20 stores sees 0x20, 0x2f, the freshly read byte as a word, and 0x20.

10. **Failure.** Invalid derived addresses or targets have no established structured recovery response. Acceptance must not invent validation, a fallback result, or rollback.

## Uncertainties

The following remain documented_only:

- Actual calling conventions and implicit entry-state requirements.
- Indirect dependency identities and contracts.
- Meaning and reliability of secondary call-result state.
- Runtime reachability and completeness of the represented control-flow boundary.
- Higher-level purpose and object layout.
- Endianness, alignment rules, address mapping, and external observability.
- Actual aliasing among globals and computed addresses.
- Signed overflow and variable-count signed right-shift semantics.
- Exact publication repetition count not asserted here.
- Dependency ownership and lifetime changes.
- Final-destination completion behavior.
- Outcomes of execution or memory-access faults.

No runtime tests have been performed.

## Evidence

The supplied static representation supports the widths, transformations, access dependencies, and relative ordering described above.

In particular, it distinguishes:

- Early byte-base capture from subsequent byte-global writes.
- The retained-word global store from later small-base byte publication.
- Publication-base capture before the 0x60 word store.
- Two initial final-region reads, an intervening 0x3a byte write, and two subsequent saved-word reads.
- Initial assignment to 0x67 from three later explicit reloads.
- Saved final-transfer arguments from later memory mutations.

No runtime evidence, resolved dependency implementation, verified structure definition, or higher-level API interpretation is available.
