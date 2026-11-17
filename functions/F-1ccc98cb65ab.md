<!-- Trusted provenance: F-1ccc98cb65ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ccc98cb65ab

## Purpose

F-1ccc98cb65ab performs an ordered sequence of unresolved indirect invocations, transforms selected machine results, updates fixed and computed memory locations, and ends with an unresolved indirect transfer.

No application-level purpose is established. The supplied representation does not establish text formatting, collection management, object initialization, or a particular numerical algorithm. Character-like constants and recurring offsets do not establish those interpretations.

This contract distinguishes:

- **Established static evidence:** explicit argument relationships, represented widths, memory accesses, value captures, and ordering.
- **[INFERENCE]:** conclusions dependent on a stated execution-model assumption.
- **documented_only:** unresolved behavior for which no definite guarantee is supported.

Every effect is conditional on execution reaching it. An invocation may not return, and an access may fail.

## Interface

### Explicit inputs

The represented interface has four positional inputs and no declared return value.

| Input | Represented width | Explicit use |
|---|---:|---|
| First | 32 bits | First argument of the initial indirect invocation, unchanged. |
| Second | 32 bits | No explicit use in the body. |
| Third | 32 bits | No explicit use in the body. |
| Fourth | Signed 32 bits | Shifted left by 7 for the initial invocation and by 25 for a later invocation. |

The absence of explicit uses for the second and third inputs does not prove their irrelevance to unresolved calling conventions or incoming machine state.

No input is established as a buffer, length, string, object reference, or allocation. No explicit null-input policy is visible.

### Implicit inputs

Execution additionally depends on:

- Incoming control context used to form the first invocation target.
- Represented control locations used to form later targets.
- Primary and secondary machine results from dependencies.
- Existing fixed-address and computed-address memory.
- Arithmetic, address-formation, alignment, byte-order, and fault semantics.

The four explicit inputs do not independently determine the observable trace.

### Outputs and completion

There is no established status result, returned object, or success indication. Observable outputs consist of invocations, mixed-width memory accesses, and the terminal transfer.

The terminal transfer has four explicit arguments, in order:

1. 0x72.
2. A 32-bit word captured during the terminal phase at offset 0x64 from a newly obtained base.
3. A 32-bit word captured earlier at offset 0x14 from twice the late link, defined below.
4. A 32-bit word read later in the terminal phase at offset 0x74 from the newly obtained base.

The fourth argument is sampled after intervening writes; it is not captured together with the second argument.

A return is represented after the transfer. Whether the target returns, and whether the represented invocation accurately describes the terminal transfer mechanism, remain **documented_only**.

### Preconditions and ownership

Uninterrupted execution requires every reached target to support the applicable invocation convention and every reached access to be valid for its width and direction. No bounds, alignment, or address-validity checks are visible.

The function’s explicit operations establish no allocation, release, reference counting, ownership transfer, or lifetime extension. Dependency ownership behavior is unknown. Dependencies must not be assumed to preserve globals, retain memory, or release memory.

## Behavior

### Invocation phase

Before its first explicit data-memory store, the function performs a long straight-line sequence of indirect invocations. No explicit conditional branch, retry, error-result check, or data-dependent loop appears.

The first target is formed from incoming control context plus 0xFFE. Its explicit arguments are the first input and the fourth input shifted left by 7. Later target expressions predominantly add 0xFFE to represented control locations.

Similar target expressions do not establish a shared dependency identity. Neither target purpose nor invocation convention is resolved.

Argument construction includes:

- Shifted primary and secondary machine results.
- Low and high 32-bit portions of represented 64-bit results.
- Repeated arguments carrying the same transformed value.
- Explicit zeros and values that simplify to zero under wrapping arithmetic.
- Captures retained across intervening invocations.
- Masks followed by shifts, and sequential shifts with intermediate truncation.

An invocation represented without explicit arguments is not proof of a zero-argument interface. A secondary machine result is not necessarily a conventional language-level return value.

Invocation order and multiplicity are observable obligations. Unused results, duplicate arguments, and zero-valued arguments do not justify eliminating invocations: their side effects and termination behavior are unknown.

### Widths and arithmetic

Explicit memory accesses include 8-bit bytes, 16-bit halfwords, and 32-bit words. Selected dependency results are represented as 64 bits.

High-half extraction shifts an unsigned 64-bit result right by 32. Other full-result right shifts include 34, 36, and 44. These are not oversized shifts of 32-bit operands.

Masks select fields including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits. Mask placement and the width of each intermediate result matter.

**[INFERENCE]:** Under a 32-bit wrapping-machine interpretation, left shifts, multiplication, subtraction, and address-like additions retain their low 32 bits. Sequential transformations retain intermediate truncation; unlimited-precision arithmetic is not equivalent.

Some operations are represented on signed 32-bit quantities. Portable signed-overflow behavior is not established. Wrapping interpretations and examples in this contract are conditional, not universal guarantees.

Byte stores retain the low 8 bits. Explicit unsigned-byte reads widened to words produce values from 0 through 255. Where such a byte supplies an address component, there is no additional implicit base.

A late byte-store value uses a signed right shift with a count taken from the low 8 bits of the publication base. Counts exceeding 31 are possible. Oversized-count behavior is **documented_only**; modulo-32 masking must not be assumed. Signed right-shift behavior also depends on the target model.

### Initial memory updates

Values retained from late invocations include:

- A result shifted left by 25, called the **shift-derived base**.
- Another result multiplied by 2²⁵, called the **multiplication-derived base**.
- A result shifted left by 8.
- A small unsigned quantity obtained by masking a shift-by-8 result with 0x7FFF and then shifting right by 6.

Under 32-bit wrapping arithmetic, the small quantity is a multiple of four from 0 through 508. Twice that quantity is a multiple of eight from 0 through 1016.

The first explicit store writes the shift-derived base as a word at the final represented 64-bit result’s high half plus 0x14. It is followed by a byte store at shift-derived base plus 9, a word store of the multiplication-derived base at the final result’s low half multiplied by 256 plus 0x54, and a second byte store at shift-derived base plus 9.

The first byte store takes its value from the final result’s high half. The second takes its value from the shift-derived base. **These are separately ordered stores with different value sources. Neither unequal source values before truncation nor unequal stored bytes are guaranteed.**

Next, the function clears the word at multiplication-derived base plus 0x14 and samples an unsigned byte at that base plus 5.

It clears the word at twice the small quantity plus 0x14, stores the sampled byte as a zero-extended word at multiplication-derived base plus 0x34, and rereads multiplication-derived base plus 0x14. It then clears the small-derived word again and performs two byte stores at twice the small quantity plus 0x15 using the low byte of the reread word.

That reread is live. Intervening stores may alias the previously cleared word, so replacing the reread with zero is not generally valid.

### Fixed-storage capture and linked updates

Fixed storage at address 0x05 is read and converted to a 32-bit unsigned quantity before the byte store at fixed address 0x01. The converted value supplies a computed base.

The fixed-storage declaration is absent. Its access width is **documented_only**: conversion to 32 bits does not establish a 32-bit memory read.

The captured base remains the earlier value after the write at 0x01. Later uses must not become fresh reads of 0x05, and the initial capture must not move past potentially overlapping writes.

Subsequent operations write a byte at captured base plus 0x15 and capture the old word at captured base plus 0x54 before replacing that word. This old word is the **retained link**.

Other linked updates use words reached at offset 0x74, self-valued word stores at offsets 0x24 and 0x14, and low-byte copies of address-like values. Several stores are explicitly duplicated.

One linked-store expression contains two reads of the word at twice the small quantity plus 0x74: one determines the destination and one supplies the stored value. Their relative evaluation order is not established. No intervening store is represented within that expression; this does not establish equality for externally changing or device-backed storage.

Additional fixed updates include a word store at 0x54 and a byte store at 0x15. Fixed and computed destinations are not established as disjoint.

Four times the retained link supplies a recurring computed base. The function also captures a word at offset 4 from this base. Later byte reads at offset 5 from that captured word’s value supply addresses for word accesses at offset 0x74. These are live reads separated by updates, not one reusable sample.

The later word obtained through this byte-based route is the **late link**. Twice and four times the late link supply subsequent bases. These names describe value relationships only; they establish no object types or ownership.

### Layout and aliasing

Recurring offsets identify access locations, not proven structure fields.

| Offset | Represented access forms |
|---:|---|
| 1 | Byte write |
| 4 | Word read; byte read during constant publication |
| 5 | Unsigned-byte read |
| 8, 9, 0x0D | Byte writes |
| 0x10 | Byte and word writes |
| 0x11 | Byte writes |
| 0x14 | Word reads and writes |
| 0x15, 0x19 | Byte writes |
| 0x20, 0x24, 0x34 | Word writes |
| 0x40 | Word read |
| 0x44, 0x54, 0x74 | Word reads and writes |
| 0x64 | Word reads |

There is no established non-aliasing relationship among these accesses. Distinct expressions may produce identical or overlapping effective addresses. Byte and halfword writes may alter later word samples. Alignment and overlapping-access consequences depend on the target.

No validation precedes the linked and scaled accesses.

### Constant writes and publication

The constant 0x3A is stored with three widths:

- A byte at four times the late link.
- A halfword at eight times the late link.
- A word at four times the late link plus 0x20.

The stores occur in that order, but their destinations need not be distinct. When the late link is zero, the byte and halfword destinations coincide.

The **adjusted value** is four times the retained link minus 0x96. The **publication base** is four times this adjusted value. These arithmetic relationships are separate from any fixed-storage capture.

At publication base plus 0x20, four successive word-store values are 0x20, 0x2F, a freshly sampled unsigned byte from publication base plus 4 widened to a word, and 0x20 again. This is repeated replacement of one destination, not an established append operation.

After these stores, a separate fixed 32-bit word is captured. Its value is the **static-derived base**. Only after this capture does the function store word value 0x20 at fixed address 0x60.

The fixed-word capture does **not** supply the publication base. If the 0x60 store overlaps its storage, subsequent operations still use the already captured static-derived base. The publication base remains the separately calculated fourfold adjusted value.

Repeated publication follows. Each group contains, in order:

1. A word store of four times the late link at a scaled destination base plus 0x54.
2. A byte store at static-derived base plus 0x19, containing the low byte of the publication base.
3. A byte store at publication base plus 0x11, containing the low byte of the static-derived base.
4. A word store at static-derived base plus 0x34, containing that group’s scaled destination base.

The initial group uses eight times the retained link as its destination base. The explicitly expanded subsequent groups use eight times the adjusted value.

These are repeated access events, not a visible data-dependent loop or one final-state assignment. Aliasing does not change the captured arithmetic operands, although it may change memory observed later. Application significance remains **documented_only**.

### Final captures and live fixed-byte reads

Immediately after repeated publication, the function reads the fixed 32-bit word at 0xA9 and then the word at four times the late link plus 0x74. It next writes the fixed byte at 0x3A.

The byte written at 0x3A is the low-byte result of eight times the adjusted value plus 0x30. Only afterward are the words at twice the late link plus 0x14 and publication base plus 4 captured. The fixed-byte write can therefore affect either following capture if their storage overlaps.

Further writes include low-byte copies, byte value 0x6C at publication base plus 8, and a self-valued word store at twice the late link plus 0x54.

The low byte of the captured fixed word from 0xA9 is assigned to fixed byte 0x67. A word is then read at twice the late link plus 0x44. Fixed byte 0x67 is subsequently read to supply the assignment to fixed byte 0x21.

Two additional live reads of 0x67 separately supply stores at publication base plus 9. The first follows computed writes; the second follows further byte and word writes and a word read at publication base plus 0x54.

These three consumers use their respective live samples. Intervening computed stores may alias 0x67, so none is generally replaceable with a cached low byte from the earlier 0xA9 capture.

Other late effects include:

- A byte write at publication base plus 0x10, later overwritten by a word sampled from publication base plus 0x40.
- Successive stores of a captured word and then the publication base at static-derived base plus 0x54.
- A store of the earlier publication-base-plus-4 capture at static-derived base plus 0x74.
- Restoration of the earlier doubled-late-link offset-0x14 capture at doubled-late-link base plus 0x54.
- Additional low-byte stores and live word reads at offsets 0x14 and 0x54.

Overwriting a byte with a later word does not eliminate the earlier byte-store event.

### Terminal access order and transfer

A word read at twice the late link plus 4 supplies the **terminal base**. A halfword value 0x65 is written at twice this terminal base.

The function then captures the word at terminal base plus 0x64. After this capture, it performs a byte write through the previously captured publication-base-plus-0x54 value and stores word value 100 at that derived base plus 0x54. Only then does it read terminal base plus 0x74.

Consequently, the terminal offset-0x74 sample observes the state after those intervening writes, subject to target memory semantics.

The remaining accesses include a word read at twice the late link plus 0x64, a byte store of the terminal base’s low byte at twice the late link plus 9, a word store at the newly read value plus 0x54, and a word store at twice the earlier terminal offset-0x64 sample.

The unresolved terminal transfer follows, using the four arguments specified in the interface.

## Side effects

Established effects comprise an ordered invocation trace and a mixed-width memory-access trace.

No transaction, rollback, lock, atomic operation, or memory barrier is established. A non-returning dependency or failing access may expose a completed prefix of effects. No explicit recovery or error-reporting path is visible.

Duplicate stores, overwritten values, and live rereads must not be collapsed merely because final ordinary-memory contents could match. Aliasing, external observers, device-backed memory, and access failures may distinguish those events.

Dependencies may alter memory before the first explicit store. No global-preservation guarantee is available.

## Dependencies

F-1ccc98cb65ab is the only supplied opaque function ID.

No opaque IDs are supplied for the indirect invocation targets or terminal target. Their identities remain **documented_only**; no additional dependency identities are invented.

Their signatures, machine-result conventions, actual result widths, side effects, ownership behavior, failure modes, and termination properties are unresolved. Arithmetic-looking arguments do not establish arithmetic-helper semantics.

## Acceptance examples

These are conditional review criteria, not executed tests.

1. **Initial arguments.** Under 32-bit wrapping shifts, fourth input 1 produces initial second argument 0x80 and later shift-by-25 argument 0x02000000.

2. **Intermediate truncation.** Under the same model, a 32-bit value shifted left by 28 and then again by 28 becomes zero. Unlimited-precision preservation of discarded bits is incorrect.

3. **Separately ordered byte stores.** If the final result’s high half and the shift-derived base both equal zero, the two early byte stores at shift-derived base plus 9 have equal source values even before byte truncation. Both store events remain required. Neither source inequality nor stored-byte inequality is an acceptance condition.

4. **Fixed 0x05 capture.** If a later write changes storage overlapping the original 0x05 read, subsequent uses of the captured base retain its earlier converted value. This criterion does not specify the original access width.

5. **Capture before 0x60.** If the word store at 0x60 overlaps the fixed word supplying the static-derived base, publication still uses the static-derived base captured before that store. The publication base remains four times the adjusted value; it is not supplied by that fixed word.

6. **Coincident constant destinations.** If the late link is zero and the accesses are valid, the byte and halfword stores of 0x3A both target address zero. Disjoint destinations are not required.

7. **Publication overwrite.** With ordinary memory, no interference, and no additional overlapping effects, the four-store sequence at publication base plus 0x20 ends with 0x20. Its trace still includes 0x2F and the sampled-byte word.

8. **Intervening fixed-byte write.** If the 0x3A write overlaps either immediately following word capture, that capture observes its effect according to target byte order and memory semantics.

9. **Live 0x67 consumers.** If a computed write changes fixed byte 0x67 between consumers, a later consumer uses the later sample rather than the original low byte from 0xA9.

10. **Terminal aliasing.** If the word-value-100 store overlaps terminal base plus 0x74, the fourth transfer argument is sampled afterward. The second argument remains the earlier offset-0x64 capture.

11. **Interrupted execution.** If a dependency does not return, later invocations and explicit stores need not occur. No rollback of earlier effects is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object identities, and field meanings.
- Indirect target identities, calling conventions, and actual result widths.
- Width and interpretation of the fixed-storage read at 0x05.
- Target signed overflow, address wrapping, alignment, byte order, signed right shifts, and oversized-shift behavior.
- Validity of low fixed addresses and computed accesses.
- Ordinary-memory versus device-backed behavior.
- External interference, concurrency, interrupts, and reentrancy.
- Relative evaluation order of multiple reads within one expression where no order is otherwise established.
- Dependency ownership, memory preservation, and termination.
- Terminal transfer mechanics and return behavior.
- Application significance of repeated publication.

**[INFERENCE]:** Recurring offsets and self-valued stores may reflect structured state. They do not establish collection invariants, disjoint storage, or ownership rules.

## Evidence

The supplied static representation supports the explicit argument relationships, access widths, arithmetic constants, capture boundaries, and ordering described here. Comments, the prior draft, and reviewer feedback are not independent behavioral authority.

The early same-address byte stores draw from the final result’s high half and the retained shift-derived base, respectively. Different sources establish separate value derivations, not unequal values.

The publication-base arithmetic is established before the separate fixed-word capture. That fixed-word capture supplies the static-derived base and precedes the word store at 0x60. These are distinct base origins and distinct ordering obligations.

No dependency inspection, implementation access beyond the supplied representation, runtime execution, smoke run, or tests have been performed. Acceptance examples describe conditional obligations, not observed outcomes.
