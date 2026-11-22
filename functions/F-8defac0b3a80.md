<!-- Trusted provenance: F-8defac0b3a80; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8defac0b3a80

## Purpose

F-8defac0b3a80 performs a long, call-interleaved sequence of integer transformations and memory accesses. Established effects include 8-bit, 16-bit, and 32-bit stores; fixed-address reads and writes; and interactions with six mapped dependencies and numerous unresolved dependencies.

No domain-level purpose is established. Scaled addressing and field extraction suggest packed-value or encoded-reference processing, but that interpretation is not an acceptance requirement.

This contract describes the supplied static representation, conditional on execution reaching each operation. The representation reports invalid instruction data, an omitted unreachable region, overlapping global symbols, and terminal control-flow truncation. It therefore does not establish a complete, normally returning executable routine.

Observable access order matters. Captured values, fresh reads, overlapping stores, and repeated equal-value writes are distinct parts of the represented behavior. Where multiple reads appear as arguments to one call, their association with argument positions is established, but their evaluation order is not.

## Interface

### Inputs

The displayed interface has no explicit parameters. It nevertheless depends on implicit execution state:

- An incoming return-link value that contributes to the initial indirect-call target.
- An incoming 32-bit register value used in the first word store and later selector-address calculations.
- Ordinary results and additional register outputs from calls.
- Mutable memory at fixed, computed, and call-derived addresses.
- An execution environment capable of performing the represented indirect calls and memory accesses.

The complete entry-state and calling-convention requirements are **documented_only**. A conventional parameterless invocation alone is not a sufficient calling contract.

### Widths and arithmetic

| Item | Established width or treatment |
|---|---|
| Byte access | 8 bits; selector bytes are interpreted as unsigned |
| Halfword store | 16 bits; retains the source’s low sixteen bits |
| Word access | 32 bits |
| Composite result | 64 bits, used as separate low and high 32-bit portions |
| Unsigned right shift | Logical, introducing zero bits |
| Byte store from a wider value | Retains only the low eight bits |

A right shift followed by a byte store does not retain every remaining bit. For example, a word shifted right by 22 contributes original bits 22–29 to the stored byte.

Address calculations support a 32-bit machine interpretation. Under that interpretation, scaling, left shifts, addition, and subtraction retain the low 32 bits. Some intermediates are presented as signed, so wraparound conclusions involving those intermediates are conditional machine-width consequences, not portable signed-arithmetic guarantees.

Byte order, alignment requirements, and fault behavior are **documented_only**.

### Outputs and completion

No ordinary return value, success indicator, error code, or caller-owned output object is established.

Outputs consist of memory effects and dependency interactions. The visible sequence ends at an invalid-instruction termination marker, not a normal return. Whether that marker corresponds to a fault, trap, invalid execution, or incomplete static recovery is unknown.

## Behavior

### Execution conditions and ordering

Each described operation requires all preceding necessary accesses to succeed and preceding calls to return. No explicit validation, bounds check, retry, rollback, or recoverable error branch is visible.

Computed addresses must not be assumed to identify conventional heap objects. Several operations explicitly reach low addresses, including 0x00, 0x01, 0x28, and 0x7C.

The contract distinguishes:

- A scalar captured before a call from a later fresh read of the same address.
- Two reads supplying separate arguments from one read duplicated into two arguments.
- Two equal-value stores from a single store.
- Fourfold-scaled and sixteen-fold-scaled destinations.
- Established sequencing between statements from unspecified evaluation order within a call’s arguments.

Unknown dependencies are not assumed to preserve globals, pointed-to storage, or any other memory.

### Initial word snapshots and replacement result

After the initial link-derived call, the incoming 32-bit register value is stored at four times an additional register output plus 0x14.

After further calls, another word store targets four times a call result plus 0x14 and stores an additional register output.

A subsequent result drives two different accesses: a word store at sixteen times that result plus 0x14, storing four times the result, followed by a word load at four times the result plus 0x14. Call this loaded value the **initial word snapshot**.

The initial word snapshot remains captured across intervening calls and supplies:

- A later word-store value obtained by logical right shift by 11.
- The address of a second word load: the snapshot logically shifted right by 22, plus 0x14.

The **second word snapshot**, not the initial snapshot, supplies a later call argument shifted left by 25.

Before the subsequent selected-byte stores, a new indirect-call result replaces the initial snapshot in the represented dataflow. That **replacement result** supplies both:

- The low byte of its fourfold-scaled value for the first selected-byte store.
- A later byte formed by masking to 29 bits, shifting right by 19, and truncating to eight bits.

Both stores obtain their selectors from the incoming register value shifted right by 11, plus one. The selector is read again for the second store after intervening calls. Neither byte-store value is derived from the initial word snapshot.

### Selected-byte access convention

A recurring operation reads an unsigned byte at a computed selector address and writes a byte at that selector’s numeric value plus 0x11.

The selected destination therefore ranges from 0x11 through 0x110 inclusive. No separate object base is shown.

Selector addresses include shifted values plus one; scaled values plus one; masked-and-shifted values plus one; and fixed address 0x01. Scaling includes factors of 4, 8, 16, 64, 1024, and larger powers of two.

Byte-store sources include zero, fourfold-scaled results, low bytes of captured words, and extracted fields. Relevant extracted byte ranges include original bits 22–29, 20–27, 19–26, and 12–19, depending on the preceding mask and shift.

Several phases write zero through a selector, make calls, then read the selector address again for another write. Neither selector equality nor destination equality is guaranteed across those calls. The zero write remains a separate effect.

### Early dependency and zero-derived phases

Before F-23325cd5e50c, an explicitly zero scalar produces a selector read at 0x01 and a zero selected-byte write. Calls intervene before another read at 0x01 supplies a later write using bits 22–29 of a retained result.

F-23325cd5e50c receives one result that is also retained for subsequent arithmetic. Following this dependency, selected-byte operations use sixteenfold scaling, a 30-bit mask, and subtraction of that retained value from a byte selector. A call receives the subtraction result twice. Under 32-bit arithmetic, underflow wraps; no rejection is shown.

Further zero assignments cause later scaled selector addresses to resolve to 0x01. These are fresh memory reads, not a permanently cached selector.

A later call result is written as a word to fixed address 0x7C before a subsequent indirect call receives that captured result twice.

Later still, a word is captured from an additional call-output-derived address plus 0x14. That captured word survives multiple calls and contributes shifted values to later stores and arguments.

### Three distinct reads at 0x28

The fixed-address 0x28 phase contains three word reads, not one:

1. After an indirect call returns, one word is captured from 0x28.
2. A following indirect call obtains its first argument from a separate word read at 0x28.
3. That same call obtains its second argument from another separate word read at 0x28.

The capture precedes both argument reads. The relative evaluation order of the two argument reads is not established.

After the argument-taking call returns, a further indirect call occurs. Only then is the original captured word stored at four times an additional register output plus 0x14.

The stored value is not either argument read and is not a fresh post-call read. The three observed values need not be equal. No assumption about dependency preservation or memory stability may merge them.

### Composite results and partial layouts

A middle composite result supplies independent low and high 32-bit portions. From the high-half-derived base, the routine reads a word at offset 8 and then a word at offset 0x0C. Both reads precede the next call.

After that call and a further call, the word captured at offset 0x0C supplies a selector address by addition of one. The low byte of the word captured at offset 8 supplies the selected-byte value.

A later word read at offset 4 from the retained low-half-derived base supplies another byte write after intervening calls.

Elsewhere, a composite result’s low half is increased by 0x0C and passed repeatedly as a call argument. Much later, after further calls, a word is read at offset 4 from that retained adjusted base. Under 32-bit address arithmetic, this corresponds to the original low half plus 0x10.

These accesses establish individual offsets, not a complete object layout, allocation size, validity guarantee, or ownership model.

### Dependency progression and repeated byte arguments

The six mapped dependencies occur in this order:

1. F-23325cd5e50c.
2. F-bbb54b5f036d.
3. F-f80b3ac065fc.
4. F-de15789688a8.
5. F-ea83f9a17404.
6. F-674e75a953e8.

Numerous unresolved calls and memory accesses separate them.

Between F-bbb54b5f036d and F-f80b3ac065fc, an unresolved dependency receives a composite result’s low half, high half, low half, and low half. Between F-de15789688a8 and F-ea83f9a17404, another unresolved dependency receives one captured value twice.

After F-f80b3ac065fc, a later composite result causes a byte store at its high half plus 0x1C, using the low byte of its low half. Following additional calls and selected-byte operations, an indirect call separately reads address 0x01 for each of its two arguments.

Those two byte reads are distinct; neither argument is specified as a duplicate of one captured byte. Their relative evaluation order is unknown. After that call, another indirect call precedes the next selected-byte zero write, whose selector address is derived from a retained composite-result low half multiplied by \(2^{25}\), plus one.

### Low-address mixed-width stores

A scalar explicitly set to zero supplies an eightfold-scaled word-store destination. This produces a 32-bit store at 0x00.

After calls and selected-byte operations, the same zero scalar, masked and shifted, supplies a halfword destination. A later call result is stored there as a 16-bit value, also at 0x00.

These overlapping stores occur in that order. Calls and byte-selector reads intervene. Since address 0x01 lies inside the word and halfword storage at 0x00, the stores can change later selector values even without callee mutation.

A subsequent composite result contributes a byte stored at a different result-derived destination plus four times the zero scalar. This is not generally another store at 0x00.

### Repeated stores and late word propagation

A later word destination, formed from a retained value shifted right by 9 plus 0x34, receives two consecutive stores: first a zero-extended byte selector, then twice that selector.

Another destination, formed from an additional register output plus 0x34, receives the same fourfold-scaled result twice consecutively.

Later phases propagate captured and scaled values through word destinations at offset 0x14. Both fourfold and sixteen-fold address scaling occur. Some stores use much larger factors, including \(2^{28}\) for an address and \(2^{26}\) for its value. No bounds or object-size interpretation is established.

### Separate reads of the unnamed fixed global

Before F-674e75a953e8, a late phase stores a retained result through a fourfold-scaled call-output address, then reads an unnamed fixed global twice:

- The first word read becomes a retained scalar.
- A separate, subsequent word read is multiplied by four to form another retained scalar.

No call intervenes between these two reads, but equality is not established. They remain distinct accesses; memory stability is not a precondition supplied by the representation.

After intervening calls, the second-read-derived scalar supplies a selector address by addition of one. Further calls separate two selector reads at that address. The first-read-derived scalar independently supplies a masked right-shift-by-10 call argument.

The two global reads must not be replaced with one snapshot used for both roles.

### Two reads at a derived offset 0x1C

The scalar formed from four times the second global read is later shifted left by 26. A subsequent indirect call separately reads a word at that shifted scalar plus 0x1C for each of its two arguments.

Both argument reads occur after the preceding indirect call and before entry to the argument-taking dependency. Their relative evaluation order is unspecified.

The shifted scalar is also multiplied by 1024. Under 32-bit arithmetic, that product is necessarily zero, although the shifted scalar itself need not be zero. After the two-read call and a further indirect call, this zero product produces a selector read at 0x01 and a selected-byte zero write. Later calls precede another selector read at that same address.

Consequently, the two word reads are not generally reads at 0x1C. Their address depends on the second global read before the later arithmetic collapses to zero.

### Two reads at a derived offset 0x14

After F-674e75a953e8 and further propagation phases, a result drives a word store at sixteen times that result plus 0x14, storing four times the result.

The next indirect call separately reads the word at four times that result plus 0x14 twice:

- One read supplies the first argument after logical right shift by 12.
- The other supplies the third argument after left shift by two.
- The middle argument is the fourfold-scaled result itself.

The two reads have no established relative evaluation order and need not yield equal words. The preceding sixteenfold-address store is not generally a store to their address.

Further calls precede a store of the retained fourfold-scaled result at another call-derived destination.

Later, a different phase captures one word from a fourfold-derived address plus 0x14, retains it across calls, and stores its right-shift-by-12 value. That single captured read must not be confused with the earlier two argument reads.

The final visible selected-byte store extracts bits 20–27 from a retained result. Additional unresolved calls follow before terminal truncation.

## Side effects

### Memory and aliasing

Direct effects include byte, halfword, and word writes. Fixed and computed locations may overlap. No accessed region is established as private scratch memory.

Repeated equal writes remain separate represented operations. Whether ordinary execution exposes them individually to hardware or other observers depends on the unknown memory environment; volatile or device-memory semantics are not established.

### Ownership and lifetime

No direct allocation, release, ownership transfer, or returned ownership is established. Dependency argument passing does not establish any of these effects.

Later use of a captured address requires continued validity, but the mechanism maintaining its lifetime is unknown.

### Errors and partial execution

There is no visible transaction or rollback. A faulting access, trapping dependency, or nonreturning call can leave earlier effects in place.

No recoverable error indication or full-sequence completion guarantee is established.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-23325cd5e50c | One displayed 32-bit argument |
| F-bbb54b5f036d | Two copies of one captured value |
| F-f80b3ac065fc | Composite low half, high half, low half, low half |
| F-de15789688a8 | Two copies of one captured value |
| F-ea83f9a17404 | Two copies of one captured word |
| F-674e75a953e8 | Two copies of one captured value |
| F-U01 | Initial link-derived indirect call |
| F-U02 | Group identifier for otherwise unresolved calls |
| F-U03 | Unresolved four-argument interaction between F-bbb54b5f036d and F-f80b3ac065fc |
| F-U04 | Unresolved equal-argument interaction between F-de15789688a8 and F-ea83f9a17404 |

The provisional IDs distinguish interactions, not verified implementations. F-U02 does not imply that grouped sites share a target.

Dependency meanings, clobbers, memory effects, ownership behavior, failure behavior, and completion are **documented_only**.

## Acceptance examples

These are conditional analytical criteria, not executed tests.

1. **Byte destination and truncation.** A selector of 0xFF selects address 0x110. A source word of 0xFFC00000 shifted right by 22 yields 0x3FF; its byte store writes 0xFF.

2. **Distinct scaling.** For a result of 3, fourfold scaling plus 0x14 gives 0x20; sixteen-fold scaling plus 0x14 gives 0x44.

3. **Initial provenance.** Changing the initial word snapshot can change the shift-by-11 store and second-load address. Holding subsequent call results fixed does not make either later byte-store value derive from that initial snapshot.

4. **Three reads at 0x28.** If the initial capture yields 0x11111111 and the argument reads yield different words, the eventual destination still receives 0x11111111. Either argument-read evaluation order remains permitted.

5. **Independent global reads.** If the first unnamed-global read yields 0x400 and the second yields 3, the retained masked-shift argument can be 1 while the scaled selector address is 0x0D. The derived offset-0x1C address is 0x3000001C under 32-bit arithmetic, although the later multiplication by 1024 yields zero.

6. **Separate argument reads.** If the two reads at 0x01 yield different bytes, the two arguments may differ. The same independence applies to the paired word reads at derived offsets 0x1C and 0x14.

7. **Repeated writes.** A captured selector of 7 causes consecutive offset-0x34 word writes of 7 and 14. The other repeated-store phase performs two writes even when both values are equal.

8. **Partial execution.** If F-de15789688a8 does not return, later accesses are not required. Earlier stores are not rolled back.

## Uncertainties

The following remain **documented_only**:

- Complete entry state, calling conventions, and additional register-output contracts.
- Executability and completeness of the represented control flow.
- Byte order, alignment, memory mapping, concurrency, and reentrancy.
- Machine behavior at signed-overflow-sensitive calculations.
- Relative evaluation order of separate reads within call arguments.
- Dependency identities and effects beyond the mapped interactions.
- Whether composite results represent 64-bit returns or register pairs.
- Object boundaries, ownership, and address lifetimes.
- The operational meaning of terminal truncation.

Packed-value processing is inferred intent only. No assumption of stable globals, equal repeated reads, normal return, or dependency-preserved memory resolves these unknowns.

## Evidence

Established static evidence consists of the supplied routine representation and dependency mapping: access widths, arithmetic, capture relationships, separate read occurrences, argument positions, statement-level ordering, fixed-address accesses, and terminal truncation.

In particular, the representation distinguishes three word reads at 0x28, two reads of the unnamed fixed global, two byte arguments read at 0x01, and paired word arguments read at derived offsets 0x1C and 0x14. It also distinguishes the initial word snapshot, the second word snapshot, and the replacement call result.

The representation’s warnings limit confidence in executable control flow. Arithmetic consequences stated under a 32-bit interpretation are conditional deductions, not runtime observations.

No implementation access, dependency inspection, runtime execution, or runtime tests have been performed.
