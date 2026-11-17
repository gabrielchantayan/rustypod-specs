<!-- Trusted provenance: F-1fbdb77c98a5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1fbdb77c98a5

## Purpose

F-1fbdb77c98a5 performs an extensive ordered sequence of indirect dependency invocations, transforms their result bit patterns, and then reads and writes shared storage and computed memory locations using mixed access widths.

No domain-level purpose is established. Printable constants do not establish formatting, serialization, or text output. Computed addresses and self-referential stores do not establish object construction, allocation, or a particular data structure.

This contract distinguishes:

- **Established static evidence:** interactions, accesses, widths, and ordering explicitly represented in the supplied body.
- **[INFERENCE]:** conclusions conditional on a machine model or interpretation.
- **documented_only:** unresolved behavior that must not be replaced by a guessed rule.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits; interpretation unspecified | No explicit consumption is visible. |
| Second argument | Signed 32 bits | Supplies an early dependency argument after a left shift of 11 bits. |
| Third argument | Signed 32 bits | Supplies early arguments after a left shift of 3 bits; that shifted value undergoes further shifts and reuse. |

Absence of explicit consumption does not prove the first argument is irrelevant to actual execution. Dependency signatures and implicit calling-convention inputs are unresolved.

### Implicit inputs

Execution depends on incoming return-link state, indirect targets, dependency-produced primary words, paired 64-bit results, additional result-register contents, shared storage, and memory reached through calculated addresses.

There is no established explicit buffer, length, object handle, or typed output parameter.

### Outputs

The represented interface has no explicit return value. Observable outputs comprise dependency interactions and memory effects.

The terminal dependency, labeled **F-U002**, receives four represented arguments:

1. 0x72.
2. A word obtained near completion through an offset-0x64 read.
3. A saved word read from offset 0x14 of a derived base before the final mutation groups.
4. A later word read through another derived base at offset 0x74.

A return is represented after this interaction. Whether the actual terminal transfer returns normally is **documented_only**.

### Preconditions

Completion requires executable indirect targets, compatible result channels, valid addresses for every reached access, and suitable alignment and memory mappings. Dependencies must return for subsequent operations to occur.

No visible validation enforces these conditions. No range of explicit input values alone is established as sufficient for safe completion. Low addresses, unaligned word locations, and potentially overlapping accesses prevent assuming an ordinary heap-object environment.

## Behavior

### Dependency interaction and arithmetic

The first indirect target depends on incoming return-link state plus 0xFFE. Later targets repeatedly depend on embedded code-position references plus the same displacement. Their identities and equivalence are unresolved.

The represented sequence has no visible conditional branch or loop before the direct-memory suffix. Invocations without represented arguments remain interactions with unknown implicit inputs, results, and effects.

The sequence repeatedly:

- Passes primary results shifted left by 8 bits.
- Consumes low and high portions of paired results separately.
- Sends the same transformed word in multiple argument positions.
- Reuses saved results across intervening calls.
- Passes explicit or derived zero arguments without skipping invocations.
- Consumes additional dependency-produced register contents independently of primary results.

These interactions must not be merged or reordered merely because represented arguments match.

The access model includes 8-bit bytes, 16-bit halfwords, 32-bit words and address-bearing quantities, and 64-bit paired results. Low and high words of a paired result mean bits 0–31 and 32–63 respectively. Unsigned extraction after shifting the full result right by 34, 36, or 44 bits also discards 2, 4, or 12 bits from its high word.

Masks select fields of several widths, including 1, 11–24 bits. Narrow stores retain the destination-width fragment.

**[INFERENCE]:** Under a 32-bit machine-bitvector model, left shifts, multiplication, and address arithmetic retain only their low 32 bits. A right shift after a truncating left shift does not recover discarded bits.

This model is not established for every represented expression. Signed left shifts, signed multiplication, signed conversions, and a variable signed right shift leave exact overflow and shift-count behavior **documented_only**. Arbitrary-precision arithmetic, saturation, or automatic rejection of negative values would introduce unsupported behavior.

### Initial mutation and fixed-storage propagation

For this section, descriptive names denote saved values, not object types:

- The **large base** is an earlier dependency result multiplied by 0x2000000.
- The **small index** is obtained by masking a dependency word’s eight-bit-left-shifted form to 15 bits, then shifting unsigned right by 6.
- The **small doubled base** is twice that index.
- The **transport word** is a retained dependency result shifted left by 8.
- The **paired result** is the last represented 64-bit dependency result before direct mutation.

The initial writes publish a retained dependency-derived word at the paired high word’s offset 0x14; publish the high word’s low byte through another dependency-derived base at offset 9; and publish the large base at offset 0x54 from the paired low word multiplied by 0x100. The offset-9 byte is then overwritten with the retained word’s low byte.

Next, the large base’s offset-0x14 word is cleared and its offset-5 byte is read. The small doubled base’s offset-0x14 word is cleared before the saved byte is zero-extended into the large base’s offset-0x34 word. The large base’s offset-0x14 word is then read back. Call this the **read-back word**: aliasing prevents assuming it is necessarily zero.

The small doubled base’s offset-0x14 word is cleared again, followed by two byte stores of the read-back word’s low byte at offset 0x15.

**Only after those duplicated byte stores**, the byte at fixed address 0x05 is read and zero-extended. Its saved value becomes the **byte-selected base**. The next fixed-storage access writes the small doubled base’s low byte to address 0x01. Thus the byte-selected base reflects the state after the duplicated computed-address stores but before the address-0x01 store.

The ensuing computed accesses publish:

- The small index’s low byte at the byte-selected base’s offset 0x15.
- The transport word’s low byte at the large base’s offset 0x11.
- The transport word at offset 0x54 from the read-back word.

The large base’s offset-0x11 byte is then cleared. The word at the byte-selected base’s offset 0x54 is read and saved as the **saved link word**, then replaced with the read-back word. The small doubled base’s offset-0x34 word receives the small index.

These accesses are ordered, not an unordered field initialization.

**[INFERENCE]:** Under 32-bit bitvector shifts, the small index is a multiple of four between 0 and 508; the small doubled base is a multiple of eight between 0 and 1016. It is not defensible to assume this base is an ordinary allocated pointer.

### Linking and subsequent fixed writes

A word read from the large base’s offset 0x74 selects a destination whose offset-0x54 word first receives another dependency-derived shifted value. The small doubled base’s offset 0x74 is also read for a self-publication through offset 0x54. That expression contains two represented reads of the selector location; they must not automatically be treated as one snapshot.

The saved link word is stored at its own offset 0x24 twice. Between those publications, the selected destination receives the saved link word’s low byte at offset 9 and the transport word at offset 0x54. The offset-9 byte publication occurs again after the second self-publication.

The small doubled base’s offset-0x14 word next receives the earlier shifted value. **Before that location receives the saved link word twice**, fixed word address 0x54 receives twice the saved link word.

The subsequent duplicated offset-0x15 stores through the small doubled base contain the low byte of four times the saved link word. **After both stores**, fixed byte address 0x15 receives the small doubled base’s low byte. These fixed writes cannot be moved away from the surrounding computed accesses: either may alias a later source or destination.

Further groups scale the saved link word by four and eight, propagate words through offsets 0x14, 0x34, 0x44, and 0x54, and traverse words selected through offsets 0x04, 0x64, and 0x74. A byte read at offset 5 is zero-extended and used directly as a base for an offset-0x74 word read.

Repeated byte reads at offset 5 are distinct observations. Intervening computed stores may change their values through aliasing. Offsets identify access locations only; they do not establish record layouts, allocation sizes, or ownership.

### Constants and entry into repeated publication

A traversal supplies an address-bearing word used for three stores of 0x3A, in this order:

1. An 8-bit store at four times that word.
2. A 16-bit store at eight times that word.
3. A 32-bit store at offset 0x20 from four times that word.

The **adjusted publication index** is four times the saved link word minus 0x96. The **publication field base** is four times that adjusted index.

At the publication field base’s offset 0x20, the represented sequence stores 0x20, stores 0x2F, reads the byte at offset 4 and stores its zero-extended value as a word, then stores 0x20 again.

**After this sequence, the shared word supplying the retained publication base is read. The fixed word store of 0x20 at address 0x60 follows that load. Both precede the initial publication stores.**

The **retained publication base** is the loaded snapshot. Neither the address-0x60 store nor later aliasing stores cause that saved base to be reloaded.

Each publication group performs four stores in order:

- A 32-bit payload at offset 0x54 from a scaled base.
- The publication field base’s low byte at retained publication base offset 0x19.
- The retained publication base’s low byte at publication field base offset 0x11.
- A scaled address-bearing word at retained publication base offset 0x34.

The payload is four times the traversal word used for the 0x3A stores. The initial group uses eight times the saved link word for its first destination base and final stored word. Subsequent explicit repetitions use eight times the adjusted publication index instead.

The repetitions are present in the supplied representation. Their exact machine-instruction multiplicity is **documented_only**; their presence does not justify collapsing the represented access order.

### Final mutation and fixed-byte rereads

The **final traversal base** is twice the traversal word that supplied the 0x3A destinations. After all repeated publications, the opening order is:

- Read the 32-bit word at fixed address 0xA9 and retain it.
- Read a word at offset 0x74 from the publication payload.
- Write fixed byte address 0x3A.
- Read and retain the final traversal base’s offset-0x14 word.
- Read and retain the publication field base’s offset-0x04 word.

The address-0x3A byte derives from eight times the adjusted publication index plus 0x30, subject to arithmetic uncertainties and byte truncation. It occurs before the last two reads and can affect either through aliasing.

Subsequent byte publications place the publication field base’s low byte at final traversal base offset 0x0D, 0x6C at publication field base offset 8, and the retained publication base’s low byte at publication field base offsets 0x0D and 9. The final traversal base’s offset-0x54 word then receives that base itself.

The saved address-0xA9 word’s low byte is written to fixed address 0x67. **A word read from final traversal base offset 0x44 intervenes before address 0x67 is read and its current byte stored at fixed address 0x21.** This is a memory reread, not an unconditional copy of the originally saved low byte.

The offset-0x44 word supplies a byte at publication field base offset 0x10 and a word at retained publication base offset 0x54. Its variable signed right shift, with the publication field base’s low byte as the count, supplies a byte at final traversal base offset 1. The retained publication base’s offset-0x54 word is then replaced with the publication field base.

A word is read from publication field base offset 0x40 before the saved offset-0x04 word is written at retained publication base offset 0x74. The offset-0x40 result is written as a word at publication field base offset 0x10, and the final traversal base is written at retained publication base offset 0x44.

**Address 0x67 is then read again**, supplying a byte store at publication field base offset 9. Between this read and the next address-0x67 read, the function:

- Publishes the publication field base’s low byte at final traversal base offset 9.
- Writes the saved address-0xA9 word at publication field base offset 0x44.
- Reads publication field base offset 0x54 to obtain another selected base.

**A further independent read of address 0x67** supplies a second byte store at publication field base offset 9. Either reread can observe a value changed by intervening computed-address stores. Neither is guaranteed to reproduce the byte stored initially at 0x67.

The selected base receives the saved payload-offset-0x74 word’s low byte at offset 0x0D. The saved final-traversal-offset-0x14 word is restored at final traversal base offset 0x54. A fresh publication-field-offset-0x14 read supplies a later byte, while further stores publish the selected base and the saved payload word’s low byte.

Finally, a word read from final traversal base offset 4 supplies a 16-bit store of 0x65 at twice that word. Reads through its offsets 0x64 and 0x74 supply a retained word and the terminal fourth argument. The selected base receives decimal 100 at offset 0x54. Another word read through final traversal base offset 0x64 selects an offset-0x54 destination for the retained word. A word store at twice the retained word immediately precedes F-U002.

No visible cleanup or rollback follows.

## Side effects

All computed destinations may alias fixed storage or other computed locations. Relevant fixed accesses include bytes at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67; words at 0x54, 0x60, and 0xA9; and the shared word supplying the retained publication base.

Endianness, volatility, atomicity, and alignment rules are not established. Consequently, the exact effect of an overlapping word store on a later byte read is unresolved.

No explicit allocation, release, reference acquisition, or ownership transfer is identifiable. Dependencies may allocate, retain, release, or invalidate memory. Saved address-bearing words do not establish continuing ownership or validity.

No explicit error result, validation path, retry, exception translation, or recovery is visible. Failure to return or a fault can prevent later effects while leaving earlier effects visible. Rollback, idempotence, thread safety, and reentrancy are not established.

## Dependencies

Opaque dependency labels are document-local and do not establish verified identities.

| ID | Established interaction | Unresolved properties |
|---|---|---|
| F-U001 | First indirect invocation; target depends on incoming return-link state plus 0xFFE. | Signature, implicit arguments, results, memory effects, termination. |
| F-U003 | Collective label for intervening unresolved indirect invocations, not a claim that they share one target. | Individual identities, complete argument/result contracts, ownership effects, failures. |
| F-U002 | Terminal represented interaction with four arguments beginning with 0x72. | Actual transfer semantics, results, side effects, normal-return behavior. |

No dependency is assumed pure, globally state-preserving, or responsible for releasing memory.

## Acceptance examples

These examples are conditional static-review criteria, not executed tests.

1. **Early scaling:** If the second argument is 1, its visible eleven-bit-shifted dependency argument is 2048. If the third argument is 1, its three-bit-shifted form is 8 and the further ten-bit-shifted form is 8192.

2. **Paired results:** A paired result with low word 1 and high word 2 supplies 256 and 2 to an interaction consuming the low word shifted by 8 and the high word unchanged.

3. **Initial fixed-byte snapshot:** If address 0x05 contains 0xFF when read after the duplicated small-base offset-0x15 stores, the saved byte-selected base is 255. The later address-0x01 write does not retroactively change that snapshot.

4. **Fixed linking writes:** If the saved link word is 7 and the small doubled base is 24, fixed word address 0x54 receives 14 at its specified position; later fixed byte address 0x15 receives 24 after the second duplicated computed offset-0x15 store.

5. **Publication snapshot:** If the shared publication-base load yields 0x1000, subsequent publication accesses continue using that saved base even if the following address-0x60 store aliases the original shared storage.

6. **Final intervening byte:** If the adjusted publication index is 2, the address-0x3A byte is 0x40 under ordinary non-overflowing arithmetic. That write precedes the final retained offset-0x14 and offset-0x04 reads.

7. **Independent rereads:** If the saved address-0xA9 word has low byte 0x12, the initial address-0x67 write contains 0x12. If intervening aliasing changes address 0x67, the address-0x21 publication and the two later offset-9 publications must use their respective reread values.

8. **Partial failure:** A fault during a later read does not require earlier stores to be undone. Reaching F-U002 requires first argument 0x72, not a particular return value.

## Uncertainties

The following remain **documented_only**:

- Domain intent and complete calling convention.
- Individual dependency identities and full interaction contracts.
- Exact signed overflow, conversion, and variable-shift semantics.
- Endianness, mapping, alignment, volatility, and concurrency behavior.
- Whether unusual addresses correspond to literal machine accesses.
- Object layouts, allocation boundaries, and ownership.
- Exact correspondence between repeated represented stores and machine instructions.
- Terminal transfer behavior and normal-return reachability.

These unknowns prevent a complete concrete input-to-output mapping. They do not permit replacing ordered accesses with guessed higher-level operations.

## Evidence

The supplied static body is the sole behavioral evidence. It represents three formal inputs, extensive indirect interactions, 32-bit and 64-bit result consumption, bitfield transformations, mixed-width memory accesses, explicit repeated publications, fixed-storage transitions, and a terminal indirect interaction.

Supplied comments report overlapping storage symbols and incomplete terminal control-flow recovery. They support caution but do not independently establish layout or execution behavior.

No dependency bodies, caller contracts, machine-instruction inspection, implementation access, or runtime observations were available. No runtime tests have been performed.
