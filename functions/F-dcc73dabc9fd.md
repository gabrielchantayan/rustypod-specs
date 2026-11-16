<!-- Trusted provenance: F-dcc73dabc9fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dcc73dabc9fd

## Purpose

F-dcc73dabc9fd accepts two 32-bit inputs, invokes a fixed sequence of unresolved dependencies, derives numeric addresses from dependency results and memory contents, performs ordered memory mutations, and makes a terminal indirect invocation.

No application-level purpose is established. Printable constants do not establish strings, formatting, or serialization. The address-relative stores suggest relationship publication; that is **inferred intent**, not a required object model.

This contract describes the supplied static representation, conditional on its executable fidelity. It does not guarantee that ordinary execution can access the represented low addresses or satisfy the unresolved calling conventions.

## Interface

### Inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | Passed unchanged to the first invocation. |
| Second input | 32 bits, represented as signed | Passed unchanged to the first invocation and shifted left by six bits for its third displayed argument. |

Neither input is established as a pointer, length, handle, or ownership token.

Implicit inputs include incoming control-link state, primary and secondary dependency-result words, fixed-location storage, calculated-address memory, and the terminal invocation target. The explicit inputs alone do not determine behavior.

The first target is formed from the incoming control-link value plus 0xFFE. Later targets use represented code-address relationships. An invocation without displayed arguments is not proven to lack implicit inputs.

### Outputs and preconditions

The declared interface returns no value. No success flag, error code, output length, or returned allocation is established.

Reaching normal completion requires compatible executable targets, defined result words, accessible memory for every reached access, and platform support for the represented address calculations and access widths. Dependency-specific preconditions are **documented_only**. No visible validation establishes these conditions.

Completion also requires the terminal target to return. Termination is not guaranteed.

## Behavior

### Arithmetic and dependency phase

The first invocation receives the original inputs and the second input shifted left by six. An extensive, statically ordered invocation sequence follows before the direct-memory phase. Its arguments include transformed earlier results, repeated values, secondary result words, and literal zeros. Earlier results remain live across later invocations.

Established transformations include masked-field extraction, fixed left shifts, unsigned right shifts, and separation of represented 64-bit results into low and high 32-bit words. Some explicitly zeroed intermediates remain zero through subsequent transformations; this does not make intervening calls removable.

Unsigned 64-bit right shifts by 34, 36, and 44 select the high word shifted right by 2, 4, and 12 respectively. Retaining twenty bits, shifting right by ten, then left by six produces multiples of 64 from 0 through 65,472. Masking must precede shifting.

Byte and halfword stores retain eight and sixteen bits respectively. Word accesses cover 32 bits. Byte reads used as numeric bases are zero-extended.

Some represented signed shifts and multiplications can overflow under a language-level interpretation. **32-bit wrapping arithmetic is a conditional machine-word interpretation**, not a verified platform property. Address arithmetic, narrowing, signed right shifts, and derived bytes retain this qualification.

There is no visible local loop or conditional branch controlling the dependency sequence. Dependencies may nevertheless branch, mutate memory, fail, or not return. The exhaustive invocation-by-invocation argument trace is **documented_only** in this contract; the transformation summary does not specify interchangeable dependencies.

### Descriptive memory roles

The following labels describe captured numeric relationships, not source identifiers or validated structures:

| Role | Meaning |
|---|---|
| Retained scaled value | A dependency-result low word shifted left by 25. |
| Large base | Another dependency result multiplied by 2²⁵. |
| Preserved value | An earlier dependency result shifted left by eight. |
| Compact field | A result shifted left by eight, masked to fifteen bits, then shifted right unsigned by six. It ranges from 0 through 508 in multiples of four. |
| Doubled compact base | Twice the compact field. |
| Early capture | The word read at large base plus 0x14. |
| Byte-derived base | The zero-extended byte read at fixed address 0x05. |
| Captured address | The word captured at byte-derived base plus 0x54 before replacement. |
| Fourfold base | Four times the captured address. |
| Adjusted base | Fourfold base minus 0x96. |
| Publication base | Four times the adjusted base. |
| Selected word | A later word selected through a zero-extended byte. |
| Double-selected base | Twice the selected word. |
| Publication payload | Four times the selected word. |
| Fixed-loaded base | A word captured from fixed-location address storage before repeated publication. |

All offsets below are byte offsets. “Low byte” means the narrowed eight-bit value, subject to the arithmetic qualification above.

### Initial mutations and capture boundaries

The direct-memory phase first stores the retained scaled value as a word at final dependency-result high word plus 0x14. Retained scaled value plus 9 receives the high word’s low byte. A word store then places the large base at final-result low word multiplied by 256, plus 0x54. Retained scaled value plus 9 subsequently receives the retained scaled value’s own low byte.

Large base plus 0x14 receives a zero word. Its offset-5 byte is read before doubled compact base plus 0x14 receives a zero word. The byte is then stored zero-extended as a word at large base plus 0x34.

The early capture is read next. The zero-word store through the doubled compact base repeats, followed by two separate byte stores of the early capture’s low byte at doubled compact base plus 0x15. Aliasing can affect the early capture; it must not simply be assumed zero.

The byte at fixed address **0x05 is read before either following mutation**: address 0x01 receives the doubled compact base’s low byte, then byte-derived base plus 0x15 receives the compact field’s low byte.

Large base plus 0x11 receives the preserved value’s low byte. Early capture plus 0x54 receives the preserved value as a word, after which large base plus 0x11 is cleared.

The captured address is then read at byte-derived base plus 0x54. Only afterward does that location receive the early capture.

### Address propagation

The propagation phase has several capture boundaries that affect later destinations.

First, doubled compact base plus 0x34 receives the compact field as a word. A word read at large base plus 0x74 selects the **propagation destination**. Its offset-0x54 word receives the large-base-producing dependency result shifted left by 26.

A separate self-publication uses **two reads** of doubled compact base plus 0x74: one supplies a destination base whose offset-0x54 word is written, and the other supplies the stored word. The relative evaluation order of these two reads within the represented expression is unresolved.

The next mutations form two repeated address/value pairs around an intervening store. Captured address plus 0x24 receives the captured address, and propagation destination plus 9 receives its low byte. Propagation destination plus 0x54 then receives the preserved value. The self-relative offset-0x24 word store and offset-9 byte store repeat afterward.

Doubled compact base plus 0x14 next receives the dependency result shifted left by 26. Fixed address **0x54** then receives the captured address shifted left by one. Doubled compact base plus 0x14 subsequently receives the captured address twice as separate word stores.

Two byte stores at doubled compact base plus 0x15 receive the fourfold base’s low byte. Fixed address **0x15** then receives the doubled compact base’s low byte.

Fourfold base plus 0x54 receives the preserved value. Doubled compact base plus 0x34 receives the fourfold base. A fresh word read at doubled compact base plus 0x74, multiplied by two, establishes the **doubled propagation base**. Fourfold base plus 0x54 then receives the fourfold base itself. Its offset-0x15 byte receives the low byte of eight times the captured address, and captured address plus 0x44 receives the doubled propagation base as a word.

An offset-5 byte read from the fourfold base now selects a word at numeric address **byte value plus 0x74**. This word establishes the **byte-selected destination**. Its offset-0x44 word receives the doubled propagation base.

Before the later offset-5 reads described below, the following relationships are published in order:

- Fourfold base plus 0x15 receives the fourfold base’s low byte; its offset-0x0D byte receives the byte-selected destination’s low byte.
- Fourfold base plus 0x14 and plus 0x54 each receive the fourfold base as words.
- A word read at fourfold base plus 0x64 selects a **secondary destination**, whose offset-0x0D byte receives the byte-selected destination’s low byte and whose offset-0x14 word receives itself.
- A word read at fourfold base plus 4 establishes the **offset-5 source base**.
- Doubled propagation base plus 9 receives the byte-selected destination’s low byte, then its offset-0x44 word receives eight times the captured address.

The offset-5 source base is read twice at offset 5, with intervening mutations. After the **first** byte read, fourfold base plus 9 receives the doubled propagation base’s low byte, and byte-selected destination plus 0x44 receives that byte zero-extended as a word. Fourfold base plus 0x14 receives eight times the captured address. Doubled propagation base plus 0x15 receives the offset-5 source base’s low byte, then its offset-0x44 word receives the fourfold base.

The **second** offset-5 byte read occurs only after these stores. Fourfold base plus 0x15 then receives the low byte of eight times the captured address. A word read at the second byte’s zero-extended value plus 0x74 establishes the selected word.

Finally, fourfold base plus 0x15 receives the offset-5 source base’s low byte; its offset-0x54 word receives the fourfold base; its offset-9 byte receives the double-selected base’s low byte; and its offset-0x14 word receives eight times the captured address.

These reads are distinct. Neither offset-5 byte capture nor any memory-selected address may be replaced with a presumed stable earlier value.

### Literal stores and repeated publication

Three calculated locations receive 0x3A: a byte at the publication payload, a halfword at eight times the selected word, and a word at publication payload plus 0x20.

Publication base plus 0x20 receives four successive word writes: **0x20, 0x2F, a zero-extended byte freshly read at publication base plus 4, and 0x20**. The byte read follows the first two writes.

The fixed-loaded base is captured next, before fixed address **0x60** receives word 0x20.

Repeated publication uses four-access groups:

1. A scaled base’s offset-0x54 word receives the publication payload.
2. Fixed-loaded base plus 0x19 receives the publication base’s low byte.
3. Publication base plus 0x11 receives the fixed-loaded base’s low byte.
4. Fixed-loaded base plus 0x34 receives the scaled base as a word.

The initial scaled base is eight times the captured address. Subsequent groups use eight times the adjusted base. The fixed-loaded base and both narrowed bytes remain captured values, not fresh source reads.

The representation contains a fixed, unrolled repetition. Its exact multiplicity is **documented_only** here; no protocol length or object count is inferred.

### Post-publication captures and fixed byte accesses

After repeated publication, the word at fixed address **0xA9** is captured, followed by the word at publication payload plus 0x74. Fixed address **0x3A** then receives an arithmetic-derived byte. Only afterward are the words at double-selected base plus 0x14 and publication base plus 4 captured.

The 0x3A byte derives from narrowing the adjusted base to a byte, multiplying by eight, and adding 0x30. Under a wrapping low-eight-bit interpretation, this equals the low byte of eight times the adjusted base plus 0x30. The representation does not establish portable signed-byte arithmetic.

Double-selected base plus 0x0D receives the publication base’s low byte. Publication base plus 8 receives 0x6C, then its offsets 0x0D and 9 receive the fixed-loaded base’s low byte. Double-selected base plus 0x54 receives that base itself as a word.

Fixed address **0x67** receives the captured 0xA9 word’s low byte. Three later reads of 0x67 have separate roles:

- After the word read at double-selected base plus 0x44, the **first read** supplies the byte written at fixed address 0x21.
- After subsequent calculated stores and the publication-base-plus-0x40 word capture, the **second read** supplies a byte written at publication base plus 9.
- After further mutations and the publication-base-plus-0x54 word capture, the **third read** supplies another byte written at publication base plus 9.

Aliasing can change 0x67 between reads. Reusing the original 0xA9 byte or an earlier 0x67 read is not equivalent.

### Late mutations and terminal capture order

The word captured at double-selected base plus 0x44 supplies the byte written at publication base plus 0x10 and the word written at fixed-loaded base plus 0x54. Its right-shifted low byte is written at double-selected base plus 1; the shift count is the publication base’s low byte. Fixed-loaded base plus 0x54 then receives the publication base instead.

The word at publication base plus 0x40 is captured. Fixed-loaded base plus 0x74 receives the earlier publication-base-plus-4 capture; publication base plus 0x10 receives the offset-0x40 capture as a word; and fixed-loaded base plus 0x44 receives the double-selected base. The second 0x67 read follows these stores.

Double-selected base plus 9 then receives the publication base’s low byte. Publication base plus 0x44 receives the captured 0xA9 word. A word read at publication base plus 0x54 selects the **previously selected destination**; the third 0x67 read follows that capture.

The earlier publication-payload-plus-0x74 capture supplies the byte written at previously selected destination plus 0x0D. The post-publication double-selected-base-plus-0x14 capture is restored at double-selected base plus 0x54.

A **fresh word capture at publication base plus 0x14** occurs next. Double-selected base plus 0x11 then receives the earlier publication-payload-plus-0x74 capture’s low byte. After that byte store, publication base plus 9 receives the previously selected destination’s low byte, and publication base plus 0x44 receives that destination as a word. Both stores precede the next address selection.

The final selection and capture boundary is:

- A word read at double-selected base plus 4 establishes the **final selected base**. A halfword 0x65 is written at twice that base.
- The word at final selected base plus **0x64 is captured before** the next two stores.
- Previously selected destination plus 0x11 receives the fresh publication-base-plus-0x14 capture’s low byte. Its offset-0x54 word then receives **100**.
- The word at final selected base plus **0x74 is captured only after those stores**.

Thus the two final-selected-base captures are not an unordered pair. Aliasing with either intervening store can affect the later offset-0x74 value and therefore the fourth terminal argument.

A word read at double-selected base plus 0x64 then establishes the **next destination**. Before its offset-0x54 word is written, double-selected base plus 9 receives the final selected base’s low byte. Next destination plus 0x54 receives the captured final-selected-base-plus-0x64 word. A word at twice that captured value receives the next destination itself.

The terminal indirect invocation receives four displayed arguments, in order:

1. 0x72.
2. The final-selected-base-plus-0x64 capture.
3. The post-publication double-selected-base-plus-0x14 capture.
4. The later final-selected-base-plus-0x74 capture.

Its target, hidden inputs, and completion behavior remain unresolved.

## Side effects

The exposed layout consists of byte offsets and access widths, not validated object types. Calculated accesses include byte, halfword, and 32-bit word operations. Fixed mutations include bytes at 0x01, 0x15, 0x21, 0x3A, and 0x67, and words at 0x54 and 0x60.

Ranges can overlap, including differently sized accesses. No non-aliasing guarantee is established. Endianness, alignment, atomicity, volatility, device behavior, and concurrency are **documented_only**; represented access widths do not establish bus-transaction widths.

No explicit allocation or release is shown. No ownership transfer, lifetime extension, retention permission, or release obligation is established. Dependencies may have such effects.

There is no visible rollback, recovery handler, null validation, or error result. A fault can leave earlier mutations in place. Thread safety and reentrancy are not established.

## Dependencies

The supplied identity map identifies only **F-dcc73dabc9fd**. No opaque function IDs are supplied for its invoked targets. Their identities are **documented_only**; assigning invented IDs would fabricate provenance.

Dependency roles include receiving entry inputs, producing primary and secondary result words, consuming transformed fields, and receiving the terminal arguments.

No dependency is assumed to preserve globals or pointed-to memory, acquire or release ownership, or return normally.

## Acceptance examples

These are conditional static obligations, not executed tests.

- **Entry arithmetic:** With second input 3 and applicable 32-bit shift semantics, the first invocation’s third argument is 192; its first two arguments remain the original inputs.
- **Small numeric base:** If the 0x05 read yields 0xE7, the byte-derived offset-0x15 destination is 0xFC. The read precedes both the 0x01 write and the 0xFC write.
- **Fixed word value:** Captured address 0x12345678 yields word 0x2468ACF0 at address 0x54 under the conditional machine-word interpretation.
- **Propagation capture:** The large-base-plus-0x74 read determines the propagation destination used by later stores. A constant or the large base itself cannot substitute for that loaded destination.
- **Distinct offset-5 reads:** If intervening stores change the offset-5 source byte, the selected word uses the second read. The first read still supplies its earlier zero-extended word store.
- **Arithmetic-derived byte:** Adjusted-base low byte 3 yields byte 0x48 at 0x3A under wrapping low-byte arithmetic. The 0xA9 and payload-plus-0x74 captures precede this write; the next two captures follow it.
- **Repeated fixed reads:** If successive 0x67 reads yield 0x11, 0x22, and 0x33, the corresponding destination writes receive those distinct values.
- **Late aliasing:** If either previously-selected-destination store changes final selected base plus 0x74, the fourth terminal argument uses the value read afterward, not a pre-store snapshot.
- **Late destination publication:** The publication-base-plus-9 byte and plus-0x44 word stores occur before selecting the final base. The double-selected-base-plus-9 byte store occurs after selecting the next destination but before writing its offset-0x54 word.
- **Partial execution:** Earlier writes remain permitted after a subsequent fault. Their occurrence does not establish completion.

## Uncertainties

The following remain **documented_only**:

- Application purpose and executable fidelity.
- Dependency identities, signatures, hidden inputs, and ownership effects.
- Exhaustive dependency-call argument trace and exact publication repetition count.
- Address width, overflow rules, signed narrowing, endianness, and alignment.
- Signed right-shift behavior and handling of shift counts of 32 or greater.
- Relative evaluation order of the two reads in the self-publication expression.
- Device, concurrency, fault, and transaction behavior.
- Terminal target selection, termination, and return behavior.

The specified capture boundaries and ordered mutations must not be collapsed merely because final ordinary-memory contents might coincide. The explicitly documented trace omissions are not claims of behavioral equivalence.

## Evidence

Established static evidence comprises displayed input and result widths, arithmetic relationships, access widths, fixed-address roles, calculated offsets, and represented ordering.

The propagation captures, three late destination-publication mutations, and separation of the final offset-0x64 and offset-0x74 captures follow from the supplied representation. Prior prose, comments, and reviewer feedback are not independent evidence.

No implementation access, instruction inspection, dependency resolution, runtime execution, or tests have been performed. Acceptance examples express conditional obligations, not observed outcomes.
