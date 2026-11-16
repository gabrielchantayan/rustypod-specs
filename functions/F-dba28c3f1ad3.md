<!-- Trusted provenance: F-dba28c3f1ad3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dba28c3f1ad3

## Purpose

F-dba28c3f1ad3 performs an ordered sequence of indirect dependency invocations, derives word-sized values and addresses from their results, mutates mixed-width memory, and makes a final four-argument indirect invocation.

The supplied representation does not establish an application-level purpose. It provides no sufficient basis for classifying the operation as initialization, serialization, rendering, allocation, or structure maintenance. Numeric constants resembling character values do not establish text processing.

This contract distinguishes:

- **Established static evidence:** interactions, arithmetic, access widths, and ordering explicitly represented in the supplied material.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior requiring declarations, dependency contracts, target semantics, or other unavailable evidence.

No runtime tests have been performed.

## Interface

### Explicit inputs and outputs

The represented interface accepts three 32-bit inputs and has no explicit return value.

| Input | Established use |
|---|---|
| First word | Passed unchanged to F-dep-001. |
| Second word | Passed unchanged to F-dep-001. |
| Third word | Shifted left by five positions and passed as the third argument to F-dep-001. |

The third input is represented with a signed type. Its shift must not be interpreted as an unrestricted mathematical multiplication without target arithmetic qualifications.

No explicit input is directly dereferenced in the visible body. Dependencies may nevertheless interpret these words as addresses or modify associated storage.

Observable outputs comprise dependency interactions, memory accesses, persistent memory changes, and the final invocation. No status, output length, returned object, or success indicator is established. Normal return requires every preceding operation to complete and the final target to return.

### Implicit inputs

Execution additionally depends on incoming control context, unresolved indirect targets, dependency-produced primary and secondary result values, existing memory, absolute-address storage, and a separately read global base.

Explicit arguments alone do not determine the outcome.

### Preconditions

Completion requires all reached indirect targets and memory accesses to be supported. These include low absolute addresses, dynamically derived addresses, mixed-width accesses, and a word read at the unaligned absolute address 0xA9.

No explicit bounds, alignment, null, target-validity, or address-validity checks establish these conditions. Invalid accesses are not converted into a represented error result.

## Behavior

### Overall ordering

The representation contains an extensive straight-line invocation phase, followed by direct memory manipulation and a final indirect invocation. No explicit conditional branch, retry, rollback, or early-return path appears.

This describes the represented body, not dependency internals or guaranteed runtime reachability. A dependency may branch, modify memory, fail, terminate execution, or not return.

Memory changes are incremental. A later failure does not establish restoration of earlier state. Different address expressions must not be presumed disjoint.

### Dependency-mediated transformations

F-dep-001 receives the initial three arguments described above. Intermediate invocations, collectively referenced as F-dep-002, consume combinations of:

- Primary 32-bit results, frequently shifted left by eight positions.
- Numerically selected lower and upper words of represented 64-bit results.
- Retained results from earlier invocations.
- Masked and shifted fields.
- Explicit zero arguments.
- Secondary result-channel values whose complete calling interface is unresolved.

Splitting a 64-bit result into bits 0–31 and bits 32–63 establishes numeric selection, not memory endianness.

Calls with unused results remain observable interactions. A call displaying no arguments does not establish absence of implicit inputs or a complete zero-input interface. Zero-valued argument derivations likewise do not make their consuming calls removable.

### Widths and arithmetic

The representation distinguishes 8-, 16-, 32-, and 64-bit quantities. Explicit unsigned byte loads widen to values from 0 through 255. Byte and halfword stores retain the low eight and sixteen bits respectively; word stores carry 32 bits.

Masks include fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. These widths do not establish object sizes or buffer lengths.

For conditional examples, **32-bit bit-vector arithmetic** means truncation to 32 bits at each word-sized intermediate and logical right shift where unsigned operands are established. Under that interpretation, a left shift by eight followed by a logical right shift by eight retains only the original low 24 bits. Two successive left shifts by 28 produce zero, without eliminating any intervening or consuming invocation.

Signed shifts, signed multiplication overflow, address overflow, and excessive shift counts require target semantics. Their unrestricted behavior remains **documented_only**; this contract does not impose portable-language guarantees on the representation.

### Address roles used below

These names describe values by origin, not source identifiers or object fields:

| Role | Origin |
|---|---|
| **Retained high-shift base** | A dependency-result lower word shifted left by 25. |
| **Scaled base** | A separate dependency result multiplied by 0x02000000. |
| **Small quantity** | Another result shifted left by eight, masked to fifteen bits, then logically shifted right by six. |
| **Small base** | Twice the small quantity. |
| **Retained shifted word** | An earlier scalar result shifted left by eight. |
| **Final pair** | The last represented 64-bit dependency result before direct stores. |
| **Address-source value** | The value read from the global at absolute address 0x05 and converted to an unsigned 32-bit quantity. |

Under 32-bit bit-vector arithmetic, the small quantity ranges from 0 through 508 in multiples of four and depends only on its source word’s low seven bits. The small base ranges from 0 through 1016 in multiples of eight.

The read at 0x05 has **unknown storage width and unknown source-type extension**. Conversion of its value to an unsigned 32-bit quantity does not establish an unsigned byte load.

### Initial stores and address-source acquisition

The initial memory effects distinguish temporary values from subsequent read-back values.

The function first stores the retained high-shift base as a word at offset 0x14 from the final pair’s upper word. A byte at offset 9 from the retained high-shift base receives the upper word’s low byte. Between that byte write and its overwrite with the retained high-shift base’s low byte, a word store writes the scaled base at offset 0x54 from 256 times the final pair’s lower word.

Next, the word at scaled-base offset 0x14 is cleared. An explicit unsigned byte read at scaled-base offset 5 supplies a later widening word store at offset 0x34. A small-base offset-0x14 clear intervenes between that byte read and widening store.

The scaled-base offset-0x14 word is then read and retained as the **clear read-back**. After another small-base offset-0x14 clear, its low byte is written twice to small-base offset 0x15. The read-back cannot be presumed zero: intervening derived accesses can alias its location.

Only after these operations does the function read the address-source global at 0x05. It then writes the small base’s low byte to absolute byte address 0x01, followed by a byte write of the small quantity’s low byte at address-source-value offset 0x15.

Thus, the address-source read precedes the 0x01 write. An overlapping write to 0x01 cannot retrospectively change the retained address-source value.

### Early absolute-global updates

After address-source acquisition, the scaled-base offset-0x11 byte receives the retained shifted word’s low byte. A word store through clear-read-back offset 0x54 follows, and the scaled-base offset-0x11 byte is then cleared.

The function reads and retains the word at address-source-value offset 0x54; call this the **retained address word**. That same location is overwritten with the clear read-back. A small-base offset-0x34 word store of the small quantity follows.

Further address-chain reads and stores precede the absolute word write at 0x54. They include a read at scaled-base offset 0x74, a store through the resulting address, a small-base offset-0x74 chain, and repeated self-address and byte stores involving the retained address word.

Immediately before the absolute 0x54 write, small-base offset 0x14 receives the dependency result underlying the scaled base shifted left by 26. Absolute word address 0x54 then receives the retained address word shifted left by one—twice that word under the stated bit-vector interpretation. This is one established write whose **value is doubled**, not two established writes to absolute 0x54.

Subsequently, small-base offset 0x14 receives the unscaled retained address word twice. Two byte writes at small-base offset 0x15 carry the low byte of four times the retained address word. Absolute byte address 0x15 then receives the small base’s low byte.

These absolute accesses are interleaved with derived accesses. They cannot be collected into an unordered initialization block because derived storage may overlap them.

### Address chains and literal writes

The following region scales the retained address word by four and eight, uses words read at offsets 0x04, 0x44, 0x64, and 0x74 as addresses, and uses explicit unsigned byte reads at offset 5 in further address formation.

It includes repeated stores, self-address stores, byte-to-word widening, and overwrites of previously written locations. No valid structure layout, graph invariant, or object separation follows from those patterns.

For the later region, define:

- **Adjusted base:** four times the retained address word, minus 0x96.
- **Working base:** four times the adjusted base.
- **Chain word:** the later word obtained at offset 0x74 from an explicitly loaded unsigned byte value.
- **Chain base:** twice the chain word.
- **Literal base:** four times the chain word.

Scaling and subtraction remain subject to the arithmetic qualifications above.

The value 0x3A is written separately as a byte at the literal base, as a halfword at eight times the chain word, and as a word at literal-base offset 0x20. These are three accesses of distinct widths, not an established text operation.

Working-base offset 0x20 receives, in order, word values 0x20, 0x2F, a widened unsigned byte read from working-base offset 4, and 0x20 again. The read occurs after the first two writes.

Only then is the separate global base read and retained. Absolute word address 0x60 receives 0x20 **after that global-base read** and before the following derived stores. If the 0x60 write overlaps the global’s storage, it does not change the already retained base.

### Repeated-store region

The region begins with a word store of the literal base at offset 0x54 from eight times the retained address word. It then writes:

- The working base’s low byte at retained-global-base offset 0x19.
- The retained global base’s low byte at working-base offset 0x11.
- Eight times the retained address word at retained-global-base offset 0x34.

Later repetitions instead use eight times the **adjusted base** for the scaled-address offset-0x54 store and the global-base offset-0x34 value. The byte values remain retained values; this region does not repeatedly reload the separate global base.

The repetition order and multiplicity matter for observable storage. Neither identical values nor absence of intervening calls permits collapsing the region to final state alone. This document does not enumerate its occurrence count; exact numeric trace multiplicity is **documented_only** here.

### Final absolute reads and byte transfers

After the repeated region, the function reads the word at absolute 0xA9 and retains it as the **absolute-source word**. It next reads the word at literal-base offset 0x74.

Absolute byte address 0x3A then receives a computed value derived from the adjusted base: eight times its byte-converted value, plus 0x30, narrowed to a byte. Under bit-vector execution, the resulting low byte equals the low eight bits of eight times the adjusted base plus 0x30.

The function subsequently retains words from chain-base offset 0x14 and working-base offset 4. Before the transfer to absolute 0x67, intervening stores write bytes at chain-base offset 0x0D and working-base offsets 8, 0x0D, and 9, followed by a self-address word at chain-base offset 0x54. The offset-8 literal is 0x6C.

Absolute byte address 0x67 then receives the absolute-source word’s low byte. A word read at chain-base offset 0x44 intervenes before a **fresh byte read of 0x67**, whose value is written to absolute byte address 0x21.

The retained absolute-source word and the byte currently stored at 0x67 are distinct sources. The word is not reread from 0xA9.

Two later byte stores to working-base offset 9 each obtain their value from another fresh read of 0x67. They must not substitute the originally extracted low byte:

- Before the first later read, intervening writes include working-base offset 0x10, retained-global-base offsets 0x54, 0x74, and 0x44, and chain-base offset 1. A word read at working-base offset 0x40 also occurs.
- Between the two later reads, stores affect chain-base offset 9 and working-base offset 0x44, followed by a word read at working-base offset 0x54.

Any intervening derived write may alias 0x67. Each fresh read therefore uses memory at its own read point. Conversely, the later word store of the absolute-source word at working-base offset 0x44 uses the retained word, regardless of changes to 0x67 or 0xA9.

### Final mutations and invocation

A byte store at chain-base offset 1 uses a signed right shift of the word read at chain-base offset 0x44. Its count is the working base’s low byte and may exceed 31. Negative-operand and excessive-count behavior remains **documented_only**.

Working-base offset 0x10 receives a byte before a later word overwrite sourced from offset 0x40. Retained-global-base offset 0x54 likewise receives successive word values rather than one final-only update.

The tail follows additional chains through working-base offsets 0x14 and 0x54 and chain-base offset 4. It writes a halfword 0x65 at twice the word obtained from chain-base offset 4, then reads through that word’s offsets 0x64 and 0x74. Further stores include a word value 100 at a derived offset 0x54 and a word write through twice a retained address value.

F-dep-003 finally receives:

1. Numeric value 0x72.
2. The word read from offset 0x64 of the address obtained at chain-base offset 4.
3. The earlier retained word from chain-base offset 0x14.
4. The later word read from offset 0x74 of the address obtained at chain-base offset 4.

No command, text, event, or ownership interpretation is established for these arguments.

## Side effects

Established direct effects include byte, halfword, and word reads or writes through derived addresses, plus:

| Absolute location | Established operation |
|---|---|
| 0x01 | Byte write of the small base’s low byte. |
| 0x05 | Address-source global read; storage width and extension unknown. |
| 0x15 | Byte write of the small base’s low byte. |
| 0x21 | Byte write sourced from a fresh 0x67 byte read. |
| 0x3A | Computed byte write. |
| 0x54 | Word write of twice the retained address word, arithmetically qualified. |
| 0x60 | Word write of 0x20 after the separate global-base read. |
| 0x67 | Byte write from the retained absolute-source word, followed by fresh byte reads. |
| 0xA9 | Word read retained for later use. |

Access offsets establish no complete object layout, allocation size, or alignment guarantee. Endianness affects overlapping accesses and remains unresolved.

No explicit allocation or release is identifiable. Dependencies have unknown ownership effects; no caller cleanup obligation, transfer rule, or lifetime guarantee is established.

No synchronization, atomicity, fault conversion, or rollback is visible. Thread safety and reentrancy are **documented_only**.

## Dependencies

These opaque IDs are contract-local references, not resolved external identities:

- **F-dep-001:** initial indirect target selected using incoming control context.
- **F-dep-002:** intermediate indirect invocations, potentially different targets and interfaces.
- **F-dep-003:** final unresolved indirect target.

For each, global preservation, memory effects, ownership, failure behavior, and return guarantees remain **documented_only**. Secondary result channels need an additional calling-convention contract.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial truncation.** Under 32-bit bit-vector shifts, a third input of 0x08000001 produces a third initial-call argument of 0x00000020.

2. **Small-address range.** A small-quantity source of either 0x0000007F or 0xFFFFFFFF produces 508 and a small base of 1016 under the stated arithmetic model.

3. **Unknown-width address source.** Acceptance must not require the value read at 0x05 to lie between 0 and 255. Its unsigned 32-bit conversion does not establish byte-sized storage.

4. **Read before absolute write.** If the 0x01 byte write overlaps the address-source global, subsequent addressing still uses the value acquired before that write.

5. **Doubled absolute word value.** If the retained address word is 0x80000001, the absolute 0x54 word write carries 0x00000002 under bit-vector arithmetic. This describes one write of a doubled value.

6. **Fresh 0x67 reads.** If 0x67 initially receives 0xAB and an intervening aliased write changes it to 0xCD, a later transfer reads 0xCD. If another intervening write changes it again, the next transfer uses that newer value.

7. **Retained word versus fresh byte.** Changing 0x67 does not change the retained absolute-source word used in the working-base offset-0x44 word store.

8. **Observable repeated writes.** Storage exposing individual write events distinguishes duplicate writes from a single final-state-equivalent write.

9. **Partial execution.** A fault or nonreturning dependency does not require the final invocation or restoration of earlier writes.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid caller population.
- Resolved targets, complete interfaces, and secondary result-channel semantics.
- Target signed arithmetic, overflow, address formation, and excessive shifts.
- Storage width and source-type extension of the global at 0x05.
- Endianness, alignment rules, address mapping, volatility, and concurrency behavior.
- Complete validity and reachability of the represented path.
- Numeric multiplicity of the repeated region as an enumerated acceptance trace.
- Dependency ownership and global side effects.
- Final-target selection and return behavior.

Structure construction and text processing remain inferred interpretations only.

## Evidence

Evidence consists solely of the supplied static function representation and dependency map. Comments, prior prose, feedback, and displayed annotations are not independent confirmation.

The represented accesses distinguish the unknown-width 0x05 global read from explicit unsigned byte loads. They also distinguish the retained 0xA9 word from successive reads of byte storage at 0x67 and establish the relative placement of the absolute-global operations described above.

No external implementation access, runtime tests, or execution traces have been used.
