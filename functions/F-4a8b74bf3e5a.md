<!-- Trusted provenance: F-4a8b74bf3e5a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4a8b74bf3e5a

## Purpose

F-4a8b74bf3e5a performs an ordered sequence of unresolved indirect calls, derives address-forming values from their results and execution context, then accesses computed storage and fixed globals. Execution concludes with an unresolved indirect transfer represented as a four-argument call.

No application-level purpose is established. Character-valued constants do not establish text processing, and self-valued stores do not establish linked structures. Allocation, parsing, formatting, initialization, and serialization are not established purposes.

This contract distinguishes represented static behavior from inferred intent and **documented_only** unknowns. No runtime tests have been performed.

## Interface

### Effective inputs and outputs

The represented interface has no explicit parameters and no declared return value. Effective inputs include:

- Incoming execution context forming the first indirect destination.
- Primary and combined dependency results.
- Additional post-call execution state.
- Current fixed-global values.
- Storage reached through computed addresses and captured values.

Outputs comprise indirect-call events and represented arguments, direct storage assignments, and the terminal transfer. No scalar result, success flag, or structured error is established.

Later events require preceding calls and accesses to complete. Completion after the terminal transfer additionally requires that dependency to return.

### Widths: values versus storage accesses

**A value’s conversion width does not establish the width of the storage access supplying or receiving it.** The excerpt omits declarations for fixed globals.

| Category | Established representation |
|---|---|
| Explicit computed byte access | 8-bit access. Explicit unsigned byte reads produce 0–255. |
| Explicit computed halfword store | 16-bit store affecting two bytes. |
| Explicit computed word access | 32-bit access affecting four bytes. |
| Combined dependency result | 64-bit representation; low and high portions are consumed separately. |
| Fixed global at 0x05 | Value converted to unsigned 32 bits. Original read width, signedness, and extension behavior are unresolved. |
| Fixed assignments at 0x01, 0x15, 0x67 | Right-hand values explicitly narrowed to eight bits; destination access widths remain unresolved. |
| Fixed assignment at 0x21 | Consumes the current global value at 0x67 without an explicit eight-bit conversion at this assignment. Both global access widths remain unresolved. |
| Fixed assignment at 0x3A | Consumes promoted arithmetic involving an eight-bit plain-character conversion. Destination width and final conversion remain unresolved. |
| Fixed assignments at 0x54 and 0x60 | Consume a represented 32-bit shifted value and decimal 32, respectively. Destination widths remain unresolved. |
| Fixed captures at 0xA9 and 0x087FBBBC | Captured into 32-bit values. Original read widths and source conversions remain unresolved. |

Thus, the fixed-global events cannot uniformly be described as byte or word accesses. The overlapping-symbol warning supplies no authoritative types, access widths, or storage extents.

### Arithmetic and execution requirements

Address-forming values are predominantly represented through 32-bit quantities. Unsigned 32-bit conversion produces a value modulo \(2^{32}\), but does not reveal whether its source was narrower, wider, signed, or unsigned.

Intermediate widths matter. A 32-bit left shift followed by a logical right shift does not recover discarded bits. Combined-result extractions use unsigned 64-bit shifts before narrowing.

Signed left shifts, multiplication, addition, and subtraction require defined operands or a separately specified target arithmetic model. Signed right shifts, oversized shift counts, pointer conversion, alignment, and address wrapping are **documented_only**. Plain-character signedness and byte order are also unresolved.

Executable indirect destinations, meaningful ABI state, and accessible reached storage are preconditions, not checks performed here. No explicit null, bounds, alignment, or overflow validation, structured recovery, or rollback is represented.

## Behavior

### Indirect-call phase

A long straight-line sequence of indirect calls precedes the explicit computed-memory mutation phase.

The first destination depends on incoming execution context plus displacement 0xFFE. Subsequent destinations use embedded execution locations or code references with the same represented displacement. No recovered dispatch table, registry, or dependency identity follows.

Calls have zero through four represented arguments. Zero represented arguments do not exclude implicit ABI inputs. Results include 32-bit values, combined 64-bit values, and additional post-call state. Calls with locally unused results remain observable.

Argument formation includes left shifts through count 31, logical right shifts, masks selecting fields up to 24 bits, unsigned combined-result extraction after shifts of 32, 34, 36, and 44 bits, explicit zeros, and reuse of saved values across intervening calls.

Repeated argument shapes do not establish identical destinations. Individual call-site enumeration and the complete argument trace remain **documented_only** coverage limitations.

### Captured-value vocabulary

These terms describe arithmetic relationships, not typed objects:

| Term | Meaning |
|---|---|
| Early magnitude | A primary result shifted left eight bits, masked to its low 15 bits, then logically shifted right six bits. |
| Early doubled base | Twice the early magnitude. |
| Global-derived base | The unsigned 32-bit conversion of the fixed global at 0x05; not established as restricted to 0–255. |
| Saved source word | Explicit 32-bit read at global-derived base +0x54, captured before that location is overwritten. |
| Adjusted quantity | Four times the saved source word minus decimal 150. |
| Adjusted base | Four times the adjusted quantity. |
| Referent word | A later explicit 32-bit read reached through an unsigned-byte-derived +0x74 access. |
| Doubled referent base | Twice the referent word. |
| Quadrupled referent base | Four times the referent word. |
| Anchor | The 32-bit captured value obtained from the fixed global at 0x087FBBBC; the global’s original read width is unresolved. |

Arithmetic relationships apply where operations are defined, or conditionally under an explicitly selected low-32-bit model.

### Opening effects and source capture

The final combined dependency result contributes to the opening destinations. Observable computed accesses begin with a 32-bit store through its high half +0x14, an eight-bit store at a previously derived base +9, a 32-bit store through its low half scaled by 256 +0x54, and another eight-bit store at the earlier +9 destination.

A computed +0x14 word is cleared before an unsigned byte is read at the same base +5. A different computed +0x14 word is cleared before the widened byte is stored as a word at the first base +0x34. The first +0x14 word is then read back. Further clearing and two identical byte stores follow.

These are ordered events, not merely final-memory requirements. Repeated stores and readback must not be discarded as redundant.

The fixed-global read at 0x05 and its unsigned 32-bit conversion precede the fixed assignment at 0x01. That assignment receives the early doubled base narrowed to eight bits; its destination width is unknown.

Intervening computed stores occur before the saved source word is read at global-derived base +0x54. The source location is then overwritten. Later use of the saved value does not constitute a fresh read of that location.

### Early scaling and dependent accesses

Before the fixed assignment at 0x54, computed effects include repeated self-valued +0x24 word stores, byte stores, and a call-derived word write at early doubled base +0x14.

The assignment at fixed global 0x54 consumes **twice the saved source word**, represented by a one-bit left shift. Its destination width and assignment conversion are unresolved.

Subsequently:

- Two word stores place the saved source word at early doubled base +0x14.
- Two byte stores place the narrowed fourfold saved-source value at early doubled base +0x15.
- The fixed assignment at 0x15 receives **the early doubled base narrowed to eight bits**, not the fourfold saved-source value.

Computed destinations need not be disjoint.

Some dependent accesses explicitly read an unsigned byte at +5 and use its numeric value as the base for a word read at +0x74. These accesses differ from the undeclared fixed-global read at 0x05: their eight-bit width and unsigned widening are established.

Other word reads at +0x74 provide bases for later +0x54 stores. One early self-valued store contains two represented reads at early doubled base +0x74. Their relative evaluation order is unresolved; neither can automatically be replaced by a cached value.

Computed offsets establish access locations, not a recovered record layout:

| Offsets | Represented computed accesses |
|---|---|
| +1, +5 | Byte store at +1; unsigned byte reads at +5. |
| +4 | Word reads and a separate unsigned byte read. |
| +8, +9, +0x0D, +0x11, +0x15, +0x19 | Byte stores. |
| +0x10 | Byte and word stores. |
| +0x14 | Word reads, clears, and other word stores. |
| +0x20, +0x24, +0x34 | Word stores, including repeated self-valued +0x24 stores. |
| +0x44, +0x54 | Word reads and stores. |
| +0x40, +0x64, +0x74 | Word reads; +0x74 also receives later stores. |

No allocation size, object extent, ownership relation, or non-aliasing invariant follows.

### Mixed-width constants and anchor boundary

Decimal 58 is stored through three computed destinations, in order:

| Width | Destination |
|---:|---|
| 8 bits | Four times the referent word. |
| 16 bits | Eight times the referent word. |
| 32 bits | Four times the referent word +0x20. |

These widths are established by explicit computed access forms. Destinations can coincide.

At adjusted base +0x20, successive word stores supply decimal 32, decimal 47, the widened unsigned byte read at adjusted base +4, and decimal 32 again. The byte read follows the 47 store and precedes its consuming word store.

The anchor capture follows the final computed 32 store and precedes the fixed assignment of 32 at 0x60. No other memory access is represented between these three boundary events.

Under defined arithmetic or low-32-bit arithmetic, adjusted base is sixteen times the saved source word minus 600. Therefore its +0x20 word starts at residue 8 modulo 16 and occupies residues 8–11. The anchor symbol’s address has residue 12.

This proves that the computed word does not include the anchor’s starting address. It does **not** establish the anchor global’s complete read footprint. A four-byte anchor read and a four-byte fixed 0x60 assignment would be disjoint from the computed word and from each other in ordinary flat-address memory, but those global widths are additional assumptions, not established declarations.

Other computed destinations can equal the anchor address. In particular, the decimal-58 word destination can do so.

### Repeated four-store groups

After the fixed 0x60 assignment, repeated groups each contain, in order:

1. A scaled-destination +0x54 word store receiving the quadrupled referent base.
2. An anchor +0x19 byte store receiving the adjusted base’s low eight bits.
3. An adjusted base +0x11 byte store receiving the anchor’s low eight bits.
4. An anchor +0x34 word store receiving the scaled value.

The first group uses eight times the saved source word as its scaled value. Later groups use eight times the adjusted quantity.

The anchor is the captured value, not a fresh global read in every group. No advancing index or traversal is represented. Exact group multiplicity is not enumerated here; this coverage limitation does not authorize collapsing repeated events.

### Late fixed captures and the 0x3A assignment

After the groups, the function captures the global at 0xA9 into a 32-bit value, then explicitly reads a word at quadrupled referent base +0x74.

The fixed assignment at 0x3A follows. Its right-hand value is formed by converting the adjusted quantity to an eight-bit plain-character value, promoting that value, multiplying by eight, and adding decimal 48.

**The complete right-hand expression is not established as eight bits.** If the converted character is signed with range −128 through 127, the promoted expression ranges from −976 through 1064. If unsigned with range 0–255, it ranges from 48 through 2088. Out-of-range signed-character conversion remains target-dependent. The undeclared destination determines any final conversion and write width.

After this assignment, the function explicitly captures the word at doubled referent base +0x14, then the word at adjusted base +4. The former supplies terminal argument three. Any actual overlapping 0x3A write footprint can affect these captures.

Following computed effects include byte stores at +0x0D and +9 destinations, decimal 108 at adjusted base +8, and a self-valued word store at doubled referent base +0x54.

### Three current reads of the global at 0x67

The global at 0x67 receives the saved 0xA9 capture explicitly narrowed to eight bits. This establishes the assignment input’s width, not the destination access width or its eventual stored representation.

Three separate current reads follow:

| Read | Consumer and ordering boundary |
|---|---|
| First | Assignment to fixed global 0x21, after the word read at doubled referent base +0x44. |
| Second | Byte store at adjusted base +9, after the adjusted-base +0x40 read and intervening computed writes. |
| Third | Another byte store at adjusted base +9, after additional stores and the adjusted-base +0x54 read. |

The first consumer has an undeclared destination and no explicit eight-bit conversion. The other two have explicit eight-bit computed destinations. None establishes the original width or signedness of the 0x67 reads.

Between the first and second reads, computed writes include adjusted base +0x10, anchor +0x54, doubled referent base +1, anchor +0x74, and anchor +0x44. The +1 byte derives from a signed word right-shifted by the adjusted base’s low eight bits. Signed shift behavior and oversized counts remain unresolved.

Between the second and third reads, a byte store at doubled referent base +9 supplies the adjusted base’s low eight bits; the saved 0xA9 capture is stored as a word at adjusted base +0x44; adjusted base +0x54 is then read.

Where computed writes overlap the actual 0x67 read footprint, the three observations can differ. The third consumer’s destination is overwritten again later.

### Remaining effects and terminal transfer

The word captured at adjusted base +0x54 becomes another base. The saved quadrupled-referent +0x74 word supplies narrowed stores at that base +0x0D and doubled referent base +0x11.

The saved doubled-referent +0x14 word is stored at doubled referent base +0x54 before adjusted base +0x14 is read. Later stores replace adjusted base +9 and +0x44 with values derived from the newly obtained base.

A word read at doubled referent base +4 supplies a final dependent base. Twice its value addresses a 16-bit store of decimal 101. Its +0x64 word is captured before intervening computed stores, including decimal 100 stored as a word. Its +0x74 word is captured afterward.

Following a word read at doubled referent base +0x64, final effects include a byte store at doubled referent base +9, a +0x54 word store through the newly read base, and a word store at twice the saved dependent +0x64 value.

The terminal transfer receives:

1. Decimal 114.
2. The saved final-dependent +0x64 word.
3. The saved doubled-referent +0x14 word captured after the 0x3A assignment.
4. The later final-dependent +0x74 word.

No returned value is consumed.

## Side effects

Fixed-global events occur in this relative order, with computed accesses intervening:

| Address | Established value-level event; global access width unresolved |
|---|---|
| 0x05 | Read and conversion to unsigned 32 bits. |
| 0x01 | Assignment input narrowed from early doubled base to eight bits. |
| 0x54 | Assignment input is the saved source word shifted left once. |
| 0x15 | Assignment input narrowed from early doubled base to eight bits. |
| 0x087FBBBC | Capture into the 32-bit anchor value. |
| 0x60 | Assignment input is decimal 32. |
| 0xA9 | Capture into a 32-bit saved value. |
| 0x3A | Assignment input is promoted character-derived arithmetic. |
| 0x67 | Assignment input narrowed to eight bits; three later current reads. |
| 0x21 | Assignment supplied by the first current 0x67 read. |

The table establishes neither adjacency nor storage isolation. Alias reasoning must distinguish a symbol’s starting address from its unknown access footprint.

No identified allocation or release primitive establishes ownership. Dependencies may allocate, release, retain, invalidate, or mutate storage. Address-like stores establish neither ownership transfer nor lifetime extension.

No lock, barrier, atomic operation, or rollback is represented. Thread safety, reentrancy, and hardware memory ordering remain **documented_only**. Earlier effects are not represented as undone when execution stops prematurely.

## Dependencies

The following are document-local opaque dependency IDs, not recovered identities:

| ID | Scope |
|---|---|
| D-001 | First context-derived indirect call; no explicit arguments represented. |
| D-002 | Remaining pre-mutation indirect call sites, collectively. |
| D-003 | Terminal four-argument indirect transfer. |

D-002 represents multiple sites, not one shared function. Identities, signatures, calling conventions, implicit inputs, preservation rules, ownership behavior, and failure modes are unresolved. No dependency is assumed to preserve globals or reachable storage.

## Acceptance examples

These are conditional static scenarios, not executed tests. Each requires completion of preceding events, valid accesses, and defined arithmetic or an explicitly chosen target model.

### Width-sensitive values

For a combined result with low half 1 and high half 2, a consumer using low-half shift by eight, unchanged high half, and high-half shift by six receives 256, 2, and 128.

Under low-32-bit shifts, shifting 0x01000001 left eight bits and logically right eight bits yields 1.

An explicit computed unsigned-byte read of 0xFF produces 255. The fixed-global read at 0x05 has no equivalent unconditional guarantee.

### Global-derived base outside the byte range

If the undeclared 0x05 global supplies numeric value 0x1234 and its unsigned conversion preserves that value, the global-derived base is 0x1234. The saved source word is read at 0x1288. Restricting the base to 0x34 would violate this conditional scenario.

### Saved source and assignment inputs

With saved source word 7 and early doubled base 0x120, the fixed 0x54 assignment input is 14, and the fixed 0x15 assignment input is the eight-bit value 0x20.

These facts alone do not prescribe either destination’s write width. Overwriting the original source location does not replace the saved 7.

### Coincident decimal-58 destinations

For referent word zero, the computed byte and halfword stores both begin at zero; the computed word store begins at 0x20. Acceptance preserves the 8-bit, then 16-bit, then 32-bit events.

### Conditional anchor observation

Assume saved source word 64, referent word 0x021FEEE7, ordinary flat-address memory, and a four-byte anchor read preserving the stored word.

The adjusted quantity is 106 and adjusted base is 0x1A8. Its +0x20 word begins at 0x1C8. Four times the referent word plus 0x20 equals 0x087FBBBC, so the decimal-58 word store begins exactly at the anchor address.

If no intervening modification affects that anchor read footprint, the capture yields 58. Without the assumed anchor width and interpretation, equal starting addresses alone do not guarantee that result.

### Conditional 0x3A alias

Assume saved source word 40, giving adjusted quantity 10, and referent word 0x13. Doubled referent base +0x14 begins at 0x3A.

The promoted 0x3A assignment input is 128. If that global is an eight-bit destination preserving 0x80, memory is little-endian, and the captured word previously contains 0x11223344, terminal argument three captures 0x11223380 absent intervening modification. A wider global write requires different byte-level expectations.

### Separate 0x67 observations

Assume an unsigned eight-bit 0x67 global and referent word 0x2F. Doubled referent base +9 is 0x67.

If the second read observes 0x42, its computed byte consumer receives 0x42. The intervening +9 byte store replaces that storage with the adjusted base’s low eight bits. Absent another modification, the third read observes the replacement.

This exact byte-level result is conditional on the global declaration; distinct read events are established regardless.

### Incomplete execution

A nonreturning earlier dependency prevents later effects. A failed access establishes neither rollback nor execution of the terminal transfer.

## Uncertainties

- **documented_only — fixed-global types:** Declarations, access widths, signedness, extension, assignment conversions, and footprints are unavailable, including for the anchor.
- **documented_only — dependencies and ABI:** Identities, implicit inputs, preservation, and return behavior remain unresolved.
- **documented_only — arithmetic:** Signed overflow, shifts, character conversion, pointer conversion, and wrapping need target semantics.
- **documented_only — memory:** Accessibility, alignment, byte order, device behavior, mappings, and concurrency are unknown.
- **documented_only — ownership and layout:** Access offsets establish neither complete objects nor retention rights.
- **documented_only — coverage:** Individual indirect calls and repeated-group multiplicity are not enumerated; this is not a complete executable trace.
- **Inferred intent only:** Text and linked-structure interpretations are possible but unnecessary and unestablished.

Unknown global types do not erase explicit computed-access widths. Conversely, explicit eight-bit assignment inputs and 32-bit capture values do not resolve fixed-global storage widths.

## Evidence

The supplied static representation supports the computed access widths, value conversions, captured-value dependencies, ordering boundaries, repeated current-global reads, and terminal argument sources described above.

The fixed-global declarations are absent. Their widths cannot be recovered solely from assignment inputs, receiving values, symbol addresses, or the overlapping-symbol warning. Accordingly, exact global footprints and byte-level alias outcomes remain conditional.

Comments, prior prose, and reviewer feedback are not independent behavioral authority. Dependency bodies, authoritative ABI and global declarations, memory-layout definitions, and executable observations are unavailable. No runtime tests have been performed.
