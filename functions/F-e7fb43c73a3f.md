<!-- Trusted provenance: F-e7fb43c73a3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7fb43c73a3f

## Purpose

F-e7fb43c73a3f represents an operation that performs an extensive ordered sequence of unresolved indirect calls, followed by mixed-width memory mutations and a final unresolved transfer.

The supplied static evidence does not establish a domain-level purpose. Recurring offsets, field extraction, and character-like constants do not establish parsing, rendering, object initialization, collection management, or a particular protocol.

This contract distinguishes:

- **Established static evidence:** represented calls, value relationships, access widths, and ordering.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** behavior not resolved by the available evidence.

Requirements are conditional on the static representation accurately describing execution. No runtime tests have been performed.

## Interface

### Explicit and implicit inputs

The represented interface accepts three 32-bit arguments.

| Input | Established consumption |
|---|---|
| First argument | No explicit consumption is represented. |
| Second argument | No explicit consumption is represented. |
| Third argument | Unsigned value whose bits 10–19 supply an early dependency argument. |

The third-argument field is shifted left by five after extraction. Its range is 0 through 32,736, in multiples of 32. Bits outside 10–19 do not affect this particular calculation.

The first two arguments cannot be declared irrelevant to the machine interface: unresolved calls have incomplete argument descriptions and may observe incoming execution state.

Additional inputs include an incoming control-location value used to calculate the first destination, an incoming register-sized quantity used in shifted arguments, call results and secondary outputs, fixed-location memory, and memory reached through calculated addresses. The final transfer destination has no established origin.

The three explicit arguments therefore do not define a reproducible invocation by themselves.

### Outputs and completion

No explicit return value is represented. Observable outputs comprise dependency interactions, memory accesses, state mutations, and a final transfer receiving four values.

The first transfer argument is 0x72. The remaining arguments are two late-loaded words and a word preserved from an earlier read. Their provenance is specified below.

A return is represented after the transfer, but whether that transfer returns normally, continues execution elsewhere, or never returns is **documented_only**.

### Preconditions

Every reached destination must support execution, and every reached memory access must be valid for its width and direction. Address calculations involve call-derived values, loaded words, zero-extended bytes, scaling, and very low numeric addresses.

No explicit null, alignment, range, lifetime, or ownership checks are established. A conventional application memory map is not sufficient evidence that these accesses are valid.

## Behavior

### Dependency-driven phase

The initial phase is a long ordered sequence of unresolved indirect calls. No explicit input-conditioned branch, retry loop, or early error return is represented.

Interactions include calls with zero through four explicit arguments, 32-bit results, 64-bit result carriers, and secondary outputs whose complete calling convention is unavailable. Lower and upper 32-bit portions of a 64-bit carrier are consumed separately. A carrier does not establish a 64-bit pointer or an eight-byte memory object.

Common argument transformations include:

- Left shifts by eight and other fixed counts.
- Masked field extraction and repositioning.
- Logical right shifts of unsigned values.
- Repeated use of one transformed quantity in several argument positions.
- Shifts of quantities explicitly set to zero.

Calls whose represented results are unused remain part of the interaction trace. They cannot be removed as pure computations: their effects, failures, and return behavior are unresolved. No dependency is assumed to preserve global state.

### Widths and arithmetic

Memory accesses use 8-, 16-, and 32-bit widths; some dependency results use 64-bit carriers.

Byte writes retain the low eight bits, halfword writes retain the low 16 bits, and word writes retain 32 bits. Unsigned byte reads widen to values between 0 and 255. Extracting the upper word of a carrier differs from logically shifting the complete carrier by 34, 36, or 44.

Many represented shifts, multiplications, and address calculations use signed 32-bit quantities. Their overflow behavior is not established as portable arithmetic. Where this contract gives modular relationships, it explicitly assumes **32-bit wrapping machine arithmetic**. Intermediate truncation matters: successive 28-bit left shifts of a 32-bit value produce zero under that interpretation.

One late operation right-shifts a **signed 32-bit loaded word** by a count taken from the low eight bits of a calculated base. Its result is narrowed to a byte. Negative-operand right-shift behavior and counts exceeding 31 are separately unresolved; the count must not silently be reduced modulo 32.

### Direct-access data relationships

The following descriptive terms identify values for this contract, not objects or ownership domains.

| Term | Represented origin |
|---|---|
| Compact base | Twice a field obtained by shifting a call result left by eight, retaining 15 bits, then logically shifting right by six. |
| Large scaled base | An earlier call result multiplied by 0x02000000. |
| Saved link | Word read at offset 0x54 from the zero-extended byte read at numeric address 0x05. |
| Fourfold base | Four times the saved link. |
| Adjusted base | Fourfold base minus 0x96. |
| Publication base | Four times the adjusted base. |
| Linked word | Later word read at offset 0x74 from a zero-extended byte obtained through an indirect offset-5 read. |
| Doubled linked base | Twice the linked word. |
| Current scaled base | Four times the linked word, used throughout the repeated-store region. |
| Fixed-derived base | Value read from a separate fixed-location word before that region. |

Under wrapping arithmetic, the compact base is a multiple of eight between 0 and 1,016. It is not established as an ordinary heap address.

These terms denote retained numeric values. They do not imply disjoint memory, stable object contents, or independently allocated records.

### Initial direct mutations and early fixed-location effects

After the last call in the dependency-driven phase, a word write at offset 0x14 through the final carrier’s upper word precedes a byte write at offset 9 through another call-derived base. A word write at offset 0x54 through the carrier’s scaled lower word intervenes before a second byte write to that offset-9 destination.

The large scaled base has its offset-0x14 word cleared before its offset-5 byte is read. That byte is widened and written as a word at offset 0x34. The compact base’s offset-0x14 word is cleared repeatedly. A subsequent read of the large scaled base’s offset-0x14 word supplies values used in later writes; it is not unconditionally zero because intervening accesses may alias it.

The earlier fixed-location dataflow has the following established relative order:

| Fixed access | Value and surrounding ordering |
|---|---|
| Byte read at 0x05 | Occurs after two byte writes at compact-base offset 0x15. Its zero-extended value supplies an address base. |
| Byte write at 0x01 | Receives the low byte of the compact base, after the read at 0x05 and before a byte write through the newly obtained byte-derived base. |
| Word write at 0x54 | Receives twice the saved link. It follows a word write at compact-base offset 0x14 and precedes two word writes of the saved link to that offset. |
| Byte write at 0x15 | Receives the low byte of the compact base. It follows those word writes and two byte writes of the fourfold base at compact-base offset 0x15. |
| Word write at 0x60 | Receives 0x20, after the fixed-derived base is read and before the repeated-store region begins. |

The saved link is read through the byte-derived base at offset 0x54 **before** that word is overwritten with the earlier clear-derived value. It therefore preserves the read result, not the later contents.

Calculated accesses can overlap these numeric locations. The table specifies represented writes, not guaranteed final contents.

### Linked and scaled mutations

The following relationships are established without establishing a unified record layout:

- Words loaded at offset 0x74 supply destinations for offset-0x54 writes.
- The saved link is written to its own offset-0x24 destination twice, with intervening accesses.
- Bytes read at offset 5 are zero-extended and used as bases for word reads at offset 0x74.
- Address-like values are narrowed into byte fields at offsets 9, 0x0D, 0x11, and 0x15.
- Scaled values, retained words, and widened bytes are published at offsets 0x14, 0x34, 0x44, and 0x54.
- A word loaded at offset 0x64 supplies another base for byte and word mutations.

Repeated reads through equivalent calculated addresses remain distinct reads. Intervening writes may change their results through aliasing.

A later group writes 0x3A as a byte at four times the linked word, as a halfword at eight times that word, and as a word at offset 0x20 from the current scaled base, in that relative order.

At publication-base offset 0x20, four word writes publish, in order: 0x20, 0x2F, the zero-extended byte read at publication-base offset 4, and 0x20. These stores do not establish a textual output stream.

### Repeated-store region

Each represented group has an ordered word write, two byte writes, and a final word write:

- The word write publishes the current scaled base at offset 0x54 through an eightfold-scaled base.
- The byte writes publish the low byte of the publication base at fixed-derived-base offset 0x19 and the low byte of the fixed-derived base at publication-base offset 0x11.
- The final word write publishes an eightfold-scaled value at fixed-derived-base offset 0x34.

The initial group uses eight times the saved link for the scaled base and final published value. Subsequent groups use eight times the adjusted base.

The fixed-derived base and byte values are retained; the fixed-location word is not represented as being reread for each group. The repetitions have no visible loop condition. An exact numeric repetition count is **documented_only** here; the represented accesses must not be collapsed into a single final-state update.

### Late fixed-location dataflow

After the repeated region, the following ordering is established:

1. A 32-bit word is read at numeric address 0xA9 and retained.
2. A word is read at current-scaled-base offset 0x74 and retained separately.
3. A byte is written at numeric address 0x3A.
4. Further words are read at doubled-linked-base offset 0x14 and publication-base offset 4, for later use.

The byte at 0x3A is the low eight bits of **eight times the adjusted base plus 0x30**. The represented calculation narrows the adjusted base before multiplication; narrowing the final expression gives the same byte. Under wrapping arithmetic, the relationship is also the low byte of 32 times the saved link minus 0x480. It is not necessarily a digit.

Before the retained 0xA9 value is used for another fixed write, byte mutations occur through the doubled linked base and publication base, including a byte write of 0x6C at publication-base offset 8. A word equal to the doubled linked base is then written at that base’s offset 0x54.

The low byte of the retained 0xA9 word is subsequently written at numeric address 0x67. A signed word is then read at doubled-linked-base offset 0x44. After that read, a fresh byte read at 0x67 supplies the byte write at 0x21.

Two additional, separate byte reads at 0x67 later supply two byte writes at publication-base offset 9. Between these reads, calculated writes and an indirect word read occur. In particular, the retained full 0xA9 word is written at publication-base offset 0x44 between the two later reads.

Consequently:

- The write at 0x67 uses the retained low byte from 0xA9.
- The write at 0x21 uses the byte actually read at 0x67 at that time.
- Each later publication-base offset-9 write uses its own fresh read at 0x67.
- None of those three reads is guaranteed to reproduce the retained low byte. Calculated destinations can alias 0x67, even without external mutation.

### Late signed-shift byte mutation

The signed word loaded at doubled-linked-base offset 0x44 supplies a byte write at publication-base offset 0x10 and a word write at fixed-derived-base offset 0x54.

**After that word write, but before the same destination is overwritten with the publication base**, the operation writes a byte at doubled-linked-base offset 1. The byte comes from right-shifting the retained signed word by the low eight bits of the publication base, then retaining the result’s low eight bits.

The source word is retained from its earlier read, not reread after the offset-0x54 write. Aliasing that destination with the original source location therefore does not change the retained operand.

The count ranges within 0–255, not necessarily 0–31. More specifically, under wrapping arithmetic, the publication-base relationship makes its low byte one of 8, 24, 40, …, 248. Counts above 31 are therefore a substantive possibility.

For nonnegative operands and an in-range count, ordinary right-shift results determine the stored byte. For negative operands, signed right-shift semantics are **documented_only**. For counts of 32 or greater, target handling is **documented_only** regardless of operand sign. No count mask, clamp, rejection, or portable result is established.

### Final mutations and transfer

The operation subsequently replaces the byte-oriented publication-base offset-0x10 contents with a full word loaded from offset 0x40. It also publishes retained values at fixed-derived-base offsets 0x74 and 0x44.

A word loaded at publication-base offset 0x54 supplies further indirect destinations. The low byte of the earlier current-scaled-base offset-0x74 read is written through that value at offset 0x0D and through the doubled linked base at offset 0x11.

The retained doubled-linked-base offset-0x14 word is restored to doubled-linked-base offset 0x54 after intervening mutations. This is a write of the saved value, not a new read of offset 0x14.

The final region also includes:

- A halfword write of 0x65 at twice a word loaded from doubled-linked-base offset 4.
- A word write of decimal 100 at offset 0x54 through another loaded base.
- A byte write from a retained publication-base offset-0x14 word.
- Further offset-0x64 and offset-0x74 word reads.
- A final word write at twice a late-loaded value.

The final transfer arguments, in order, are:

1. 0x72.
2. The word read at offset 0x64 through the value obtained from doubled-linked-base offset 4.
3. The preserved word from doubled-linked-base offset 0x14.
4. The later word read at offset 0x74 through that same offset-4-derived value.

No explicit success check follows the transfer.

## Side effects

### Layout and access semantics

Established relative byte offsets include 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, and 0x19. Word offsets include 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. Halfword accesses use scaled addresses.

These are access relationships, not one verified structure definition. Fixed-location accesses and calculated accesses belong to a potentially overlapping memory space.

Order matters when accesses alias, fault, expose device effects, or encounter external mutation. A later overwrite does not erase an earlier observable write. Endianness, alignment rules, atomicity, volatility, and synchronization are not established.

### Ownership and errors

No explicit allocation, release, or ownership transfer is identified. Dependency behavior does not establish retention, lifetime, cleanup, or global-preservation guarantees.

No explicit diagnostic, error result, rollback, or recovery path is represented. Earlier effects may remain after a late fault or nonreturning call. Completion is not established as transactional.

## Dependencies

F-e7fb43c73a3f is the only established opaque function ID in the supplied mapping. No opaque IDs are available for the other destinations, so none are invented.

For every unresolved dependency, the following remain **documented_only**:

- Identity and whether multiple sites reach the same operation.
- Complete arguments and calling convention.
- Primary and secondary output meaning.
- Global and memory effects.
- Ownership and lifetime behavior.
- Fault, exception, and nonreturning behavior.

The final transfer is likewise unresolved. Recursion and association with a known API are not established.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input field boundaries.** Third-argument values 0–1,023 produce zero for the established field argument; 1,024 produces 32; 0x000FFFFF produces 32,736. Changing only bits 20–31 leaves that calculation unchanged, not necessarily the entire invocation.

2. **Early fixed writes.** If the compact base is 0x108 and the saved link is 0x1234, wrapping arithmetic gives byte 0x08 at 0x01, later word 0x2468 at 0x54, and later byte 0x08 at 0x15. These are ordered write events, not unconditional final-memory assertions.

3. **Calculated byte at 0x3A.** With an adjusted base of 2, the represented byte write is 0x40. The preceding 0xA9 word read and current-scaled-base offset-0x74 word read must already have occurred.

4. **Fixed-byte rereads without aliasing.** If the saved 0xA9 word is 0x12345678 and no intervening access or external action changes 0x67, its subsequent reads yield 0x78.

5. **Fixed-byte rereads with aliasing.** If a calculated write changes 0x67 between two reads, the later consumer uses the changed byte. Reusing a cached 0x78 would not preserve that dataflow.

6. **In-range shift.** With a retained positive word 0x12345678 and publication-base low byte 8, the offset-1 byte write is 0x56. It occurs between the fixed-derived-base offset-0x54 word write of the retained operand and its overwrite with the publication base.

7. **Out-of-range shift.** With publication-base low byte 40, no particular result byte is established. Treating 40 as 8 requires target evidence absent here.

8. **Observable overwrite sequence.** The publication-base offset-0x20 word writes remain distinguishable as 0x20, 0x2F, widened byte, 0x20 when memory exposes the trace.

9. **Late failure.** A fault after completed stores does not establish rollback. Even fault-free earlier execution does not establish that the final transfer returns.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, complete machine interface, and boundary accuracy.
- Dependency identities and effects.
- Signed overflow, signed right-shift rules, and out-of-range shift handling.
- Actual address validity, object boundaries, aliases, and lifetimes.
- Endianness, alignment, atomicity, and external mutation.
- Exact repeated-store count asserted by this contract.
- Final destination and completion semantics.
- Thread safety, reentrancy, and interrupt interaction.

Inferred intent is limited to possible field manipulation and state publication. No object model or textual-output interpretation is required.

## Evidence

The supplied static representation supports the explicit argument widths, third-argument field extraction, ordered unresolved calls, mixed-width accesses, retained-value relationships, fixed-location dataflow, late signed-shift byte mutation, and final four-argument transfer described above.

It does not provide dependency bodies, a verified calling convention, an authenticated memory map, target shift semantics, or execution observations. Embedded comments and earlier descriptions are not independent verification.

No runtime tests have been performed. No successful execution or provenance hash is asserted.
