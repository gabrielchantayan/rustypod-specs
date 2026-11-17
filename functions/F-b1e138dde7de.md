<!-- Trusted provenance: F-b1e138dde7de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1e138dde7de

## Purpose

F-b1e138dde7de performs an ordered sequence of indirect invocations, transforms dependency-derived values, reads and updates memory through computed addresses and fixed-location symbols, and finishes with a four-argument indirect invocation.

No application-level purpose is established. The supplied body does not establish text output, serialization, allocation, initialization, or formatting. Constants that correspond to printable characters do not establish such interpretations.

This contract distinguishes:

- **Established static evidence:** represented operations, operand widths, value dependencies, and ordering visible in the supplied body.
- **Conditional machine interpretation:** arithmetic consequences under explicitly stated assumptions, particularly 32-bit words and modulo-\(2^{32}\) arithmetic.
- **documented_only:** behavior unresolved by the supplied material or not exhaustively enumerated here.

All effects are conditional on execution reaching them. No successful execution, valid memory mapping, or normal completion has been demonstrated.

## Interface

### Explicit inputs

| Argument position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit reference in the body |
| Second | 32 bits | No explicit reference in the body |
| Third | Unsigned 32 bits | Supplies transformed arguments to early indirect invocations |

The first two arguments have no established semantic role. Their absence from explicit expressions does not prove that dependencies ignore the incoming machine state associated with them.

The third argument participates in two visible transformations:

- Left shift by 24 bits.
- Extraction of its low eight bits, followed by left shift by five bits.

Under 32-bit unsigned arithmetic, both transformed values depend only on the low byte. This does not establish whole-function equivalence for arguments sharing that byte.

### Implicit inputs

Behavior also depends on:

- Incoming control-flow context used to form the initial indirect target.
- Executability and invocation conventions of later indirect targets.
- Primary results, represented 64-bit results, and secondary machine-result values.
- Existing memory at computed addresses.
- Values associated with fixed-location symbols whose declarations are absent.
- A global-derived base loaded before the repeated update region.
- The unresolved final indirect target.

The explicit arguments alone do not determine execution.

### Outputs and completion

No explicit return value is represented. Outputs consist of dependency effects, direct memory updates, fixed-location assignments, and the final invocation.

The final invocation receives, in order:

1. The numeric value 0x72.
2. A saved 32-bit value read at offset 0x64 from a late-loaded base.
3. A saved 32-bit value read earlier at offset 0x14 from another base.
4. A saved 32-bit value read at offset 0x74 from the same base used for the second argument.

A return follows in the representation. Whether the final dependency returns, faults, blocks, or permanently transfers control is **documented_only**.

### Preconditions and ownership

Every reached invocation target must support the actual invocation convention. Every reached memory access must be permitted, including low numeric addresses and potentially unaligned destinations. No explicit validation establishes these conditions.

No allocation size, buffer capacity, object layout, ownership transfer, retention rule, or release protocol is established. A numeric value used as an address is not thereby an owned pointer.

Dependencies must not be assumed to preserve globals, maintain memory lifetimes, or refrain from allocating or releasing memory.

## Behavior

### Indirect invocation phase

Execution begins with a long, explicitly ordered sequence of indirect invocations. The initial target is formed from incoming control-flow context plus 0xFFE. Subsequent targets predominantly use represented continuation values or code-location values plus the same displacement.

Target formation is distinct from arithmetic performed on values passed to dependencies.

The invocations have differing represented argument counts, including no represented arguments. Some results are discarded. Discarding a result does not establish that an invocation is redundant: its side effects, failures, and termination behavior remain unresolved.

No explicit loop or conditional branch appears in the supplied body. Dependencies may nevertheless branch, block, fault, or fail to return.

Three result categories must remain distinct:

- A primary value represented as 32 bits.
- A represented 64-bit value whose low and high 32-bit halves are separately consumed.
- A secondary machine-result value whose complete production and invocation convention are unresolved.

A secondary result must not be substituted with zero, the primary result, or an assumed preserved register value.

### Width-sensitive transformations

Dependency-derived values undergo shifts, multiplication, masking, narrowing, and extraction. Masks select fields including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit.

The following distinctions affect observable arguments and addresses:

- Shifting a represented 64-bit value before narrowing differs from shifting a 32-bit value.
- Extracting the high half of a represented 64-bit result differs from consuming an unresolved secondary result.
- Unsigned right shifts introduce zeros.
- Masking before shifting can remove bits that would otherwise survive.
- Sequential 32-bit left shifts can discard information before subsequent operations.
- Explicit eight-bit and 16-bit destinations retain only the corresponding low bits.

Some address expressions and left shifts use represented signed intermediates. Exact executable behavior for overflow is not independently established. Modulo-\(2^{32}\) conclusions in this contract are conditional interpretations, not assertions of portable signed arithmetic.

### Opening memory relationships

The initial direct-memory region depends on several saved quantities:

- A dependency-derived word shifted left by 25.
- Another dependency-derived result multiplied by \(2^{25}\).
- A separate result shifted left by eight.
- A compact quantity obtained by shifting a result left by eight, masking with 0x7FFF, then shifting right by six.
- The low and high halves of the last represented 64-bit dependency result.

A 32-bit store at offset 0x14 from that result’s high half precedes an eight-bit store at offset 9 from the 25-bit-shifted base. A 32-bit store at offset 0x54 from the low half multiplied by 0x100 intervenes before the offset-9 destination is written again. The first byte value comes from the high half; the second comes from the saved 25-bit-shifted quantity.

The \(2^{25}\)-scaled base has its offset-0x14 word cleared before an explicit unsigned-byte read at offset 5. That byte is widened to unsigned 32 bits and stored at offset 0x34.

A separate word destination, offset 0x14 from twice the compact quantity, is cleared before the widened-byte store and cleared again afterward. Between those clears, the function reads back the offset-0x14 word from the \(2^{25}\)-scaled base.

The saved readback is subsequently used both as a byte value and as an address base. It is not unconditionally zero: intervening computed writes may overlap the cleared word.

### Compact addresses and byte truncation

Under 32-bit bit-vector arithmetic, the compact quantity is a multiple of four between 0 and 508. Twice that quantity is therefore a multiple of eight between 0 and 1016.

The represented accesses through this doubled quantity use low numeric addresses. No additional allocation base or object-pointer interpretation is supplied.

A 32-bit value shifted left by 25 has a zero low byte under the same interpretation. An explicit eight-bit store of that value therefore writes zero, even when the full word is nonzero.

### Fixed-location read associated with 0x05

After two consecutive byte stores at offset 0x15 from the doubled compact base, the function reads the value associated with fixed location 0x05 and converts it to unsigned 32 bits. This occurs before the fixed-location assignment associated with 0x01.

The converted value is saved and used as a base for:

- An eight-bit store at offset 0x15.
- A 32-bit read at offset 0x54.
- A subsequent 32-bit store at offset 0x54.

**The fixed-location source storage width and signedness are not supplied.** Conversion to unsigned 32 bits does not establish an eight-bit load, an eight-bit mask, or zero extension from a byte. Any source-side widening, sign extension, or truncation is **documented_only**.

This is distinct from the explicitly represented unsigned-byte reads at computed offset-5 addresses.

### Saved words, dependent addresses, and overwrites

The word read at offset 0x54 from the converted 0x05-associated base is saved before that destination is overwritten. The saved word subsequently supplies values and address bases scaled by two, four, and eight.

Computed accesses mix widths:

| Explicit access width | Representative offsets or forms |
|---|---|
| Eight bits | 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Destinations formed by scaling loaded values |
| 32 bits | 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

This is not a recovered structure layout. Shared offsets do not establish common types, separate objects, or minimum allocation extents.

Later address chains include 32-bit reads through previously loaded words and through explicitly read unsigned bytes. Those particular byte-derived bases are restricted to 0–255. That restriction does not apply to the converted fixed-location value associated with 0x05.

Repeated stores remain distinct represented events. These include consecutive duplicate byte stores, duplicate self-referential word stores, and successive writes of different values to the same word destination.

One assignment reads a computed offset-0x74 word both to obtain the stored value and to form its destination base. The representation contains repeated reads within that expression; it does not independently establish their machine-level evaluation order.

### Assignment associated with fixed location 0x54

The fixed-location symbol associated with 0x54 receives the represented expression formed by shifting the saved offset-0x54 word left by one.

This assignment occurs after an offset-0x14 computed word destination receives another shifted value and before that computed destination receives the saved word itself.

The right-hand expression uses a represented 32-bit signed value. Its arithmetic overflow semantics remain unresolved. More importantly, **the declaration of the fixed-location destination is absent**. Neither the right-hand expression’s width nor the numeric location establishes a 32-bit memory store there.

The assigned expression and its ordering are established; the destination storage width, assignment conversion, and actual memory access width are **documented_only**.

### Constant and repeated-update regions

A later region stores 0x3A through three explicitly sized computed destinations:

- Eight bits at a four-times-scaled address.
- 16 bits at an eight-times-scaled address.
- 32 bits at offset 0x20 from the four-times-scaled base.

A separate offset-0x20 word destination receives four ordered writes: 0x20, 0x2F, an unsigned byte read from offset 4 of its base and widened to 32 bits, then 0x20 again. The byte read follows the first two writes.

Next, a global-derived base is read into a represented 32-bit signed value. The fixed-location symbol associated with 0x60 then receives the integer constant 0x20, before the repeated computed updates begin.

**The assignment at 0x60 has no independently established destination width.** The integer constant does not establish a 32-bit store. The global-derived base’s represented local width likewise does not establish the source global’s storage or load width.

Each repeated update group preserves this order:

1. A 32-bit store at offset 0x54 of a scaled base.
2. An eight-bit store at offset 0x19 of the saved global-derived base.
3. An eight-bit store at offset 0x11 of another saved base.
4. A 32-bit store at offset 0x34 of the global-derived base.

The initial group uses a different scaled base and offset-0x34 value from subsequent groups. No dependency invocation or global-base reload is represented within this region.

The exact repetition count and exhaustive event trace are **documented_only in this contract**. This limitation does not permit eliminating repeated writes.

### Late fixed-location read associated with 0xA9

After the repeated region, the value associated with fixed location 0xA9 is saved in a represented 32-bit temporary. A computed 32-bit read at offset 0x74 follows.

**The temporary’s width does not establish the fixed-location source storage width or memory-read width.** The global declaration is absent; source signedness and assignment conversion are unresolved.

The saved 32-bit value later supplies an explicitly eight-bit-narrowed value for the fixed-location assignment associated with 0x67, and an explicitly 32-bit computed store at offset 0x44. These downstream uses do not retroactively establish a 32-bit global read at 0xA9.

### Late processing and final invocation

The two reads just described precede the fixed-location assignment associated with 0x3A. Its represented value is formed from an eight-bit character conversion of a saved quantity, multiplication by eight, and addition of 0x30. Character signedness, destination conversion, and the global’s storage width remain unresolved.

The function then saves the offset-0x14 word later used as the final invocation’s third argument, followed by a word read at offset 4 of another saved base.

Subsequent effects include:

- An explicit eight-bit store of 0x6C.
- Eight-bit computed stores and fixed-location assignments involving narrowed values.
- A 32-bit read at offset 0x44, followed by stores using that saved value.
- A signed right shift whose count is a saved base’s low byte.
- A 32-bit read at offset 0x40 before stores at offsets 0x74 and 0x10.
- Further 32-bit stores overlapping earlier byte destinations.
- A 16-bit store of 0x65 through twice a newly loaded base.
- A 32-bit store of decimal 100 at offset 0x54.
- Final 32-bit stores through newly loaded and doubled bases.

The variable shift count ranges from 0 to 255. Behavior for negative signed operands and counts of at least 32 is not independently established.

The symbol associated with 0x67 is read again for later copies. Its value must not be replaced unconditionally with the earlier saved value: intervening computed accesses may alias its storage.

The final second argument is saved from offset 0x64 before the fourth argument is saved from offset 0x74. Intervening memory effects do not replace these saved values. The third argument remains the earlier saved offset-0x14 word.

## Side effects

Direct effects include explicitly sized computed writes and assignments to fixed-location symbols associated with 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67.

For undeclared fixed-location symbols, expression width and destination storage width must remain separate:

- At 0x54, the assigned expression is the saved word shifted left by one; actual destination width is unresolved.
- At 0x60, the assigned value is the integer constant 0x20; actual destination width is unresolved.
- At 0xA9, the read value is saved in a represented 32-bit temporary; actual source storage and read widths are unresolved.
- At 0x05, the value is converted to unsigned 32 bits; actual source storage and read widths are unresolved.
- Explicit eight-bit narrowing before other global assignments establishes the narrowed value, not the globals’ complete declarations.

Computed addresses may overlap fixed-location storage or one another. No non-aliasing guarantee is established. Comments about overlapping symbols do not independently establish storage extents.

Endianness, alignment, volatility, concurrency, and write-sensitive behavior are unresolved. Represented ordering does not establish universal byte-level outcomes for overlapping wider stores.

No explicit validation-failure path, error result, rollback, or cleanup handler is represented. Invalid targets, inaccessible addresses, forbidden writes, or arithmetic restrictions may prevent completion. Earlier effects may remain. Atomicity, thread safety, and reentrancy are not established.

## Dependencies

Only F-b1e138dde7de has an authoritative supplied opaque identity. The following IDs are document-local references to unresolved dependency boundaries.

| Opaque ID | Boundary | Established information |
|---|---|---|
| F-unknown-01 | Initial indirect invocation | Target depends on incoming control-flow context |
| F-unknown-02 | Intermediate invocation family | Ordered invocations consume constants, transformed results, and secondary machine values |
| F-unknown-03 | Final indirect invocation | Receives the four arguments described above |

F-unknown-02 denotes multiple invocation sites, not one proven target or shared implementation.

No dependency contract establishes purity, global preservation, ownership, allocation, release, retention, memory lifetime, or normal return.

## Acceptance examples

These are conditional static requirements, not executed tests.

### Explicit argument transformations

For third arguments 0x000000AB and 0x123456AB, unsigned 32-bit left shift by 24 produces 0xAB000000 in both cases. Low-byte extraction followed by left shift by five produces 0x1560.

This establishes equality of those expressions only.

### Compact-address boundary

For the relevant dependency result 0x7F, the compact transformation produces 508. Doubling and adding 0x14 produces numeric address 0x40C.

Introducing an additional object base would change the represented address relationship.

### Fixed-location conversion without byte narrowing

If the value associated with 0x05 converts to unsigned 32-bit value 0x00001234, the saved base is 0x1234, not 0x34. Its offset-0x54 address is 0x1288, absent address wraparound.

Whether that premise is realizable depends on the missing declaration.

### Fixed-location writes without invented widths

If the saved word used by the 0x54-associated assignment is 0x1234, its left-shifted expression is 0x2468 under the stated arithmetic interpretation. That assignment occurs between the surrounding computed word writes described above.

Acceptance does not require four destination bytes to be written at 0x54. Likewise, the assignment of 0x20 at 0x60 must retain its position before the repeated region, but its actual store width cannot be selected from the constant alone.

### Saved value from 0xA9

If the fixed-location read and conversion yield saved 32-bit value 0x11223344, explicit eight-bit narrowing contributes 0x44 to the assignment associated with 0x67. The later computed 32-bit offset-0x44 store uses the full saved value.

Neither consequence establishes a four-byte source read at 0xA9 or a one-byte global store at 0x67.

### Truncation and ordered overwrites

An explicit eight-bit store of 0x06000000 contributes 0x00.

For write-sensitive memory, the offset-0x20 sequence contains four writes: 0x20, 0x2F, the widened byte, and 0x20. Keeping only the last write does not preserve that sequence.

### Readback and saved final argument

If a cleared offset-0x14 word is ordinary memory and no intervening access modifies it, its readback is zero. Later use of that saved zero as a base can target numeric location 0x54; no null check is represented.

If the earlier final-argument read yields 0x11223344, later changes to that location do not change the final invocation’s third argument.

## Uncertainties

The following remain **documented_only**:

- Application-level intent and runtime reachability.
- Complete target identities and invocation conventions.
- Exhaustive invocation and memory-event traces, including the repetition count.
- Source declarations, signedness, extents, and access widths of undeclared globals.
- Conversion rules at fixed-location reads and assignments, particularly 0x05, 0x54, 0x60, and 0xA9.
- Executable signed-overflow, signed-shift, and oversized-shift behavior.
- Endianness, alignment, address mappings, volatility, and concurrent modification.
- Ownership, allocation, release, retention, and lifetime requirements.
- Dependency effects on globals and previously obtained addresses.
- Normal return and any unconditional successful-completion postcondition.

## Evidence

The supplied static body supports the represented argument widths, indirect invocation structure, arithmetic relationships, explicitly sized computed accesses, ordering relationships, saved-value dependencies, and final four-argument invocation.

Fixed-location declarations are not supplied. A represented 32-bit expression, a 32-bit receiving temporary, an integer constant, or an eight-bit narrowing does not independently establish the associated global’s physical access width.

Prior prose, reviewer feedback, and embedded comments are not independent behavioral authority.

No tools, external implementation inspection, runtime execution, smoke runs, or tests were performed. No successful execution or complete final memory state is claimed.
