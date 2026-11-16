<!-- Trusted provenance: F-fafda0e934ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fafda0e934ec

## Purpose

F-fafda0e934ec has no declared arguments or result. It performs an extended sequence of indirect dependency invocations, transforms ordinary and secondary call-associated results, mutates absolute and computed storage, and invokes a final unresolved dependency with four arguments.

The established behavior is call orchestration and memory mutation. No domain-level purpose, record schema, or ownership protocol is established. Recurring offsets and self-reference stores do not establish that the addressed storage contains valid objects.

This contract distinguishes:

- **Established static evidence:** relationships directly supported by the supplied function representation.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **documented_only:** unresolved behavior or detail not exhaustively specified here.

Ordering below is ordering in the supplied representation. It does not independently establish machine-level access granularity or observability. No runtime tests have been performed.

## Interface

### Entry and completion

The declared interface has no explicit parameters and no result value. Implicit inputs include incoming return-control state, dependency outputs, secondary call-associated values, absolute storage, and memory reached through derived addresses.

The first dependency destination is incoming return-control state plus 4,094 bytes. Intermediate destinations use embedded location-like values plus that same displacement. Their identities and calling conventions are unresolved.

The final dependency receives four represented 32-bit arguments. A return follows that invocation in the representation. Whether the dependency returns normally is **documented_only**.

### Widths, storage, and conversions

| Quantity or access | Established represented width |
|---|---:|
| Explicit computed byte access | 8 bits |
| Explicit computed halfword store | 16 bits |
| Explicit computed word access | 32 bits |
| Retained ordinary scalar or address-like value | 32 bits |
| Paired call result | 64 bits |
| Explicit unsigned conversion of the value obtained at absolute address 5 | 32 bits |
| Original accesses through undeclared named globals | **documented_only** |

Offsets are byte offsets. Explicit unsigned-byte reads produce values from 0 through 255. Explicit one-byte destinations constrain the destination footprint, unlike a narrowed expression assigned to a global whose declaration is absent.

**The original types and access widths of the named globals at absolute addresses 1, 5, 21, 33, 58, 84, 96, 103, and 169 are not supplied.** The same limitation applies to the separately stored base at 0x087fbbbc.

Consequently:

- An explicitly narrowed assignment expression does not establish a one-byte global write.
- Assignment of a global value into a 32-bit retained scalar does not establish a four-byte global read.
- A global-to-global assignment does not establish either access width or the intervening conversion.
- Source types, assignment conversions, and complete alias footprints remain **documented_only** unless independently constrained by an explicit conversion.

The address-5 read is followed by an explicit unsigned 32-bit conversion. That establishes the resulting address-base width, not the source access width or an eight-bit range.

For paired results, “low” and “high” mean the low and high 32-bit portions. Unsigned 64-bit right shifts by 34, 36, and 44 produce the high portion shifted right by two, four, and twelve. Narrowing before those wide shifts is not equivalent.

Unsigned arithmetic has finite-width behavior. Signed overflow, problematic signed shifts, out-of-range signed conversions, and pointer validity remain **documented_only**. Mathematical scaling descriptions below do not grant unbounded arithmetic or portable wrapping semantics.

Byte order, alignment, address mapping, and external modification are unresolved. Absolute and computed storage may alias. Low addresses are represented storage locations, not established null sentinels.

### Preconditions and errors

Progress requires every reached dependency target and memory access to be supported by the execution environment. The function does not establish or validate those prerequisites.

No bounds check, normal error result, rollback, or recovery path is represented. Faults, dependency exceptions, and nontermination remain **documented_only**.

## Behavior

### Dependency orchestration

The function starts with a long straight-line sequence of indirect invocations and scalar transformations. No explicit data-dependent branch or loop appears.

Calls without displayed arguments remain ordered dependency events. An unused ordinary result does not establish purity or absence of secondary outputs. Calls must not be discarded or reordered on that basis.

Established argument relationships include:

- A secondary call-associated value shifted left by five.
- Paired low portions shifted left by eight.
- High portions forwarded unchanged or shifted separately.
- The same transformed value repeated in multiple argument positions.
- Explicit zeros alongside unknown outputs.
- Retained values reused after intervening calls.
- Masks and extraction operations applied before further shifts.

An early two-argument consumer receives the low portion shifted left by eight, then the high portion shifted left by five. Another receives the shifted low portion, unchanged high portion, high portion shifted left by five, and the shifted low portion again.

Later consumers include three-argument forms receiving the low portion shifted left by eight, unchanged high portion, and high portion shifted left by six. Other forms use different shifts, secondary channels, retained results, or explicit zeros; those forms are not interchangeable.

Masks select low fields of widths including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits, or a single bit. Operation order matters. A retained 32-bit intermediate cannot be replaced by an unbounded expression when discarded bits could affect later operations.

Under a 32-bit bit-vector interpretation, two successive left shifts by 28 discard all original bits. That observation does not resolve the portable behavior of the signed expressions representing the chain. Explicitly initialized zeros remain zero through their represented shifts; unknown secondary outputs must not replace them.

The complete intermediate call-site signature ledger is **documented_only** in this contract. The relationships above do not authorize combining dependency events.

### Initial direct mutation

The final paired result supplies a high-portion base and a low-portion-derived base scaled by 256. Earlier retained values supply another address-like value, a heavily scaled base, a doubled small index, and payloads.

The first direct write is a 32-bit store at the high-portion base plus 20. An explicit byte store at the retained address-like value plus nine follows, using a character conversion of the high portion. A word store at the scaled low-portion base plus 84 receives the heavily scaled value. The offset-nine byte is then overwritten using another retained value.

The heavily scaled base’s offset-20 word is cleared before an unsigned byte is read at its offset five. The doubled-index base’s offset-20 word is then cleared. The saved byte is widened into a word at the heavily scaled base plus 52, after which that base’s offset-20 word is reread.

That reread need not be zero: intervening writes may alias the cleared word. Another doubled-index offset-20 clear and two identical explicit offset-21 byte stores precede the first explicit global read.

### Absolute-storage events

The table describes expressions and ordering, **not inferred global access footprints**.

| Absolute location | Represented value relationship | Ordering anchor |
|---|---|---|
| 5 | Read a global value and explicitly convert it to unsigned 32 bits | After the duplicate doubled-index offset-21 stores |
| 1 | Assign an explicitly one-byte-converted doubled-index value | Immediately after the address-5 conversion |
| 84 | Assign the saved computed offset-84 value shifted left by one | Between a computed offset-20 store and two further offset-20 stores |
| 21 | Assign an explicitly one-byte-converted doubled-index value | After those word stores and two additional computed offset-21 byte stores |
| 0x087fbbbc | Read a global value into a retained 32-bit scalar | After the adjusted working region’s final offset-32 store of 32 |
| 96 | Assign the integer value 32 | After the stored-base snapshot |
| 169 | Read a global value into a retained 32-bit scalar | After the repeated cross-reference region |
| 58 | Assign arithmetic performed after converting the adjusted anchor to a character value | After a computed offset-116 snapshot |
| 103 | Assign an explicitly one-byte-converted version of the retained address-169 value | After the late linked-base offset-84 self-reference store |
| 33 | Assign a newly read address-103 global value | After the linked-base offset-68 word read |

For every row, the original global declaration and access width are **documented_only**. Destination conversion is unresolved where the destination declaration is missing. Known expression widths or retained widths do not repair that omission.

### Address-5-derived and linked updates

The **converted address-5 base** is the unsigned 32-bit conversion of the value read from address 5. It is not necessarily that value’s low byte.

After the address-1 assignment, this base supplies an explicit offset-21 byte destination. Subsequent computed writes precede a word read at the converted base plus 84. That word is retained before the same word destination is overwritten.

Intervening updates include:

- An offset-17 byte assignment later overwritten with zero.
- An offset-116 word read through the heavily scaled base.
- Writes through the resulting address.
- A self-reference update using offset-116 word values read through the doubled-index base.
- Repeated offset-36 self-reference stores and offset-nine byte stores before the absolute address-84 assignment.

Separate occurrences of a computed read remain separate evaluations. The representation does not establish a finer ordering between multiple operand reads within the same assignment.

After the address-21 assignment, linked processing uses offsets 4, 5, 9, 13, 20, 21, 52, 68, 84, 100, and 116. An explicitly typed unsigned-byte read at a computed offset five supplies the base for an offset-116 word read. This genuinely byte-based traversal is distinct from the unresolved-width global read at absolute address 5.

Recurring offset-five reads are separate snapshots. Intervening writes can change their results.

### Constants and repeated cross-references

A linked word supplies the scale for three explicit stores of 58: a byte at four times that word, a halfword at eight times it, and a word at four times it plus 32. The byte store precedes the halfword store; the word store follows.

An earlier retained fourfold value is reduced by 150 to form the **adjusted anchor**. Four times that anchor is the **working base**.

The working base’s offset-32 word successively receives:

- 32;
- 47;
- the widened unsigned byte read at working base plus four;
- 32 again.

The offset-four byte read occurs after the first two writes.

The separately stored base is then read and retained as a 32-bit scalar, called the **stored base**. The global assignment at address 96 follows.

Repeated cross-reference groups contain these explicit computed accesses:

| Destination | Payload | Access width |
|---|---|---:|
| Eightfold destination plus 84 | Retained fourfold value derived from the linked word used for the stores of 58 | 32 bits |
| Stored base plus 25 | One-byte-converted working base | 8 bits |
| Working base plus 17 | One-byte-converted stored base | 8 bits |
| Stored base plus 52 | Eightfold destination | 32 bits |

Within each group, the offset-84 word store precedes the two byte stores, which precede the offset-52 word store. The first group uses an earlier unadjusted eightfold destination; later groups use eight times the adjusted anchor.

The stored base is retained rather than reloaded for each group. Its source global need not remain unchanged. Literal repetition is represented; its complete multiplicity is **documented_only** here. Repetition is not permission to collapse observable accesses.

### Late snapshots and address-58 arithmetic

For the late sequence, **linked base** means the retained doubled linked value. These names describe value provenance, not object types or ownership.

After the repeated region:

1. The global at address 169 is read into a retained 32-bit value.
2. A word is saved from offset 116 of the repeated region’s retained fourfold payload base.
3. The global at address 58 is assigned.
4. Words are saved from linked base plus 20 and working base plus four.

The address-58 expression converts the **adjusted anchor itself** to an eight-bit character value before multiplication by eight and addition of 48. The character value undergoes integer promotion for the arithmetic. There is **no explicit final eight-bit narrowing of the arithmetic result**.

Character signedness and out-of-range character conversion are unresolved. Under signed eight-bit character interpretation, the promoted arithmetic can range from −976 through 1,064; under unsigned interpretation, from 48 through 2,088. The missing destination declaration determines the final assignment conversion and write width.

### Late updates and three address-103 reads

Computed byte writes affect linked base plus 13 and working base plus eight, thirteen, and nine. Working base plus eight receives 108. The latter two working-base destinations receive the retained one-byte conversion of the stored base. Linked base plus 84 receives the linked base as a word.

The address-103 global assignment then uses a one-byte conversion of the retained address-169 snapshot. This establishes the assignment expression, not a byte-wide global destination or the value a later global read must produce.

A word is next read at linked base plus 68. **The first separate address-103 read** supplies the global assignment at address 33. Both global types and the assignment conversion remain unresolved.

The offset-68 word supplies an explicit byte store at working base plus 16 and a word store at stored base plus 84. It also supplies a character-converted byte store at linked base plus one after a right shift controlled by the working base’s low eight bits. Signed right-shift details and invalid shift counts are **documented_only**. Stored base plus 84 is then overwritten with the working base.

A word is read at working base plus 64. The saved working-base offset-four word is stored at stored base plus 116. The new offset-64 word is stored at working base plus 16, overlapping the earlier explicit byte destination. Stored base plus 68 receives the linked base.

**The second address-103 read** supplies an explicit byte store at working base plus nine; the destination imposes its one-byte conversion. A linked-base offset-nine byte store follows. The retained 32-bit address-169 snapshot is stored at working base plus 68.

A word is then read at working base plus 84. **The third address-103 read** supplies another explicit byte store at working base plus nine.

Exactly three separate address-103 reads are represented. Their source widths are not established. The second and third feed one-byte destinations, but that does not make their global source reads byte-wide. Intervening computed writes may affect subsequent global values through aliases whose footprints are unresolved.

### Final updates and invocation

The word read from working base plus 84 becomes a further destination base. Its offset 13 receives a one-byte conversion of the saved computed offset-116 word.

Linked base plus 84 is restored from the saved linked-base offset-20 word. A word is saved from working base plus 20. Linked base plus 17 receives the same converted offset-116 payload. Working base plus nine receives a character conversion of the further destination base, and working base plus 68 receives that base as a word.

The final traversal reads a new base from linked base plus four. Twice the new base receives an explicit halfword store of 101. A word is saved from new base plus 100. The further destination’s offset 17 receives a character conversion of the saved working-base offset-20 word; its offset 84 receives the word value 100.

A word is saved from new base plus 116, followed by a destination read from linked base plus 100. A linked-base offset-nine byte update precedes a word store of the saved offset-100 value at the latest destination plus 84. The latest destination is also stored as a word through twice the saved offset-100 value.

F-U003 receives, in order:

1. 114.
2. The saved word from new base plus 100.
3. The earlier saved word from linked base plus 20.
4. The saved word from new base plus 116.

These are retained snapshots, not final rereads.

## Side effects

Established direct effects include explicitly typed mixed-width computed writes, overlapping destinations, self-reference stores, repeated writes, and assignments through undeclared globals.

Global footprints must remain unresolved throughout alias analysis. In particular, assignments at addresses 1, 21, 33, 58, 84, 96, and 103 cannot be modeled as byte or word stores solely from their expressions. Reads at 103 and 169 cannot be assigned footprints from their consumers.

Earlier writes may affect later reads where actual storage overlaps. Retained snapshots remain unchanged by subsequent source-memory changes, although later reads may observe different values.

No atomicity, synchronization, reentrancy, idempotence, rollback, or object-lifetime guarantee is established. No explicit allocation or release protocol is identified. Dependencies may allocate, release, or mutate storage; address use does not establish ownership transfer or continued validity.

## Dependencies

| Opaque ID | Role |
|---|---|
| F-U001 | Initial indirect dependency derived from incoming control state |
| F-U002 | Family of intermediate indirect dependencies |
| F-U003 | Final unresolved dependency receiving four arguments beginning with 114 |

F-U002 does not assert that intermediate targets identify one function.

Identities, full interfaces, secondary-result channels, global preservation, ownership effects, exceptions, and termination are **documented_only**. No argument-free invocation establishes purity or a complete machine interface.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Address-5 conversion

If the global read produces unsigned value 0x1234, its explicit unsigned 32-bit conversion is 4,660, not 52. Computed offsets 21 and 84 use that converted base. This does not establish the original read width.

### Global expression versus footprint

If a one-byte-converted expression supplied to address 1 equals 0x34, the assignment supplies that expression to the global’s destination conversion. A one-byte write and preservation of adjacent storage are not acceptance requirements without its declaration.

Likewise, retaining the address-169 value in 32 bits does not establish a four-byte source read.

### Address-58 promotion

If the adjusted anchor is 32, its character conversion preserves 32 under either signed or unsigned eight-bit interpretation. The promoted arithmetic result is 304.

If the missing destination is an unsigned 16-bit object, the assignment can retain 304. If it is unsigned eight-bit storage, assignment conversion yields 48. The excerpt does not select either destination type. Requiring unconditional final narrowing to 48 is unsupported.

### Paired transformations

For a consumer receiving low portion shifted left by eight, unchanged high portion, and high portion shifted left by six, low portion 1 and high portion 2 yield 256, 2, and 128.

Keeping the low 24 bits of 0xFF123456 and shifting left by six yields 0x048D1580.

A paired value with high portion 0x100 yields zero after an unsigned 64-bit right shift by 44, but 16 after a shift by 36.

### Rereads and snapshots

If the address-169 snapshot is 0x12345612, the explicit one-byte conversion supplied to the address-103 assignment is 0x12. Its later global-read value still depends on the missing declaration, assignment conversion, and intervening effects.

If the second address-103 read produces 0x134 and conversion at its explicit one-byte destination retains the low eight bits, working base plus nine receives 0x34. No byte-wide source read is implied.

If aliases modify a cleared computed word before its reread, the reread need not return zero. If execution reaches F-U003, its arguments use the specified snapshots even if their source locations have changed.

## Uncertainties

The following remain **documented_only**:

- Original global declarations, read/write widths, implicit conversions, and alias footprints.
- Character signedness and out-of-range character conversions.
- Complete intermediate call ledger and repeated-region multiplicity not enumerated here.
- Dependency conventions, secondary-channel lifetimes, and memory-preservation behavior.
- Whether paired outputs represent one result or multiple machine channels.
- Signed overflow, problematic shifts, pointer validity, and address arithmetic fidelity.
- Byte order, alignment, external modification, and actual aliases.
- Machine-level access observability, ownership, allocation lifetime, and final-target return behavior.

Cross-reference construction is an inferred possible intent, not a required interpretation.

## Evidence

The supplied function representation supports the described dependency relationships, explicit computed-access widths, ordered global evaluations and assignments, retained snapshots, three address-103 reads, and final four-argument invocation.

It supplies no declarations establishing the named globals’ original access widths. Explicit narrowing on assignment and 32-bit retention on reading therefore do not establish global footprints.

The address-58 expression places character conversion before promoted multiplication and addition, with no explicit final one-byte conversion. Final destination conversion remains unresolved.

Comments, prior prose, and reviewer feedback do not independently establish behavior. No tools, runtime execution, tests, dependency inspection, or instruction-level validation were performed.
