<!-- Trusted provenance: F-33c9ec1e2aee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-33c9ec1e2aee

## Purpose

F-33c9ec1e2aee performs an ordered sequence of indirect invocations, derives values from implicit entry state and dependency outputs, mutates memory using mixed access widths, and reaches a final indirect transfer with four represented arguments.

No application-level purpose is established. The evidence does not justify describing this operation as initialization, serialization, rendering, allocation, or container maintenance. Pointer-like values, repeated offsets, and printable constants do not establish those meanings.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, constants, value relationships, and ordering.
- **[INFERENCE]:** conclusions requiring a 32-bit machine interpretation or faithful recovery of the represented control flow.
- **documented_only:** unresolved behavior or explicitly unenumerated detail for which this document makes no stronger acceptance claim.

No runtime tests have been performed.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. Its implicit inputs include:

- Incoming control state used to form the first invocation target.
- Two incoming 32-bit quantities used in early shifted arguments.
- Primary results and additional machine outputs from indirect invocations.
- Memory addressed by dependency results, scaled values, and byte-derived bases.
- Fixed-address byte and word storage.
- A late-loaded global word used as an address base.
- An unresolved final transfer target.

This is not a complete callable ABI. Absence of explicit parameters does not imply absence of inputs.

Observable outputs comprise the invocation trace, dependency effects, explicit memory accesses and mutations, and the final four-argument transfer. A nominal return follows only if that transfer returns normally. No success flag, error code, returned pointer, or byte count is represented.

### Widths and layouts

| Item | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Composite dependency result | 64 bits |
| Ordinary address-bearing quantities | Apparently 32 bits |

Composite results are numerically separated into low and high 32-bit words. This does not establish memory endianness.

Offsets are byte displacements. Frequently used offsets include 0x04, 0x05, 0x09, 0x0D, 0x14, 0x15, 0x34, 0x44, 0x54, 0x64, and 0x74. These accesses do not establish a structure definition, allocation extent, alignment guarantee, or non-aliasing rule.

Byte stores retain the low eight bits. Halfword stores retain the low sixteen bits. Explicit unsigned byte reads widened for word stores are zero-extended.

### Preconditions

Normal completion requires every reached indirect target to be callable and every reached memory access to be valid for its represented width. Derived addresses may be zero, small integers, byte values, or scaled results.

No entry validation establishes these conditions. No ordinary process-address-space assumptions make the fixed low addresses automatically valid.

## Behavior

### Dependency-driven preparation

The represented top-level flow is straight-line: no conditional branch or loop is shown. Dependencies may nevertheless branch, fail, mutate state, or not return.

The first invocation target is formed from incoming control state plus 0xFFE. Later preparation targets repeatedly use code-associated values plus that displacement. The meaning of this targeting convention remains unresolved.

Calls with no represented arguments remain observable calls. Unused results do not establish effect-free execution. Repeated-looking calls must not be merged, reordered, or treated as equivalent. In particular, unknown dependencies may change globals subsequently read by this operation.

Arguments include:

- Incoming quantities shifted left by 3, 25, and 28 bits.
- Low result words shifted left by 8.
- Unmodified and shifted high result words.
- Duplicate arguments and explicit zeros.
- Masked fields subsequently shifted or rescaled.

Field extraction includes bits 10–19, low 24-bit fields, and narrower fields of several widths. Unsigned 64-bit right shifts by 34, 36, and 44 occur before narrowing. Shifting a composite result right by 44 selects its high word shifted right by 12, not a field of its low word.

The complete preparation-call schedule and individual target identities are **documented_only** here. This limitation does not make their multiplicity or effects optional.

### Arithmetic interpretation

**[INFERENCE]** With 32-bit bit-vector arithmetic, shifts and scaling retain the low 32 bits. Consequently:

- A left shift by 25 preserves at most seven input bits.
- A left shift by 8 discards the original high byte.
- A 24-bit field shifted left by 6 occupies at most 30 bits.
- A 22-bit field shifted left by 8 also occupies at most 30 bits.
- Successive 32-bit left shifts by 28 and 28 yield zero.

These are conditional machine-width conclusions. They are not claims about portable signed arithmetic. Signed overflow, address wraparound, and instruction-specific shift semantics remain **documented_only**.

### Memory-phase vocabulary

The following descriptive terms identify retained values without asserting object types:

- **Terminal result:** the last composite preparation result.
- **Retained shifted word:** an earlier low result word shifted left by 25.
- **Retained argument word:** another earlier result shifted left by 8 and retained across later calls.
- **Preparation scalar:** an earlier result whose multiplication by 2²⁵ supplies the **shifted base**.
- **Compact precursor:** a result shifted left by 8, masked to fifteen low bits, then shifted right by 6.
- **Compact base:** twice the compact precursor.
- **Byte-selected base:** the zero-extended byte read from fixed address 0x05.
- **Loaded link word:** the word subsequently read at byte-selected base plus 0x54.
- **Quadrupled link:** four times the loaded link word.
- **Adjusted scalar:** the quadrupled link minus decimal 150.
- **Derived base:** four times the adjusted scalar.
- **Late global base:** a global word loaded after the constant-store region.

All address calculations remain subject to the arithmetic qualifications above.

### Initial mutations and fixed-byte input

The terminal high word supplies a destination base: its offset-0x14 word receives the retained shifted word. The byte at retained shifted word plus 0x09 first receives the terminal high word’s low byte. Between that store and its overwrite with the retained shifted word’s own low byte, a word at terminal low word multiplied by 256 plus 0x54 receives the shifted base.

The shifted-base word at 0x14 is cleared before the byte at shifted base plus 0x05 is read. The compact-base word at 0x14 is then cleared, and the byte just read is zero-extended into shifted base plus 0x34. Next, shifted base plus 0x14 is read as a word. That read is not contractually replaced by zero: intervening accesses can matter under aliasing or mapped-storage behavior.

The compact-base word at 0x14 is cleared again. Two successive byte stores at compact base plus 0x15 receive the low byte of the word just read.

**The byte at fixed address 0x05 is then read and retained before fixed byte address 0x01 receives the narrowed compact base.** The retained byte supplies the byte-selected base for later accesses; changing address 0x05 afterward does not retroactively change this retained value.

The byte-selected base’s offset-0x15 byte receives the compact precursor’s low byte. At shifted base plus 0x11, the retained argument word’s low byte is stored, then later replaced with zero. Between those byte stores, the word read from shifted base plus 0x14 supplies a destination base whose offset-0x54 word receives the retained argument word.

The loaded link word is read from byte-selected base plus 0x54 before that location is overwritten with the earlier shifted-base offset-0x14 read. Compact base plus 0x34 receives the compact precursor.

### Linked mutations and the fixed word at 0x54

A word read at shifted base plus 0x74 supplies a further base. Its offset-0x54 word first receives the preparation scalar shifted left by 26, then later receives the retained argument word. Intervening operations include compact-base offset-0x74 dereferences and repeated self-valued stores at loaded link word plus 0x24, interleaved with byte stores at the reached base plus 0x09.

The next ordering is explicit:

- Compact base plus 0x14 receives the preparation scalar shifted left by 26.
- **Fixed address 0x54 receives a 32-bit word: the loaded link word shifted left by one.**
- Compact base plus 0x14 is overwritten with the loaded link word, and receives that word again.

The fixed-address store is distinct from every dynamically addressed offset-0x54 store. It is not conditional on an inferred structure layout.

Subsequent accesses use doubled, quadrupled, and octupled values directly as bases. They include duplicate byte stores at compact base plus 0x15, a fixed byte store at 0x15, and word mutations at offsets 0x34, 0x44, and 0x54.

One linked path reads a byte at quadrupled link plus 0x05, uses that byte as a base for a word read at 0x74, and propagates the reached word’s low byte through offset-0x0D stores. Another path reads quadrupled link plus 0x64. Self-valued offset-0x14 stores occur on separately reached bases.

A word read at quadrupled link plus 0x04 supplies another base. Its offset-0x05 byte is read twice, with intervening writes. The second read supplies a base for a further offset-0x74 word read. These are separate observations, not a cached-byte contract.

### Constants and repeated publication

The newly reached word supplies scaled destinations for three stores of 0x3A: a byte at four times that word, a halfword at eight times that word, and a word at four times that word plus 0x20. The adjusted scalar is formed between the byte and halfword stores.

At derived base plus 0x20, successive word values are 0x20, 0x2F, a zero-extended byte read from derived base plus 0x04, and 0x20 again. The byte read occurs after the first two stores.

The late global base is then loaded. Fixed address 0x60 receives a 32-bit 0x20.

The publication region repeatedly stores a relationship through four destinations, in this order: a scaled base plus 0x54, late global base plus 0x19 as a byte, derived base plus 0x11 as a byte, and late global base plus 0x34 as a word.

The payload is four times the recently reached offset-0x74 word. The first publication uses eight times the original loaded link word as its scaled base and final word value. Subsequent publications use eight times the adjusted scalar instead. The byte values are the narrowed derived base and narrowed late global base, respectively.

The repetitions are finite and textually explicit, not shown as a count-controlled loop. Their exact count is **documented_only** in this contract. Repetition and access order must not be collapsed to final-state assignments.

### Final-region reads and retained provenance

The final region begins with these ordered observations:

1. The 32-bit word at fixed address 0xA9 is read and retained.
2. A word at publication payload plus 0x74 is read. Its low byte is used later in dynamic stores; it is not the final transfer’s second argument.
3. Fixed byte address 0x3A is written.
4. The word at doubled recently reached link plus 0x14 is read and retained for the final transfer’s third argument.
5. The word at derived base plus 0x04 is read and retained for a later word store.

The fixed-byte value at 0x3A is the low eight bits of eight times the narrowed adjusted scalar plus 0x30. It is not a word store.

The doubled recently reached link will be called the **final working base** below.

The operation writes narrowed derived and late-global-base values through the final working base and derived base, including byte 0x6C at derived base plus 0x08. It also performs a self-valued word store at final working base plus 0x54.

Fixed byte address 0x67 then receives the low byte of the retained fixed-address-0xA9 word. **This store does not establish the value of every later read at 0x67.**

A word is next read at final working base plus 0x44. Only after that read does fixed byte address 0x21 receive a **fresh byte read from 0x67**. The intervening dynamic read can itself matter for mapped storage.

### Remaining mutations and final transfer

The offset-0x44 word supplies both a byte store at derived base plus 0x10 and a complete word store at late global base plus 0x54. A byte at final working base plus 0x01 receives a variable-right-shift result from that word; the shift count is the derived base’s low eight bits. The late-global-base offset-0x54 word is then overwritten with the derived base.

A word read at derived base plus 0x40 is retained while late global base plus 0x74 receives the earlier derived-base offset-0x04 word. The offset-0x40 value is then stored as a word at derived base plus 0x10. Late global base plus 0x44 receives the final working base.

Next, derived base plus 0x09 receives a fresh read of fixed byte 0x67. Intervening stores write the narrowed derived base at final working base plus 0x09 and the retained fixed-address-0xA9 word at derived base plus 0x44. After a word read at derived base plus 0x54, derived base plus 0x09 receives **another fresh read of 0x67**.

The word just read at derived base plus 0x54 becomes a destination base. Its offset-0x0D byte receives the low byte retained from publication payload plus 0x74. Final working base plus 0x54 receives the retained offset-0x14 word destined for the final third argument.

A later word read at derived base plus 0x14 supplies a distinct value, subsequently narrowed into a byte store. It must not be confused with the retained final third argument.

After further byte and word mutations, a word read at final working base plus 0x04 supplies another base. A halfword 0x65 is stored at twice that base. Its offset-0x64 word is read and retained as the final second argument. The recently reached destination’s offset-0x11 byte receives the narrowed derived-base offset-0x14 read, and its offset-0x54 word receives decimal 100.

The final fourth argument is then read through offset 0x74 of the base obtained from final working base plus 0x04. A separate offset-0x64 read through the final working base supplies another destination. Remaining effects include a byte store at final working base plus 0x09, a word store at that destination plus 0x54, and a word store at twice the retained final second argument.

The unresolved final target receives, in order:

- 0x72.
- The retained late offset-0x64 word.
- The earlier retained final-working-base offset-0x14 word.
- The late offset-0x74 word.

No result interpretation or continuation guarantee is established.

## Side effects

Dependency effects are unknown. No dependency is assumed to preserve globals, preserve memory, allocate memory, release memory, or transfer ownership.

Explicit accesses can overlap. A byte store at offset 0x15 can overlap a word stored at offset 0x14. Fixed storage can also alias dynamically computed destinations. Endianness determines the byte composition of overlapping mixed-width stores and remains unresolved.

Fixed-address effects specifically include byte reads or writes at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, word writes at 0x54 and 0x60, and a word read at 0xA9. The latter is not naturally word-aligned; permitted alignment remains unknown.

No rollback, lock, bounds check, null check, or recovery path is represented. Failure does not establish reversal of preceding effects. No allocation or release is identifiable; ownership and lifetime changes are **documented_only**.

## Dependencies

The only supplied resolved opaque function ID is **F-33c9ec1e2aee**, the subject of this contract. No resolved IDs are available for its indirect dependencies; none are invented here.

| Dependency surface | Established interaction | Unresolved contract |
|---|---|---|
| Entry-dependent target | Initial indirect invocation | Identity, ABI, effects, termination |
| Preparation targets | Receive transformed words, zeros, duplicates, or no represented arguments | Individual identities and complete argument conventions |
| Result producers | Supply primary values, composite values, and additional machine outputs | Whether composite values represent genuine 64-bit returns or paired outputs |
| Final target | Receives four arguments beginning with 0x72 | Dispatch meaning, ownership, return behavior |

Naming only the subject in the supplied identity map does not establish recursion.

## Acceptance examples

These are conditional static obligations, not executed tests.

- **Narrowing and widening:** A byte store receiving 0x12345678 writes 0x78. An unsigned byte read of 0xFE widened into a word supplies 0x000000FE.
- **Composite extraction:** For numeric result 0x12345678ABCDEF01, the high word is 0x12345678. A 64-bit right shift by 44 yields 0x12345.
- **Early fixed-byte ordering:** If the read at 0x05 yields 0x7C, the byte-selected base is 0x7C. A subsequent write at 0x01 does not replace that retained input.
- **Distinct fixed-word mutation:** If the loaded link word is 0x12345678, the fixed 32-bit store at 0x54 receives 0x2468ACF0. It occurs after compact base plus 0x14 receives the shifted preparation scalar and before that destination receives the loaded link word.
- **Final-region ordering:** An access trace must place the 0xA9 word read and publication-payload offset-0x74 read before the fixed byte write at 0x3A. Both retained offset-0x14 and derived-base offset-0x04 reads follow that write.
- **Fresh fixed-byte reads:** If 0x67 yields 0x11 when copied to 0x21, then yields 0x22 and 0x33 at the two later dynamic stores, those destinations receive their respective observations. Unconditionally reusing the original low byte from 0xA9 is insufficient.
- **Ordered overwrites:** Observable storage at derived base plus 0x20 can distinguish 0x20, 0x2F, the widened byte observation, and 0x20. Matching only the final value is insufficient.
- **Failure and continuation:** An invalid reached address has no specified graceful error return. If all accesses complete and the final target returns normally, the represented nominal return is reached without an explicit result.

## Uncertainties

The following remain **documented_only**:

- Application purpose and complete callable ABI.
- Dependency identities, contracts, and additional-output validity.
- Accuracy of represented control-flow recovery.
- Complete preparation-call enumeration and publication repetition count in this document.
- Endianness, alignment requirements, signed overflow, and address wrapping.
- Variable-right-shift behavior, particularly counts outside 0–31 and sign treatment.
- Validity and mapped-storage meaning of low addresses.
- Object boundaries, aliasing restrictions, ownership, and lifetimes.
- Final target identity, termination, concurrency safety, and reentrancy.

These uncertainties do not weaken the established fixed-address identities, access widths, retained-value provenance, or fresh-read ordering described above.

## Evidence

This contract is based on independent static inspection of the supplied representation. Prior prose and reviewer feedback are not treated as proof.

The strongest evidence is the explicit memory widths, constants, numeric result separation, data dependencies, and relative ordering of accesses. The fixed word store at 0x54, early read at 0x05 before the write at 0x01, and distinct later reads at 0x67 are directly represented.

No executable image, dependency bodies, runtime observations, or external ABI specification were available. No tools or runtime tests were used. No snapshot hash is asserted.
