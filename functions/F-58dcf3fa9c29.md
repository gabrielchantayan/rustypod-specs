<!-- Trusted provenance: F-58dcf3fa9c29; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-58dcf3fa9c29

## Purpose

F-58dcf3fa9c29 performs a fixed sequence of indirect dependency invocations, transforms their result channels, mutates storage through computed and loaded addresses, and reaches an unresolved final indirect transfer.

The supplied representation does not establish an application-level purpose. Repeated offsets and character-sized constants do not establish text construction, object initialization, collection management, or a protocol.

This contract distinguishes:

- **Established static evidence:** represented operations, explicit access types, value conversions, retained values, and ordering relationships.
- **[INFERENCE]:** possible intent, not a behavioral requirement.
- **documented_only:** behavior whose resolution requires missing declarations, dependency contracts, machine semantics, or trace detail not enumerated here.

The principal observable results are dependency invocations and storage effects. No useful returned value is established.

## Interface

### Inputs and result channels

The represented interface has no explicit parameters and no explicit returned value. Implicit inputs include incoming return-link state, dependency-result channels, reachable memory, fixed-address globals, a separately stored publication-base value, and the context of the final transfer.

Some dependency results are represented as 64-bit composites. Their low and high 32-bit words are consumed separately. This does not establish an eight-byte memory layout or serialization format. Other secondary result channels have incompletely established definitions.

No explicit parameters does not mean independence from caller state.

### Widths: accesses versus conversions

Explicitly typed pointer accesses establish the following representation-level widths:

| Access category | Width | Value interpretation |
|---|---:|---|
| Unsigned-byte read | 8 bits | Zero-extended when converted to a word |
| Byte or character store | 8 bits | Stores the destination byte representation |
| Halfword store | 16 bits | Includes constants 0x003a and 0x0065 |
| Word read or store | 32 bits | Signed, unsigned, or uninterpreted according to its represented use |
| Composite result channel | 64 bits | Supports full-width logical extraction |

**An undeclared global is different from an explicitly typed pointer access.** Converting its value to 32 bits does not establish a 32-bit load. Assigning a value first narrowed to eight bits does not establish an eight-bit destination.

For the fixed-address globals, the established facts are:

| Location | Established value operation | Unresolved storage facts |
|---|---|---|
| 0x05 | Its value is converted to unsigned 32 bits and used as an address base | Original width, signedness, and load extent |
| 0x01 | Receives a value first converted to eight bits | Destination width and assignment conversion |
| 0x15 | Receives a value first converted to eight bits | Destination width and assignment conversion |
| 0x54 | Receives a signed-word shift-derived value | Destination width and assignment conversion |
| 0x60 | Receives the numeric value 0x20 | Destination width and signedness |
| 0xa9 | Its value is captured in a 32-bit uninterpreted result | Original width, signedness, and conversion details |
| 0x3a | Receives a character-conversion-derived arithmetic value | Destination width, signedness, and final conversion |
| 0x67 | Initially receives a value first narrowed to eight bits; is subsequently read three times | Storage width, signedness, and read extent |
| 0x21 | Receives the then-current value of the global at 0x67 | Source and destination declaration-dependent conversions |

The separately stored publication-base global is captured in a signed 32-bit value. Its declaration is unavailable; a 32-bit capture does not independently establish its underlying load width.

These unresolved storage properties are **documented_only**. Fixed addresses remain relevant to aliasing, but their complete occupied address ranges cannot be derived from their names.

### Arithmetic, errors, and ownership

Unsigned operations follow their represented widths. Intermediate truncation matters: a 32-bit left shift followed by a logical right shift does not recover discarded high bits. Signed overflow, signed left shifts, signed right shifts, character signedness, and oversized variable-shift counts require machine-level clarification.

Byte order, alignment requirements, atomicity, and memory-mapped behavior are **documented_only**.

Normal progression requires each reached dependency to be callable and return, each reached memory access to be permitted, and the final transfer to be valid. No represented null, bounds, alignment, or lifetime checks enforce those requirements.

There is no established status result, error-reporting protocol, rollback, allocation, release, reference-counting, or ownership transfer. Unknown dependencies must not be assumed to preserve globals, retain object lifetimes, or refrain from acquiring or releasing memory.

## Behavior

### Indirect-invocation prefix

The first dependency, F-58dcf3fa9c29-D001, is invoked through a destination derived from incoming return-link state plus 0x0ffe. Subsequent invocations use distinct embedded-anchor-derived target expressions with the same displacement. They are represented collectively by F-58dcf3fa9c29-D002, not as one interchangeable helper.

The prefix is straight-line in the supplied representation. No input-sized loop, conditional choice, retry, or early-return decision is shown. Every invocation remains significant, including those with unused results, zero-valued represented arguments, or no shown arguments. The last category does not prove that incoming register state is irrelevant to the callee.

The first composite result supplies four represented positions to the next invocation: its low word shifted left by eight, its unshifted high word, and two copies of its low word shifted left by fifteen.

Later arguments combine fresh results with retained earlier results. Relationships include:

- Low-word shifts by eight and other fixed counts.
- High-word left shifts, including variants from six through sixteen.
- Repeated transformed arguments and explicit zeros.
- Arguments derived from incompletely defined secondary results.
- Masks selecting low-bit fields, including widths from eleven through twenty-four bits and single-bit selection.
- Full 64-bit logical shifts by 34, 36, and 44 bits.

A full-result shift by 44 extracts the high word shifted right by twelve. At least one extraction uses an older retained composite rather than the newest result.

Masking to twenty bits before logically shifting right by ten selects bits 10–19. Masking to twenty-four bits before shifting left by six differs from shifting an unrestricted word.

Under truncating 32-bit semantics, successive left shifts totaling at least 32 clear a value when nothing intervenes to change it. Explicitly initialized zeros also remain zero through subsequent shifts. Neither observation permits removal of surrounding calls.

The complete per-invocation argument schedule is **documented_only** in this contract. These relationships do not define a substitute executable schedule.

### Retained address relationships

The following names describe numeric relationships, not object types:

| Name | Relationship |
|---|---|
| Packed base | A retained dependency low word shifted left by 25 |
| Scaled base | A later dependency word multiplied by 2²⁵ |
| Small component | A dependency word shifted left by eight, masked to fifteen bits, then logically shifted right by six |
| Small base | Twice the small component |
| Converted global base | The value of the global at 0x05 converted to unsigned 32 bits |
| Chain word | An explicitly typed 32-bit read at converted-global-base offset 0x54 |
| Fourfold base | Four times the chain word |
| Biased value | Fourfold base minus decimal 150 |
| Adjusted base | Four times the biased value |
| Publication base | A separately captured signed 32-bit value from an undeclared global |
| Terminal lookup word | A later 32-bit read at offset 0x74 through an explicitly unsigned-byte-derived base |
| Element base | Four times the terminal lookup word |
| Doubled-link base | Twice the terminal lookup word |

The converted global base is **not established to lie between 0 and 255**. Its unsigned 32-bit conversion does not establish an unsigned-byte read. This differs from the terminal lookup chain, whose relevant address-producing read is explicitly an unsigned-byte access.

Under truncating word arithmetic, the packed base has its low 25 bits clear. The small component is a multiple of four from 0 through 508; the small base is a multiple of eight from 0 through 1016.

The chain word and terminal lookup word are observations made after preceding mutations, not immutable entry inputs.

### Initial publications and fixed-address globals

The direct-mutation phase begins with a 32-bit publication of the packed base at offset 0x14 from the final composite result’s high word. An explicit byte store then publishes that high word’s character-converted value at packed-base offset 9.

A 32-bit store publishes the scaled base at offset 0x54 from the final composite low word multiplied by 256. Another byte store overwrites packed-base offset 9 with the packed value’s character-converted representation. Both byte stores remain separate events.

The scaled-base offset-0x14 word is cleared before an explicit unsigned-byte read at scaled-base offset 5. That observed byte is later zero-extended into a 32-bit publication at scaled-base offset 0x34. Small-base offset 0x14 is also cleared, including a repeated clear. The scaled-base offset-0x14 word is read after the intervening publications, and repeated byte stores use small-base offset 0x15.

The global at absolute 0x05 is read and converted to unsigned 32 bits **before** the assignment to the global at 0x01. The latter receives the small base first narrowed to eight bits; its actual storage width is unknown.

An explicit byte publication at converted-global-base offset 0x15 precedes the 32-bit read at that base’s offset 0x54. That read supplies the chain word, and the same word location is subsequently replaced. Accesses through other retained bases intervene; they cannot be presumed disjoint.

The assignment to the global at 0x54 precedes the assignment to the global at 0x15. Their destination widths are not established. Between them are explicitly typed stores through the small base, including repeated byte stores at offset 0x15. When the small base is zero, those computed stores reach absolute 0x15, but they remain distinct from the later global assignment.

### Chained mutations and constants

Loaded words and explicitly read unsigned bytes become bases for further accesses. The mutation graph uses offsets including 0x04, 0x05, 0x09, 0x0d, 0x11, 0x14, 0x15, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74. It is not confined to one contiguous record.

Established effects include self-valued word publications, repeated stores, doubling and quadrupling, eightfold scaling, repeated reads after intervening mutations, and overlapping mixed-width accesses. An explicit byte store at offset 0x15 overlaps a 32-bit word beginning at offset 0x14 when their bases agree. Which word bits change depends on byte order.

Some chained expressions reread an offset-0x74 word rather than using a permanently cached link. Repeated unsigned-byte reads likewise remain separate observations.

The constant-0x3a group contains three explicitly typed stores:

- Eight bits at the element base.
- Sixteen bits at eight times the terminal lookup word.
- Thirty-two bits at element-base offset 0x20.

At adjusted-base offset 0x20, successive 32-bit publications carry 0x20, 0x2f, the zero-extended result of an explicit unsigned-byte read at adjusted-base offset 4, and finally 0x20. Intermediate effects remain part of the trace.

### Publication-base capture and repeated publications

The publication-base global is captured **after** the final adjusted-base offset-0x20 publication and **before** the global assignment of 0x20 at absolute 0x60. Neither global’s underlying access width is established.

Subsequent operations use the captured 32-bit publication-base value. No reload after the 0x60 assignment is represented, even if that assignment changes the storage originally supplying the publication base.

The repeated publication region has four ordered roles within each group:

1. A 32-bit publication of the element base at offset 0x54 from an eightfold-scaled base.
2. An explicit byte publication of the adjusted base’s low eight bits at publication-base offset 0x19.
3. An explicit byte publication of the publication base’s low eight bits at adjusted-base offset 0x11.
4. A 32-bit publication at publication-base offset 0x34.

The first group uses eight times the chain word for its scaled base and final published value. Subsequent groups use eight times the biased value.

This is a finite represented run, not an input-sized traversal. Its exact repetition count is **documented_only** here. This coverage limitation does not authorize replacing repeated accesses with final-state assignments.

### Final global observations and mutable storage at 0x67

The value of the global at absolute 0xa9 is captured in a 32-bit result **before** the explicit 32-bit read at element-base offset 0x74. Both precede the assignment to the global at 0x3a.

The 0x3a assignment uses the biased value converted to a character, multiplied by eight, and increased by 0x30. It is not an assignment of the constant 0x3a. Character signedness and destination conversion remain unresolved. Only conditionally, if the destination ultimately retains the low eight bits, does its byte pattern equal the low eight bits of eight times the biased value plus 0x30.

The final region captures 32-bit words from doubled-link-base offset 0x14 and adjusted-base offset 4. It also performs byte publications through these bases, writes 0x6c through an explicit byte destination at adjusted-base offset 8, and publishes the doubled-link base as a word at its offset 0x54.

The global at 0x67 is mutable storage, **not an established byte object**. Its initial assigned value is the saved 0xa9 result narrowed to eight bits. Its destination width and signedness remain **documented_only**.

Three subsequent global reads are represented:

- After a 32-bit read at doubled-link-base offset 0x44, a read supplies the assignment to the global at 0x21.
- A later read supplies an explicit byte store at adjusted-base offset 9.
- Another read supplies a second explicit byte store at that same destination.

The 0x21 assignment has declaration-dependent conversion semantics. The two adjusted-base stores are established eight-bit destinations, so only their stored byte representations survive those destination conversions.

The earlier adjusted-base offset-9 byte publication, before the initial 0x67 assignment, instead uses the retained publication base’s low eight bits.

Intervening computed accesses can affect the storage supplying 0x67. Before the first later adjusted-base publication, these include byte and word stores at adjusted-base offset 0x10, publications at publication-base offsets 0x54, 0x74, and 0x44, and a signed-shift-derived byte store through the doubled-link base.

Between the two later 0x67 reads, an explicit byte store publishes the adjusted base’s low eight bits at doubled-link-base offset 9; a word store publishes the saved 0xa9 result at adjusted-base offset 0x44; and a word is read at adjusted-base offset 0x54.

Each global read observes its own trace point. Neither declaration uncertainty nor repeated use permits unconditional reuse of an earlier observation.

### Remaining effects and final transfer

The final region includes:

- A byte store derived from a signed-word right shift whose count is the adjusted base’s low eight bits.
- A 32-bit store at adjusted-base offset 0x10 after a byte store at that address.
- Further byte and word publications through the word loaded at adjusted-base offset 0x54.
- An explicit 16-bit store of 0x0065 at twice a word loaded from doubled-link-base offset 4.
- A 32-bit store of decimal 100 at offset 0x54 through a retained loaded base.
- Further loaded-base publications and a final 32-bit store through twice another loaded word.

The unresolved final dependency, F-58dcf3fa9c29-D003, receives four represented argument positions: 0x72, a recently loaded word, the retained word from doubled-link-base offset 0x14, and a later word read at offset 0x74 through another loaded base.

A return is represented afterward. Whether the transfer returns, and whether any returned result is meaningful, remain **documented_only**.

## Side effects

Established direct effects comprise explicitly typed mixed-width pointer accesses, global observations and assignments of unresolved storage width, repeated publications, overlapping accesses, and mutations through loaded addresses.

Captured scalar values remain retained unless a reload is represented. Conversely, the three later reads of the global at 0x67 must not be collapsed into one observation.

Unknown global widths prevent a complete alias-footprint model. They do not establish disjointness from computed stores.

No transaction, synchronization, cleanup, rollback, idempotence, reentrancy, or thread-safety guarantee is shown. Earlier effects can remain after a fault or non-returning invocation.

Ordering here is representation-level ordering, not a guarantee about hardware bus transactions, volatile accesses, or concurrent visibility.

## Dependencies

| Opaque ID | Role | Resolution |
|---|---|---|
| F-58dcf3fa9c29-D001 | Entry-link-derived invocation | Destination and contract documented_only |
| F-58dcf3fa9c29-D002 | Ordered family of subsequent indirect invocations | Individual destinations and contracts documented_only |
| F-58dcf3fa9c29-D003 | Final unresolved indirect transfer | Destination, interface, effects, and return behavior documented_only |

D002 is a document-local family reference, not evidence of one shared dependency. Missing information includes actual calling conventions, argument counts, secondary-result definitions, error behavior, global effects, and memory ownership rules.

## Acceptance examples

These are conditional analytical examples, not executed tests.

### Composite separation

If D001 supplies low word 1 and high word 2, with ordinary 32-bit shift semantics, the next invocation’s four shown positions contain 0x100, 2, 0x8000, and 0x8000. One shifted 64-bit argument is not equivalent.

### Converted global is not an unsigned byte

If the missing declaration and current state make the global at 0x05 evaluate to 0x1234, its unsigned 32-bit conversion supplies base 0x1234. The chain-word read is then at 0x1288, assuming valid arithmetic and storage.

Restricting that base to 0x34 merely because the global’s address is 0x05 is unsupported.

### Source-value narrowing does not establish destination width

If the saved 0xa9 result is 0x123456ab, the value supplied to the initial 0x67 assignment is first narrowed to eight bits, yielding 0xab. This alone does not require a one-byte store or establish the values of neighboring storage.

### Reobserving mutable global storage

Suppose 0x67 is independently established to be an unsigned-byte object, the doubled-link base is 0x5e, and the adjusted base is 0x08. The intervening byte store at doubled-link-base offset 9 reaches 0x67 and writes 0x08.

Absent other changes, the following global read supplies 0x08 to the second adjusted-base offset-9 byte store. Without that declaration, the effect on the complete global value remains declaration- and byte-order-dependent.

### Widths, boundaries, and order

Under truncating 32-bit semantics, shifting 0x12345678 left by eight and logically right by eight yields 0x00345678. The small base can equal 1016, but not 1022.

An accepted trace preserves publication-base capture before the 0x60 global assignment, the 0xa9 global observation before the element-base offset-0x74 read, and the 0x54 global assignment before the 0x15 global assignment.

The explicitly typed 0x3a constant group requires eight-, sixteen-, and thirty-two-bit stores. This width requirement does not extend to the separately named global at 0x3a.

If an invocation never returns or an access faults, later effects need not occur. No rollback or returned error code is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid invocation context.
- Dependency destinations, calling conventions, secondary results, and ownership effects.
- Complete prefix argument schedule and repeated-publication count within this document.
- Undeclared globals’ storage widths, signedness, occupied ranges, and assignment conversions.
- Byte order, alignment, atomicity, memory-mapped semantics, concurrency, and faults.
- Signed overflow, character conversions, and variable-shift behavior.
- Accessible address ranges, object boundaries, lifetimes, and alias restrictions.
- Final transfer destination and return behavior.

Initialization or publication intent is **[INFERENCE]** only. It does not justify replacing unusual accesses with a conventional data structure.

## Evidence

The supplied static representation supports the explicit pointer-access widths, value conversions, retained-result relationships, invocation structure, and ordering boundaries described above.

It distinguishes an unsigned 32-bit conversion of the undeclared global at 0x05 from explicitly typed unsigned-byte reads elsewhere. Likewise, narrowing the value assigned to 0x67 establishes that source conversion, not the global’s storage width. The same distinction applies to the other undeclared fixed-address globals and the publication-base capture.

The representation establishes three reads of 0x67 after its initial assignment, publication-base capture before the 0x60 assignment, and the 0xa9 observation before the element-base offset-0x74 read and 0x3a assignment.

Comments, prior prose, and reviewer assertions are not independent behavioral evidence. No dependency inspection, machine-level verification, runtime execution, or tests have been performed. No independently verified provenance is asserted.
