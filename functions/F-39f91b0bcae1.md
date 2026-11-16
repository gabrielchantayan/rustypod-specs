<!-- Trusted provenance: F-39f91b0bcae1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-39f91b0bcae1

## Purpose

F-39f91b0bcae1 performs an ordered sequence of indirect dependency invocations, derives values from their results and ambient machine state, performs mixed-width memory mutations, and reaches an unresolved four-argument control transfer.

No application-level purpose is established. Character-like constants do not establish text processing; recurring offsets do not establish objects, collections, or a protocol.

This contract separates:

- **Established static evidence:** represented interactions, widths, data relationships, and access ordering.
- **[INFERENCE]:** interpretations not required for acceptance.
- **documented_only:** unresolved behavior or explicitly unenumerated detail.

All behavior is conditional on reaching the relevant operation. Successful access, dependency return, and ordinary completion are not guaranteed.

## Interface

### Inputs and outputs

The represented interface declares no parameters and no return value. Effective inputs nevertheless include:

- Incoming control state used to calculate the first indirect target.
- An ambient 32-bit machine-state value used in shifted arguments.
- Primary results, 64-bit results, and secondary machine-state outputs of dependencies.
- Contents of computed-address and fixed-location memory.
- The unresolved terminal target.

Observable outputs are dependency interactions, memory reads and writes, and the terminal transfer. No success code, error object, output-buffer contract, or length result is established.

### Widths and numeric interpretation

| Quantity | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access and most arithmetic | 32 bits |
| Selected dependency results | 64 bits |

Unsigned byte loads contribute values from 0 through 255. Narrowing a word for a byte store retains its low eight bits. Numeric extraction of high and low words from a 64-bit result does not establish an aggregate-return memory layout or byte order.

For this document, **32-bit bit-vector arithmetic** means arithmetic retaining only the low 32 bits. Conclusions requiring that interpretation are conditional. Signed-looking shifts, multiplication, subtraction, address conversion, and overflow are not independently resolved into portable language semantics.

### Preconditions, lifetime, and ownership

Every executed target must support the invocation, and every executed address must support the represented access width. No explicit validation establishes null safety, alignment, accessible extent, overflow safety, or dependency success.

Small addresses, including fixed locations near zero, are used directly. Ordinary process-memory assumptions are not sufficient preconditions.

No allocation, release, reference counting, or ownership transfer is established. Dependencies may alter globals and memory lifetimes. Their missing implementations do not imply preservation, harmlessness, or responsibility for releasing resources.

## Behavior

### Dependency interactions

The initial phase is a long straight-line sequence of indirect invocations, without an explicit represented loop or conditional branch.

The first target is calculated from incoming control state plus 0xFFE bytes. Subsequent targets use represented continuation-like constants or symbolic locations with the same displacement. This calculation pattern does not identify dependency behavior or establish that similar sites share a target.

The first three invocations have no explicit arguments. The third primary result is shifted left by eight and retained. The fourth invocation receives the ambient word shifted left by four. A later invocation supplies the retained shifted result, shifted left by seven again, in two argument positions.

Subsequent interactions consume combinations of:

- Primary words shifted left, frequently by eight.
- Unmodified high words of 64-bit results.
- Independently transformed low and high words.
- Retained results carried across intervening invocations.
- Repeated argument values.
- Explicit zeros.
- Secondary machine-state outputs whose producers are not fully expressed by the declared interfaces.

An invocation with no explicit arguments remains an interaction. It does not prove that incoming machine state is irrelevant. Similar argument shapes do not justify merging, removing, or reordering invocations.

### Numeric constraints

Visible transformations include masks selecting one-bit and low 11-, 12-, 13-, 14-, 15-, 16-, 17-, 18-, 19-, 20-, 21-, 22-, and 24-bit portions. Masking before shifting remains significant.

Unsigned right shifts are logical. Right shifts of unsigned 64-bit results by 34, 36, and 44 correspond to logical right shifts of their original high words by two, four, and twelve.

Under 32-bit bit-vector arithmetic:

- A left shift by eight followed by a logical right shift by eight retains the original low 24 bits.
- Two successive left shifts by 28 yield zero.
- A left shift by 31 followed by a left shift by 30 yields zero.
- Locally retained zero values remain zero through subsequent local shifts and masks.

These local relationships do not establish preservation of unrelated globals or memory by dependencies.

A late signed-looking right shift uses the low byte of a derived address as its count. Counts can exceed 31. Oversized-count handling and signed right-shift behavior remain documented_only; no masking or saturation rule is imposed.

### Memory-phase terminology

The following descriptive names identify data relationships, not object types:

| Name | Meaning |
|---|---|
| **shift-derived base** | A retained dependency result multiplied by 2²⁵ |
| **retained shifted base** | A retained low dependency-result word shifted left by 25 |
| **small component** | A later result shifted left by eight, masked to 15 bits, then logically shifted right by six |
| **small base** | Twice the small component |
| **captured base** | A word captured from offset 0x54 of an early byte-derived base |
| **fourfold base** | Four times the captured base |
| **eightfold base** | Eight times the captured base |
| **adjusted value** | The fourfold base reduced by decimal 150 |
| **derived base** | Four times the adjusted value |
| **publication base** | A word loaded from fixed publication storage |
| **paired base** | Twice a later word loaded through a byte-derived address |
| **constant-write base** | Four times that same later loaded word |

Under 32-bit bit-vector arithmetic, the small component is a multiple of four from 0 through 508; the small base is a multiple of eight from 0 through 1,016.

### Initial mutation and fixed-location effects

The final dependency result before direct mutation supplies low and high words. A word store at high-word address offset 0x14 publishes the retained shifted base. A byte store at retained-shifted-base offset 0x09 then publishes the high word’s low byte. A word store through the latest low word multiplied by 256, at offset 0x54, publishes the shift-derived base. A second byte store at retained-shifted-base offset 0x09 publishes that retained base’s own low byte.

The shift-derived base’s word at offset 0x14 is cleared before its unsigned byte at offset 0x05 is read. Between that byte read and the subsequent word read at offset 0x14:

- Small-base offset 0x14 is cleared.
- The captured byte is extended unsigned and stored as a word at shift-derived-base offset 0x34.

After the offset-0x14 word is captured, small-base offset 0x14 is cleared again. The captured word’s low byte is written twice to small-base offset 0x15.

The function next reads the byte at **fixed location 0x05** and uses its unsigned value as a base. **After this read**, it writes the small base’s low byte to **fixed location 0x01**. Subsequent accesses through the byte-derived base therefore use the captured byte, not a fresh read of fixed location 0x05.

The byte-derived base receives the small component’s low byte at offset 0x15. Further updates include two byte writes to shift-derived-base offset 0x11—first from a retained shifted result, then zero—with a computed-address word store between them.

The byte-derived base’s word at offset 0x54 is captured as the captured base before that location is overwritten. Small-base offset 0x34 receives the small component.

Later, a scaled retained value is written to small-base offset 0x14. A **32-bit write at fixed location 0x54** then publishes twice the captured base. Only after that fixed-location write is small-base offset 0x14 overwritten with the captured base, twice. The fixed word write is not an independent or deferrable side effect: it lies between computed-address word stores and may alias them.

After two further byte writes at small-base offset 0x15, a byte write at **fixed location 0x15** publishes the small base’s low byte. It precedes the ensuing scaled-address word updates, including stores at fourfold-base offset 0x54 and small-base offset 0x34.

No non-aliasing assumption permits these fixed-location operations to be omitted or moved.

### Computed-address propagation

The intervening propagation region uses word offsets 0x04, 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74, together with byte offsets 0x05, 0x09, 0x0D, 0x11, and 0x15.

Its visible relationships include:

- Capturing the shift-derived base’s word at offset 0x74 and updating through that loaded base.
- Repeated self-value word stores at captured-base offset 0x24, interleaved with byte stores through another loaded base.
- Propagating fourfold and eightfold captured-base values through offsets 0x14, 0x34, 0x44, and 0x54.
- Reading unsigned bytes at offset 0x05 and treating them as address components for word reads at offset 0x74.
- Re-reading an offset-0x05 byte after intervening mutations rather than retaining one universal value.
- Loading the fourfold base’s word at offset 0x04 and subsequently accessing memory through that captured word.

One update contains two distinct word reads from small-base offset 0x74: one supplies a destination base, the other the stored value. They must not be collapsed into one read. Their relative evaluation order within that combined update is not independently resolved.

Decimal offset 100 means **0x64**, not 0x100.

### Constants and publication region

A later word obtained through a byte-derived address determines the paired base and constant-write base. The function writes numeric 0x3A as:

- An 8-bit value at the constant-write base.
- A 16-bit value at twice the constant-write base.
- A 32-bit value at constant-write-base offset 0x20.

At derived-base offset 0x20, the ordered word-write sequence is 0x20, 0x2F, an unsigned byte read from derived-base offset 0x04, and 0x20. The byte read occurs after the 0x2F write and before the corresponding propagation write.

The **publication-base word is loaded before** a 32-bit write of 0x20 to **fixed location 0x60**. Later publication operations use that captured base. Even if the fixed write aliases its storage, the base is not reloaded.

An initial publication group writes the constant-write base to eightfold-base offset 0x54, then writes:

- The derived base’s low byte to publication-base offset 0x19.
- The publication base’s low byte to derived-base offset 0x11.
- The eightfold base as a word to publication-base offset 0x34.

Subsequent repeated groups use eight times the adjusted value, rather than the eightfold captured base, for both the offset-0x54 destination base and the word published at publication-base offset 0x34. Within each group, the word store precedes the offset-0x19 byte store, which precedes the offset-0x11 byte store, which precedes the offset-0x34 word store.

Repeated writes remain separate represented effects. Their multiplicity is not enumerated by this contract; that detail is documented_only, not permission to coalesce the region.

### Final capture interleaving

The final region begins with this access ordering:

1. Capture the 32-bit word at **fixed location 0xA9**.
2. Capture the word at constant-write-base offset 0x74.
3. Write a byte at **fixed location 0x3A**, retaining the low eight bits of eight times the adjusted value plus 0x30.
4. Capture the word at paired-base offset 0x14.
5. Capture the word at derived-base offset 0x04.

Thus the offset-0x14 and offset-0x04 captures occur **after** the fixed byte mutation. They are not members of an immutable pre-mutation snapshot.

The function then writes the derived base’s low byte at paired-base offset 0x0D; writes 0x6C at derived-base offset 0x08; writes the publication base’s low byte at derived-base offsets 0x0D and 0x09; and writes the paired base itself as a word at paired-base offset 0x54.

### Final fixed-byte flow and later mutations

The low byte of the captured fixed-location word from 0xA9 is written to **fixed location 0x67**. A word is then read from paired-base offset 0x44. Only afterward is the **current byte at 0x67** read and written to **fixed location 0x21**.

The word captured from paired-base offset 0x44 supplies the next byte store at derived-base offset 0x10 and word store at publication-base offset 0x54. Its shifted low-byte result is then stored at paired-base offset 0x01, using the unresolved shift semantics described above. Publication-base offset 0x54 is subsequently overwritten with the derived base.

Next, the word at derived-base offset 0x40 is captured. The previously captured derived-base offset-0x04 word is written to publication-base offset 0x74. The offset-0x40 capture is written as a **word** at derived-base offset 0x10, after the earlier byte store there. Publication-base offset 0x44 receives the paired base.

There are then two additional, distinct reads of fixed location 0x67:

- The first supplies a byte store at derived-base offset 0x09. After that store, paired-base offset 0x09 receives the derived base’s low byte, and derived-base offset 0x44 receives the captured full word from fixed location 0xA9.
- A word is then captured from derived-base offset 0x54 as a **loaded destination base**. Only after this word read is fixed location 0x67 read again, supplying another byte store at derived-base offset 0x09.

The three reads of 0x67—one for fixed location 0x21 and two for computed-address stores—are not interchangeable with the low byte initially written there. Intervening accesses may alias 0x67 or otherwise affect its current contents.

The low byte of the earlier constant-write-base offset-0x74 capture is written at loaded-destination-base offset 0x0D. Paired-base offset 0x54 receives the previously captured paired-base offset-0x14 word.

After a word is captured from derived-base offset 0x14, further stores publish the earlier offset-0x74 capture’s low byte at paired-base offset 0x11, then the loaded destination base at derived-base offset 0x09 as a byte and offset 0x44 as a word.

### Terminal propagation and transfer

A word read from paired-base offset 0x04 becomes the **late base**. Before reading late-base offset 0x64, the function performs a 16-bit store of 0x65 through twice the late base.

The late-base offset-0x64 word is retained as the second terminal argument. Before the late-base offset-0x74 word is read, the function writes the derived-base offset-0x14 capture’s low byte at loaded-destination-base offset 0x11 and writes decimal 100 as a word at loaded-destination-base offset 0x54.

After capturing late-base offset 0x74, it reads paired-base offset 0x64 as another destination base. It then writes the late base’s low byte at paired-base offset 0x09, publishes the retained second argument at that other destination base’s offset 0x54, and finally stores that destination base through twice the retained second argument.

The terminal target receives, in order:

1. 0x72.
2. The retained late-base offset-0x64 word.
3. The earlier paired-base offset-0x14 capture.
4. The retained late-base offset-0x74 word.

The representation includes a return afterward. Whether the transfer actually returns, dispatches elsewhere, or has different continuation semantics remains documented_only.

## Side effects

The function exposes mixed-width, potentially overlapping accesses. Recurring offsets establish an access vocabulary, not field names, structure size, allocation extent, or independent storage.

Meaningful fixed-location effects include the early byte read at 0x05, byte writes at 0x01 and 0x15, word writes at 0x54 and 0x60, the late word read at 0xA9, byte writes at 0x3A and 0x67, three subsequent reads of 0x67, and the byte write at 0x21. Their interleavings are specified above.

No atomicity, synchronization, reentrancy, or thread-safety guarantee is established. Trace requirements concern represented operations; whether each operation is externally observable depends on the execution environment.

There is no explicit rollback or recovery. A nonreturning dependency or failed access can leave all earlier completed effects intact.

## Dependencies

F-39f91b0bcae1 is the only supplied opaque function ID. No opaque IDs or contracts identify its indirect dependencies. No additional identities are invented.

Established dependency interactions comprise ordered invocation, transformed explicit arguments, primary-result consumption, high-word extraction, secondary-output consumption, and the terminal four-argument transfer.

Dependency memory effects, global preservation, register preservation, ownership, exceptions, termination, and calling conventions remain documented_only.

## Acceptance examples

These are conditional examples, not executed tests.

### Unsigned extension and truncation

If the relevant offset-0x05 byte is 0xE7, its word propagation is 0x000000E7, not a sign-extended value.

Under 32-bit bit-vector semantics, shifting 0xAB123456 left by eight and then logically right by eight yields 0x00123456. A 64-bit unsigned result whose high word is 0x80000000 yields 0x20000000 when shifted right by 34 and narrowed.

### Fixed-location ordering

An acceptable initial trace preserves the fixed-0x05 byte read before the fixed-0x01 byte write. It also preserves the fixed-0x54 word write between the specified small-base offset-0x14 word stores and the fixed-0x15 byte write before subsequent scaled-address updates.

If fixed location 0x60 aliases publication-base storage, publication still uses the word captured before the 0x20 write. Reloading afterward is not equivalent.

### Capture after mutation

Suppose fixed location 0x3A lies within either subsequent capture’s storage. Acceptance requires the paired-base offset-0x14 and derived-base offset-0x04 reads to occur after that byte write. Values captured before it cannot substitute universally. Exact affected word values depend on byte order and overlap.

### Changing fixed-byte contents

Suppose the three specified reads of 0x67 return 0xA1, 0xB2, and 0xC3. Fixed location 0x21 receives 0xA1; the two later derived-base offset-0x09 stores receive 0xB2 and 0xC3 respectively. Intervening memory effects remain in place. The example does not assert that these are the only stores to offset 0x09.

### Width and overwrite visibility

The 0x3A operations require distinct 8-, 16-, and 32-bit stores. Three byte stores do not satisfy that contract.

The derived-base offset-0x20 sequence preserves 0x20, 0x2F, the intervening unsigned-byte propagation, and 0x20. A matching final word alone is insufficient.

### Partial execution and transfer

If an early dependency never returns, later memory mutation and terminal transfer are not required.

If the terminal target is reached, its third argument is the earlier paired-base offset-0x14 capture, not a fresh read after the subsequent stores.

## Uncertainties

The following remain **documented_only**:

- Application purpose and accuracy of the represented function boundary.
- Indirect dependency identities, ABI details, and some secondary-output producers.
- Address mapping, alignment support, byte order, and actual memory-access observability.
- Signed-overflow, address-arithmetic, oversized-shift, and signed-right-shift semantics.
- Dependency-induced ownership and lifetime changes.
- Relative evaluation order of the two reads within the combined offset-0x74 update.
- The unenumerated publication-region repetition count and complete per-site dependency trace.
- Actual continuation after the terminal transfer.

These limitations do **not** leave the specified fixed-location interleavings optional. Unknown aliasing requires preserving them, not treating storage as independent.

**[INFERENCE]** Recurring offsets may suggest structured state manipulation. No object semantics or invariants are established.

## Evidence

The supplied static representation supports the described widths, constants, arithmetic relationships, captured-value reuse, fixed-location effects, and ordered access boundaries.

Prior prose, reviewer feedback, comments, warnings, and symbolic names are not independent behavioral authority. Claims above are based on the represented operations rather than those annotations.

No dependency implementations, instruction-level semantics, ABI documentation, or executable observations were supplied. No runtime tests have been performed.
