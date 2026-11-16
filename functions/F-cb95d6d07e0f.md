<!-- Trusted provenance: F-cb95d6d07e0f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb95d6d07e0f

## Purpose

F-cb95d6d07e0f presents a four-word interface, performs an extensive sequence of unresolved indirect invocations and integer transformations, then accesses computed-address and fixed-address storage before a terminal indirect transfer.

The static material does not establish an application-level purpose. Constants that resemble characters do not establish formatting, string construction, decoding, or pathname handling. Offset-based accesses do not establish an allocator, container, or object model.

This contract distinguishes:

- **Established static evidence:** operations, value dependencies, and sequencing explicitly represented in the supplied body.
- **[INFERENCE]:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior for which the available evidence does not support an operational guarantee.

The contract describes the represented behavior conditionally; it does not certify that the supplied representation faithfully captures executable control flow. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface accepts four 32-bit inputs.

| Position | Interpretation in the representation | Explicit use |
|---|---|---|
| First | Uninterpreted 32-bit word | Forwarded unchanged to the first indirect invocation. |
| Second | Signed 32-bit integer | Forwarded unchanged and shifted left by six bits for that invocation. |
| Third | Uninterpreted 32-bit word | No explicit body-level use. |
| Fourth | Signed 32-bit integer | Shifted left by six bits for early invocations. |

None is established as a pointer, buffer length, ownership token, or status value. The third input’s lack of explicit use does not establish machine-level noninterference: unresolved invocations may consume state beyond their displayed arguments.

### Additional entry state

The first invocation target is constructed from incoming linkage state plus 4094 bytes. Subsequent operations consume secondary result-state words whose producers and calling-convention meanings are incompletely represented.

Consequently, the four explicit inputs are insufficient to determine behavior. Secondary state must not be assumed to contain zeros, to survive arbitrary invocations, or to be an ordinary return value.

### Outputs and preconditions

There is no explicit return value. Observable outputs comprise indirect invocations, direct memory effects, fixed-global effects, and arguments supplied to the terminal transfer. No success flag, output extent, or error result is established.

Traversing the represented path requires executable indirect targets and accessible storage for every reached access. No validation of these prerequisites is shown. Address-space mappings, alignment requirements, and low-address accessibility are documented_only.

## Behavior

### Invocation and transformation phase

The body begins with a long straight-line sequence of indirect invocations. The first receives the first input, the unchanged second input, and the second input shifted left by six bits. Early later invocations incorporate the similarly shifted fourth input.

Most later targets are formed from fixed continuation-like locations plus 4094 bytes. This is a target-construction pattern, not proof that the invocations share a callee. Invocations displayed without arguments remain observable dependency boundaries; absent displayed arguments do not establish absence of inputs or side effects.

The data flow consumes represented 32-bit results, 64-bit results, and secondary result-state words. Transformations include:

- Eight-bit left shifts of low result words.
- Six-bit and other left shifts of high result words.
- Masked extraction of low-order fields.
- Logical right shifts of unsigned intermediates.
- Duplicated arguments and explicit zero arguments.

No argument-validation branch, retry mechanism, or explicit early error return is shown. Unknown callees may alter globals, computed-address storage, or external state before the direct-access phase.

### Widths and arithmetic

Explicit pointer dereferences establish accesses of 8, 16, or 32 bits. An eight-bit destination retains the low eight bits; a sixteen-bit destination retains the low sixteen bits. Explicit unsigned-byte loads used numerically are zero-extended.

These facts do **not** establish the access widths of named globals whose declarations are absent. In particular:

- The global value associated with absolute address 0x05 is converted to an unsigned 32-bit value. The conversion does not establish an eight-bit load.
- The global value associated with absolute address 0xA9 is assigned to a four-byte temporary. The temporary’s width does not independently establish the global load width.
- Assigning an eight-bit-converted value to another named global does not independently establish an eight-bit memory store.

Global access widths and any declaration-dependent conversions are documented_only unless independently explicit. No requirement here selects or prohibits an eight-bit load at 0xA9.

For unsigned 64-bit result extraction, shifting by 32 selects the high word. Shifting by 34, 36, or 44 corresponds respectively to logically shifting that high word right by 2, 4, or 12 bits. These are not low-word shifts.

The scalar and address data paths are represented using 32-bit quantities. Signed left shifts, signed multiplication, and signed address expressions do not independently establish portable overflow behavior. Conditional examples below use low-32-bit machine arithmetic only where stated. Unbounded, saturated, or checked arithmetic is not automatically equivalent.

A late byte value depends on a signed right shift with a count taken from the low eight bits of a computed address value. That count can reach or exceed 32. Negative-operand and oversized-count behavior are documented_only; neither arithmetic sign extension nor count reduction modulo 32 is guaranteed by the available target evidence.

### Initial direct-access linkage

The following descriptive labels identify value provenance, not recovered structures:

- **Link base:** an earlier low result word shifted left by 25 bits.
- **Primary base:** a separate earlier result multiplied by \(2^{25}\).
- **Small base:** twice the quantity obtained by shifting a prior result left eight bits, masking to 15 bits, and shifting right six bits. This extracted quantity is at most nine bits wide.
- **Payload:** another earlier result shifted left eight bits.
- **Final pair:** the low and high words of the last represented 64-bit result before direct accesses.

The first direct-access group writes the link base as a word at the final pair’s high word plus 0x14, then writes that high word’s low byte at link base plus 9. A word write of the primary base at 256 times the final pair’s low word plus 0x54 follows; the next byte write to link base plus 9 takes the low byte of the link base itself.

These two writes to the same byte address are separate effects.

The primary-base word at offset 0x14 is cleared before an explicit unsigned-byte read at offset 5. The small-base word at offset 0x14 is then cleared, and the byte read from the primary base is stored, zero-extended, as a word at primary base plus 0x34.

A fresh word read at primary base plus 0x14 precedes another clear at small base plus 0x14. Two successive byte writes at small base plus 0x15 use the fresh word read’s low byte. They cannot be replaced by assumed-zero writes: aliasing or externally meaningful storage may affect the read.

### Fixed-global and linked-field effects

The value read from the global at 0x05 becomes an unsigned 32-bit address-like value. That read precedes the assignment to the global at 0x01, whose assigned expression is the small base converted to eight bits. Computed-address writes follow, including a byte at the address-like value plus 0x15, byte updates at primary base plus 0x11, and a word update at a destination derived from the earlier word read.

A word read at the address-like value plus 0x54 is saved before that field is overwritten. Call this saved word the **link value**. It subsequently supplies destinations and scaling expressions throughout the linked-field phase.

The represented linked accesses include word fields at offsets 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74, together with byte fields at offsets 1, 5, 9, 0x0D, 0x11, and 0x15. Some fields receive temporary values before replacements; some updates are repeated.

The global assignment at 0x54 receives the link value shifted left one bit. It occurs between an initial word assignment to small base plus 0x14 and later assignments of the unscaled link value to that field. The global assignment at 0x15 receives the small base converted to eight bits, after two byte writes at small base plus 0x15.

No homogeneous layout is established. In particular, an explicit unsigned-byte read at offset 5 becomes a numerical address base for a word read at offset 0x74. No pointer table or index-to-pointer conversion intervenes in the representation.

An expression containing two word reads of the same offset-0x74 field does not establish which operand is evaluated first or that both reads return the same value. The finer evaluation order is documented_only.

### Constant writes and shared-anchor repetition

A subsequently obtained linked word supplies three destinations for constant 0x3A: an eight-bit store at four times that word, a sixteen-bit store at eight times it, and a thirty-two-bit store at four times it plus 0x20.

For the remaining description:

- **Adjusted source:** four times the saved link value, less 0x96.
- **Computed region:** four times the adjusted source.
- **Linked region:** twice the linked word used for the constant-write destinations.
- **Constant region:** four times that same linked word.
- **Anchor:** a value read from separate shared global storage.

The computed region receives successive word writes at offset 0x20: 0x20, 0x2F, an explicit unsigned byte freshly read at offset 4 and zero-extended, then 0x20 again.

**The shared-anchor global is read after those writes and before the assignment of 0x20 to the global at absolute address 0x60.** The anchor is retained for the subsequent repeated region; that region does not explicitly reread the anchor global. The global access widths at both locations remain unresolved.

The repeated region contains ordered groups consisting of a computed-destination word write at offset 0x54, a byte write at anchor plus 0x19, a byte write at computed region plus 0x11, and a word write at anchor plus 0x34.

The first group uses eight times the original link value for its scaled destination and anchor-field value. Later groups use eight times the adjusted source. The linked word written at offset 0x54 is the constant-region value. The two byte sources are respectively the low bytes of the computed region and the anchor.

The represented repetitions remain separate accesses. No final-state-only equivalence is established. Their actual executable multiplicity and the reliability of the represented control-flow boundary remain documented_only.

### Finalization: saved reads and fixed-global ordering

Finalization begins by reading the global associated with 0xA9, retaining its converted four-byte value. A word read at constant region plus 0x74 follows. **Both reads precede the assignment to the global at 0x3A.**

That assignment’s expression multiplies the adjusted source’s character-converted value by eight and adds 0x30. The absent global declaration prevents fixing the destination access width or its final stored representation.

**Only after the 0x3A assignment** does the function save the word at linked region plus 0x14 and the word at computed region plus 4. These saved values must not be treated as reads performed before the global assignment.

The next computed-address effects include byte writes at linked region plus 0x0D and computed region plus 8, 0x0D, and 9; the offset-8 byte is 0x6C. A word write at linked region plus 0x54 stores the linked-region value itself.

The global at 0x67 is then assigned the eight-bit-converted value retained from the earlier 0xA9 read. The following word read at linked region plus 0x44 occurs before a **fresh read of the global at 0x67** used for the assignment to the global at 0x21. This is not specified as reuse of the earlier assigned value.

The recently read linked word supplies a byte at computed region plus 0x10 and a word at anchor plus 0x54. A signed-right-shift-derived byte is written at linked region plus 1, before anchor plus 0x54 is replaced with the computed-region value.

A word read at computed region plus 0x40 supplies a later word write at computed region plus 0x10. Between those accesses, the saved offset-4 word is written to anchor plus 0x74; afterward, anchor plus 0x44 receives the linked-region value.

The global at 0x67 is then **read again** for a byte write at computed region plus 9. Between this read and the next read of that global are:

- A byte write at linked region plus 9.
- A word write at computed region plus 0x44 using the saved four-byte value originating from 0xA9.
- A word read at computed region plus 0x54.

A further **fresh read of 0x67** supplies the second later byte write at computed region plus 9. Intervening computed-address stores may alias global storage, so neither later read may be replaced by the initially assigned eight-bit value without additional evidence.

### Remaining updates and terminal transfer

The word read at computed region plus 0x54 supplies a later linked destination. Its offset-0x0D byte receives the low byte of the earlier constant-region offset-0x74 word. The saved linked-region offset-0x14 word is restored at linked region plus 0x54.

After another word read at computed region plus 0x14, the function performs further byte updates, replaces computed region plus 0x44 with the recently obtained linked destination, and reads a word at linked region plus 4.

That last word determines a sixteen-bit store of 0x65 at twice its value. Its offset-0x64 field supplies a word retained for terminal dispatch; its offset-0x74 field supplies the terminal fourth argument. Intervening effects include a byte update at the newer linked destination plus 0x11 and a word store of decimal 100 at that destination plus 0x54.

A word subsequently read at linked region plus 0x64 supplies another destination. Before dispatch, the body updates linked region plus 9, writes the retained dispatch word at that destination plus 0x54, and writes the destination word at twice the retained dispatch word.

The terminal indirect transfer receives four represented arguments:

1. Constant 0x72.
2. The word read from offset 0x64 of the base obtained from linked region plus 4.
3. The saved word read from linked region plus 0x14 after the 0x3A assignment.
4. The later word read from offset 0x74 of the base obtained from linked region plus 4.

These arguments retain their read provenance despite intervening writes. The transfer target is unresolved. A represented return afterward does not independently establish normal return or the original transfer mechanism.

## Side effects

Explicit pointer accesses retain their represented widths. Named-global accesses retain their represented value dependencies and ordering, but their widths remain documented_only where declarations are missing.

No non-aliasing guarantee is supplied. Computed destinations may overlap each other or fixed-global storage. Earlier saved reads remain saved values; later explicit reads occur at their own positions and may observe intervening changes.

Byte and word fields can overlap, including offsets 0x14 and 0x15. Byte order is unresolved, so access traces—not invented final numerical words—define these updates.

The body identifies no explicit allocation, release, ownership transfer, lock, rollback, or reference-count protocol. Unknown dependencies may perform such operations. Memory lifetime, retained references, reentrancy, and thread safety are documented_only.

No explicit recovery path is shown. A fault or non-returning dependency may prevent later effects. No rollback, portable exception result, or successful completion indicator is established.

## Dependencies

Only F-cb95d6d07e0f has a supplied opaque function identity. No other target is resolved to an opaque function ID; inventing dependency IDs would imply unsupported identity resolution.

Unresolved dependency boundaries comprise the entry-linkage-derived invocation, continuation-derived invocations, secondary-result producers, and terminal transfer.

No dependency is established to preserve globals, leave memory untouched, own or release arguments, retain references, or return normally.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Initial forwarding.** With second input 3, a reached first invocation receives the unchanged first input, 3, and 192. This does not prove independence from the third input at machine level.

2. **Conditional truncation.** Under low-32-bit shift semantics, 0x04000000 shifted left six bits becomes zero. Keeping an unbounded result would differ.

3. **Explicit byte widening.** If the primary-base offset-5 byte read returns 0xFE, the following offset-0x34 word store receives 254, not −2.

4. **Unresolved global widths.** Neither the conversion of the 0x05 global to 32 bits nor assignment of the 0xA9 global to four-byte storage is sufficient to accept a particular global load width.

5. **Anchor snapshot timing.** If the assignment at 0x60 aliases shared-anchor storage, the repeated region still uses the anchor obtained before that assignment.

6. **Finalization boundary.** If the 0x3A assignment changes storage later read at linked region plus 0x14, the saved third terminal argument reflects the post-assignment read. The earlier 0xA9 and constant-region offset-0x74 reads precede that assignment.

7. **Fresh 0x67 reads.** If an intervening computed-address store changes storage read through global 0x67, a later offset-9 byte store uses the newly read global expression, subject to unresolved global conversion semantics—not a cached copy of the initial assignment.

8. **Trace visibility and failure.** Repeated writes and intermediate constants remain represented effects. If a later access fails, the evidence does not require rollback or establish that earlier effects were absent.

## Uncertainties

The following remain documented_only:

- Application purpose and logical data model.
- Calling convention and secondary result-state semantics.
- Indirect target identities, contracts, and return behavior.
- Named-global declarations, access widths, and declaration-dependent conversions.
- Executable control-flow fidelity and repeated-region multiplicity.
- Address mapping, alignment, byte order, and device-memory semantics.
- Signed overflow, negative signed right shifts, and oversized shift counts.
- Fine operand-evaluation ordering within compound access expressions.
- Ownership, lifetime, concurrency, and fault delivery.

**[INFERENCE]** Unusual target construction, extensive repetition, and low-address effects may indicate an imperfect representation of executable structure. This possibility does not justify replacing visible effects with a guessed algorithm.

## Evidence

The supplied static body is the sole behavioral evidence. It supports the represented input widths, indirect-invocation pattern, explicit pointer-access widths, integer transformations, memory-value provenance, and the fixed-global ordering described above.

Comments, warnings, prior prose, and reviewer feedback are not independent proof. Missing global declarations prevent deriving memory access widths from destination temporaries or assigned expression widths alone.

No executable, dependency implementation, verified memory map, calling-convention record, or runtime trace was available. No runtime tests, smoke runs, or fault observations have been performed.
