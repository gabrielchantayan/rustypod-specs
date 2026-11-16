<!-- Trusted provenance: F-0dc15de6a1a1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0dc15de6a1a1

## Purpose

F-0dc15de6a1a1 accepts two 32-bit inputs, invokes a long sequence of unresolved indirect dependencies, performs ordered mutations across fixed and computed addresses, and reaches a final unresolved indirect transfer.

No domain-level purpose is established. The available evidence does not justify describing the operation as initialization, formatting, decoding, collection management, or memory management. Character-like constants and recurring offsets do not establish such meanings.

This contract distinguishes:

- **Established static evidence:** represented input relationships, access widths, arithmetic expressions, and relative ordering.
- **[INFERENCE]:** consequences dependent on an explicitly stated execution or memory model.
- **documented_only:** unresolved interfaces or behavior, without invented requirements.

Completion and safety depend on information beyond the explicit inputs.

## Interface

### Explicit inputs and output

| Item | Width | Established use |
|---|---:|---|
| First input | 32 bits | Passed unchanged to the first indirect invocation. |
| Second input | 32 bits, represented as signed | Passed unchanged and also shifted left by six positions. |
| Declared output | No value | No explicit status, error, or result object is returned. |

The first invocation receives three represented arguments, in order: the first input, the second input, and the second input shifted left six positions.

Neither input has an established pointer, handle, size, index, or ownership interpretation.

### Implicit inputs and preconditions

Execution also consumes incoming execution context, indirect-call results, secondary result state, existing memory, a separately stored global address word, and a final indirect target.

Conditional completion requires every reached target and memory access to be supported, and every preceding dependency to return. No explicit null, bounds, alignment, permissions, input-range, or dependency-result checks establish these conditions.

The explicit arguments alone cannot establish memory safety.

### Widths and storage views

Direct accesses include:

- 8-bit reads and writes.
- 16-bit writes.
- 32-bit reads and writes.
- 64-bit dependency-result containers whose low and high 32-bit portions are consumed separately.

Offsets such as 0x04, 0x05, 0x09, 0x0D, 0x11, 0x14, 0x15, 0x19, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74 are observed access offsets, not a validated object layout.

Byte and word views can overlap. In particular, offset 0x15 lies inside a 32-bit word beginning at offset 0x14. Fixed-address word accesses include an unaligned read at 0xA9. Endianness, alignment tolerance, mapping, and permissions remain **documented_only**.

## Behavior

### Dependency prefix and arithmetic

A long ordered sequence of indirect invocations precedes the first explicit data-memory store. Many targets are formed from execution-location values using displacement 0xFFE. A common displacement does not establish common target identity.

The prefix includes invocations with no represented arguments, literal-zero arguments, shifted primary results, secondary result state, and separate portions of 64-bit results. There is no represented conditional branch or loop selecting success, failure, or recovery.

The full per-site interface remains **documented_only**. In particular, an invocation without represented arguments is not established to be effect-free.

Working widths are predominantly 32 bits. Unsigned right shifts are logical. Masks retain selected low bits before subsequent shifts: retaining 24 bits before shifting left six positions produces at most 30 significant bits; retaining 22 bits before shifting left eight positions also produces at most 30.

**[INFERENCE]:** under a 32-bit machine-bit-vector model, arithmetic and left shifts retain their low 32 bits. Under that model:

- Two successive left shifts by 28 positions produce zero.
- Left shifts by 14 and then 23 positions produce zero.
- Left shifts by 31 and then 30 positions produce zero.
- A left shift by eight followed by an unsigned right shift by eight retains the original low 24 bits.

These reductions do not authorize eliminating dependency invocations.

Signed overflow and signed left shifts are not established as portable language behavior. A terminal signed right shift uses an address-derived low byte as its count. Negative operands and counts of 32 or greater remain execution-model questions; modulo-32 count masking is not established.

### Descriptive values

The following names describe relationships, not source identifiers or object types:

| Descriptive value | Relationship |
|---|---|
| Scaled seed | An earlier 32-bit dependency result retained into the mutation region. |
| Scaled base | Scaled seed multiplied by 2²⁵. |
| Shifted base | A different dependency result shifted left 25 positions. |
| Retained shifted result | Another dependency result shifted left eight positions. |
| Small index | A result shifted left eight positions, masked to 15 bits, then logically shifted right six positions. |
| Doubled index | Twice the small index. |
| Final pair | The last 64-bit dependency result before explicit stores, viewed as low and high 32-bit portions. |
| Saved link | A word saved through an early byte-derived base at offset 0x54. |

**[INFERENCE]:** under 32-bit bit-vector arithmetic, the small index ranges from 0 through 508 in multiples of four; the doubled index ranges from 0 through 1016 in multiples of eight.

All address arithmetic remains subject to the execution model. Distinct descriptive names do not establish disjoint memory.

### Initial mutations and reached fixed-address read

The first explicit word destination is the final pair’s high portion plus 0x14; it receives the shifted base. A byte at shifted base + 0x09 then receives the high portion’s low byte. Between that byte write and a second write to the same byte destination, a word at 256 times the final pair’s low portion + 0x54 receives the scaled base. The second byte write receives the shifted base’s low byte.

The word at scaled base + 0x14 is cleared before reading the byte at scaled base + 0x05. A zero word store at doubled index + 0x14 precedes publication of that zero-extended byte to scaled base + 0x34. The word at scaled base + 0x14 is then reloaded. Another zero word store at doubled index + 0x14 precedes two writes of the reloaded word’s low byte to doubled index + 0x15.

Only after these duplicated byte writes is fixed address 0x05 read as an unsigned byte and widened to a 32-bit base. Fixed address 0x01 then receives the doubled index’s low byte.

This reached read of 0x05 is not an entry-time snapshot. Earlier overlapping writes must be accounted for under the applicable memory model.

The byte-derived base + 0x15 receives the small index’s low byte. Other early effects write the retained shifted result’s low byte to scaled base + 0x11, write the full retained shifted result through the reloaded word at offset 0x54, and clear scaled base + 0x11.

The word at byte-derived base + 0x54 is then saved as the saved link before that location receives the earlier reload from scaled base + 0x14. Doubled index + 0x34 receives the small index.

**[INFERENCE]:** the cleared word at scaled base + 0x14 reloads as zero only for ordinary memory without relevant overlapping or external mutation. In that case, the store through the reload reaches fixed address 0x54. This conditional store is distinct from the later explicit fixed-address 0x54 mutation.

### Saved-link mutations and explicit fixed-address writes

A word read from scaled base + 0x74 supplies another destination base. Its offset-0x54 word first receives the scaled seed shifted left 26 positions. Additional effects use words read at doubled index + 0x74 as address ingredients, write the saved link into saved link + 0x24 twice, and write its low byte to the newly loaded destination’s offset 0x09 twice. The retained shifted result also replaces that destination’s offset-0x54 word.

The following mutation cluster has an explicit, significant order:

- The 32-bit word at doubled index + 0x14 receives the scaled seed shifted left 26 positions.
- **Fixed address 0x54 receives a 32-bit write of the saved link shifted left one position.**
- The word at doubled index + 0x14 receives the saved link twice.
- The byte at doubled index + 0x15 receives the low byte of four times the saved link twice.
- **Fixed address 0x15 receives an 8-bit write of the doubled index’s low byte.**

The fixed-address 0x54 write therefore occurs after the shifted dependency-result write and before both saved-link word writes. It is not merely a location reached conditionally through an address loaded from memory.

The fixed-address 0x15 write occurs after both four-times-saved-link byte writes and before the subsequent indirect word stores. Those following stores write the retained shifted result at four times the saved link + 0x54 and four times the saved link at doubled index + 0x34.

Neither fixed mutation may be discarded or moved on the assumption that its storage is unrelated to computed destinations. For example, doubled index + 0x14 can itself equal 0x54, and doubled index + 0x15 can equal 0x15.

### Linked region and constant-width effects

Four times the saved link becomes a recurring base. Subsequent effects combine:

- Word destinations reached through offsets 0x54, 0x64, and 0x74.
- Repeated reads at the same computed address.
- Bytes read at offset 0x05, widened, and used as bases for offset-0x74 word reads.
- Address-derived values written into their addressed regions.
- Repeated byte writes interleaved with word writes.

A word read at doubled index + 0x74, multiplied by two, supplies an intermediate base. Through the four-times-saved-link region, another byte-derived offset-0x74 read supplies an earlier linked value. Its low byte is propagated to offset-0x0D and offset-0x09 destinations. Word writes establish self-address-derived contents at offsets 0x14 and 0x54.

A further base comes from the word at four times the saved link + 0x04. Its offset-0x05 byte is read twice, with intervening writes. The later byte read supplies another offset-0x74 word read. That word is the **linked value** used below; twice it is the **companion base**.

Repeated reads are distinct observations. Intervening byte and word writes may alias their storage.

Three writes of 0x3A follow: an 8-bit write at four times the linked value, a 16-bit write at eight times the linked value, and a 32-bit write at four times the linked value + 0x20. These are width-specific memory effects, not established character emissions.

The **adjusted value** is four times the saved link minus 0x96. The **publication base** is four times the adjusted value. No lower-bound check protects the subtraction.

Publication base + 0x20 receives successive 32-bit writes of 0x20, 0x2F, a zero-extended byte read from publication base + 0x04, and 0x20 again. The byte read occurs after the first two word writes and observes any applicable aliases.

### Captured global base and repeated publication

After the final 0x20 write above, a separate global address word is loaded and retained as the **captured global base**. Fixed address 0x60 then receives a 32-bit write of 0x20. Publication stores follow.

The load precedes the fixed-address store. A later reload is not equivalent without a preservation and aliasing argument.

Each publication group has four ordered effects:

| Effect | Width |
|---|---:|
| Store four times the linked value at derived base + 0x54 | 32 bits |
| Store publication base’s low byte at captured global base + 0x19 | 8 bits |
| Store captured global base’s low byte at publication base + 0x11 | 8 bits |
| Store the derived base at captured global base + 0x34 | 32 bits |

The initial derived base is eight times the saved link. Subsequent groups use eight times the adjusted value.

The repetitions are finite and explicitly represented, not controlled by a represented loop condition. This contract does not enumerate their exact cardinality; that detail remains **documented_only** here. Repetition cannot be collapsed merely because successive groups use equal values.

### Terminal snapshots and fresh global-byte reads

The terminal region first saves the word at fixed address 0xA9, then the word at four times the linked value + 0x74.

Fixed address 0x3A next receives the low eight bits of eight times the adjusted value plus 0x30. Words at companion base + 0x14 and publication base + 0x04 are saved afterward, before the following indirect overwrites.

Subsequent mutations write publication base’s low byte at companion base + 0x0D; write 0x6C at publication base + 0x08; write captured global base’s low byte at publication base + 0x0D and +0x09; and write the companion base at companion base + 0x54.

Fixed address 0x67 receives the saved 0xA9 word’s low byte. A word read at companion base + 0x44 intervenes before a fresh byte read of 0x67 supplies fixed address 0x21. The latter write is not established to equal the earlier saved byte.

The intervening word supplies:

- Its low byte to publication base + 0x10.
- Its full value to captured global base + 0x54.
- A signed-right-shift result, narrowed to one byte, to companion base + 0x01.

The shift count is publication base’s low byte. Publication base then replaces the word at captured global base + 0x54.

A word read at publication base + 0x40 supplies a later overwrite of publication base + 0x10. Between that read and overwrite, the saved publication-base + 0x04 word is written to captured global base + 0x74. Captured global base + 0x44 receives the companion base.

A second fresh read of 0x67 supplies publication base + 0x09. Companion base + 0x09 then receives publication base’s low byte, and publication base + 0x44 receives the full saved 0xA9 word. A word is loaded from publication base + 0x54. A third fresh read of 0x67 then supplies another write to publication base + 0x09.

These reads cannot be replaced with a cached value: intervening indirect writes may overlap 0x67.

### Terminal links and final transfer

The word loaded from publication base + 0x54 becomes the **terminal destination base**. Its offset-0x0D byte receives the low byte of the saved four-times-linked-value + 0x74 word. Companion base + 0x54 receives the earlier saved companion-base + 0x14 word.

After saving publication base + 0x14, the operation writes the saved linked-region word’s low byte to companion base + 0x11. Publication base + 0x09 receives the terminal destination base’s low byte, and publication base + 0x44 receives its full value.

A word read at companion base + 0x04 supplies a **terminal lookup base**. A 16-bit write of 0x65 at twice that base precedes the word read at terminal lookup base + 0x64.

The terminal destination receives the saved publication-base + 0x14 word’s low byte at offset 0x11 and decimal 100 as a 32-bit word at offset 0x54.

The remaining word reads obtain terminal lookup base + 0x74 and companion base + 0x64. After companion base + 0x09 receives terminal lookup base’s low byte, the latter loaded address receives the saved terminal-lookup-base + 0x64 word at offset 0x54. The final word store writes that loaded address at twice the saved offset-0x64 value.

The final indirect transfer receives four represented arguments:

1. 0x72.
2. The saved terminal-lookup-base + 0x64 word.
3. The earlier saved companion-base + 0x14 word.
4. The recently read terminal-lookup-base + 0x74 word.

No further explicit effect follows except the represented return. Actual return reachability remains **documented_only**.

## Side effects

Fixed-address effects include byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67; word writes at 0x54 and 0x60; a byte read at 0x05; repeated byte reads at 0x67; and a word read at 0xA9.

The explicit 0x54 write stores the saved link shifted left one position. The explicit 0x15 write stores the doubled index’s low byte. Computed accesses may independently reach these same locations or overlap them.

Saved snapshots remain distinct from later fresh reads. Byte, halfword, and word effects retain their respective widths; equal values do not merge separate observable writes.

Unknown dependencies may change memory, globals, devices, execution state, ownership, or lifetimes. No preservation guarantee is available.

No explicit allocation, release, retention, or ownership transfer is established. Access does not establish ownership.

There is no represented error result, rollback, cleanup branch, or recovery path. An unsupported access, invalid target, or non-returning dependency may stop execution after earlier effects. Atomicity, synchronization, reentrancy, and idempotence are not established.

## Dependencies

Only F-0dc15de6a1a1 has a supplied opaque function ID. IDs for indirect dependencies remain **documented_only**; no additional identities are assigned.

| Boundary | Established interaction | Unresolved contract |
|---|---|---|
| First indirect invocation | Three input-derived arguments | Target ID, ABI, effects, results |
| Remaining prefix invocations | Ordered consumption of shifted, masked, split, secondary, and zero values | Per-site identities, preservation, failures, ownership |
| Final indirect transfer | Four arguments beginning with 0x72 | Target ID, effects, return behavior |

Distinct invocation sites must not be conflated without independent evidence.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input shift.** With second input 3, the first dependency receives 3 unchanged and 192 as the shifted argument.

2. **Width-dependent truncation.** With second-input bit pattern 0x04000000, the six-position left shift produces zero under 32-bit bit-vector arithmetic. No saturation is represented.

3. **Reached fixed-address read.** If byte 0x05 equals 0x7E when reached, the byte-derived base is 0x7E. Its offset-0x15 byte destination is 0x93, and its offset-0x54 word location is 0xD2. An entry-time byte cannot replace this read.

4. **Explicit fixed word mutation and aliasing.** If the small index is 32, doubled index + 0x14 equals 0x54. With saved link 0x11, that location receives the shifted scaled-seed value, then the distinct fixed-address write of 0x22, then two writes of 0x11. Later byte writes may further modify the word’s storage.

5. **Explicit fixed byte mutation.** If the small index is zero and the saved link is 0x11, doubled index + 0x15 equals 0x15. The two byte writes there store 0x44; the following fixed-address byte write stores zero. Subsequent indirect stores occur afterward. Retaining only 0x44 would be incorrect.

6. **Captured global base.** Publication uses the address word captured before the 0x60 store, even if interacting storage changes afterward.

7. **Fresh reads of 0x67.** If the saved 0xA9 low byte is 0x12 but an intervening aliasing write changes 0x67 to 0x34, a subsequent reached read supplies 0x34, not cached 0x12.

8. **Partial completion and terminal snapshots.** A non-returning prefix dependency prevents subsequent actions without undoing earlier effects. If the terminal transfer is reached, its third argument is the saved companion-base + 0x14 word, not necessarily that location’s current contents.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and meaningful input constraints.
- Complete dependency identities, ABI contracts, and secondary-result associations.
- Whether 64-bit result containers represent scalars or paired machine results.
- Address width, execution-state conventions, endianness, alignment, and mapping.
- Signed overflow, signed shifts, and oversized shift-count behavior.
- Concrete aliases, external mutation, device semantics, ownership, and lifetimes.
- Exact publication cardinality, not enumerated in this contract.
- Intended meanings of character-like constants.
- Final target and normal-return reachability.

These gaps do not authorize validation, repaired addresses, suppressed accesses, reordered fixed-address mutations, or cached replacements for fresh reads.

## Evidence

The supplied static representation is the sole behavioral evidence. Prior prose, reviewer feedback, and embedded comments are not independent proof.

The fixed-address 0x54 and 0x15 mutations are established by their represented destination, width, value relationship, and position among surrounding indirect writes. Their concrete alias consequences depend on reached addresses and the memory model.

Other established claims derive from represented widths, constants, dependencies, and access order. Arithmetic consequences requiring an execution model are labeled **[INFERENCE]**.

No dependency implementations, independent ABI definitions, execution traces, or memory-layout specifications were available. No runtime tests have been performed.
