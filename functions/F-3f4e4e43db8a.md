<!-- Trusted provenance: F-3f4e4e43db8a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f4e4e43db8a

## Purpose

F-3f4e4e43db8a performs an ordered sequence of unresolved indirect invocations, derives address-valued quantities from dependency results, modifies memory through mixed-width accesses, and reaches a final unresolved control transfer.

**Established static evidence:** the supplied representation displays no formal parameters or explicit scalar return value. Its inputs include incoming execution context, dependency results, global storage, and memory at computed addresses. No conditional validation, explicit error result, or rollback is visible.

**[INFERENCE]** Some accesses resemble linked-record maintenance, and some literal values resemble character data. These similarities do not establish a record type, output format, device, or application purpose.

This contract describes represented behavior, not demonstrated execution. Reachability and the accuracy of the supplied representation remain unresolved.

## Interface

### Inputs and outputs

The apparent parameterless interface has implicit inputs:

- An incoming control-context value used to derive the first indirect destination.
- Two incoming 32-bit quantities used in early shifted arguments.
- Primary and secondary results of dependency invocations, including results represented as combined 64-bit quantities.
- Existing bytes and words at computed and fixed addresses.
- Global storage containing a base used during the repeated-write region.
- The destination and execution requirements of the final unresolved transfer.

No complete calling convention, register-preservation contract, or dependency signature is established.

Observable outputs consist of dependency invocations, memory reads, memory writes, and the final transfer. There is no established scalar success or failure result. A displayed return follows the final transfer, but normal return is conditional on that destination permitting continuation.

### Widths and layout

| Item | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Ordinary integer and word access | 32 bits |
| Combined dependency result | 64 bits |
| Integer-derived address | Apparently 32 bits |

All offsets are byte offsets. Repeated offsets do not prove a shared structure layout.

Byte-to-word conversions are zero-extending where an unsigned byte is read. Byte stores retain the low eight bits; halfword stores retain the low sixteen bits and affect two bytes. Word stores affect four bytes. Endianness is not established.

Fixed-address effects include byte accesses at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, and word accesses at 0x54, 0x60, and 0xA9. Computed accesses use offsets including 0x01, 0x04, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74.

### Execution preconditions

Completion requires every reached indirect destination to be executable and compatible with the actual execution context. Every reached memory access requires storage supporting its address, width, access direction, and applicable alignment restrictions.

These are environmental requirements, not checks performed by the function. No null, bounds, length, alignment, ownership, or lifetime validation is visible. Low numeric addresses must not be silently reinterpreted as valid allocated-object pointers.

## Behavior

### Invocation-driven preparation

The preparation region contains a long, fixed sequence of indirect invocations, with no visible conditional selection between paths.

The first target is derived from incoming control context using a displacement of 0xFFE. Subsequent destinations are derived from embedded control-location or address-valued references with the same displacement. This construction does not establish common dependency identity or semantics.

Displayed invocations range from no explicit arguments to four word-sized arguments. An invocation displaying no arguments does not establish the absence of implicit register inputs.

The preparation dataflow includes:

- Left-shifted incoming values.
- Primary results shifted before later invocations.
- Low and high 32-bit portions of combined 64-bit results.
- Secondary results whose complete signatures are unresolved.
- Values retained across intervening invocations.
- Explicit zero arguments in numerous later positions.

Invocation order is part of the represented behavior. Similar-looking calls cannot be merged, eliminated, or reordered on the assumption that they are pure or interchangeable.

### Arithmetic constraints

Unsigned 32-bit left shifts retain only the low 32 result bits. Subsequent right shifts do not recover discarded bits. A left shift by 25 followed by a logical right shift by two therefore differs from shifting the original value left by 23.

Logical shifts of combined 64-bit results by 34, 36, and 44 select the high result word shifted right by 2, 4, and 12 respectively. They are not oversized shifts of a 32-bit operand.

Masks select various low-order regions, including 24-, 22-, 21-, 20-, 19-, 18-, 17-, 16-, 14-, 13-, 12-, and 11-bit regions. These numerical selections do not establish semantic field meanings.

Signed-looking shifts, multiplications, and address additions occur as well. Their behavior on overflow or otherwise invalid signed operations is **documented_only**. Conditional examples using 32-bit modular arithmetic do not establish that model for every represented expression.

### Address families and initial stores

The late preparation establishes several address families:

- A dependency result shifted left by eight.
- Another result multiplied by 2²⁵.
- An earlier result shifted left by 25.
- A small index derived from another result.
- Twice that small index.

Under a 32-bit bit-vector interpretation, the small index selects **bits 0–6 of the original dependency result and multiplies that selection by four**. Equivalently, after shifting the result left by eight, the extraction selects bits 8–14 of that shifted quantity and rescales them. It does not select bits 8–14 of the original result.

The small index ranges from 0 through 508 in steps of four. Its doubled base ranges from 0 through 1016 in steps of eight.

The first direct stores have this relative order:

- A word containing the retained left-shift-by-25 value is written at offset 0x14 from the final dependency’s secondary-result base.
- That secondary result’s low byte is written at offset 0x09 from the left-shift-by-25 base.
- The 2²⁵-scaled value is written as a word at offset 0x54 from a base formed by scaling the final primary result by 256.
- The byte at the earlier offset-0x09 destination is overwritten with the retained shifted value’s low byte.

Under the bit-vector interpretation, the last byte is zero. The preceding byte store remains a distinct access.

### Interleaved memory updates

The following region clears offset 0x14 of the 2²⁵-scaled base, reads its unsigned byte at offset 0x05, clears offset 0x14 of the doubled-index base, and stores the zero-extended byte at offset 0x34 of the scaled base. It then reloads the scaled base’s offset-0x14 word before clearing the doubled-index destination again.

The reloaded word participates in later addresses and duplicate byte stores. It cannot unconditionally be replaced by zero: the intervening computed stores may overlap the cleared word.

Further effects include:

- Consecutive duplicate byte stores at offset 0x15 of the doubled-index base.
- Reading the byte at fixed address 0x05 and using its zero-extended value as a base.
- Writing the doubled-index base’s low byte to fixed address 0x01.
- Byte updates at offsets 0x15 and 0x11.
- Writing through offset 0x54 of the reloaded word-valued base.
- Reading offset 0x54 of the byte-derived base before overwriting that location.
- Storing the small index at offset 0x34 of the doubled-index base.

Call the word captured before that offset-0x54 overwrite the **captured link**. Subsequent effects use it both numerically and as an address. They include repeated self-valued word stores at its offset 0x24, byte stores through another loaded link, and successive updates of offset 0x14 of the doubled-index base.

Fixed address 0x54 receives twice the captured link as a word. Fixed address 0x15 receives a byte derived from the doubled-index base.

The next region uses fourfold and eightfold versions of the captured link, words read through offset 0x74, and byte-derived bases. It modifies offsets 0x09, 0x0D, 0x14, 0x15, 0x34, 0x44, and 0x54. Some destinations are overwritten with different categories of value: doubled links, zero-extended bytes, and scaled address-valued words.

Word-valued accesses at offsets 0x04 and 0x64 supply further bases. Repeated reads of byte positions can observe intervening writes. These operations establish an access history, not merely final field values.

### Literal-valued writes and repeated groups

A later loaded link supplies three destinations receiving 0x3A: a byte at its fourfold address, a halfword at its eightfold address, and a word at offset 0x20 from its fourfold address.

An **adjusted fourfold base** is formed by subtracting 0x96 from the earlier fourfold captured-link quantity, then multiplying by four. Its offset 0x20 receives four successive word stores: 0x20, 0x2F, a zero-extended byte read from its offset 0x04, and 0x20 again. The byte read occurs after the first two stores.

The repetition base is then **loaded from global storage before** the word value 0x20 is written to fixed address 0x60. Subsequent repeated groups use that captured base. Moving its load after the 0x60 store is not equivalent when the store overlaps the global storage.

Each repeated group has four ordered stores:

- A word at offset 0x54 of a scaled base, containing the fourfold version of the later loaded link.
- A byte at offset 0x19 of the captured repetition base, containing the adjusted fourfold base’s low byte.
- A byte at offset 0x11 of the adjusted fourfold base, containing the repetition base’s low byte.
- A word at offset 0x34 of the repetition base, containing the group’s scaled destination base.

The first group uses eight times the captured link for the scaled base. Later groups use eight times the subtraction-adjusted quantity. The representation repeats accesses to these same derived destinations; it does not establish traversal over distinct records.

The exact repetition count is **documented_only** in this contract. No invented iteration bound or single-store replacement is accepted.

### Final-phase capture boundaries

The final region retains a base previously obtained by doubling a loaded link. Here it is called the **retained record base**, without implying an actual record type.

Before the byte store to fixed address 0x3A, the function captures:

- The word at fixed address 0xA9.
- The word at offset 0x74 from the fourfold later-link base used in the repeated groups.

It then writes to 0x3A the low byte of eight times the subtraction-adjusted quantity plus 0x30. No decimal-digit range restriction is visible.

**After** this byte store, it captures:

- The word at offset 0x14 of the retained record base.
- The word at offset 0x04 of the adjusted fourfold base.

These later captures must observe memory after the 0x3A write, including any overlap. The saved offset-0x14 value is subsequently reused both as stored data and as the transfer’s third displayed argument.

### Final updates and fresh fixed-address reads

The following updates include a byte at offset 0x0D of the retained record base, literal byte 0x6C at offset 0x08 of the adjusted fourfold base, and repetition-base-derived bytes at its offsets 0x0D and 0x09. The retained record’s offset 0x54 is set to its own base value.

The captured 0xA9 word’s low byte is then stored at fixed address 0x67. Next, the retained record’s offset-0x44 word is loaded. Fixed address 0x21 receives a **fresh byte read from 0x67**, not a stipulated copy of the original capture.

The loaded offset-0x44 value supplies a byte at adjusted-base offset 0x10 and a word at repetition-base offset 0x54. A byte at retained-record offset 0x01 is derived from a signed-looking right shift of that loaded word, using the adjusted base’s low byte as the count. The repetition base’s offset 0x54 is then overwritten with the adjusted base.

The function also:

- Captures the adjusted base’s offset-0x40 word.
- Stores the earlier offset-0x04 capture at repetition-base offset 0x74.
- Stores the offset-0x40 capture as a word at adjusted-base offset 0x10.
- Stores the retained record base at repetition-base offset 0x44.

Adjusted-base offset 0x09 next receives another fresh byte read from 0x67. After intervening computed stores, including the captured 0xA9 word at adjusted-base offset 0x44, the adjusted base’s offset-0x54 word is loaded. Adjusted-base offset 0x09 then receives **another** fresh byte read from 0x67.

Thus, three later accesses read 0x67: one for fixed address 0x21 and two for computed byte destinations. Intervening computed stores may overlap 0x67; the values need not all equal the initially stored byte.

The saved offset-0x74 capture supplies low-byte stores through the newly loaded link and at retained-record offset 0x11. The retained record’s offset 0x54 receives the saved offset-0x14 word. A fresh word from adjusted-base offset 0x14 supplies a later low-byte store.

### Terminal stores and unresolved transfer

A word loaded from retained-record offset 0x04 becomes the **terminal link base**. A halfword containing 0x65 is written at twice that base. Its offset-0x64 word is then captured as the transfer’s second displayed argument.

The previously loaded adjusted-base link receives a byte at offset 0x11 and the word value 100 at offset 0x54. The terminal link base’s offset-0x74 word is captured as the transfer’s fourth displayed argument.

The final memory effects then include:

- Loading a destination base from retained-record offset 0x64.
- Writing the terminal link base’s low byte at retained-record offset 0x09.
- Writing the saved second-argument word at offset 0x54 of the destination base.
- **Writing that destination base itself as a word at twice the saved second-argument value.**

The last item is a separate direct store, not merely address traversal or argument preparation. It occurs before the unresolved transfer.

The transfer receives four displayed arguments: 0x72, the saved terminal-link offset-0x64 word, the earlier retained-record offset-0x14 capture, and the saved terminal-link offset-0x74 word. Later aliasing writes do not turn these saved arguments into fresh reads.

## Side effects

Direct writes modify addressed storage without any established allocation, release, ownership transfer, or lifetime extension.

Dependencies may mutate globals, alter subsequently accessed memory, allocate, release, perform I/O, fail, or not return. None is assumed to preserve global storage or memory ownership.

Duplicate and overwritten stores remain represented effects. Intermediate accesses may matter for overlapping storage, device-backed memory, observers, or faults. No atomicity, synchronization, reentrancy, or thread-safety guarantee is established.

Failure can leave a prefix of effects committed. No transactional boundary, recovery path, or explicit error mapping is visible.

## Dependencies

Only the subject function has an authoritative supplied opaque identity. The following local opaque labels distinguish unresolved dependency classes:

| Opaque ID | Role | Status |
|---|---|---|
| F-U001 | First context-derived indirect destination | documented_only |
| F-U002 | Subsequent preparation destinations | documented_only |
| F-U003 | Final unresolved transfer destination | documented_only |

F-U002 represents multiple call sites, not one interchangeable helper. Their precise identities, signatures, preservation rules, ownership effects, failure behavior, and completion behavior are unresolved.

## Acceptance examples

These are conditional static examples, not executed tests.

### Original-result bit selection

Under 32-bit bit-vector arithmetic, original dependency results 0 and 0x100 both produce small index zero. Original result 127 produces 508 and doubled base 1016. An interpretation selecting original bits 8–14 incorrectly distinguishes the first two inputs.

### Mixed-width literals

Given writable, nonoverlapping destinations, the three 0x3A stores affect one, two, and four bytes respectively. Widening all three to word stores violates the access contract.

### Captured repetition base

If repetition-base storage overlaps the word store at 0x60, the groups still use the value loaded before that store. Reloading afterward can select a different base and is not equivalent.

### Capture across the 0x3A write

If the 0x3A byte destination overlaps the retained record’s offset-0x14 word, the saved third argument reflects the post-write word. Conversely, the 0xA9 and earlier offset-0x74 captures precede the write and retain their captured values.

### Fresh reads at 0x67

If the initial byte stored at 0x67 is 0x12, but an intervening computed store changes that byte to 0x34 before a later computed-byte copy, that copy must use the fresh read. Reusing 0x12 for every destination violates the represented dependency on current memory.

### Final direct store

If the saved second argument is 0x1000 and the final destination-base load yields 0x2000, successful execution writes word 0x1000 at 0x2054 and word 0x2000 at 0x2000 before F-U003. Both are required effects.

### Partial failure and saved arguments

A nonreturning preparation dependency prevents any guarantee of later stores. A fault in the final direct store prevents any guarantee of reaching F-U003, without rolling back earlier effects. If the transfer is reached, its saved arguments must not be replaced by new memory reads.

## Uncertainties

The following remain **documented_only**:

- Application purpose and meaningful object layouts.
- Accuracy, completeness, and reachability of represented control flow.
- Full invocation trace, dependency identities, and calling conventions.
- Exact repeated-group count as specified by this document.
- Endianness, alignment, address validity, and device-memory semantics.
- Signed overflow, signed left-shift behavior, and negative right-shift behavior.
- Variable right-shift behavior for counts at least 32; count masking is not established.
- Concrete alias relationships and asynchronous memory changes.
- Dependency allocation, release, ownership, and lifetime effects.
- Whether the final destination returns, dispatches elsewhere, or terminates execution.

Overlap cautions do not independently establish particular aliases. Conversely, the absence of an established alias does not justify reordering accesses or replacing fresh reads with saved values.

## Evidence

Established claims derive from static inspection of the supplied representation: operand widths, masks, shifts, displayed invocation ordering, access offsets, literal values, capture boundaries, fresh reads, and the final stores and transfer.

The small-index selection follows from applying the mask after the eight-bit shift. The global-base ordering follows from its capture preceding the 0x60 store. The final capture boundaries and 0x67 behavior follow from their distinct read positions. The doubled-second-argument destination is supported by an explicit final word store.

No implementation access, dependency inspection, runtime execution, smoke scenario, or runtime test has been performed. Acceptance examples describe conditional consequences, not observed outputs.
