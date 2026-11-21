<!-- Trusted provenance: F-ab5efc45bb13; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab5efc45bb13

## Purpose

F-ab5efc45bb13 invokes an ordered sequence of unresolved indirect dependencies, derives address-like and scalar values from their results and incoming machine state, mutates memory, and invokes a final unresolved operation with four displayed arguments.

No domain-level purpose is established. Constants resembling characters, repeated stores, and address-like results do not establish formatting, allocation, initialization, or a particular data structure.

This contract distinguishes established static evidence, conditional analytical consequences, and unresolved behavior marked **documented_only**. All later effects depend on earlier calls returning and earlier memory accesses completing.

## Interface

### Inputs and outputs

The presented interface declares no explicit parameters and no result value. Its implicit inputs include:

- An incoming 32-bit register word.
- Incoming continuation-like state used to derive the first indirect target.
- Dependency results represented as 32-bit words or 64-bit pairs.
- Additional register outputs outside conventional displayed return values.
- Contents of fixed and computed memory locations.

No scalar success or failure result is established. Observable outputs comprise dependency invocations, ordered memory accesses, and the final four-argument indirect operation.

### Widths, addressing, and arithmetic

Explicit accesses distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Represented 64-bit results are separated into low and high 32-bit halves. Addresses and arithmetic operands are represented as 32-bit quantities.

Arithmetic descriptions use a **32-bit bit-pattern model**: shifts to the left, multiplication, addition, and subtraction retain the low 32 bits. Successive shifts truncate separately. This model expresses the displayed dependencies; it does not establish portable signed-overflow behavior or executable address wrapping.

Byte stores retain eight bits; unsigned byte loads expand to 0–255. Halfword and word stores of small constants include zero-valued upper bits within their respective widths. Endianness and alignment requirements are **documented_only**.

One late right shift operates on a signed word with a count derived from an address’s low byte. Counts can exceed 31. Neither excessive-count handling nor executable signed-shift semantics is established.

### Preconditions and failure

Reaching the final operation requires valid indirect targets, compatible machine-level calling conventions, returning dependencies, and accessible memory for every executed access width.

No visible null check, bounds check, error status, retry, recovery, or rollback exists. Small and zero derived addresses are not rejected. A fault or non-returning dependency can leave only a prefix of the effects completed.

## Behavior

### Dependency phase

The function begins with a long, straight-line sequence of indirect invocations. The first target uses incoming continuation-like state plus 0xFFE. Later targets use embedded continuation-like values with the same displacement. No conditional choice among these invocations is displayed.

An empty displayed argument list does not establish absence of machine-level inputs. Unknown dependencies may observe registers or shared memory.

Visible argument transformations include:

| Input | Transformation | Consequence under the bit-pattern model |
|---|---|---|
| Incoming word | Logical right shift by 15 | At most 17 significant bits |
| Incoming word | Left shift by 13 | Low 13 bits cleared |
| Incoming word | Mask with 0x7FFFF, then right shift by 2 | Range 0–131071 |
| Incoming word | Mask with 0x3FFF, then left shift by 3 | Range 0–131064, in steps of eight |
| Returned word | Mask with 0xFFFFFF, then right shift by eight | Extracts bits 8–23 |
| Returned word | Mask with 0xFFFFFF, then right shift by 12 | Extracts bits 12–23 |
| Returned word | Mask with 0x1FFFF, then right shift by four | Extracts bits 4–16 |

Other arguments combine retained values, shifted low halves, unchanged or shifted high halves, and additional register outputs. Explicit zeros appear in several argument positions. Some successive shifts discard every original bit; replacing them with wider intermediate arithmetic would change the represented behavior.

Dependency identities, side effects, preservation rules, and full machine-level interfaces remain **documented_only**.

### Values entering direct mutation

The following descriptive names identify retained values, not source identifiers or established data structures:

- **Shifted address:** a dependency result shifted left by 25.
- **Scale-producing word:** a separate dependency result.
- **Scaled base:** the scale-producing word multiplied by 0x02000000.
- **Selector:** a returned word shifted left by eight, masked with 0x7FFF, then logically shifted right by six.
- **Selector base:** twice the selector.
- **Retained shifted word:** another dependency result shifted left by eight.
- **Final pair:** the low and high words of the last represented 64-bit result before direct mutation.

The shifted address and scaled base are multiples of 0x02000000 and retain seven source bits. The selector ranges from 0 to 508 in steps of four; its doubled base ranges from 0 to 1016 in steps of eight.

The initial stores occur in this order: the shifted address goes to final-pair-high plus 0x14; the high word’s low byte goes to shifted-address plus 9; the scaled base goes to 256 times final-pair-low plus 0x54; and the shifted address’s low byte replaces the byte at shifted-address plus 9. That replacement is zero under the stated model. The scaled-base word at offset 0x14 is then cleared.

Repeated or replacing stores remain distinct accesses, even when their final contents coincide.

### Clears, fixed-location input, and seed capture

After the scaled-base clear, an unsigned byte is read at scaled-base plus 5. The selector-base word at offset 0x14 is cleared, the expanded byte is stored as a word at scaled-base plus 0x34, and scaled-base offset 0x14 is reloaded.

With stable ordinary memory, that reload is zero. The intervening potentially overlapping selector-based store also writes zero, and the same-base offset-0x34 publication is disjoint from offset 0x14. Aliasing alone does not justify a nonzero result. External modification or device behavior could distinguish the accesses.

The selector-base word at offset 0x14 is cleared again. The reloaded word’s low byte is then stored twice at selector-base plus 0x15.

The fixed symbol at address 0x05 is next read and expanded into a 32-bit base. Its declaration width is unavailable. This read precedes the byte write of selector-base’s low byte to fixed address 0x01. The selector’s low byte is then written at the loaded base plus 0x15.

Before reading that base’s offset-0x54 field, the function:

- Writes the retained shifted word’s low byte at scaled-base plus 0x11.
- Writes the complete retained shifted word at the earlier reloaded word plus 0x54.
- Clears scaled-base plus 0x11.

The word subsequently read at the fixed-symbol-derived base plus 0x54 is retained as the **saved seed**. That field is then replaced with the earlier reloaded word. Seed capture follows the preceding computed stores; non-aliasing is not assumed.

### Seed-derived mutations and lookup chains

Selector-base offset 0x34 receives the selector. A word loaded at scaled-base plus 0x74 supplies an **early destination**; its offset-0x54 field receives the scale-producing word shifted left by 26.

Another expression reads selector-base offset 0x74 for both destination formation and the value stored at the resulting base plus 0x54. A single stable capture is not established, nor is an executable evaluation order between those two reads.

The saved seed is stored at saved-seed plus 0x24 twice. Between these stores, its low byte goes to early-destination plus 9 and the retained shifted word goes to early-destination plus 0x54. The byte write is repeated afterward.

Next, selector-base offset 0x14 receives the scale-producing word shifted left by 26. Fixed address 0x54 then receives a word containing twice the saved seed. Selector-base offset 0x14 receives the saved seed twice; selector-base offset 0x15 receives the low byte of four times the saved seed twice. Fixed address 0x15 then receives selector-base’s low byte.

Let **fourfold seed** and **eightfold seed** denote four and eight times the saved seed. Subsequent effects are:

- Fourfold-seed offset 0x54 receives the retained shifted word, then later receives the fourfold seed itself.
- Selector-base offset 0x34 receives the fourfold seed.
- A fresh selector-base offset-0x74 word is doubled into an **intermediate base**.
- Fourfold-seed offset 0x15 receives the eightfold seed’s low byte.
- Saved-seed offset 0x44 receives the intermediate base.

An unsigned byte at fourfold-seed plus 5 supplies a base for a word lookup at offset 0x74. Call this result the **first lookup**. Its offset-0x44 field receives the intermediate base.

Fourfold-seed offset 0x15 then receives the fourfold seed’s low byte. Its offset 0x0D receives the first lookup’s low byte, its offset 0x14 receives the fourfold seed, and its offset 0x54 receives the fourfold seed again. A word read at fourfold-seed plus 0x64 supplies another destination, whose offset 0x0D receives the first lookup’s low byte and whose offset 0x14 receives that destination itself.

A word loaded at fourfold-seed plus 4 becomes a **secondary base**. Intermediate-base offset 9 receives the first lookup’s low byte, and offset 0x44 receives the eightfold seed. The secondary-base byte at offset 5 is then read.

After that first byte read, fourfold-seed offset 9 receives the intermediate base’s low byte; first-lookup offset 0x44 receives the expanded byte; fourfold-seed offset 0x14 receives the eightfold seed; and intermediate-base offsets 0x15 and 0x44 receive, respectively, the secondary base’s low byte and the fourfold seed.

The secondary-base byte at offset 5 is read again. After a byte write of the eightfold seed at fourfold-seed offset 0x15, this second byte supplies another word lookup at offset 0x74. Call its result the **late link**.

Fourfold-seed offset 0x15 then receives the secondary base’s low byte, offset 0x54 receives the fourfold seed, offset 9 receives the doubled late link’s low byte, and offset 0x14 receives the eightfold seed.

The two secondary-base byte reads are fresh reads separated by writes. They need not agree when locations alias.

### Constants and repeated publication

Define **doubled-link base** as twice the late link and **fourfold-link base** as four times it. The function writes:

- Byte 0x3A at fourfold-link base.
- Halfword 0x003A at eight times the late link.
- Word 0x0000003A at fourfold-link base plus 0x20.

The **adjusted value** is the fourfold seed minus 0x96. The **adjusted-fourfold base** is four times that adjusted value.

At adjusted-fourfold offset 0x20, successive word stores contain 0x20, 0x2F, a freshly loaded unsigned byte from adjusted-fourfold offset 4, and 0x20 again. Under ordinary byte addressing, offset 4 is disjoint from the offset-0x20 word; those word stores alone cannot change that byte.

A word read at fixed address 0x087FBBBC supplies the **publication base**. After this load, fixed address 0x60 receives word 0x20.

The ensuing repeated region consists of ordered four-write groups:

| Position within each group | Effect |
|---|---|
| First | Fourfold-link base is stored as a word at the selected destination plus 0x54 |
| Second | Adjusted-fourfold base’s low byte is stored at publication-base plus 0x19 |
| Third | Publication base’s low byte is stored at adjusted-fourfold plus 0x11 |
| Fourth | The selected destination is stored as a word at publication-base plus 0x34 |

The first selected destination is the eightfold seed. Every subsequent group selects eight times the adjusted value. This is a fixed straight-line repetition, not established traversal or indexed iteration. Its numerical repetition count is **documented_only** in this contract; repeated accesses must not be collapsed.

### Late captures and the fixed-byte chain

After publication, the function captures a complete word from fixed address 0xA9, then a word from fourfold-link plus 0x74. Fixed address 0x3A receives the low byte of eight times the adjusted value plus 0x30. The function then captures words from doubled-link offset 0x14 and adjusted-fourfold offset 4.

The following mutations precede the fixed-byte chain: doubled-link offset 0x0D receives adjusted-fourfold’s low byte; adjusted-fourfold offset 8 receives 0x6C; its offsets 0x0D and 9 receive publication-base’s low byte; and doubled-link offset 0x54 receives the doubled-link base itself.

Fixed address 0x67 receives the captured 0xA9 word’s low byte. A word is then loaded at doubled-link offset 0x44, and a fresh byte read from 0x67 is copied to fixed address 0x21.

The loaded word’s low byte goes to adjusted-fourfold offset 0x10, and the complete word goes to publication-base offset 0x54. Doubled-link offset 1 receives the low byte of that word’s signed right shift, using adjusted-fourfold’s low byte as the count. Publication-base offset 0x54 is then replaced with adjusted-fourfold.

Next, a word is captured from adjusted-fourfold offset 0x40. Publication-base offset 0x74 receives the previously captured adjusted-fourfold offset-4 word; adjusted-fourfold offset 0x10 receives the newly captured offset-0x40 word; and publication-base offset 0x44 receives doubled-link.

A second fresh read of 0x67 feeds adjusted-fourfold offset 9. Doubled-link offset 9 then receives adjusted-fourfold’s low byte. The complete captured 0xA9 word is stored at adjusted-fourfold offset 0x44.

Only then is adjusted-fourfold offset 0x54 loaded as the **destination base**. A third fresh read of 0x67 again feeds adjusted-fourfold offset 9.

These three reads of 0x67 are not interchangeable with one retained byte. Intervening computed stores may alias 0x67.

### Final mutation and indirect operation

The destination base’s offset 0x0D first receives the low byte of the captured fourfold-link offset-0x74 word. Doubled-link offset 0x54 is restored from the earlier captured doubled-link offset-0x14 word.

The function next captures adjusted-fourfold offset 0x14. Call this retained word the **captured field**. Before the new-base lookup, three further mutations occur, in order:

- Doubled-link offset 0x11 receives the captured fourfold-link offset-0x74 word’s low byte.
- Adjusted-fourfold offset 9 receives destination-base’s low byte.
- Adjusted-fourfold offset 0x44 receives the complete destination base.

A word is then read at doubled-link offset 4 to obtain the **new base**. Halfword 0x0065 is written at twice that new base. The word at new-base offset 0x64 is read and retained as the **final second argument**.

After that read, destination-base offset 0x11 receives the captured field’s low byte. Destination-base offset 0x54 then receives word 100. Only after this store is new-base offset 0x74 read and retained as the **final fourth argument**.

This order is significant: the new-base lookup, halfword store, and offset-0x64 read precede the store of 100; the offset-0x74 read follows it.

A word subsequently read at doubled-link offset 0x64 supplies the **last destination**. Doubled-link offset 9 then receives new-base’s low byte, before the last two word stores:

- Last-destination offset 0x54 receives the retained final second argument.
- Twice the retained final second argument receives the last destination as a word.

The final indirect operation receives four displayed arguments:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | Retained new-base offset-0x64 word |
| Third | Earlier captured doubled-link offset-0x14 word |
| Fourth | Retained new-base offset-0x74 word |

The third argument is retained, not reloaded after subsequent mutations. Whether this operation returns, transfers control permanently, or dispatches elsewhere remains **documented_only**.

## Side effects

Memory effects include fixed-location accesses, computed reads and writes, overlapping byte and word fields, repeated stores, and fresh reads separated by mutations. Distinct descriptive names do not establish distinct storage.

Access order is contractual where established by the representation. A final memory image cannot substitute for the access sequence, especially under aliasing, device semantics, or partial completion.

No allocation, release, reference counting, ownership transfer, or lifetime guarantee is established. Address use does not establish ownership. Unknown dependencies may mutate globals, retain addresses, allocate, release memory, or fail to return.

No transactional boundary is visible. Failure need not undo completed writes.

## Dependencies

F-ab5efc45bb13 is the only supplied opaque function identity. No opaque identities are supplied for the indirect targets; their identities remain **documented_only** rather than receiving invented mappings.

Unresolved dependencies include the entry-state-derived invocation, continuation-derived sequence, word and 64-bit result providers, additional-register-output providers, and final four-argument operation.

No dependency is assumed pure, returning, global-preserving, or ownership-neutral.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Extraction:** Masking 0xAB123456 to 24 bits and shifting right by eight produces 0x1234.
- **Selector boundary:** A selector-producing word of 0x7F produces selector 508 and selector base 1016. Selector 511 is inconsistent with the transformation.
- **Truncation:** Shifting 0x80 left by 25 produces zero in the 32-bit model.
- **Observable replacement:** If the initial byte at shifted-address plus 9 receives 0x7B, the later store still writes zero. A two-write observer must not receive one coalesced write.
- **Fixed seed store:** For saved seed 0x12345678, fixed address 0x54 receives word 0x2468ACF0. Selector 508 causes the later fixed byte at 0x15 to receive 0xF8.
- **Stable clear:** Under stable ordinary memory, the scaled-base offset-0x14 reload is zero, including when the selector-based zero store overlaps it.
- **Fresh fixed reads:** If a computed store changes 0x67 between reads, a later read uses the changed byte rather than necessarily the original captured low byte.
- **Pre-lookup mutation:** If adjusted-fourfold offset 0x44 aliases doubled-link offset 4, the destination-base word written there precedes the new-base lookup. Under stable ordinary memory, the lookup observes that replacement.
- **Read before 100:** If new-base offset 0x64 aliases destination-base offset 0x54, its retained value is captured before the store of 100. The second final argument is not automatically 100.
- **Read after 100:** If new-base offset 0x74 aliases destination-base offset 0x54, stable ordinary memory makes the fourth final argument 100 after that store.
- **Captured byte source:** If the captured field is 0x123456AB, destination-base offset 0x11 receives 0xAB even if the original capture location changes afterward.
- **Last byte-store ordering:** The new-base byte write at doubled-link offset 9 precedes both final word stores. If it overlaps either destination, those later stores can replace it.
- **Failure:** An unmapped early destination provides no promise of final invocation or rollback.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, structures, and meanings of constants.
- Complete machine-level interface and indirect dependency contracts.
- Declaration width of the fixed symbol at 0x05.
- Endianness, alignment, executable overflow rules, and excessive-shift handling.
- Memory mappings, external mutation, device behavior, and runtime reachability.
- Numerical publication-group count as specified by this contract.
- Executable evaluation order within the expression reading selector-base offset 0x74 twice.
- Final-operation return or transfer behavior.
- Fidelity of the supplied static representation to executable behavior.

“Publication,” “seed,” “link,” and “base” describe data flow only. The final-phase stores and their ordering described above are established static effects, not unresolved behavior.

## Evidence

Evidence is limited to the supplied static representation. It supports explicit access widths, arithmetic dependencies, retained captures, fixed addresses, and the described ordering of separate accesses.

Ordinary-memory conclusions and acceptance examples are conditional analytical consequences, not runtime observations. Dependency behavior, successful completion, and external output are not established.

No runtime tests have been performed.
