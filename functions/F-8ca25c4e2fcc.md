<!-- Trusted provenance: F-8ca25c4e2fcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8ca25c4e2fcc

## Purpose

F-8ca25c4e2fcc performs a call-mediated sequence of value transformations followed by mixed-width memory mutations and an unresolved terminal transfer. It has three explicit 32-bit inputs and no established scalar result.

No domain-level purpose is established. Repeated offsets support describing address relationships, not asserting an object model. Numeric constants that resemble characters do not establish text processing, encoding, or string construction.

This contract separates:

- **Established static evidence:** represented calls, arithmetic, access widths, captured values, and ordering.
- **Inferred intent:** possible interpretations that impose no acceptance requirements.
- **documented_only unknowns:** unresolved dependency contracts, execution semantics, and explicitly identified limits of this document.

All behavior is conditional on preceding calls returning and memory accesses completing. Static representation alone does not prove runtime reachability.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit body reference. |
| Second input | 32 bits | No explicit body reference. |
| Third input | Signed 32-bit representation | Shifted left by 30 and supplied as the third argument of an early four-argument call. |

No explicit input validation is represented.

The first two inputs are not proven irrelevant. Unresolved calling conventions and entry-state interactions prevent a whole-operation independence guarantee.

Under a 32-bit bit-vector interpretation, the identified transformation of the third input retains only its lowest two bits. Portable signed-overflow behavior is not established.

### Implicit inputs and prerequisites

Behavior also depends on entry control state, an incoming register-like value, secondary result channels, represented 64-bit call results, existing memory, a globally stored address, and an unresolved terminal target.

Each reached access requires suitable readable or writable storage. Each reached call requires an executable target under the actual calling convention. No null, bounds, alignment, or permission checks are visible.

Literal low-address accesses are represented. An environment that leaves these addresses unmapped may fault rather than complete.

### Outputs and ownership

Observable outputs comprise dependency interactions, memory reads and writes, and terminal control transfer. There is no explicit status result.

No ownership transfer, allocation policy, release responsibility, or lifetime guarantee is established. Dependencies cannot be assumed to preserve globals or unrelated memory, or to acquire or release referenced storage.

Normal return, nonreturning transfer, exceptions, and faults remain **documented_only**.

## Behavior

### Execution and arithmetic model

The represented body contains an extended ordered call sequence, a direct-memory phase, and a terminal unresolved call with four displayed arguments. No explicit conditional branch or input-sized loop appears in this body. Dependencies may contain arbitrary control flow.

Calls without displayed arguments remain observable interactions. Neither omitted arguments nor ignored results establish purity.

Represented scalar transformations generally operate on 32-bit values. A 64-bit result is sometimes split numerically into bits 0–31 and bits 32–63; this does not establish memory endianness.

The call phase includes zero arguments, shifted prior results, result halves, and masked fields. Supported extractions include:

| Extraction | Conditional unsigned result |
|---|---|
| Retain 19 bits, shift right 2 | 0–131,071 |
| Retain 14 bits, shift left 3 | Multiples of 8 through 131,064 |
| Retain 24 bits, shift right 8 | Original bits 8–23 |
| Retain 18 bits, shift right 11 | Original bits 11–17 |
| Retain 17 bits, shift right 4 | Original bits 4–16 |
| Retain 24 bits, shift right 12 | Original bits 12–23 |

A **non-exhaustive** inventory of displayed call-argument left shifts includes 6, 7, 8, 9, 13, 14, 15, 16, **17**, **21**, 22, 23, 24, 25, and 30 positions. The shifts by 17 and 21 are represented operations, not architecture-dependent unknowns. Some operate on values already reduced to zero.

Repeated shifts must retain their intermediate width constraints. Under bit-vector semantics, discarded bits cannot be recovered by subsequent shifts. Unbounded-integer simplification is not generally equivalent.

### Access widths and layout

Observed widths are 8, 16, and 32 bits. Dynamic byte offsets include 1, 5, 8, 9, 13, 16, 17, 21, and 25. Dynamic word offsets include 4, 16, 20, 32, 36, 52, 64, 68, 84, 100, and 116.

These observations do not establish structure boundaries, disjoint objects, alignment, or byte order. A word at offset 16 may overlap a byte at the same offset.

Byte stores retain the low eight bits. Explicit byte-to-word conversions zero-extend. Halfword stores affect two bytes; word stores affect four bytes, including when the numeric payload resembles a character.

### Transition into direct memory

The initial direct-memory operations preserve several distinct relationships:

- A word derived from an earlier result is stored at offset 20 from the high half of the final represented 64-bit call result.
- A derived offset-9 byte destination first receives the low byte of that high half.
- An offset-84 word destination, based on the final low half multiplied by 256, receives a separately scaled earlier result.
- The offset-9 byte destination is then overwritten with another retained value.

A derived base subsequently has its offset-20 word cleared, followed by a byte read at offset 5. Another offset-20 word is cleared, the captured byte is stored as a zero-extended word at offset 52 of the first base, and the first base’s offset-20 word is read. The other offset-20 location is cleared again.

Thus, even a read following a clear is not unconditionally zero: intervening stores may alias it.

A further base derives from a call result shifted left eight, masked to 15 bits, and shifted right six, then doubled. Under 32-bit bit-vector semantics, the extracted field ranges from 0 through 508 in steps of four; the doubled base ranges from 0 through 1,016 in steps of eight. Two consecutive byte stores target offset 21 of that doubled base.

### Early fixed-location accesses and cross-linked mutation

The byte at **address 5 is read first** and zero-extended. Only afterward is **address 1 written as a byte**, using the low byte of the doubled masked-field base. These are different accesses, not a read-modify-write of one global.

The captured byte from address 5 becomes a dynamic base. Its offset-21 byte receives the low byte of the masked field. Other dynamic stores then occur before a word is captured from offset 84 of this byte-derived base. That offset-84 location is subsequently overwritten.

For the remaining description, **anchor** means this captured word. It is a snapshot, not an assertion that the originating memory remains unchanged.

The intervening cross-linked updates include:

- A byte at offset 17 written and then cleared.
- Word loads at offset 116, with loaded words used as later bases.
- A store through an offset-116-derived base whose payload also requires an offset-116 read.
- Repeated self-valued word stores at offset 36.
- Repeated offset-9 byte stores and offset-20 word overwrites.

These accesses must not be reduced to a cached pointer graph without noninterference evidence.

Within this phase, **address 84 receives a 32-bit word equal to twice the anchor**, under the applicable arithmetic semantics. This fixed-location write occurs after a dynamic offset-20 store and before two further dynamic offset-20 stores of the anchor.

Later, two dynamic offset-21 byte stores are followed by a **byte write at address 21**, carrying the low byte of the doubled masked-field base. Address 21 is not merely another notation for either preceding dynamic destination.

Subsequent updates use four and eight times the anchor as bases or payloads. They include word stores at offsets 20, 52, 68, and 84, byte stores at offsets 9, 13, and 21, and dependent loads at offsets 4, 5, 100, and 116. Several byte reads at offset 5 occur at different sequence positions and cannot automatically share a cached value.

### Constants and publication groups

For the later phases, these descriptive quantities identify retained values:

| Quantity | Meaning |
|---|---|
| Reduced base | Four times the anchor, minus 150 |
| Publication base | Four times the reduced base |
| Leaf value | A word loaded at offset 116 from a base obtained by zero-extending a dynamically read byte |
| Doubled-leaf base | Twice the leaf value |
| Publication payload | Four times the leaf value |
| Global base | Address-like word read from global storage before publication |

These are descriptive labels, not validated types. Multiplication, subtraction, and address formation have unresolved overflow and wrapping semantics.

The numeric value 58 is stored as a byte at four times the leaf value, as a halfword at eight times the leaf value, and as a word at offset 32 from four times the leaf value.

The word at publication-base offset 32 receives, in order:

1. 32.
2. 47.
3. A zero-extended byte read at publication-base offset 4.
4. 32 again.

The global base is then loaded. **Address 96 receives a 32-bit word containing 32** after that load and before the publication groups.

The initial publication group is distinct from its successors:

| Position within group | Initial group | Recurring groups |
|---|---|---|
| First | Word at eight-times-anchor offset 84 receives the publication payload | Word at eight-times-reduced-base offset 84 receives the publication payload |
| Second | Byte at global-base offset 25 receives the low publication-base byte | Same |
| Third | Byte at publication-base offset 17 receives the low global-base byte | Same |
| Fourth | Word at global-base offset 52 receives eight times the anchor | Receives eight times the reduced base |

The recurring offset-84 store is part of **each** recurring group and precedes its two byte stores and offset-52 word store. The initial offset-84 destination must not be conflated with the recurring destination, even though particular arithmetic values could make them alias.

The groups reuse retained values and do not advance through array elements. No additional call or explicit read is displayed between their constituent stores.

**Coverage limit:** the exact number of recurring groups and the complete expanded write trace are unspecified by this document, classified **documented_only**. This is a specification-coverage limit, not a claim that the supplied representation lacks repetitions. No exact-count acceptance requirement is asserted.

### Final captures and fixed-byte propagation

After publication, the ordering of the following observations is established:

| Order | Access |
|---:|---|
| 1 | Capture a 32-bit word at fixed **address 169**. |
| 2 | Capture a 32-bit word at publication-payload offset 116. |
| 3 | Write a byte at fixed **address 58**, using the low eight bits of eight times the reduced base plus 48. |
| 4 | Capture a word at doubled-leaf-base offset 20. |
| 5 | Capture a word at publication-base offset 4. |

The dynamic read between the address-169 capture and address-58 write is significant. The subsequent word captures occur after the address-58 write and may observe overlapping storage.

Dynamic mutations then include a byte at doubled-leaf-base offset 13, byte value 108 at publication-base offset 8, low-global-base bytes at publication-base offsets 13 and 9, and a self-valued word at doubled-leaf-base offset 84.

Next, **address 103 is written as a byte**, using the low byte of the word captured at address 169. A dynamic word is then read at doubled-leaf-base offset 68. Only after that read is **address 103 read as a byte and its value written to address 33**.

This is a memory read from address 103, not unconditional propagation of the earlier captured address-169 byte.

The captured dynamic offset-68 word supplies further stores and the operand of a represented signed right shift. The shift count is the low byte of the publication base and may exceed 31. Count masking, large-count behavior, and signed right-shift semantics remain **documented_only**.

Additional mutations and captures intervene before each of two later reads of address 103:

- The first later read supplies a byte store at publication-base offset 9, after updates involving global-base offsets 68, 84, and 116 and replacement of the word at publication-base offset 16.
- A word carrying the address-169 snapshot is subsequently stored at publication-base offset 68, followed by a word capture at publication-base offset 84.
- The second later read of address 103 supplies another byte store at publication-base offset 9.

Each read observes memory at its own position. Dynamic stores may overlap address 103; substituting the original byte written there is not generally supported.

The publication-base offset-16 replacement uses a word loaded at offset 64. The global-base offset-116 update uses the earlier publication-base offset-4 snapshot, not a fresh read.

### Terminal mutations and arguments

The tail captures another word at publication-base offset 20 and uses its low byte in a later offset-17 store. The address-like value captured at publication-base offset 84 becomes the base for an offset-13 byte store, an offset-17 byte store, and an offset-84 word store of 100.

A word read at doubled-leaf-base offset 4 becomes a further base. Twice that value receives a halfword containing 101. Its offset-100 word is captured before its offset-116 word. A subsequent word capture at doubled-leaf-base offset 100 provides another destination base.

The final visible stores include:

- A byte at doubled-leaf-base offset 9 containing the low byte of the further base.
- A word at offset 84 from the last captured destination base, containing the earlier offset-100 capture.
- A word through twice that earlier offset-100 capture, containing the last destination-base capture.

The terminal unresolved transfer receives four explicit values:

1. Constant 114.
2. The earlier offset-100 word captured from the further base.
3. The word captured at doubled-leaf-base offset 20 immediately after the address-58 write.
4. The later offset-116 word captured from the further base.

These are retained snapshots. Replacing them with terminal-time reloads can change behavior.

## Side effects

Established side effects include ordered unresolved calls, fixed and dynamic memory reads, mixed-width stores, repeated overwrites, and control transfer.

Meaningful fixed accesses include the byte read at address 5; byte writes at addresses 1 and 21; word writes at addresses 84 and 96; the word capture at address 169; the byte write at address 58; and the distinct write and later reads at address 103, including propagation to address 33.

No synchronization, atomicity, transactional rollback, or reentrancy guarantee is visible. Intermediate writes may matter through aliasing, concurrent observation, or device-memory semantics. Static ordering here describes the represented access sequence, not a proven hardware visibility model.

There is no explicit error status, validation failure branch, or recovery path. A fault or nonreturning dependency can prevent all subsequent effects. Earlier effects are not shown as rolled back.

## Dependencies

F-8ca25c4e2fcc is the only resolved opaque function ID supplied. No distinct dependency ID is established; unresolved targets are not assigned fabricated identities.

| Dependency surface | Established interaction | Unresolved contract |
|---|---|---|
| Entry-state-derived target | First call target depends on incoming control state | Identity, arguments, effects, termination |
| Extended unresolved call sequence | Receives constants, shifts, masked values, and result halves | Calling convention, memory effects, result meaning |
| Secondary result channels | Values consumed by later argument construction | Formal results versus incidental architectural state |
| Global storage | Supplies bytes, words, and the publication global base | Initialization, lifetime, aliasing, external mutation |
| Terminal target | Receives four explicit values | Destination, ownership effects, return behavior |

No dependency is assumed to preserve globals, preserve storage, or own or release referenced memory.

## Acceptance examples

These are conditional deductions, not executed tests.

### Third-input contribution

With 32-bit bit-vector shifts, third-input values 0, 1, 2, and 3 contribute respectively 0, 1,073,741,824, 2,147,483,648, and 3,221,225,472 to the identified early argument. Adding four leaves that argument unchanged, but does not prove whole-operation equivalence.

### Extraction and width

Masking 0xAB123456 to 24 bits and shifting right eight produces 0x1234. Sign extension or retaining the upper byte would violate this extraction.

If the constant-58 group is reached, its byte, halfword, and word stores remain distinct. A string write or three byte stores is not equivalent.

### Fixed-access ordering

An access observer must distinguish the read at address 5 from the later write at address 1. Likewise, it must distinguish the word capture at address 169, the intervening dynamic word read, the byte write at address 58, and the subsequent word captures.

### Address-103 rereads

Suppose the byte written to address 103 is 0x12, and an intervening dynamic store changes that address to 0x34 before a later represented read. Under ordinary memory semantics, that read supplies 0x34. Reusing 0x12 is not equivalent.

### Publication bases

If eight times the anchor and eight times the reduced base are distinct valid addresses, the initial and recurring offset-84 stores target distinct locations. Every recurring group retains its offset-84, offset-25, offset-17, offset-52 order. This document does not specify the total group count.

### Failure and shift boundaries

An unwritable derived address need not produce a graceful status result or restore prior state.

If the variable shift count is 40, neither interpreting it as eight nor forcing a zero result is established without target semantics.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and external success criteria.
- Runtime reachability and accuracy of the static representation.
- Dependency identities, effects, calling conventions, and secondary-result semantics.
- Signed overflow, address wrapping, endianness, alignment, and variable-shift behavior.
- Memory-map validity, object boundaries, aliasing, and device semantics.
- Ownership, storage duration, concurrency, and terminal return behavior.
- Exact publication repetition count and complete expanded access trace as specified by this document.

Repeated offsets suggest structured state; character-like constants suggest possible text-related intent. Neither inference is an acceptance requirement.

## Evidence

The supplied static material supports the represented widths, transformations, fixed addresses, dynamic offsets, capture relationships, ordering constraints, and unresolved interaction boundaries described above.

The prior draft and reviewer feedback are not independent behavioral evidence. Arithmetic examples depend on explicitly stated bit-vector or ordinary-memory assumptions.

No dependency implementation, architecture specification, executable observation, or runtime memory map was supplied. No runtime tests have been performed. No snapshot hash or independently verified provenance is asserted.
