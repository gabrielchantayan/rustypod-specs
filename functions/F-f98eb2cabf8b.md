<!-- Trusted provenance: F-f98eb2cabf8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f98eb2cabf8b

## Purpose

F-f98eb2cabf8b invokes a long sequence of indirect dependencies, transforms inherited and returned machine-state values, mutates memory through fixed and calculated addresses, and finishes with an unresolved indirect invocation.

The established behavior is an ordered, effectful machine-state operation. The supplied representation does not establish an application-level purpose, conventional object model, or standalone calling convention. Interpreting the operation as formatting, initialization, serialization, or linked-object manipulation would require additional evidence.

This contract distinguishes:

- **Established static evidence:** explicitly represented accesses, transformations, invocation arguments, and their ordering.
- **[INFERENCE]:** consequences requiring stated arithmetic or memory assumptions.
- **documented_only:** behavior whose detailed contract remains unresolved or is not exhaustively specified here.

No runtime tests have been performed.

## Interface

### Inputs and outputs

The displayed interface has no explicit parameters or return value. Its effective inputs nevertheless include:

- Two inherited machine-state words used in shifts, masks, and dependency arguments.
- An inherited return-link value used to calculate the first indirect target.
- Primary, paired, and secondary machine-state results observed after dependency invocations.
- Memory read through fixed addresses and addresses calculated from those results.

Arbitrary entry-state values are not established as valid inputs.

The final indirect invocation has four explicit word arguments. A return is represented afterward, but whether that target returns normally, consumes additional machine state, or transfers control elsewhere is **documented_only**. No conventional success value or error result is established.

### Widths and arithmetic

| Surface | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access, address, and most arguments | 32 bits |
| Paired dependency result | 64 bits |

A paired result is separated numerically into low and high 32-bit words. This separation does not establish memory byte order.

Byte stores retain the low eight bits; halfword stores retain the low sixteen. Explicit unsigned byte reads widen to values from 0 through 255.

Unsigned right shifts are logical. Other expressions involve signed words, left shifts, multiplication, and a variable signed right shift. Target overflow, signed-shift behavior, and oversized-count behavior remain **documented_only**. Expressions must not silently acquire unlimited precision.

### Memory and ownership preconditions

Continuation requires each reached indirect target to be executable and each reached memory access to be permitted. No visible validation establishes these conditions.

There is no established allocation, release, lifetime, ownership-transfer, buffer-length, or resource-acquisition contract. A loaded word subsequently used as an address is not proof of an owned object. Unknown dependencies must not be assumed to preserve globals, leave reachable memory unchanged, or allocate or release memory.

Calculated bases may overlap one another and fixed-address storage. Alignment, memory-map permissions, device behavior, concurrency, and access atomicity remain **documented_only**.

## Behavior

### Indirect-invocation phase

The operation begins with an indirect invocation whose target is derived from the inherited return-link value by adding 4094. Subsequent targets are predominantly derived from embedded address-like values using the same displacement.

The visible progression contains no explicit loop, status-based early exit, retry, rollback, or validation branch. Each later invocation depends on earlier invocations returning. Dependencies themselves may branch, fault, modify memory, or prevent continuation.

Arguments include inherited words, zero constants, single-word results, low and high portions of paired results, and values retained across intervening invocations. Transformations include:

- Left-shifting low result words by eight bits.
- Forwarding high result words unchanged or shifted.
- Passing the same transformed value in multiple positions.
- Extracting masked fields before further shifts.
- Logical right shifts of unsigned words.
- Further shifts of values already narrowed by earlier word-width operations.

Mask placement is significant. Retaining 24 low bits before shifting is not interchangeable with retaining a field after a previous shift.

**[INFERENCE]** Under 32-bit bit-vector semantics, shifting a word left by 25 and then shifting that result left by seven yields zero. Wider arithmetic that retains discarded bits would not reproduce that consequence.

The exhaustive invocation trace, dependency identities, formal signatures, and secondary-result conventions are **documented_only** in this contract. Similar argument shapes do not establish identical targets.

### Roles used for the memory phase

These names describe numeric roles, not recovered objects:

| Role | Meaning |
|---|---|
| Retained high-scale word | An earlier paired result’s low word shifted left by 25 |
| Shift-derived base | A later single-word result multiplied by 33,554,432 |
| Small field | A word shifted left by eight, masked to its low 15 bits, then logically shifted right by six |
| Small base | Twice the small field |
| Saved shifted word | A later single-word result shifted left by eight |
| Loaded link | Word read at offset 84 from the base obtained by widening the byte at address 5 |
| Fourfold link base | Four times the loaded link |
| Adjusted base | Fourfold link base minus 150 |
| Publication base | Four times the adjusted base |
| External base | Word read from separate fixed storage at address 0x087FBBBC |

The small field is bounded by the mask and shift. Under ordinary word-width evaluation it ranges from 0 through 508 in multiples of four; the small base ranges from 0 through 1016 in multiples of eight.

All scaling and address calculations remain subject to the arithmetic qualifications above.

### Initial mutations and early fixed-address effects

After the last paired-result invocation, direct memory effects begin without an intervening visible call.

The first cluster stores the retained high-scale word at offset 20 from the latest high result word. It writes that high word’s low byte at offset 9 from the retained high-scale word, stores the shift-derived base at offset 84 from the latest low result word scaled by 256, and overwrites the earlier offset-9 byte with the retained high-scale word’s low byte.

The shift-derived base’s offset-20 word is then cleared. An unsigned byte is read at its offset 5. A clear at small-base offset 20 precedes publication of that widened byte as a word at shift-derived-base offset 52. The shift-derived-base offset-20 word is read; another small-base offset-20 clear follows. The read word’s low byte is then stored twice at small-base offset 21.

**[INFERENCE]** With ordinary stable memory and no external mutation, the read following the clear yields zero. The represented operation does not suppress subsequent accesses using that value as an address base.

The next fixed-address and derived effects have this relative order:

- The byte at address **5** is read and widened.
- The low byte of the small base is written at address **1**.
- The small field’s low byte is written at offset 21 from the widened byte-derived base.
- The saved shifted word’s low byte is written at shift-derived-base offset 17.
- The saved shifted word is stored at offset 84 from the previously read offset-20 word.
- Shift-derived-base offset 17 is cleared with a byte store.
- The loaded link is read at offset 84 from the byte-derived base; that location is then overwritten with the previously read offset-20 word.

Thus the loaded link is captured after the address-1 write and the intervening derived stores, not from an entry-time snapshot.

Further updates write small-base offset 52, follow offset-116 words, and write offsets 84, 36, and 9 through resulting bases. Some word and byte stores occur twice.

After a transformed dependency value is written at small-base offset 20, a word store at fixed address **84** publishes **twice the loaded link**. Small-base offset 20 is subsequently written twice with the loaded link. Two byte stores of the fourfold link base’s low byte at small-base offset 21 precede the fixed byte write at address **21**, which contains the **small base’s low byte**.

The fixed word at address 84 must not be conflated with the earlier calculated offset-84 store. Their addresses can coincide under particular states, but they are separate represented accesses at different points.

### Scaled-link updates and literal stores

The following region uses twofold, fourfold, and eightfold scalings of loaded words. It writes byte fields at offsets 9, 13, and 21 and words at offsets 20, 52, 68, and 84. It follows words at offsets 4, 100, and 116, including an offset-116 word reached through an unsigned byte read at offset 5.

Repeated byte reads are evaluated against memory at their respective positions. Intervening stores can change later pointer-following behavior.

A word obtained through the later byte-mediated offset-116 lookup determines three destinations for the numeric value 58:

- A byte store at four times that word.
- A halfword store at eight times that word.
- A word store at offset 32 from its fourfold-scaled address.

For the remainder of this contract:

- **Literal-link word** means that later byte-mediated offset-116 result.
- **Working base** means twice the literal-link word.
- **Publication value** means four times the literal-link word.

At publication-base offset 32, the ordered overwrites are a word containing 32, a word containing 47, a word containing the unsigned byte freshly read at publication-base offset 4, and another word containing 32. The byte read is after the first two stores.

### External-base load and repeated publication

The external base is loaded after the offset-32 overwrite cluster. A word containing **32** is then written at fixed address **96**, before the first repeated publication store.

This placement matters: the address-96 store can overlap calculated destinations, while the external base has already been captured.

Each publication group has the same access-width and destination ordering:

1. A word store at an eightfold-scaled base plus 84, containing the publication value.
2. A byte store at external-base offset 25, containing the publication base’s low byte.
3. A byte store at publication-base offset 17, containing the external base’s low byte.
4. A word store at external-base offset 52, containing the eightfold-scaled base.

The first group uses eight times the loaded link. Later groups use eight times the adjusted base. The external base is not reloaded between groups.

The representation contains repeated groups, not an explicit termination-controlled loop. Their exact count is **documented_only** here. Their writes must not be collapsed to final-state assignments in an ordering-sensitive interpretation.

### Final fixed-address reads and address-58 write

After repeated publication, the final cluster begins with this established ordering:

- A word is read at fixed address **169** and saved.
- A word is read at offset 116 from the **publication value** and saved.
- A byte is written at fixed address **58**.
- The word at working-base offset **20** is read and saved.
- The word at publication-base offset **4** is read and saved.

The address-58 byte is the low eight bits of **eight times the adjusted base plus 48**. It is not the literal byte 58.

The address-58 write precedes both saved derived-word reads. If its byte overlaps either word, those reads observe memory after that write, subject to memory semantics. Describing all four saved words as a simultaneous snapshot is incompatible with this ordering.

### Final byte copying, rereads, and derived updates

The next group writes the publication base’s low byte at working-base offset 13, literal byte 108 at publication-base offset 8, and the external base’s low byte at publication-base offsets 13 and 9. A word containing the working base is stored at its offset 84.

Only then is the low byte of the saved address-169 word written to fixed address **103**.

The working-base offset-68 word is subsequently loaded. The byte at address **103** is then read to populate fixed address **33**. The address-33 value therefore comes from that byte read, not directly from the saved address-169 word.

The loaded offset-68 word supplies a byte store at publication-base offset 16 and a word store at external-base offset 84. Its signed right-shift result, narrowed to a byte, is written at working-base offset 1; the shift count is the publication base’s low byte. External-base offset 84 is then overwritten with the publication base.

The publication-base offset-64 word is loaded. The saved publication-base offset-4 word is stored at external-base offset 116; the newly loaded offset-64 word is stored at publication-base offset 16; and the working base is stored at external-base offset 68.

The next address-103 reread supplies a byte store at publication-base offset 9. A byte store at working-base offset 9 and a word store of the saved address-169 word at publication-base offset 68 intervene before a word is loaded from publication-base offset 84. Another fresh address-103 reread then supplies another publication-base offset-9 byte store.

There are **three explicit reads of address 103 in this final cluster**: one for address 33 and two for publication-base offset 9. A subsequent publication-base offset-9 byte store instead uses the low byte of the word loaded from publication-base offset 84. It is not another address-103 reread.

Consequently, neither the address-33 store nor the two later reread-fed stores are unconditional copies of the original saved word’s low byte. Intervening derived stores may overlap address 103; the read used to populate address 33 also precedes a potentially overlapping write to address 33.

### Final word following and transfer

The word loaded from publication-base offset 84 becomes another destination base. Its offset 13 receives the low byte of the saved publication-value offset-116 word.

Working-base offset 84 receives the word saved earlier from **working-base offset 20**. This is not established as restoring the former contents of offset 84.

A fresh publication-base offset-20 word is then read. Further stores update working-base offset 17 and publication-base offsets 9 and 68.

The word at working-base offset 4 is next followed. A halfword containing 101 is written at twice that loaded word. Its offset-100 word is loaded and retained for the final invocation. Subsequent effects include:

- A byte derived from the freshly read publication-base offset-20 word.
- A word containing 100 at offset 84 from the base loaded through publication-base offset 84.
- A word read at offset 116 from the base obtained through working-base offset 4.
- A word read at working-base offset 100.
- A byte write at working-base offset 9.
- A word store at offset 84 from the working-base offset-100 result.
- A word store at twice the retained offset-100 result, containing the working-base offset-100 result.

The final indirect invocation receives, in order:

1. Word value **114**.
2. The retained offset-100 word reached through working-base offset 4.
3. The word saved earlier from working-base offset 20, after the address-58 write.
4. The late offset-116 word reached through working-base offset 4.

No success check or cleanup is visible afterward.

## Side effects

Established effects include indirect invocations, byte/halfword/word stores, fixed-address accesses, and unresolved control transfer.

The layout surface includes byte reads at offsets 4 and 5; word reads at offsets 4, 20, 64, 68, 84, 100, and 116; byte writes at offsets 1, 8, 9, 13, 16, 17, 21, and 25; and word writes at offsets 16, 20, 32, 36, 52, 68, 84, and 116. Scaled-address stores also occur. These offsets do not define one contiguous object.

Meaningful fixed storage includes bytes at addresses 1, 5, 21, 33, 58, and 103, and words at addresses 84, 96, and 169. These accesses participate in the ordering described above.

No transaction or rollback is represented. If a dependency fails to return or an access faults, earlier effects may remain. Fault delivery, recovery, atomicity, and external observation rules are **documented_only**.

## Dependencies

The supplied identity mapping establishes only **F-f98eb2cabf8b**. No opaque IDs are supplied for the indirect dependencies; this contract does not invent them.

| Dependency surface | Established contract | Unresolved |
|---|---|---|
| Entry indirect target | Derived from inherited return-link state | Identity, formal inputs, effects |
| Long invocation sequence | Ordered invocations consuming transformed machine-state values | IDs, signatures, preservation, ownership, errors |
| Final indirect target | Four explicit word arguments beginning with 114 | Identity, additional inputs, effects, return behavior |

A displayed invocation without explicit arguments does not establish that its target consumes no inherited state.

## Acceptance examples

These examples are conditional static consequences, not executed tests.

- **Paired separation:** If a paired result has high word 0x12345678 and low word 0xAB, a forwarding site preserving the high word and shifting the low word by eight supplies 0x12345678 and 0xAB00 in their represented argument positions.

- **Narrowing:** A byte store supplied with 0x12345678 writes 0x78. An unsigned byte read of 0xFE widens to 254.

- **Repeated shifts:** Under 32-bit bit-vector semantics, shifting 1 left by 25 and then seven produces zero.

- **Early fixed effects:** A trace placing the address-1 write before the address-5 read is incompatible. The fixed word at address 84 contains twice the captured loaded link at its store point; later aliased writes may change it.

- **Publication prelude:** The external-base read precedes the address-96 word store of 32, which precedes the first publication group. Moving the fixed store after that group changes the trace.

- **Address-58 dependency:** If the adjusted base is 3, the address-58 byte is 72. If that destination overlaps a later saved input word, the input is read after the byte write, not before it.

- **Address-103 rereads:** If the saved address-169 word has low byte 0xA5, address 103 initially receives 0xA5. If an intervening derived store changes that byte to 0x7E before a later reread, that reread supplies 0x7E. The original saved word remains independently available for its later word store.

- **Interruption:** If an early dependency never returns, later memory effects and the final invocation are not reached. No synthetic completion or rollback is required.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid callers, and inherited-state calling convention.
- Dependency identities, formal signatures, result channels, preservation, and ownership.
- Exhaustive invocation trace and exact publication repetition count in this contract.
- Signed overflow, address wraparound, signed shifts, and oversized shift counts.
- Byte order, alignment, memory permissions, devices, alias relationships, and object boundaries.
- Fault handling, concurrency, reentrancy, atomicity, and final-target return behavior.

The variable right shift uses a signed word and a count derived from an eight-bit value, potentially exceeding 31. No masking or rejection rule is established.

**[INFERENCE]** Low-address accesses and repetitive mixed-width mutations make conventional object-level interpretations unreliable. They do not justify removing accesses or substituting a guessed application algorithm.

## Evidence

This contract is based on the supplied static operation representation and identity mapping. Embedded comments, prior prose, and reviewer feedback are not independent proof.

The strongest evidence comprises explicit access widths, numeric paired-result separation, ordered overwrites, fixed-address operations, fresh address-103 reads, the address-58 write preceding saved input loads, and the final invocation’s four arguments.

The representation supports three address-103 reads in the final cluster, but only two feed later publication-base offset-9 stores. The subsequent store there has a different loaded-word source; treating it as a third reread-fed store would change the established data dependency.

No implementation access, tools, or runtime tests were used. Executable validity, reachable safe entry states, and a complete replacement implementation are not established.
