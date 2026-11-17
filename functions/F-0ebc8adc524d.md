<!-- Trusted provenance: F-0ebc8adc524d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ebc8adc524d

## Purpose

F-0ebc8adc524d invokes an extended sequence of indirect dependencies, transforms their results, performs ordered mixed-width memory updates, and ends with an unresolved indirect invocation.

No application-level purpose is established. Address-bearing stores may suggest linked state, but describing the function as initialization, allocation, rendering, serialization, or container manipulation would exceed the evidence.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented in the supplied material.
- **Conditional consequences:** results that follow under a stated width, memory, or execution model.
- **documented_only:** unresolved behavior for which no executable acceptance rule is established.

The observable contract is not merely a final memory image. Repeated writes, intermediate reads, and possible aliasing between computed destinations and fixed globals affect behavior.

No runtime tests have been performed.

## Interface

### Entry, implicit inputs, and completion

The represented interface has no explicit parameters and no explicit return value. Its implicit inputs include:

- An incoming control-address value used to construct the first indirect target by adding 0xFFE.
- An incoming 32-bit register value used in an early argument shifted left by 25 bits.
- Primary and secondary register results supplied by dependencies.
- Shared state read through fixed and computed addresses.
- A fixed shared word later captured as a publication base.
- An unresolved final control target.

Several represented dependency results contain 64 bits, with the low and high 32-bit portions consumed independently. Other later arguments depend on secondary register results not presented as ordinary primary returns. Those secondary values must not be assumed zero or preserved across unrelated calls.

Normal completion requires every reached dependency and memory access to complete, and the final target to return. A return is represented after the final invocation, but no success code, failure code, or returned ownership-bearing value is established.

### Widths and layout

| Quantity | Represented width | Interpretation |
|---|---:|---|
| Byte access | 8 bits | A byte store retains the low eight bits |
| Halfword store | 16 bits | Writes two bytes, with endianness unresolved |
| Word access | 32 bits | Used for values and address-bearing state |
| Combined dependency result | 64 bits | Supplies independently extracted portions |
| Address calculation | Apparently 32 bits | Actual pointer and overflow semantics remain unresolved |

Unsigned byte reads explicitly converted to words are zero-extended. A byte value of 0xE1 therefore becomes 0x000000E1.

The 64-bit result representation does not establish an eight-byte object in memory, a serialized layout, or an ownership convention.

Computed offsets do not define a uniform structure. In particular, a computed-base access at offset 0x01 is a **byte destination only**. The separate byte write to absolute address 0x01 is not a field access at that offset.

### Preconditions

Fault-free completion requires:

- Executable indirect targets under the actual calling convention.
- Readable and writable effective addresses for each represented access width.
- Any required alignment.
- Dependencies that return when subsequent behavior is expected.
- A memory environment supporting the represented low-address accesses.

No visible checks enforce these conditions. No buffer extent, object size, ownership parameter, or validated address range is established.

## Behavior

### Indirect preparation

An extended, ordered chain of indirect invocations precedes the sustained explicit memory-update phase. Most targets are constructed from code-related anchors plus 0xFFE; the first uses the incoming control-address value instead.

The represented invocations have zero through four explicit arguments. Argument sources include earlier primary results, high and low portions of combined results, secondary register results, saved values, the incoming register value, and explicit zeros.

Argument position is significant. Equal shifted values passed in two positions are two arguments, not one interchangeable value. Calls with unused primary results remain observable dependency invocations; their effects are unknown.

No conditional branch, retry, or early error return is represented in this preparation sequence. This does not constrain internal dependency behavior or guarantee that a dependency returns.

The exact per-call invocation ledger is **documented_only** in this contract. No semantic replacement for the preparation chain is established.

### Arithmetic boundaries

Under a 32-bit fixed-width interpretation:

- Left shifts discard bits exceeding the word width.
- Sequential shifts truncate at their represented word-sized intermediates.
- Unsigned right shifts insert zeros.
- Masks select only their stated low-order bits: 0xFFFFFF selects 24 bits, 0x3FFFFF selects 22, 0xFFFFF selects 20, and 0x7FFFF selects 19.
- Masking to 20 bits and shifting right by 10 yields a value from 0 through 1023.
- Extracting a 64-bit result after shifts of 34, 36, or 44 bits cannot be replaced by shifting its low word.

A late bounded index is derived from a word shifted left by eight, masked to 15 bits, then shifted right by six. Under the fixed-width model, the index equals four times the producer’s low seven bits. Its range is 0–508 in multiples of four; its doubled value ranges from 0–1016 in multiples of eight.

Signed-looking shifts, multiplication, subtraction, and address addition may exceed their apparent signed ranges. Original machine semantics are not established independently. Modular arithmetic examples below are conditional, not claims of verified execution semantics. Checked arithmetic and saturation are not represented.

### Opening memory updates and the first fixed-global capture

For clarity, the **saved scaled base** means the earlier dependency result multiplied by 0x02000000. It is distinct from the final combined result’s low word multiplied by 256.

The opening updates preserve the following relationships and ordering:

- The high word of the final combined result supplies a base for a word store at offset 0x14.
- A saved shifted value supplies a base for two byte writes at offset 0x09. Between those writes, the saved scaled base is stored at offset 0x54 from the final low word multiplied by 256.
- The saved scaled base’s offset-0x14 word is cleared before its offset-0x05 unsigned byte is read.
- An index-based offset-0x14 word is cleared before that captured byte is published, zero-extended, at offset 0x34 from the saved scaled base.
- The saved scaled base’s offset-0x14 word is then read. Another index-based clear follows, then two consecutive byte writes of the captured word’s low byte to offset 0x15 from the doubled index.

Only **after those duplicated index-based byte writes** is the fixed scalar at absolute address 0x05 captured and converted to a 32-bit unsigned value. That captured value subsequently supplies a computed base.

The fixed byte write to absolute address 0x01 occurs **after** this capture and receives the doubled index’s low byte. Further computed writes then use the captured address-0x05 value, including a byte destination at its offset 0x15 and a word source/destination at its offset 0x54.

This timing matters even without intervening calls: preceding computed writes may alias absolute address 0x05. The value is not an entry snapshot. Conversely, later mutation of address 0x05 does not replace the already captured value.

The scalar’s underlying declaration is absent from the supplied material. Its conversion to a 32-bit value is established; the exact storage width of that fixed-global read remains **documented_only**.

### Address-bearing updates and access layout

The next region repeatedly loads words and uses their values as subsequent bases. It also uses unsigned bytes as address inputs.

| Computed offset | Represented accesses |
|---|---|
| 0x01 | Byte store only |
| 0x04 | Word reads; a separate byte read also occurs |
| 0x05 | Unsigned byte reads |
| 0x08, 0x09, 0x0D | Byte stores |
| 0x10 | Byte and word stores |
| 0x11, 0x15, 0x19 | Byte stores |
| 0x14 | Word reads, clears, and replacements |
| 0x20 | Word stores |
| 0x24 | Word self-reference stores |
| 0x34 | Word stores |
| 0x40 | Word read |
| 0x44, 0x54 | Word reads and stores |
| 0x64, 0x74 | Word reads |

Offsets are byte displacements. This table does not imply disjoint fields, common object ownership, or a single valid structure layout.

Significant state transitions include:

- Capturing an offset-0x54 word before replacing it.
- Loading an offset-0x74 word and using it as a base for offset-0x54 updates.
- Storing address-derived self-references at offset 0x24.
- Repeating equal byte writes and equal word writes.
- Replacing the same offset-0x54 destination with differently derived values.
- Reading an unsigned byte at offset 0x05 and using that byte as a base for an offset-0x74 word read.
- Repeating an offset-0x05 byte read after intervening stores rather than unconditionally retaining its earlier value.

One compound update refers to the same offset-0x74 source both when selecting the destination base and when selecting the stored value. Exact machine read multiplicity and expression evaluation order are **documented_only**.

Two explicit fixed updates occur in this region: a 32-bit store to absolute address 0x54 and a byte store to absolute address 0x15. They remain distinct from computed fields with those offsets.

### Constants, shared-base capture, and repeated publication

A retained word from the earlier offset-0x54 capture is scaled by four, then reduced by decimal 150. Call that result the **adjusted value**. Its fourfold scaling supplies the later **working base**.

A separately loaded word supplies a **tail base** equal to twice that word and a **payload base** equal to four times that word. These are descriptive roles, not established object types.

The constant-writing region includes:

- A byte store of 0x3A at the payload base.
- A halfword store of 0x003A at twice the payload base.
- A word store of 0x0000003A at payload-base offset 0x20.
- Successive word stores at working-base offset 0x20 of 0x20, 0x2F, an unsigned byte read from working-base offset 0x04, and 0x20.

The byte read supplying the third overwrite occurs after the first two overwrites. Possible overlap therefore matters.

Next, the word at fixed address **0x087FBBBC** is captured as the **publication base**. This capture occurs **before** the fixed 32-bit store of 0x20 to absolute address **0x60**.

All earlier computed stores may affect the captured shared word if they alias it. Later operations use the captured publication base rather than implicitly rereading that global.

The repeated publication region writes, in order within each group:

- A scaled payload word to an offset-0x54 destination.
- The working base’s low byte to publication-base offset 0x19.
- The publication base’s low byte to working-base offset 0x11.
- A scaled word to publication-base offset 0x34.

The initial group uses eight times the retained word for its offset-0x54 base and final stored word. Subsequent groups use eight times the adjusted value for those roles.

The repetitions are statically explicit, not a represented data-dependent loop. Their exact count is **documented_only** here; collapsing them into one group is not an established equivalence. The constants do not establish text encoding or text output.

### Final fixed-global reads and dependent updates

The final region has three distinct kinds of state: captured words, fresh computed-memory reads, and fresh fixed-byte reads. They must not be conflated.

First, the 32-bit word at absolute address **0xA9** is captured. Then a word is read at payload-base offset 0x74. Only afterward is the byte at absolute address **0x3A** written with the low eight bits of eight times the adjusted value plus 0x30.

Subsequent computed captures read tail-base offset 0x14 and working-base offset 0x04, in that order. Thus the absolute-address-0x3A write precedes those two captures, but follows both the address-0xA9 capture and the payload-offset-0x74 read.

Computed byte updates follow, including 0x6C at working-base offset 0x08 and address-derived bytes at offsets 0x0D and 0x09. A word self-reference is stored at tail-base offset 0x54.

The captured address-0xA9 word’s low byte is then written to absolute address **0x67**. The later observations of that location are separate reads:

1. After reading the word at tail-base offset 0x44, the byte at absolute address 0x67 is read for the byte write to absolute address **0x21**.
2. After further computed stores, a variable-shift byte update, a working-base offset-0x40 word capture, and additional word replacements, address 0x67 is read again for a byte store to working-base offset 0x09.
3. After a byte store to tail-base offset 0x09, a word store of the captured address-0xA9 value to working-base offset 0x44, and a word read at working-base offset 0x54, address 0x67 is read a third time for another byte store to working-base offset 0x09.

These are not unconditional reuses of the originally captured low byte. Intervening computed stores may overlap absolute address 0x67, including when that address lies within a word destination.

The variable-shift update writes a byte at tail-base offset 0x01. Its source is a signed-looking right shift of the captured tail-offset-0x44 word, with a count selected from the working base’s low eight bits. Counts above 31 are possible in the representation; effective shift behavior remains **documented_only**.

Further tail updates restore a previously captured word at tail-base offset 0x54 and read working-base offset 0x14 before later replacements. A newly loaded word from tail-base offset 0x04 supplies the base for:

- A halfword store of 0x0065 at twice that loaded word.
- Word reads at offsets 0x64 and 0x74.

The final updates include a byte publication from the working-offset-0x14 capture, a word store of decimal 100 at another offset-0x54 destination, a word read at tail-base offset 0x64, and address-bearing word stores culminating in a store at twice the word loaded from the former offset-0x64 source.

### Final invocation

The final unresolved target receives four represented arguments, in order:

1. 0x72.
2. The word loaded at offset 0x64 from the base obtained through tail-base offset 0x04.
3. The earlier word captured at tail-base offset 0x14.
4. The later word loaded at offset 0x74 from that same offset-0x04-derived base.

The fourth argument is captured before the last computed stores; it is not an unconditional reread at invocation time.

Target identity, internal effects, resource handling, and return behavior remain unresolved.

## Side effects

Direct effects include byte, halfword, and word stores; overlapping-width updates; stores through loaded address-bearing values; repeated overwrites; and explicit fixed-global accesses.

Established fixed-address stores include:

| Absolute address | Width |
|---|---:|
| 0x01, 0x15, 0x21, 0x3A, 0x67 | 8 bits |
| 0x54, 0x60 | 32 bits |

Meaningful fixed reads include the scalar at 0x05, the publication-base word at 0x087FBBBC, the word at 0xA9, and the three later byte reads at 0x67. Their ordering is specified above.

Dependencies may introduce additional effects. No assumption that they preserve globals is supported.

No transaction, rollback, locking, bounds validation, or error cleanup is represented. A fault or nonreturning dependency may leave a completed prefix of effects.

No direct allocation or release is established. Unknown dependencies may allocate or release memory. Address-bearing stores do not establish ownership transfer, retained-reference validity, or lifetime guarantees.

Atomicity, barriers, concurrency safety, reentrancy, and device-memory semantics are **documented_only**.

## Dependencies

F-0ebc8adc524d is the only supplied resolved opaque function ID. No additional opaque dependency IDs are available.

Dependencies therefore remain unresolved:

- The entry-relative indirect target.
- The preparation targets derived from code-related anchors.
- The final indirect target.

No source symbol is promoted into a dependency identity. Distinct call sites are not assumed to reach distinct functions.

Dependency calling conventions, memory effects, secondary-result behavior, ownership behavior, failures, and completion guarantees are **documented_only**.

## Acceptance examples

These are conditional criteria, not executed tests.

1. **Unsigned widening:** An explicit unsigned byte read of 0xE1 is published as word 0x000000E1, not 0xFFFFFFE1.

2. **Word-width shift:** Under 32-bit truncating shifts, 0xAB123456 shifted left by eight produces 0x12345600.

3. **Bounded index:** Producer low-seven-bit value 127 yields index 508 and doubled index 1016. Higher producer bits do not affect that derivation under the stated model.

4. **Offset versus absolute address:** Acceptance includes a computed byte destination at offset 0x01 and a separate byte write at absolute address 0x01. It does not introduce a word read at computed offset 0x01.

5. **Early fixed capture:** If a duplicated index-based byte write changes the scalar at absolute address 0x05, its subsequent capture observes the changed state, subject to the unresolved scalar width. Capturing it at entry is not equivalent.

6. **Publication-base timing:** If earlier computed stores change the word at 0x087FBBBC, the publication base reflects that state when captured, before the fixed word write at 0x60.

7. **Address-0xA9 timing:** The word at 0xA9 is captured before the byte write at 0x3A. A later overlapping computed write does not retroactively alter that captured word.

8. **Fresh address-0x67 reads:** Suppose the initial low-byte publication writes 0x12 to 0x67. If an intervening computed store changes that byte to 0x34, the next specified read observes 0x34 under ordinary sequential memory semantics. Reusing 0x12 solely because it was originally published is insufficient.

9. **Overwrite sequence:** At working-base offset 0x20, acceptance preserves 0x20, 0x2F, byte-derived word, 0x20 in order where those writes are observable.

10. **Partial progress and completion:** A later fault need not undo earlier writes. If all preceding operations complete, the final invocation receives the four arguments in their specified positions; successful return remains conditional on the target.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention, entry-state requirements, and function-boundary accuracy.
- Unresolved dependency identities and contracts.
- Exact preparation-call ledger and repeated-publication count in this contract.
- Original semantics of overflowing signed-looking arithmetic.
- Pointer representation, endianness, alignment, and valid address ranges.
- Storage width of the fixed scalar at 0x05.
- Excessive-count variable-shift behavior.
- Exact machine read multiplicity and evaluation order inside compound expressions.
- Object boundaries, aliasing restrictions, ownership, and lifetimes.
- Meaning of low-address mappings and possible device semantics.
- Concurrency guarantees and final-target completion.

These gaps do not authorize replacing the represented accesses with a conventional higher-level operation.

## Evidence

The contract relies on the supplied static representation’s indirect invocations, result widths, arithmetic, explicit accesses, fixed-global ordering, repeated stores, and final invocation arguments.

The corrected access classification follows the represented byte destination at computed offset 0x01 and the distinct fixed byte store at absolute address 0x01. Fixed-global ordering follows the explicit placement of the address-0x05 capture, publication-base capture, address-0xA9 capture, and separate address-0x67 reads among computed accesses.

Comments, warnings, the prior draft, and reviewer feedback are not independent behavioral evidence. No dependency bodies, calling contexts, memory map, executable traces, or runtime observations were supplied.

No tools or runtime tests were used. Arithmetic examples are conditional deductions, not measured results.
