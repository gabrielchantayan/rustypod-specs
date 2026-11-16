<!-- Trusted provenance: F-411bafadeff1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-411bafadeff1

## Purpose

F-411bafadeff1 performs an extensive ordered sequence of indirect calls, transforms captured results, updates memory using mixed access widths, and finishes with an unresolved indirect control transfer.

No application-level purpose is established. The numeric constants written near the end do not establish formatting, text output, serialization, initialization, or object management.

This contract separates:

- **Established static evidence:** behavior expressly represented in the supplied function body.
- **[INFERENCE]:** conclusions conditional on a specified arithmetic or execution model.
- **documented_only:** unresolved behavior or detail not fully specified here.

All ordering below describes the supplied static representation, conditional on execution reaching the relevant operations. No runtime tests have been performed.

## Interface

### Inputs

The represented interface has no explicit parameters. It nevertheless consumes substantial implicit state:

| Input | Width or form | Role |
|---|---|---|
| Incoming control-link value | Address-sized, represented with 32-bit arithmetic | Determines the opening indirect destination after adding 0xFFE. |
| Incoming working word | Unsigned 32-bit word | Supplies shifted arguments to opening dependencies. |
| Primary dependency results | 32-bit words or represented 64-bit values | Supply arguments, retained data, and address components. |
| Auxiliary dependency-result state | Primarily 32-bit words | Supplies additional arguments outside ordinary assigned results. |
| Existing memory | 8-, 16-, and 32-bit accesses | Supplies data and further effective addresses. |
| Fixed-location state | Bytes and 32-bit words | Participates in address selection, repeated writes, and final updates. |
| Final indirect destination | Unresolved executable destination | Receives four represented arguments. |

An empty displayed argument list does not establish that a dependency has no implicit inputs.

### Outputs and completion

There is no established scalar return value, status code, or success indicator. Observable outputs include dependency calls, memory accesses, dependency effects, and the final transfer.

The representation includes a return after treating the final transfer as a call. Whether the underlying transfer returns normally is **documented_only**.

### Preconditions

A completing execution requires valid implicit entry state, executable reached destinations, and accessible memory at every reached address and width. Alignment, address mapping, calling conventions, and any device-memory requirements must be satisfied by the environment.

No null checking, bounds checking, address validation, or failure recovery is represented. Effective addresses are not established as checked array elements or valid object fields.

## Behavior

### Control and arithmetic model

The body has no explicit conditional branches or loops in the supplied representation. Its major phases are indirect calls, direct memory updates, and the final transfer. Dependencies may prevent later operations from being reached.

Most indirect destinations use embedded address-like values or entry references plus 0xFFE. The opening destination instead depends on the incoming control-link value. Distinct call sites must not be merged into one dependency identity without evidence.

The opening primary result is shifted left by eight for the next displayed call. Other opening arguments are the incoming working word shifted left by 29 and that word logically shifted right by 22, then left by 29. Later arguments include captured primary results, upper and lower portions of 64-bit results, auxiliary result state, explicit zeros, masks, and shifts.

The working width is principally 32 bits. Separating a represented 64-bit result into upper and lower 32-bit portions does not establish an eight-byte memory layout or byte order.

Masks retain fields ranging from 11 to 24 bits; shifts extend through 31 bits. Masking, shifting, and truncation order must be preserved. Several captured words are explicitly zeroed before later transformations; under fixed-width arithmetic their shifted values remain zero, but surrounding calls remain potentially observable.

**[INFERENCE]:** A 32-bit bit-vector interpretation gives truncating left shifts and arithmetic modulo \(2^{32}\). Some represented operations instead have signed arithmetic forms. Overflow, negative operands, and oversized signed shifts require actual instruction semantics; this document does not impose portable-language behavior on them.

### Memory-phase terminology

The following are descriptive contract labels, not recovered identifiers or type declarations:

| Label | Meaning |
|---|---|
| Scaled base | A saved call result multiplied by 0x02000000. |
| Compact index | A saved result shifted left by eight, masked with 0x7FFF, then logically shifted right by six. |
| Small base | Twice the compact index. |
| High-shift base | A saved lower result word shifted left by 25. |
| Saved shifted word | An earlier primary result shifted left by eight. |
| Terminal pair | The lower and upper words of the final represented 64-bit result before direct stores. |
| Retained word | The word captured from the byte-derived base at offset 0x54, as described below. |
| Fourfold base | Four times the retained word. |

Under 32-bit bit-vector arithmetic, the compact index ranges from 0 through 508 in multiples of four. The small base ranges from 0 through 1016 in multiples of eight. Low-address accesses are therefore possible.

All offsets below are byte offsets. A “word” access is 32 bits; a byte is eight bits; a halfword is 16 bits. Stored bytes retain only the low eight bits.

### Opening memory updates

The first word store uses the terminal upper word plus 0x14 as its address and receives the high-shift base. The byte at high-shift base plus nine then receives the terminal upper word’s low byte. Between that byte write and its overwrite with the high-shift base’s low byte, a word at 256 times the terminal lower word plus 0x54 receives the scaled base.

The scaled-base word at offset 0x14 is cleared before reading the byte at scaled base plus five. Next, the small-base word at offset 0x14 is cleared, and the captured byte is widened unsigned into the scaled-base word at offset 0x34. The scaled-base word at offset 0x14 is then read and retained temporarily. Another small-base clear follows, then two consecutive byte writes to small base plus 0x15 receive that captured word’s low byte.

The function next reads the byte at absolute address 0x05 and writes the small base’s low byte to absolute address 0x01. The captured byte becomes a base: its offset-0x15 byte receives the compact index’s low byte.

The scaled-base byte at offset 0x11 receives the saved shifted word’s low byte. A word addressed by the captured offset-0x14 value plus 0x54 receives the saved shifted word, after which the scaled-base byte at offset 0x11 is cleared.

The word at the byte-derived base plus 0x54 is captured as the retained word before that location is overwritten with the earlier offset-0x14 value. The small-base word at offset 0x34 receives the compact index.

### Linked updates and absolute-address effects

A word read at scaled base plus 0x74 supplies a linked base. Its offset-0x54 word first receives the original scaled-base-producing result shifted left by 26.

A separate nested access reads the small-base word at offset 0x74 twice, using one result as an address component and one as stored data. The relative evaluation order of those two reads is unresolved; replacing them with one read is not justified.

The retained word then supplies both addresses and data. Its offset-0x24 word receives the retained word itself twice, with other writes between those stores. The linked base’s offset-nine byte twice receives the retained word’s low byte; its offset-0x54 word is overwritten with the saved shifted word between those byte writes.

The following ordering includes fixed-location effects that are not ordinary fields:

- The small-base word at offset 0x14 receives the original result shifted left by 26.
- A **32-bit store to absolute address 0x54** receives the retained word shifted left by one.
- The small-base word at offset 0x14 receives the retained word, then receives it again.
- Two consecutive byte stores to small base plus 0x15 receive the low byte of four times the retained word.
- A **byte store to absolute address 0x15** receives the small base’s low byte.

These absolute stores remain distinct when their addresses coincide with derived locations. In particular, a zero small base makes absolute 0x15 identical to small base plus 0x15, so the absolute write overwrites the preceding byte writes.

Subsequent linked updates use twice, four times, and eight times the retained word. The fourfold-base offset-0x54 word receives the saved shifted word, then fourfold-derived values. The small-base offset-0x34 word receives the retained word shifted left by two.

A fresh small-base offset-0x74 read, doubled, supplies another link. That link is written through retained-word and byte-derived chains at offset 0x44. Reads through fourfold-base offsets 0x04 and 0x64 select further bases. Bytes at offsets five are read separately at different points; intervening writes mean these reads must not be cached together.

The accessed offsets include byte locations 0x01, 0x05, 0x09, 0x0D, 0x11, 0x15, and 0x19, and word locations 0x04, 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74. This is an access inventory, not evidence of one structure layout or allocation extent.

### Constants and repeated-write run

A later fresh byte read selects a base whose offset-0x74 word is read. Call this captured value the **publication link**. Its double is the **doubled link**, and its quadruple is the **publication value**.

The numeric value 0x3A is written using three distinct widths:

- A byte at four times the publication link.
- A halfword at eight times the publication link.
- A word at four times the publication link plus 0x20.

The **adjustment seed** is the fourfold base minus 0x96. The **adjusted base** is four times that seed. At adjusted base plus 0x20, successive word stores receive 0x20, 0x2F, a freshly read byte from adjusted base plus four widened unsigned, and 0x20 again.

The next fixed-location dependency is explicit: the function reads a **32-bit stored base from absolute address 0x087FBBBC before writing the 32-bit value 0x20 to absolute address 0x60**. Call the captured result the **loaded base**. Later writes do not cause this base to be reloaded.

Before the repeated groups, a word at eight times the retained word plus 0x54 receives the publication value. The first publication group writes, in order:

- The adjusted base’s low byte to loaded base plus 0x19.
- The loaded base’s low byte to adjusted base plus 0x11.
- Eight times the retained word to loaded base plus 0x34.
- The publication value to eight times the adjustment seed plus 0x54.

Following groups preserve the byte-write relationship but store eight times the adjustment seed at loaded base plus 0x34. Complete groups also repeat the offset-0x54 publication-value store. The final group ends after the offset-0x34 word store, without that trailing offset-0x54 store.

The exact expanded repetition count is **documented_only** here. It is not permission to select an arbitrary count or collapse repeated accesses.

### Final reads and updates

Immediately after the repeated-write run, the function reads a **32-bit word at absolute address 0xA9**. This precedes the word read at publication value plus 0x74 and the byte write to absolute address 0x3A.

The 0x3A byte receives eight times the adjustment seed plus 0x30, truncated to eight bits. After that store, the function captures the doubled-link word at offset 0x14 and the adjusted-base word at offset four. The former is retained for restoration and the final transfer.

Further byte stores place the adjusted base’s low byte at doubled link plus 0x0D, 0x6C at adjusted base plus eight, and the loaded base’s low byte at adjusted-base offsets 0x0D and nine. The doubled-link word at offset 0x54 temporarily receives the doubled link itself.

The absolute-address 0x67 accesses have separate capture points:

1. Absolute 0x67 receives the low byte of the word previously captured at absolute 0xA9.
2. A word is read at doubled link plus 0x44.
3. A fresh byte read from absolute 0x67 supplies the byte written to absolute 0x21.

After that copy, the newly captured offset-0x44 word supplies a byte write at adjusted base plus 0x10, a word write at loaded base plus 0x54, and a signed-right-shift-derived byte at doubled link plus one. The shift count is the adjusted base’s low eight bits; unresolved instruction semantics apply to negative operands and large counts.

The loaded-base offset-0x54 word is then overwritten with the adjusted base. A word read at adjusted base plus 0x40 precedes copying it to adjusted base plus 0x10. Between that read and copy, loaded base plus 0x74 receives the earlier captured adjusted-base offset-four word. Loaded base plus 0x44 then receives the doubled link.

Only now does a **new read of absolute 0x67** supply a byte store to adjusted base plus nine. Following it, doubled link plus nine receives the adjusted base’s low byte; adjusted base plus 0x44 receives the captured 0xA9 word; and adjusted base plus 0x54 is read to obtain a new linked base. A **second new read of absolute 0x67** then supplies another byte store to adjusted base plus nine.

Neither later 0x67 read is guaranteed to equal the initial 0x67 store or the value copied to 0x21. Intervening derived accesses may overlap 0x67 or have device effects. These are fresh reads, not retained-byte reuse.

The publication-value offset-0x74 word captured earlier supplies low-byte updates at the new linked base plus 0x0D and doubled link plus 0x11. The doubled-link offset-0x54 word is restored from its earlier captured value. A fresh adjusted-base offset-0x14 read supplies another later byte update.

Final linked operations include a halfword store of 0x65 at twice a newly read base, a word store of decimal 100 at the new linked base plus 0x54, and fresh reads through offsets 0x64 and 0x74. Additional word stores update a reached offset-0x54 location and twice a captured offset-0x64 value.

The final indirect destination receives these four represented arguments, in order:

1. 0x72.
2. The word captured through the final newly selected base’s offset 0x64.
3. The earlier saved doubled-link word at offset 0x14.
4. The word freshly captured through that selected base’s offset 0x74.

## Side effects

Direct effects include mixed-width memory writes, repeated overwrites, absolute-address accesses, and captured-value-dependent address selection. No direct allocation, release, reference counting, ownership transfer, or lifetime extension is established.

Dependencies have unavailable contracts. They must not be assumed to preserve globals, implicit state, memory, or ownership.

Aliasing is material throughout. Byte stores at offset 0x15 overlap word accesses at offset 0x14 for the same base. Absolute locations may overlap dynamically selected accesses. Word interpretation after overlapping writes depends on endianness, which is unresolved.

Repeated accesses remain part of the represented trace even when their final ordinary-memory value is unchanged. Device behavior, faults, or aliasing may expose intermediate effects.

No transaction, rollback, explicit diagnostic, retry, or recovery path is represented. If an operation fails or a dependency does not return, earlier effects may remain.

## Dependencies

The supplied identity map establishes only **F-411bafadeff1**. It provides no opaque function IDs for the indirect destinations.

Accordingly, dependency identities, complete signatures, auxiliary-result conventions, ownership behavior, and final-transfer return conventions are **documented_only**. No replacement IDs or semantic dependency names are invented.

Displayed result consumption establishes required widths at particular uses, not complete callable interfaces. Calls with unused displayed results may still have observable effects.

## Acceptance examples

These are conditional requirements, not observed executions.

### Small-base alias with absolute 0x15

If the compact index is zero, the small base is zero. The two linked-phase byte writes to small base plus 0x15 and the following absolute-0x15 write target the same byte. Acceptance requires both initial writes followed by the small base’s low byte, zero. Keeping only the first pair is incorrect.

### Absolute 0x54 ordering

If the retained word is one and 32-bit shifting is defined as truncating, the absolute-0x54 word store receives two. It occurs after the specified small-base offset-0x14 high-shift store and before the subsequent retained-word stores there. Treating absolute 0x54 merely as an offset in an unrelated object loses this requirement.

### Global-read boundaries

An accepted access trace loads the stored base at 0x087FBBBC before the absolute-0x60 word store. At the end of the repeated run, it reads the word at 0xA9 before the publication-value offset-0x74 read and the absolute-0x3A byte store. Moving either global read across those boundaries is unsupported.

### Fresh values from absolute 0x67

Suppose the initial 0x67 write stores 0x12, while intervening aliased operations make the later two reads yield 0x34 and 0x56. The corresponding adjusted-base offset-nine stores must receive 0x34 and 0x56, not a cached 0x12. This example assumes those memory effects are possible in the selected environment.

### Width and captured-value preservation

A byte store from 0x12345678 stores 0x78. A byte widened into a word contributes an unsigned value from zero through 255. Neither operation justifies a wider copy.

If the doubled-link offset-0x14 memory changes after its initial final-phase capture, the final transfer still receives the captured word as its third argument, not a replacement reread.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention, implicit entry requirements, and complete dependency identities.
- Authenticity and completeness of the represented control-flow boundary.
- Full indirect-call trace at dependency-identity granularity.
- Exact expanded repeated-write count in this contract.
- Dependency side effects, failures, memory ownership, and preservation guarantees.
- Endianness, alignment requirements, mapped ranges, and device semantics.
- Signed-overflow, address-wrap, and oversized-shift instruction behavior.
- Object layouts, allocation extents, alias relationships, and normal-return guarantees.
- Application-level meaning of constants and memory updates.

These limits prevent treating this document as a complete executable replacement specification.

## Evidence

The supplied function body supports the stated access widths, constants, separate capture points, represented operation order, and final argument ordering. In particular, it expressly represents the absolute 0x54 and 0x15 stores, the stored-base read before the absolute 0x60 store, the absolute 0xA9 read, and separate post-copy reads of absolute 0x67.

Prior prose, reviewer claims, comments, warnings, and apparent naming patterns are not independent evidence of intent. Arithmetic conclusions are conditional where machine semantics are unavailable.

No runtime execution, dependency execution, memory observation, or test validation has been performed.
