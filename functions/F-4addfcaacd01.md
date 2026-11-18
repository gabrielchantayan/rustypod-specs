<!-- Trusted provenance: F-4addfcaacd01; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4addfcaacd01

## Purpose

F-4addfcaacd01 is a parameterless entry that performs an ordered sequence of unresolved indirect calls, derives numeric fields from their results, modifies fixed-address and computed-address storage, and reaches a final unresolved indirect transfer.

No application-level purpose is established. Character-like constants and recurring offsets do not establish formatting, serialization, initialization, or an object-maintenance protocol.

This contract distinguishes:

- **Established static evidence:** accesses, calculations, and ordering explicitly represented in the supplied material.
- **Inferred intent:** interpretations that impose no acceptance requirement.
- **documented_only:** behavior for which the available evidence does not support a complete requirement.

This is a partial behavioral contract, not a complete executable replacement specification. Dependency identities, parts of the execution model, and some sequence details remain unresolved.

## Interface

### Inputs and outputs

The represented entry has no explicit parameters and no explicit return value. Its effective inputs nevertheless include:

- Incoming execution context used to form the first indirect target.
- Scalar, composite, and additional result state produced by dependencies.
- Memory contents at each reached read, including fixed-address storage.
- Memory mappings, permissions, alignment rules, and address arithmetic.
- Dependency effects occurring before later calls or memory accesses.

No success indicator, error code, returned allocation, or designated output buffer is established. Memory effects and dependency interactions are the visible outputs.

### Widths and layout rules

| Category | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Ordinary scalar, address, and word access | 32 bits |
| Composite call result | 64 bits |

Byte and halfword stores retain the corresponding low-order bits. Explicit unsigned byte widening produces values from 0 through 255.

Composite results are separated numerically into low and high 32-bit portions. Some other extractions begin at bit positions 34, 36, or 44. These operations do not establish memory endianness.

A word access affects four bytes, including when its address is unaligned. Whether such an access succeeds, faults, or has other platform-specific behavior is **documented_only**. Endianness is also unresolved.

### Preconditions and failure boundary

An uninterrupted traversal requires executable indirect targets, compatible calling conventions, meaningful consumed result state, and accessible storage for every reached access at its represented width.

No visible validation establishes these conditions. They are execution prerequisites, not checked input restrictions. A failure may occur after earlier calls and writes have already taken effect.

## Behavior

### Dependency phase

The opening phase is a long, straight-line sequence of indirect calls. The first target depends on incoming execution context plus displacement 0xFFE. Later targets use embedded location-related values with the same displacement. The available map does not identify the callees.

Call sites represent zero through four explicit arguments. Arguments include zero constants, shifted scalar results, portions of composite results, masked fields, duplicated values, and state retained across intervening calls. Additional result state is sometimes consumed separately from the ordinary scalar return.

Calls with no explicit arguments remain observable interactions. Ignored return values do not imply purity. Similar argument patterns do not establish target equality.

No represented status branch selects an abbreviated sequence. If a dependency does not return normally, subsequent behavior is not required by this contract.

The exact complete call-site inventory, argument association, and actual ABI remain **documented_only** here. This limitation does not authorize omitting, merging, or reordering calls.

### Arithmetic constraints

Numeric selections include masks of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Masking and shifting are distinct operations whose intermediate widths matter.

Established examples include:

- Low 20 bits followed by an unsigned right shift of ten: range 0–1023.
- Low 24 bits followed by an unsigned right shift of eight: range 0–65535.
- Low 17 bits followed by an unsigned right shift of four: range 0–8191.

Unsigned right shifts are zero-filling. For unsigned 32-bit calculations, discarded high bits do not survive later operations. Signed left shifts, overflowing signed multiplication, and mixed signed/address calculations cannot all be assigned wrapping semantics from the supplied representation alone.

No saturation, overflow rejection, or unbounded-integer interpretation is established. Successive shifts must not be combined without preserving intervening width and truncation.

A late signed value is right-shifted by a distance derived from the low byte of another value. Negative operands and distances outside the ordinary 32-bit shift range remain **documented_only**; no count-masking rule is established.

### Opening direct-memory effects

The following semantic labels describe retained values without asserting structure or ownership:

- **Primary base:** a retained scalar call result multiplied by 2²⁵.
- **Compact base:** twice the field obtained by shifting another scalar left eight, selecting its low 15 bits, and shifting right six.
- **Packed value:** an earlier scalar shifted left 25.
- **Terminal portions:** the low and high 32-bit portions of the final composite result before direct mutation.

Under a 32-bit bit-vector interpretation, the compact base ranges from 0 through 1016 in increments of eight, and the packed value has a zero low byte. Neither property establishes address validity.

The initial direct effects preserve this order:

- A word at terminal-high-plus-0x14 receives the packed value.
- A byte at packed-value-plus-0x09 receives the low byte of the terminal high portion.
- A word at 256 times the terminal low portion, plus 0x54, receives the primary base.
- The byte at packed-value-plus-0x09 is overwritten with the low byte of the packed value.

Next, primary-base-plus-0x14 is cleared as a word before the byte at primary-base-plus-0x05 is read. Compact-base-plus-0x14 is then cleared as a word before the captured byte is widened and stored as a word at primary-base-plus-0x34.

The word at primary-base-plus-0x14 is subsequently read. Another compact-base-plus-0x14 word clear precedes two byte stores at compact-base-plus-0x15, both using the low byte of that captured word.

These destinations need not be distinct. In particular, the captured word cannot universally be replaced with zero: the intervening widened-byte store may overlap it.

### Early fixed-address effects and linked updates

After those two compact-base byte stores, the function reads the byte at absolute address **0x05** and retains its unsigned widened value. Only afterward does it write the low byte of the compact base to absolute address **0x01**.

The captured byte from 0x05 then serves as a numeric address base, not merely as data. A byte at that base plus 0x15 receives the low byte of half the compact base. Later, a word at that base plus 0x54 is captured before that word is replaced. Call this captured word the **saved link**.

Between the fixed read and that word capture, computed-address writes occur. The capture therefore observes memory at its actual position in the sequence, not necessarily entry-state memory.

Further linked updates use the saved link and bases scaled by two, four, or eight. The significant fixed-address progression is:

| Fixed effect | Ordering and value |
|---|---|
| Byte read at 0x05 | Before the byte write at 0x01; the read value is retained for subsequent address formation. |
| Byte write at 0x01 | Low byte of the compact base. |
| Word write at 0x54 | Twice the saved link, represented as a left shift of one. |
| Byte write at 0x15 | Low byte of the compact base, later than the word write at 0x54. |

Around the 0x54 write, compact-base-plus-0x14 first receives a shifted retained call result, then receives the saved link twice. Two further byte stores at compact-base-plus-0x15 use the low byte of four times the saved link; the absolute 0x15 write follows those stores.

The word write at absolute 0x54 is not interchangeable with a write to a field at some computed base plus 0x54. They coincide only if the reached address calculation makes them coincide.

The remaining linked-update region reads words through offsets including 0x04, 0x44, 0x54, 0x64, and 0x74. Loaded bytes can become bases for word reads. Effects include self-address-derived stores, word replacement, low-byte address fragments, and updates through both retained and freshly loaded bases.

These patterns establish neither list membership nor ownership fields. The complete intervening computed-access trace is **documented_only** in this contract; its effects must not be assumed absent when evaluating later reads.

### Constant-bearing writes and pointer capture

A later linked word supplies a value whose doubled form is retained as the **doubled-link base**. Its fourfold form becomes the **scaled-link base**.

The function writes 0x3A at three distinct widths: a byte at the scaled-link base, a halfword at twice that base, and a word at that base plus 0x20.

Define the **adjusted base** as four times the saved link minus 0x96, and the **working base** as four times the adjusted base, subject to the unresolved signed-arithmetic qualifications above.

The word at working-base-plus-0x20 receives, in order, 0x20, 0x2F, a widened byte read from working-base-plus-0x04, and 0x20 again. That byte read occurs after the first two stores.

Only then is a 32-bit pointer-valued word captured from absolute address **0x087FBBBC**. Call its retained value the **captured pointer**. The function subsequently writes the word 0x20 to absolute address **0x60**.

This capture-before-write relationship is significant. Later updates use the captured pointer, not repeated reads of its source storage. If the 0x60 write or subsequent computed writes alter the pointer’s source storage, they do not by themselves change the retained pointer.

### Repeated updates

Following the 0x60 write, an initial update group stores the scaled-link base at eight times the saved link plus 0x54, writes low-byte fragments at captured-pointer-plus-0x19 and working-base-plus-0x11, and writes eight times the saved link at captured-pointer-plus-0x34.

The subsequent repeated group instead uses eight times the adjusted base for the scaled destination and the final stored word. Its internal order remains:

- Word update at the eightfold-scaled destination plus 0x54.
- Byte update at captured-pointer-plus-0x19.
- Byte update at working-base-plus-0x11.
- Word update at captured-pointer-plus-0x34.

The exact repetition count is **documented_only** here. Preserving only final memory contents is not an established substitute: access multiplicity can matter for device mappings, faults, instrumentation, or concurrent observers.

### Late fixed-address reads, writes, and rereads

After the repeated region, the function captures a **32-bit word at absolute address 0xA9**. Call this the **fixed word**. It also captures a word at scaled-link-base-plus-0x74.

A byte write to absolute address **0x3A** follows those reads. Its value is the low eight bits of eight times the adjusted base plus 0x30. This is an absolute byte destination, distinct from the earlier computed-address stores containing the numeric value 0x3A.

The next captures include a word at doubled-link-base-plus-0x14, retained for the final transfer, and a word at working-base-plus-0x04. Computed writes then affect doubled-link-base-plus-0x0D, working-base-plus-0x08, working-base-plus-0x0D, working-base-plus-0x09, and doubled-link-base-plus-0x54. The 0x08 byte receives 0x6C.

Only after those writes does absolute address **0x67** receive the low byte of the fixed word captured at 0xA9. The function then reads the word at doubled-link-base-plus-0x44 before assigning the current byte at **0x67** to the byte at absolute address **0x21**.

Thus, the established fixed-address order is:

**word read at 0xA9 → byte write at 0x3A → byte write at 0x67 → byte read from 0x67 and write to 0x21**, with the intervening computed accesses described above.

The initial 0x67 write is not the last use of that storage. Later expressions reference 0x67 again:

- The first later reference supplies a byte store at working-base-plus-0x09 after intervening computed byte and word writes, including a word write at working-base-plus-0x10 and an update at captured-pointer-plus-0x44.
- The second later reference supplies another byte store at working-base-plus-0x09 after the first such store, a byte store at doubled-link-base-plus-0x09, a word store of the fixed word at working-base-plus-0x44, and a word read at working-base-plus-0x54.

These are later references to storage, unlike the once-captured pointer. Computed writes may overlap 0x67, so neither later reference may automatically be replaced with the initially written low byte of the fixed word. The copy to 0x21 likewise denotes the byte observed at its own source-read point.

The retained fixed word remains a separate 32-bit value. Later changes to 0x67 do not retroactively modify that captured word.

### Final transfer

Further late effects include restoring a retained word through doubled-link-base-plus-0x54, additional byte and word updates, a halfword store of 0x65 through twice a loaded base, a word store of decimal 100, and a word store through twice another loaded value.

The final unresolved transfer receives four explicit arguments:

1. 0x72.
2. A word captured from a computed base at offset 0x64.
3. The earlier word captured at doubled-link-base-plus-0x14.
4. A word captured later from the computed base at offset 0x74.

The 0x74 argument is captured before the final intervening writes; it is not a required reread at transfer time.

The representation ends after this transfer with an apparent return. Actual normal return, further dispatch, tail transfer, and non-returning behavior remain **documented_only**.

## Side effects

### Storage relationships

Observed computed offsets are access locations, not a recovered record definition:

| Width | Representative offsets |
|---|---|
| Byte | 0x01, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| Halfword | Doubled or eightfold-scaled destinations |
| Word | 0x04, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

Meaningful absolute locations include 0x01, 0x05, 0x15, 0x21, 0x3A, 0x54, 0x60, 0x67, 0xA9, and the pointer source at 0x087FBBBC.

Byte and word access ranges may overlap. Fixed storage and computed-address storage are not separate namespaces. An aliasing write affects subsequent reads according to its width, position in the sequence, and the unresolved memory model.

### Ownership, errors, and concurrency

No allocation, release, retain, or ownership transfer is identifiable in the direct-memory operations. Dependencies may nevertheless allocate, release, or invalidate storage; no preservation or lifetime guarantee is established.

No rollback, retry, error conversion, or explicit error reporting is visible. Partial effects may survive failure.

No synchronization or atomic-access protocol is established. Thread safety, reentrancy, and guarantees under concurrent mutation are **documented_only**.

## Dependencies

Only F-4addfcaacd01 has a supplied established identity. The following opaque references label unresolved dependency roles, not recovered functions:

| Opaque reference | Role |
|---|---|
| F-UNRESOLVED-001 | Initial execution-context-derived target |
| F-UNRESOLVED-002 | Subsequent dependency-phase targets; potentially multiple distinct functions |
| F-UNRESOLVED-003 | Final indirect-transfer target |

Target equality, actual signatures, global preservation, memory effects, allocation behavior, and ownership obligations remain **documented_only**.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Byte widening:** If primary-base-plus-0x05 yields 0xE7, the subsequent widened word store receives 231, not a sign-extended negative value.

2. **Width-sensitive extraction:** Selecting the low 20 bits of 0xFFFFFFFF and shifting right ten yields 1023. Shifting the original signed value instead does not preserve that extraction.

3. **Early fixed read:** If the read at 0x05 yields 0xE7, subsequent uses of that captured value use numeric base 231. A later write affecting 0x05 does not change the retained value.

4. **Absolute versus relative destination:** The word write at absolute 0x54 must not be redirected to compact-base-plus-0x54 merely because both expressions contain the same offset.

5. **Captured pointer:** If the pointer-source read yields a particular address and later writes change its source storage, repeated-region updates still use the captured address unless execution is otherwise interrupted.

6. **Late alias-sensitive reread:** If an intervening computed word write overlaps 0x67, a later reference to 0x67 must observe the resulting byte under the applicable memory model. It cannot universally reuse the byte initially copied from the fixed word.

7. **Fixed-word independence:** If the captured word at 0xA9 is 0x123456AB, the initial byte write at 0x67 stores 0xAB. This does not reduce later uses of the retained word to eight bits.

8. **Access-width distinction:** A halfword store of 0x3A is not replaceable by a byte store solely because their low bytes agree.

9. **Failure:** If an early dependency does not return normally, later fixed writes and the final transfer are not required.

10. **No invented text:** The constants 0x3A, 0x2F, 0x65, 0x6C, and 0x72 do not require any particular string or encoding.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and object interpretation.
- Complete call-site inventory and argument/result association.
- Target identities, equivalence, and actual ABI.
- Signed overflow, signed left-shift, and variable-distance signed-shift semantics.
- Endianness, alignment behavior, mappings, and low-address validity.
- Complete intervening computed-access trace and exact repetition count.
- Permitted aliasing, storage boundaries, ownership, and lifetimes.
- Dependency effects on globals and memory.
- Final-transfer continuation and termination.
- Determinism, idempotence, reentrancy, and concurrency guarantees.

The fixed-address ordering, once-only pointer capture, and later storage references at 0x67 are established at the represented level. Their concrete alias-dependent values remain unresolved where address values or memory semantics are unknown.

## Evidence

This contract rests solely on static inspection of the supplied representation of F-4addfcaacd01. Prior prose, comments, inferred labels, and reviewer statements are not independent proof.

The represented accesses support the stated widths, fixed-address chronology, retained captures, later 0x67 references, arithmetic selections, repeated updates, and final explicit arguments.

No implementation access, dependency inspection, machine-level validation, runtime tests, or smoke execution has been performed.
