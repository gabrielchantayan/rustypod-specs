<!-- Trusted provenance: F-db4fab01567e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db4fab01567e

## Purpose

F-db4fab01567e invokes an ordered series of indirect dependencies, transforms their results, and performs mixed-width memory updates through computed addresses. It also accesses named global storage and finishes with an unresolved four-argument indirect invocation.

**Established static evidence:** The supplied representation has no explicit parameters or result, shows a straight-line dependency phase followed by a direct-memory phase, and shows a return after the terminal invocation.

**[INFERENCE]** The offset-based reads, scaled addresses, and repeated publications may implement a state-construction or state-update operation. No application-level purpose, object model, protocol, or resource-management role is established.

This contract describes static behavioral obligations, conditional on the supplied representation accurately reflecting execution. It does not treat recovered types, comments, or the prior draft as independent proof of machine behavior. Unresolved details are marked **documented_only**. No runtime tests have been performed.

## Interface

### Inputs and outputs

No explicit arguments are represented. Implicit inputs include:

- Incoming execution context selecting the first indirect target.
- Primary and secondary dependency outputs.
- Contents of computed memory locations at their respective read times.
- Named global values, including a stored address captured before the repeated publication block.

No explicit return value is represented. Observable outputs are dependency invocations, memory accesses, changes to storage, and the terminal control transfer. Reaching the shown return requires the terminal dependency to return normally.

### Widths and layout

| Access or quantity | Established representation |
|---|---|
| Explicit byte dereference | 8-bit access |
| Explicit halfword dereference | 16-bit access |
| Explicit word dereference | 32-bit access |
| Combined dependency result | 64-bit representation, often split into two 32-bit portions |
| Integer values used as addresses | Represented principally as 32-bit quantities |
| Named global access without a supplied declaration | Memory-access width **documented_only** |

An explicit unsigned byte read widened to a word contributes a value from 0 through 255. Explicit one-byte destinations retain an eight-bit representation; two-byte and four-byte destinations have distinct footprints.

A conversion of a named global’s value is not evidence of its memory-access width. In particular:

- The value obtained from global storage at 0x05 is converted to unsigned 32 bits. This does not establish an eight-bit read.
- The value obtained from global storage at 0xA9 is captured in a 32-bit quantity. This does not establish a 32-bit read.
- Assigning an eight-bit converted value to an undeclared global does not establish an eight-bit write.

Offsets are byte displacements, not demonstrated structure fields. Explicit accesses include offsets through 0x74, but no common allocation size or independent-object layout is established. Numeric selection of the upper and lower halves of a combined result does not establish memory byte order.

### Preconditions and ownership

Completion without interruption requires callable indirect targets, returning dependencies wherever continuation is needed, accessible storage for every reached access, and platform support for the encountered alignment and arithmetic.

The function does not visibly validate those requirements. No ownership transfer, buffer-size agreement, lifetime guarantee, synchronization contract, or reentrancy guarantee is established. No allocation or release is explicitly shown; dependencies may nevertheless allocate, retain, mutate, or release resources.

## Behavior

### Dependency phase

The represented dependency phase has no explicit conditional branch or loop. Results influence later arguments and addresses, rather than visibly selecting alternate paths.

The first invocation selects its target using incoming execution context. Its primary result, shifted left by eight positions, supplies the next invocation’s represented argument. Later invocations include:

- Calls with no represented arguments.
- Calls with explicit zeros in one or several positions.
- Shifted primary results and shifted upper portions of combined results.
- Earlier captured values reused after intervening calls.
- Equal values supplied in separate argument positions.
- Masked values, unsigned right shifts, and extractions from combined outputs.

Calls whose represented results are unused remain observable. A call displaying no arguments is not thereby proven pure or fully characterized as a zero-argument interface. Similar-looking call sites are not assumed to share a target.

Several arguments use secondary outputs whose producer interfaces are incomplete. These outputs are not established to be zero, preserved globals, or copies of primary results.

### Arithmetic contract

Most represented transformations operate on 32-bit quantities. Unsigned 32-bit shifts and arithmetic retain the width appropriate to those operations; unsigned right shifts insert zeros. Mask placement matters: a mask before a shift generally cannot be moved after it.

Visible transformations include retaining low-order bit ranges, selecting bits 10 through 19, and unsigned shifts of combined 64-bit quantities by 34, 36, or 44 positions. The dependency phase also carries captured values through multiple successive shifts; those intermediate widths must not be replaced with unlimited-precision arithmetic.

**documented_only:** Signed left shifts, multiplication, addition, subtraction, and conversions outside a signed range do not establish portable wraparound semantics. Machine-level truncation is plausible but is not an unconditional guarantee.

Address scaling by two, four, and eight occurs without visible bounds checks. A subtraction of 0x96 forms an adjusted value used throughout the later phase.

One late signed right shift takes its count from the low eight bits of a derived address. Behavior for negative shifted values or counts of at least 32 is unresolved. The contract does not assume that hardware masks the count.

### Initial direct-memory publications

The opening direct effects have an established order:

- A 32-bit publication at offset 0x14 from the upper portion of the final combined dependency result.
- An eight-bit publication at offset 0x09 from an earlier shifted-result base.
- A 32-bit publication at offset 0x54 from the final primary result scaled by 0x100.
- Another eight-bit publication to the same offset-0x09 destination.

A derived base’s offset-0x14 word is then cleared before its offset-0x05 byte is read. That explicit byte read is later published, zero-extended, to the base’s offset-0x34 word. Clears at another derived base’s offset 0x14 occur between these operations. A word is subsequently captured from the first cleared location; it must reflect the ordered accesses, not an assumed entry snapshot.

Two consecutive byte writes publish that captured word’s character-converted value to the other base’s offset 0x15.

### Early globals and linked values

The named global at 0x05 is read and its value converted to unsigned 32 bits **before** the assignment to the named global at 0x01. The latter assignment receives an eight-bit conversion of an earlier doubled index. Neither global’s declaration is supplied.

The converted value from 0x05 becomes an address base. Following intervening byte and word writes, a 32-bit value is captured from that base’s offset 0x54; that location is then overwritten with the earlier captured clearing-stage word. In this contract, the newly captured offset-0x54 value is the **captured link**.

The captured link supplies later self-publications, address calculations, and scaling. Its fourfold value is eventually reduced by 0x96; this is the **adjusted link**. These descriptive labels identify captured numeric values, not independent objects or ownership relationships.

Before that adjustment:

- The named global at 0x54 receives the captured link shifted left by one.
- Derived offset-0x14 destinations receive the captured link, including repeated writes.
- Derived offset-0x15 bytes receive character-converted fragments of scaled values.
- The named global at 0x15 receives an eight-bit conversion of the doubled index.

The assignment to global 0x54 therefore depends on the earlier offset-0x54 load through the converted 0x05 value. The assignment to global 0x15 depends on the doubled index, not the immediately preceding byte destination’s address or current contents.

Global access widths remain **documented_only**. Their ordering and value dependencies remain established despite that limitation.

### Linked-memory updates

The function reads words at offsets including 0x04, 0x54, 0x64, and 0x74 and uses those values as further bases or scaled numeric values. Explicit bytes read at offset 0x05 also supply address bases for offset-0x74 word reads.

Linked updates include:

- Writing an offset-0x74 value to its own offset-0x54 destination.
- Repeated self-publications at offset 0x24 and associated byte publications at offset 0x09.
- Publishing scaled values at offsets 0x14, 0x34, 0x44, and 0x54.
- Copying low-byte fragments to offsets 0x09, 0x0D, and 0x15.
- Re-reading an offset-0x05 byte after intervening writes before using it for another linked read.

The explicit rereads cannot generally be replaced with earlier captured values. A nested expression also contains two represented reads of the same offset-0x74 location; equality of their results is not guaranteed for externally changing storage, and their internal evaluation order is not established.

The displacement 100 is 0x64. It must not be confused with the later literal value 100 stored at offset 0x54.

### Constant publications and stored-address capture

A word obtained through a byte-derived offset-0x74 read supplies scaled destinations for three writes of 0x3A:

| Destination relationship | Width |
|---|---:|
| Fourfold-scaled address | 8 bits |
| Eightfold-scaled address | 16 bits |
| Fourfold-scaled address plus 0x20 | 32 bits |

The adjusted link’s fourfold value forms a separate publication base. Its offset-0x20 word receives, in order, 0x20, 0x2F, an explicit offset-0x04 byte widened to 32 bits, and 0x20 again.

After those accesses, the function captures a named stored-address value into a represented 32-bit quantity. This capture occurs **before** assigning 0x20 to the named global at 0x60. The declaration and read width of the stored-address global, and the write width at 0x60, are **documented_only**.

Consequently, aliasing between the 0x60 assignment and stored-address storage does not justify reloading the address after that assignment.

### Repeated publication block

The captured address is reused throughout a long fixed repetition. Publications use:

- The low eight bits of the adjusted link’s fourfold value at captured-address offset 0x19.
- The low eight bits of the captured address at the fourfold-adjusted base’s offset 0x11.
- A scaled word at captured-address offset 0x34.

The initial offset-0x34 publication uses eight times the unadjusted captured link. Subsequent publications use eight times the adjusted link. Between publication groups, another offset-0x54 word repeatedly receives a captured fourfold-scaled linked value.

The addresses do not visibly advance as array elements. Replacing the block with one final store per destination is not a generally equivalent trace. Exact authoritative repetition count remains **documented_only**; the distinct initial source, repeated address relationships, and ordering within the recurring pattern are established.

### Post-repetition global ordering

The named global at 0xA9 is read after the repeated block and captured as a represented 32-bit value. This read precedes both:

1. An explicit 32-bit read at offset 0x74 from the captured fourfold-scaled linked base.
2. The assignment to the named global at 0x3A.

The 0x3A assignment depends on the **adjusted link**, converted to the represented character type, multiplied by eight, and increased by 0x30. The character conversion occurs before the multiplication. It is not a literal assignment of 0x3A, and it is unrelated to the earlier three writes whose value is 0x3A.

The assignment’s final memory width is unknown because the global declaration is absent. Character signedness and out-of-range conversion behavior also remain unresolved.

The function then captures words from another derived base’s offset 0x14 and from the fourfold-adjusted base’s offset 0x04. Following byte publications and a self-publication at offset 0x54, the captured 0xA9 value supplies an eight-bit conversion assigned to global 0x67.

An explicit derived offset-0x44 word read occurs **after** that assignment and **before** the copy from global 0x67 to global 0x21. Thus, the copy uses a fresh global read, not the previously captured 0xA9 value.

Two subsequent explicit byte publications at the fourfold-adjusted base’s offset 0x09 each obtain their value from a fresh read of global 0x67. Intervening accesses separate those reads. Neither can unconditionally be replaced with the original assignment value or the value used for the 0x21 copy.

### Late updates and terminal invocation

Late effects include:

- A literal byte 0x6C at offset 0x08.
- Byte publications at offsets 0x09, 0x0D, 0x10, and 0x11.
- The variable-count shifted character publication described above.
- Replacement of an offset-0x10 word with a captured offset-0x40 word.
- Publication of the captured 0xA9-derived 32-bit value at offset 0x44, later overwritten there by another captured link.
- Restoration of the earlier captured offset-0x14 word to an offset-0x54 destination.
- A 16-bit write of 0x65 through a doubled address.
- A 32-bit write of literal 100 at offset 0x54.
- A final 32-bit publication through another doubled address.

“Restoration” means reuse of one captured value, not rollback.

The terminal dependency receives four represented arguments:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | A 32-bit word loaded late from offset 0x64 |
| Third | The post-repetition word captured earlier from offset 0x14 |
| Fourth | A 32-bit word loaded late from offset 0x74 |

The second and fourth arguments use the base obtained from a late offset-0x04 word read. The third argument is the captured value, not a terminal-time reload. No status check, result conversion, or explicit cleanup follows.

## Side effects

### Named low-address storage

All entries below are named globals whose declarations are unavailable. Their addresses do not independently prove access widths.

| Address | Represented access and dependency | Memory width |
|---|---|---|
| 0x05 | Read; value converted to unsigned 32 bits and used as a base | **documented_only** |
| 0x01 | Assigned an eight-bit conversion of the doubled index, after the 0x05 read | **documented_only** |
| 0x54 | Assigned the captured link shifted left by one | **documented_only** |
| 0x15 | Assigned an eight-bit conversion of the doubled index | **documented_only** |
| 0x60 | Assigned 0x20 after the stored-address capture | **documented_only** |
| 0xA9 | Read before the derived offset-0x74 read; captured as 32 bits | **documented_only** |
| 0x3A | Assigned the character-converted adjusted link scaled by eight plus 0x30 | **documented_only** |
| 0x67 | Assigned an eight-bit conversion of the 0xA9 capture; subsequently read three times | **documented_only** |
| 0x21 | Assigned from the first subsequent fresh 0x67 read | **documented_only** |

These globals are distinct from explicitly typed computed dereferences, whose represented widths are supported. The supplied overlapping-storage warning prevents assuming that separate storage names identify disjoint objects.

Ordering matters under aliasing. Word writes can affect later byte reads; captured values remain captures, while fresh reads observe storage at their own access times. Ordinary memory, volatile storage, device registers, and concurrently changing storage can give repeated accesses different significance. No synchronization or atomicity is established.

No error result, rejection branch, retry, exception handler, rollback, or compensating cleanup is represented. A fault or nonreturning dependency can leave earlier effects visible and prevent all later effects.

## Dependencies

Only document-local opaque references are assigned here; they do not resolve external identities.

| Opaque ID | Role | Established interface fragment |
|---|---|---|
| F-000000000001 | Entry-selected dependency | No represented arguments; primary output feeds the opening shift |
| F-000000000002 | Immediate consumer | Receives that output shifted left by eight |
| F-000000000003 | Terminal dependency | Receives the four arguments specified above |

Intervening targets, complete calling conventions, secondary-output provenance, and resource effects remain **documented_only**. No dependency is assumed to preserve globals, preserve pointed-to memory, or acquire or release ownership.

## Acceptance examples

These are conditional static acceptance examples, not observed executions.

### Opening shift

If F-000000000001 returns 0x00001234 and the represented shift applies, F-000000000002 receives 0x00123400.

### Explicit byte widening

If the clearing-stage explicit offset-0x05 byte read produces 0xE7, its later word publication contains 0x000000E7. This rule does not imply that the separate named global at address 0x05 is read as one byte.

### Global-derived address

If the value read from global 0x05 converts to 0x00000120, the subsequent offset-0x54 capture through that base addresses 0x00000174, assuming the represented address arithmetic. Acceptance requires the global read before the 0x01 assignment; it does not prescribe the global read’s width.

### Captured link and adjustment

If the captured link is 0x40 and these operations are representable, the value assigned to global 0x54 is 0x80. The adjusted link is 0x6A. The later 0x3A assignment therefore derives from character-converting 0x6A, multiplying by eight, and adding 0x30. Its final stored representation remains conditional on the global’s declared type.

### Stored address versus overwrite

If the stored-address global aliases storage changed by the 0x60 assignment, the publication block still uses the address captured before that assignment. Reloading afterward can select a different destination.

### Fresh global reads

If global 0x67 changes between its initial assignment and the copy to 0x21, the copy uses the later read value. Changes before either subsequent offset-0x09 publication can likewise affect that publication independently.

### Ordered overwrites

Absent relevant interference, the four offset-0x20 word writes leave 0x20. A trace-sensitive observer must still account for 0x2F and the widened-byte publication.

### Width and terminal capture

The three constant 0x3A publications require their respective 8-, 16-, and 32-bit footprints. Separately, modifying the terminal third argument’s original offset-0x14 location after capture does not change that captured argument.

### Interrupted execution

An inaccessible computed destination does not establish graceful rejection. Previously completed effects may remain; no rollback or error code is promised.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended state invariants.
- Dependency identities, complete interfaces, and secondary-output provenance.
- Named-global declarations, access widths, and assignment conversions.
- Signed overflow, character conversion boundaries, and excessive shift counts.
- Endianness, alignment requirements, and address-space interpretation.
- Object extents and exact alias relationships.
- Whether apparent low-address globals correspond to ordinary mapped storage.
- Authoritative repetition count and machine-level observability of repeated accesses.
- Concurrency, synchronization, reentrancy, ownership, and lifetimes.
- Whether the terminal target returns normally.

These limits do not erase established ordering or value dependencies. They prevent turning those dependencies into an unsupported complete executable model.

## Evidence

Evidence is limited to the supplied static representation of F-db4fab01567e. Direct support exists for the represented interface, dependency-driven transformations, explicitly typed mixed-width accesses, named-global ordering, captured-value reuse, fresh global reads, and final four-argument invocation.

Missing global declarations constrain width claims. Overlapping-storage warnings and incompletely represented call outputs constrain alias and dependency claims. Neither the prior draft nor reviewer feedback independently establishes behavior.

No implementation access, target resolution, runtime execution, memory tracing, or tests were performed. Acceptance examples express conditional obligations derived from static evidence only.
