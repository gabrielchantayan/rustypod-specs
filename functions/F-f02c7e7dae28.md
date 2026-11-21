<!-- Trusted provenance: F-f02c7e7dae28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f02c7e7dae28

## Purpose

F-f02c7e7dae28 performs an ordered sequence of unresolved indirect invocations, transforms entry-context and dependency-produced bit patterns, then performs mixed-width memory reads and writes before a final unresolved control transfer.

**Established static evidence:** The supplied representation declares no explicit arguments or result. It contains no visible validation, conditional recovery, or ordinary error-reporting path. Observable effects include repeated dependency invocations, overlapping memory accesses, repeated writes to fixed computed destinations, and accesses to low absolute addresses.

**Inferred intent:** Some accesses resemble record updates, and some constants have character interpretations. Neither observation establishes a collection-management, text-output, allocation, or serialization API.

This contract describes supported data-flow and ordering constraints without assigning semantic object types to address-like values. It does not establish that the operation is safely callable through a conventional no-argument interface.

## Interface

### Inputs and entry context

No explicit parameters are declared. Nevertheless, execution depends on:

- An incoming 32-bit register-carried word.
- An incoming return-link value used to form the first indirect target.
- Results and secondary register outputs of indirect dependencies.
- Memory contents at computed addresses and absolute locations.
- An address-like word captured from a stored-address global.
- An unresolved final transfer target.

The actual calling convention, required implicit register state, and interpretation of secondary outputs are **documented_only**.

### Outputs

No explicit returned value is established. Observable outputs consist of the dependency-invocation trace, argument bit patterns, direct memory effects, dependency effects, and final control transfer.

There is no established success flag, error code, ownership token, or guarantee of ordinary return.

### Widths and arithmetic

| Quantity | Supported interpretation |
|---|---|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Combined dependency result | Represented as 64 bits, consumed through low and high 32-bit portions |
| Address-forming quantities | Represented through 32-bit arithmetic |
| Byte store from a wider quantity | Low 8 bits |
| Widened byte read | Unsigned value from 0 through 255 |

The combined-result representation does not establish a genuine 64-bit dependency return type; it may aggregate multiple register outputs.

Arithmetic examples use a conditional 32-bit bit-pattern interpretation: left shifts and scaling retain the low 32 bits. This interpretation supports the stated bit extractions but does not resolve signed overflow or all machine shift behavior. Byte order, alignment requirements, address mapping, and invalid-access behavior remain **documented_only**.

### Preconditions

Reaching any later operation requires all preceding dependencies to return normally and all preceding accesses to complete. Every reached target must be callable, and every reached memory location must support the indicated width and access direction.

These requirements include low absolute addresses, scaled addresses, and addresses obtained from memory contents. Ordinary non-null-pointer requirements are insufficient.

No dependency is assumed to preserve globals, leave memory unchanged, or retain or release resources in any particular way.

## Behavior

### Dependency phase

The first indirect target incorporates the incoming return-link value plus 0xFFE. The next invocation receives the incoming word logically shifted right by 15 and shifted left by 13.

A long straight-line invocation sequence follows. Some occurrences display no arguments; others display one through four. A missing displayed argument list does not establish a recovered zero-argument machine signature.

Significant transformations include:

- Low result words shifted left by 8, sometimes accompanied by unchanged high words.
- Low, high, or secondary-output words shifted by 6, 7, 9, 13, 14, 15, 16, 17, 22, 23, 24, 25, or 30, depending on the occurrence.
- Explicit zero arguments and duplicate transformed arguments.
- Values captured from earlier results and reused across intervening invocations.
- The incoming word’s low 19 bits shifted right by 2.
- The incoming word’s low 14 bits shifted left by 3.
- Result bits 8–23, saved high-word bits 11–17, other saved high-word bits 4–16, and result bits 12–23.

These projections are distinct. An unrestricted right shift must not replace an extraction that first masks upper bits.

Under the stated bit-pattern interpretation, shifting the incoming word left by 13 and then left by 30 produces zero. Other quantities are explicitly reset to zero before later transformations. Such zero-valued arguments do not make the receiving or surrounding invocations removable.

No visible local branch checks dependency success, retries an invocation, or maps a dependency outcome to a defined error response.

### Address-like quantities entering the memory phase

The following descriptive names identify captured values, not recovered object types:

- **Shifted seed:** an earlier low result word shifted left by 25.
- **Scaled base:** a separate result multiplied by \(2^{25}\).
- **Saved shifted word:** another result shifted left by 8.
- **Small component:** a result shifted left by 8, masked to 15 bits, then logically shifted right by 6.
- **Small doubled base:** twice the small component.
- **Final pair:** the low and high words of the last combined result before direct stores.

The small component equals four times its producer’s low seven bits under the stated interpretation. Its possible values are 0, 4, …, 508; its doubled base ranges from 0 through 1016 in steps of eight.

These quantities become addresses or address components without visible validation.

### Initial stores and readback

The opening direct effects preserve this order: a word store of the shifted seed at the final high word plus 0x14; a byte store of that high word at the shifted seed plus 9; a word store of the scaled base at 256 times the final low word plus 0x54; and another byte store at the shifted seed plus 9.

The second byte value is the shifted seed’s low byte, which is zero under 32-bit shift semantics. The preceding byte store remains a distinct event.

Next, a word at the scaled base plus 0x14 is zeroed, and a byte at the scaled base plus 5 is read. A zero word is then stored at the small doubled base plus 0x14 before the captured byte is widened and stored at the scaled base plus 0x34.

The word at the scaled base plus 0x14 is subsequently read. Another zero word is written at the small doubled base plus 0x14, followed by two byte stores of the readback’s low byte at that base plus 0x15.

The byte at offset 0x15 overlaps a word beginning at offset 0x14. Byte order affects its numerical significance, not the overlap. The readback is not unconditionally zero: that conclusion requires ordinary-memory, aliasing, and external-mutation assumptions.

### Early absolute accesses and derived updates

The byte at absolute address 0x05 is read and widened **before** the byte at absolute address 0x01 is written. The latter receives the small doubled base’s low byte. The captured byte from 0x05 subsequently acts as an address base, including for accesses at offsets 0x15 and 0x54. It is not reread after the write to 0x01.

The saved shifted word is published both as a byte at the scaled base plus 0x11 and as a word through an address obtained from the earlier readback. The byte destination at offset 0x11 is then explicitly zeroed.

A word captured from offset 0x54 of the widened-byte base is retained before that location is overwritten. Call this captured word the **unadjusted seed**. Its double, fourfold, and eightfold forms participate in later accesses.

Further updates follow word-valued links at offset 0x74, publish address-like values back into memory, and repeatedly write byte fields at offsets 9 and 0x15. The word at absolute address 0x54 receives twice the unadjusted seed. This absolute store occurs between updates of the word at the small doubled base plus 0x14. Later, absolute byte address 0x15 receives the small doubled base’s low byte.

The representation also contains a destination and stored value that each depend on a read at the small doubled base plus 0x74. A single immutable link must not be assumed where distinct reads are represented; their internal evaluation order is not independently established.

### Alias-sensitive record-like accesses

The fourfold unadjusted seed becomes another frequently used base. Operations through it include word stores at offsets 0x14 and 0x54 and byte stores at offsets 9, 0x0D, and 0x15.

A byte read at its offset 5 is widened and used to locate a word at offset 0x74. Separately, a word read at its offset 4 becomes a base for byte reads at offset 5. Those byte reads recur with intervening writes, so later reads cannot be replaced by an earlier captured byte.

A later such byte determines another word read at offset 0x74. Call that word the **late seed**, twice it the **late doubled base**, and four times it the **publication value**.

Before publication, the operation writes:

- Byte 0x3A at four times the late seed.
- Halfword 0x003A at eight times the late seed.
- Word 0x0000003A at four times the late seed plus 0x20.

The fourfold unadjusted seed is reduced by 0x96 to form the **adjusted seed**. Four times the adjusted seed is the **publication base**.

At the publication base plus 0x20, the visible sequence is word 0x20, word 0x2F, a byte read at offset 4 whose widened value is stored as a word at offset 0x20, and word 0x20 again. Intermediate writes and the byte read remain observable.

These accesses do not establish structure layouts, allocation boundaries, or linked-record invariants.

### Stored-address capture and publication groups

The supplied representation captures an address-like word from the **stored-address global**, then writes word 0x20 at absolute address 0x60, then begins publication.

This ordering is material. The supplied feedback describes the opposite relationship between the 0x60 write and global read, but that relationship is not supported by the displayed source sequence. This contract retains the displayed ordering rather than silently reversing it.

The initial publication group differs from the homogeneous groups that follow it:

| Effect within group | Initial group | Subsequent homogeneous groups |
|---|---|---|
| First word destination | Eight times the unadjusted seed plus 0x54 | Eight times the adjusted seed plus 0x54 |
| First word value | Publication value | Publication value |
| Second effect | Byte at captured stored address plus 0x19 receives publication base’s low byte | Same |
| Third effect | Byte at publication base plus 0x11 receives captured stored address’s low byte | Same |
| Fourth effect | Word at captured stored address plus 0x34 receives eight times the unadjusted seed | Same destination receives eight times the adjusted seed |

All four effects retain their intra-group order.

The following homogeneous block repeatedly uses the captured address and fixed arithmetic values. It has no visible advancing index. Its destinations are not successive array elements, and the initial group must not be folded into the block as an identical repetition.

The block’s exact numeric repetition count is **documented_only** in this contract; its multiplicity must not be treated as one merely because the writes repeat.

### Final captures and low-address propagation

Immediately after publication, the operation captures a word from absolute address 0xA9, then a word at the publication value plus 0x74. Only after those reads does it write the byte at absolute address 0x3A.

That byte is the low eight bits of eight times the adjusted seed plus 0x30 under the bit-pattern interpretation. After this byte write, the operation captures, in order:

- A word at the late doubled base plus 0x14, retained for later publication and the final transfer.
- A word at the publication base plus 4.

Thus the two later captures must not be moved ahead of the 0x3A write under unresolved aliasing or memory-device assumptions.

Subsequent effects include byte stores at the late doubled base plus 0x0D and the publication base plus 8, 0x0D, and 9. The byte at offset 8 receives 0x6C. A word at the late doubled base plus 0x54 receives that base itself.

The captured word from 0xA9 supplies the **initial** byte written at absolute address 0x67. Propagation after that write uses fresh byte reads:

- A word at the late doubled base plus 0x44 is read before the current byte at 0x67 is read and copied to absolute address 0x21.
- Several computed writes and a word read at the publication base plus 0x40 occur before another current-byte read from 0x67 supplies a byte store at the publication base plus 9.
- Further computed stores, including a word store of the captured 0xA9 value at the publication base plus 0x44, precede a word read at the publication base plus 0x54.
- After that word read, another current-byte read from 0x67 supplies another byte store at the publication base plus 9.

The initial 0xA9 capture therefore does not determine all later propagated bytes. Intervening computed writes may alias 0x67; reads may also have device-specific effects. The same captured word is independently used for the full-word store at offset 0x44, regardless of subsequent byte changes at 0x67.

### Remaining final updates and transfer

The word read from the late doubled base plus 0x44 supplies a byte at the publication base plus 0x10, an initial word publication through the captured stored address plus 0x54, and a shifted-byte store at the late doubled base plus 1.

That shift uses the publication base’s low eight bits as its count. Counts of 32 or more are possible. Whether the right shift is arithmetic and how excessive counts behave are **documented_only**; masking or zero-producing behavior must not be invented.

The word publication through the stored address plus 0x54 is subsequently replaced by the publication base. Other effects publish the captured publication-base-plus-4 word at stored address plus 0x74, copy the word read at publication base plus 0x40 to publication base plus 0x10, and store the late doubled base at stored address plus 0x44.

The word later read at publication base plus 0x54 becomes another address base. It receives byte and word updates, including word decimal 100 at offset 0x54. A word captured from publication base plus 0x14 supplies a later byte at this new base plus 0x11.

A word read at late doubled base plus 4 determines a halfword destination at twice that word; it receives 0x0065. Its offsets 0x64 and 0x74 are read at separate later points, with intervening stores. A further word read at late doubled base plus 0x64 supplies a destination for a word store at offset 0x54 and a value stored through twice the captured offset-0x64 word.

The final unresolved invocation receives four displayed arguments:

1. 0x72.
2. The captured word from offset 0x64 of the base obtained from late doubled base plus 4.
3. The earlier capture from late doubled base plus 0x14.
4. The captured word from offset 0x74 of the same late-derived base.

These are retained captures, not permission to reread the fields at invocation time. Ordinary return from the final transfer is unresolved.

## Side effects

Direct effects include 8-, 16-, and 32-bit accesses, overlapping stores, repeated stores, pointer-like reads, and low-address accesses.

Meaningful absolute data flow includes the 0x05 read before the 0x01 write; ordered updates to 0x54 and 0x15; the stored-address capture before the 0x60 word write; and the final 0xA9 capture, 0x3A write, initial 0x67 write, and later 0x67 reads feeding 0x21 and computed destinations.

No atomicity, synchronization, transaction boundary, or rollback is established. A failed access or dependency may leave partial effects.

No explicit allocation or release operation is identified. Dependency results becoming addresses establish neither ownership transfer nor allocation provenance, lifetime, or permission to release memory. Dependency resource effects remain unknown.

No local error response is established for invalid targets, invalid addresses, unsupported alignment, arithmetic ambiguity, or excessive shift counts.

## Dependencies

Unresolved invocation occurrences may be identified as **F-opaque-001**, **F-opaque-002**, and subsequent ordinal occurrence IDs. These denote positions in the invocation trace, not recovered distinct function identities. The final unresolved transfer is **F-opaque-final**.

The supported dependency-facing requirements are to preserve invocation order, displayed argument transformations, explicit zero and duplicate arguments, and captured result values reused after intervening calls.

Signatures, target identities, secondary-output conventions, global preservation, ownership behavior, and normal return are **documented_only**. Apparently unused results do not make an invocation effect-free.

## Acceptance examples

These are conditional static examples, not executed tests.

### Bit extraction

For incoming word 0xFFFFFFFF, logical right shift by 15 yields 0x0001FFFF and 32-bit left shift by 13 yields 0xFFFFE000. The low-19-bit projection shifted right by 2 yields 0x0001FFFF; the low-14-bit projection shifted left by 3 yields 0x0001FFF8.

For result word 0x12345678, extracting bits 8–23 yields 0x3456, not 0x123456.

### Small component

If the relevant producer’s low seven bits are 127, the small component is 508 and its doubled base is 1016. A component of 511 violates the visible extraction.

### Separate observable writes

If the final high result’s low byte is 0xAB, the initial repeated byte destination receives 0xAB and later zero, assuming both accesses succeed. Recording only the final zero omits an observable event.

### Absolute read ordering

A model in which writing 0x01 changes a device value at 0x05 must still use the byte captured before that write for subsequent widened-byte addressing.

Likewise, the final words captured after the 0x3A write may reflect that write if their locations alias it; the preceding 0xA9 and publication-value-plus-0x74 captures cannot.

### Current-byte propagation

Suppose the 0xA9 capture ends in 0xAB and a later computed write changes 0x67 to 0xCD before a subsequent byte read. The initial 0x67 write receives 0xAB, but that later consumer receives 0xCD. Substituting the original capture for the current-byte read is not equivalent.

### Initial versus homogeneous publication

For unadjusted seed 0x100, the adjusted seed is 0x36. The initial group uses scaled value 0x800 for its first destination base and fourth stored word; subsequent groups use 0x1B0. Only the homogeneous block repeats the same destinations and values.

### Readback and final arguments

A previously zeroed word reads as zero only under appropriate ordinary-memory and noninterference assumptions. Similarly, final arguments retain their specified captured origins even if intervening stores alter the original locations.

## Uncertainties

The following remain **documented_only**:

- Actual entry convention and implicit register requirements.
- Dependency identities, signatures, effects, and secondary outputs.
- Whether combined results are genuine 64-bit returns.
- Byte order, alignment, address mapping, device behavior, and external mutation.
- Object layouts, allocation boundaries, ownership, and lifetime.
- Signed overflow and unresolved right-shift behavior.
- Internal ordering of multiple reads within a single represented expression where not otherwise established.
- Numeric multiplicity of the homogeneous publication block.
- Final target and return behavior.
- Fidelity of the supplied representation to original execution.

The disagreement about the stored-address capture and 0x60 write is a source-versus-feedback discrepancy, not evidence for an additional read or reordered execution.

## Evidence

**Established static evidence:** The supplied representation supports the indirect-call prefix, width-specific transformations, ordered direct-memory effects, distinct initial and homogeneous publication groups, and final captured-argument transfer described above.

In particular, it places the stored-address global capture before the 0x60 write. It also distinguishes the initial 0x67 assignment from later reads of that byte, and places the final 0x3A write between two pairs of word captures.

**Inferred intent:** Record-like mutation and character-related output remain interpretations only.

**Verification status:** Static analysis only. No runtime tests, executable inspection, memory traces, or dependency implementations were used. Acceptance examples are conditional arithmetic and ordering consequences, not observed execution results.
