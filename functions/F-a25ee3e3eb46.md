<!-- Trusted provenance: F-a25ee3e3eb46; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a25ee3e3eb46

## Purpose

F-a25ee3e3eb46 forwards three inputs through an ordered series of indirect operations, transforms selected results, performs address-dependent memory updates, and finishes with an indirect operation receiving four represented arguments.

No application-level purpose is established. Character-like constants do not establish text output. Address-following operations do not establish a container format, object model, or allocation scheme.

This contract distinguishes:

- **Established static evidence:** represented input widths, explicitly typed indirect accesses, arithmetic relationships, retained observations, and operation order.
- **Inferred intent:** possible interpretations that impose no behavioral requirements.
- **documented_only:** unresolved properties, including dependency identities and access widths for globals whose declarations are absent.

“Read,” “write,” “observation,” and “assignment” describe represented evaluations. They do not independently establish physical transaction size, atomicity, or special-purpose storage behavior. No runtime tests have been performed.

## Interface

### Inputs and result

| Component | Represented width | Use |
|---|---:|---|
| First input | 32 bits | Forwarded unchanged to the initial indirect operation. |
| Second input | 32 bits | Forwarded unchanged to the initial indirect operation. |
| Third input | Signed 32 bits | Forwarded initially and later shifted left by five. |
| Result | No explicit value | No status, address, or count is explicitly returned. |

The initial operation receives the three inputs in order, followed by zero. Its target depends on incoming execution context beyond those inputs. The input tuple alone does not determine execution.

Intermediate operations supply represented 32-bit values, 64-bit values with separately used lower and upper 32-bit portions, and secondary result state. An operation with no represented explicit arguments may still have hidden inputs.

### Storage widths and layout

Offsets are byte offsets. A **word** means an explicitly represented 32-bit indirect access.

| Storage category | Established access width |
|---|---:|
| Explicit byte and character accesses | 8 bits |
| Explicit halfword accesses | 16 bits |
| Explicit word accesses | 32 bits |
| Named globals without declarations | **documented_only** |

Byte accesses include offsets +1, +4, +5, +8, +9, +0x0D, +0x10, +0x11, +0x15, and +0x19. Word accesses include +4, +0x10, +0x14, +0x20, +0x24, +0x34, +0x40, +0x44, +0x54, +0x64, and +0x74. Halfword stores use doubled and eightfold-scaled addresses.

The named locations associated with 0x01, 0x05, 0x15, 0x21, 0x3A, 0x54, 0x60, 0x67, 0xA9, and 0x087FBBBC have no supplied declarations. Their receiving values and assignment operands do not establish their access widths. In particular, converting an observation to 32 bits does not prove a 32-bit load, and assigning an eight-bit-narrowed operand does not prove a byte store.

Computed accesses may overlap. At a common base, a word at +0x14 intersects a byte at +0x15; a word at +0x10 intersects bytes at +0x10 and +0x11. Separately derived bases may also coincide. No general non-aliasing guarantee is established.

Normal completion requires callable targets, returning dependencies, and accessible storage for every reached access. No visible validation establishes these conditions. Address validity, alignment, byte order, and address-space interpretation remain unresolved.

## Behavior

### Indirect operations and arithmetic

A long, ordered indirect-operation phase precedes the direct memory-update region. The final indirect operation follows that region. No explicit input-controlled loop or conditional branch appears in the represented body. Dependencies may branch, fault, or fail to return.

The indirect phase propagates shifted primary results, secondary result state, portions of 64-bit results, explicit zeros, masked fields, and retained values. Operations whose primary results are unused remain part of the sequence; purity is not established. No result is visibly checked for an error sentinel.

The following arithmetic qualifications apply throughout:

- Unsigned 32-bit left shifts retain the low 32 bits; unsigned right shifts introduce zeros.
- Extraction of bits 10–19 yields 0 through 1023 before subsequent scaling.
- A 24-bit field shifted left by six occupies at most 30 bits.
- Unsigned 64-bit right shifts by 34, 36, and 44 correspond to logical right shifts of the upper 32-bit portion by two, four, and twelve.
- Successive truncating 32-bit left shifts by 28 and 28 retain no bits; they do not create a 56-bit intermediate.
- Explicit eight- and sixteen-bit destinations retain the corresponding low bits.
- An explicit unsigned-byte observation converted to an unsigned word is zero-extended. This does not establish the extension behavior of an undeclared global.

Several signed shifts and multiplications can exceed signed range. Their actual overflow semantics are **documented_only**. Wraparound-dependent conclusions are conditional, not unconditional guarantees.

### Initial memory window

Two separately produced dependency results supply two large address-forming values, each scaled by \(2^{25}\). Call them the **first large base** and **second large base**, according to provenance.

**Separate provenance does not establish numerical inequality.** The bases may have the same value. They must nevertheless be tracked separately unless equivalence is established for the relevant execution; equality does not justify merging or omitting their producing operations. Under truncating 32-bit arithmetic, both have zero low bytes.

A separate **small value** is obtained through an eight-bit left shift, a 15-bit mask, and a logical right shift by six. Under the represented bitwise interpretation, its range is 0 through 508 in multiples of four. The **small doubled base** ranges from 0 through 1016.

The initial effects preserve this order:

- A result-derived +0x14 word receives the first large base.
- First-large +9 receives the low byte of a returned upper portion.
- Another result-derived +0x54 word receives the second large base.
- First-large +9 receives the first large base’s low byte, conditionally zero under truncating arithmetic.

Second-large +0x14 is then cleared. A byte is observed at second-large +5; small-doubled +0x14 is cleared; the zero-extended byte is stored as a word at second-large +0x34.

The word at second-large +0x14 is subsequently retained as the **early saved word**. Small-doubled +0x14 is cleared again, followed by two byte writes at small-doubled +0x15 using the early saved word’s low byte.

A zero early saved word requires that intervening aliasing or environmental effects have not changed the relevant storage. Equal large bases do not permit deleting accesses or ignoring their represented order.

### Global-derived base and early chains

After the duplicate byte writes, the global associated with 0x05 is observed and converted to an unsigned 32-bit **global-derived base**. Its range is not established as 0–255.

The global at 0x01 is next assigned the small doubled base explicitly narrowed to eight bits; its destination width remains unknown. An explicit byte at global-derived +0x15 receives the small value’s low byte.

An important interleaving follows:

| Order | Destination | Value |
|---:|---|---|
| 1 | Second-large +0x11, byte | Low byte of a retained result shifted by eight |
| 2 | Early-saved-word +0x54, word | Full retained shifted result |
| 3 | Second-large +0x11, byte | Zero |

The intervening word store cannot be moved outside the byte-write pair without an additional equivalence argument.

Global-derived +0x54 is read as a word and then overwritten with the early saved word. Its original observation becomes the **saved link**. This is not necessarily a load through a byte-derived address.

Further chains follow +0x74 words, write through resulting +0x54 destinations, and repeat +0x24 word and +9 byte writes. One compound relationship contains two separate word observations at small-doubled +0x74. Their internal evaluation order is not independently resolved; replacing both with one cached read is not established as equivalent.

Among repeated small-doubled +0x14 word writes, the global at 0x54 receives twice the saved link through a signed shift. The global destination width is unknown. Two later small-doubled +0x15 byte writes precede assignment of the eight-bit-narrowed small doubled base to the global at 0x15.

### Subsequent linked updates

Before adjustment of the saved link, several relationships determine later addresses:

- Four times the saved link supplies a repeatedly used base.
- A small-doubled +0x74 word, doubled, supplies another update base.
- An explicit unsigned byte at the fourfold-saved-link base +5 supplies the base for a +0x74 word observation.
- That resulting word receives chained +0x44 updates and supplies retained low-byte writes through +0x0D destinations.
- A +0x64 word supplies another base for +0x0D and +0x14 writes.
- A word at fourfold-saved-link +4 supplies a base whose +5 byte is observed twice, with intervening writes.

The two +5 observations remain separate: their values may differ if intervening effects overlap their storage. The later observation supplies a zero-extended base for another +0x74 word read. Call that word the **link value**. Its doubled and quadrupled forms are retained for the later region.

These relationships establish address derivation and update order, not object ownership or a linked-container abstraction.

### Constants and repeated region

A constant group stores 0x3A as a byte at four times the link value, a halfword at eight times the link value, and a word at four-times-link +0x20.

Define the **adjusted value** as four times the saved link minus decimal 150, and the **working base** as four times the adjusted value, subject to the signed-arithmetic qualifications above.

Working +0x20 receives, in order, 0x20, 0x2F, a freshly observed and zero-extended byte from working +4, and 0x20 again.

The global associated with 0x087FBBBC is then observed into a signed 32-bit **external base**. The observation’s source width is unknown. Only afterward does the global at 0x60 receive 0x20.

The expanded repeated region preserves this four-store order:

| Position within occurrence | Effect |
|---:|---|
| 1 | A scaled-base +0x54 word receives the quadrupled link value. |
| 2 | External +0x19 receives the retained working-base low byte. |
| 3 | Working +0x11 receives the retained external-base low byte. |
| 4 | External +0x34 receives an eightfold-scaled value. |

The initial occurrence uses eight times the saved link for the first destination’s base and the fourth store’s value. Subsequent occurrences use eight times the adjusted value.

These are expanded repetitions, not an input-controlled loop. Exact repetition multiplicity is **documented_only** in this contract. Replacing the region with final-state assignments is not established as equivalent, especially where destinations overlap or storage has externally observable access behavior.

### Post-repetition observations

After the repeated region, the order is:

1. Observe the global at 0xA9 into a retained 32-bit value.
2. Read a word at quadrupled-link +0x74.
3. Assign the global at 0x3A a character-derived arithmetic expression.

The 0xA9 observation is not established as a 32-bit load. For the 0x3A expression, the adjusted value is first converted to an eight-bit character representation, multiplied by eight, and increased by 0x30. Character signedness, arithmetic conversions, destination type, and final conversion remain unresolved. No final explicit byte narrowing is established for this global assignment.

Doubled-link +0x14 and working +4 are next read as words, in that order. The ensuing effects preserve:

| Order | Destination | Value |
|---:|---|---|
| 1 | Doubled-link +0x0D, byte | Retained working-base low bits |
| 2 | Working +8, byte | 0x6C |
| 3 | Working +0x0D, byte | Retained external-base low bits |
| 4 | Working +9, byte | Same retained external-base bits |
| 5 | Doubled-link +0x54, word | Doubled-link base |
| 6 | Global at 0x67 | Saved 0xA9-derived value explicitly narrowed to eight bits |

The sixth assignment’s destination width remains unknown.

### Three separate observations of 0x67

Doubled-link +0x44 is read as a word after the 0x67 assignment. The first fresh observation of 0x67 then supplies assignment to the global at 0x21. Neither global’s width nor their intervening declared-type conversion is established.

Before the second observation, the saved +0x44 word supplies a byte write at working +0x10, a word write at external +0x54, and a shifted byte write at doubled-link +1. External +0x54 is then overwritten with the working base.

The right-shift count is the working base masked to its low eight bits. Under truncating arithmetic, the fourfold scaling restricts this count to multiples of four from 0 through 252. Signed-right-shift behavior and counts of at least 32 are **documented_only**.

Working +0x40 is next read as a word. The saved working +4 word is stored at external +0x74, the newly read +0x40 word at working +0x10, and the doubled-link base at external +0x44. The second fresh 0x67 observation supplies a byte write at working +9.

Before the third observation, doubled-link +9 receives the retained working-base byte, working +0x44 receives the saved 32-bit 0xA9-derived value, and working +0x54 is read as a **newly loaded base**. The third fresh 0x67 observation then supplies another byte write at working +9.

These are three represented global observations, not three established byte loads. Their values need not agree.

### Final updates and transfer

The saved quadrupled-link +0x74 word’s low byte is written at newly-loaded +0x0D. The earlier doubled-link +0x14 word is restored at doubled-link +0x54.

Working +0x14 is read and retained. Before its low byte is written at newly-loaded +0x11:

- Doubled-link +0x11 receives the saved quadrupled-link byte.
- Working +9 receives the newly loaded base’s low byte.
- Working +0x44 receives that base as a word.
- A word at doubled-link +4 supplies the **final base**.
- A halfword constant 0x65 is stored at twice the final base.
- Final-base +0x64 is read and retained.

Newly-loaded +0x11 then receives the saved working +0x14 low byte, and newly-loaded +0x54 receives decimal 100 as a word.

Final-base +0x74 is read and retained. Doubled-link +0x64 is separately read, after which doubled-link +9 receives the final base’s low byte.

The last direct word stores place the saved final-base +0x64 value at separately-loaded +0x54, then place that separately loaded value at twice the saved final-base +0x64 value.

The final indirect operation receives:

| Position | Represented argument |
|---:|---|
| 1 | 0x72 |
| 2 | Saved word from final-base +0x64 |
| 3 | Earlier saved word from doubled-link +0x14 |
| 4 | Saved word from final-base +0x74 |

These arguments use retained observations, not transfer-time reloads. A return is represented afterward; the actual transfer mechanism and return behavior remain unresolved.

## Side effects

Direct effects include byte, halfword, and word accesses through returned, loaded, and scaled addresses, plus named-global observations and assignments.

| Named location | Represented action | Width qualification |
|---|---|---|
| 0x01, 0x15 | Assignment of explicitly eight-bit-narrowed operands | Destination widths unknown |
| 0x05 | Observation converted to unsigned 32 bits | Source width and extension unknown |
| 0x21 | Assignment from a fresh 0x67 observation | Both global types unknown |
| 0x3A | Assignment of character-derived arithmetic | Destination width and final conversion unknown |
| 0x54, 0x60 | Assignments of a shifted value and 0x20 | Not established as word stores |
| 0x67 | Assignment, then three observations | Global access widths unknown |
| 0xA9, 0x087FBBBC | Observations retained in 32-bit values | Not established as word loads |

No transaction, rollback, synchronization, or atomicity mechanism is represented. Earlier effects may remain if a later operation faults or fails to return.

No direct allocation or release operation is identified. Returned addresses are not proven newly allocated or owned. Input forwarding does not establish ownership transfer. Unknown dependencies may mutate, retain, allocate, or release storage.

No explicit error-reporting or recovery path is established.

## Dependencies

F-a25ee3e3eb46 is the only supplied opaque function ID. Dependency IDs are unavailable and remain **documented_only**; none are invented.

| Dependency role | Established interface evidence | Unresolved properties |
|---|---|---|
| Initial indirect target | Three inputs followed by zero | Identity, hidden inputs, calling convention, effects |
| Intermediate indirect targets | Transformed arguments, primary results, paired portions, secondary state | Exact signatures, preservation, ownership, failures |
| Final indirect target | Four arguments beginning with 0x72 | Identity, result meaning, transfer and return behavior |

Different call sites are not assumed to share targets. No dependency is assumed to preserve globals, memory, or ownership. Equal returned values do not establish equivalent dependency behavior.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Input forwarding and small-value boundary

Given distinct 32-bit input patterns and a valid initial target, the initial argument sequence preserves those patterns followed by zero.

If the small-value construction receives a value whose low seven bits are all set, and the shift has the represented truncating interpretation, the small value is 508 and the small doubled base is 1016.

### Equal large bases with separate provenance

If each producing result contributes numeric value 1 to its respective \(2^{25}\) scaling, both large bases can equal 0x02000000. Equality is permitted.

The first-large +9 writes then address 0x02000009, while second-large +0x14 addresses 0x02000014. Subsequent accesses use the same shared base where their formulas coincide. The producing operations and all represented accesses remain required; equal scaled values do not establish purity or authorize merging calls.

### Global-derived address range

If the 0x05 observation converts to unsigned 32-bit value 0x1234, the designated byte write uses 0x1234 +0x15. The base is not restricted to 0x34, and no eight-bit source observation is required.

### Ordering and narrowing

The early trace preserves second-large +0x11 byte write, early-saved-word +0x54 word write, then second-large +0x11 byte clear.

If the 0xA9 observation converts to retained value 0x12345678, the later assignment operand for 0x67 is eight-bit pattern 0x78. This establishes neither a word load at 0xA9 nor a byte store at 0x67.

If the three subsequent 0x67 observations yield 0x12, 0x134, and 0x256, the 0x21 assignment is sourced from 0x12 subject to its unknown destination conversion. The two designated working +9 byte writes receive 0x34 and 0x56.

### Retention and partial failure

The final third argument uses the saved doubled-link +0x14 word even if later overlapping writes change that storage.

If an earlier effect succeeds and a later access fails, neither rollback nor a status result is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object model, and data-format meaning.
- Dependency identities, signatures, hidden inputs, preservation, ownership, and failures.
- Named-global declarations, widths, signedness, and conversions.
- Extension and range of the 0x05-derived base.
- Actual equality and overlap among computed addresses for a particular execution.
- Byte-level overlap and physical transactions involving named globals.
- Character interpretation and final conversion for the 0x3A assignment.
- Signed overflow, signed right shifts, and oversized shift counts.
- Address validity, alignment, byte order, and special-purpose storage behavior.
- Exact expanded repetition multiplicity in this contract.
- Allocation, release, retention, lifetime, and final transfer behavior.

A state-construction or state-transformation purpose is **inferred intent**, not an established application contract.

## Evidence

The supplied static body establishes explicit input widths, typed indirect-access widths, arithmetic relationships, separate result provenance, represented evaluation order, repeated observations, explicit narrowing, and retained final-argument provenance.

The two large address-forming values originate in separate dependency results. Nothing establishes that those results, or their scaled values, are numerically unequal. The contract therefore preserves both provenance and the possibility of equal addresses without treating equality as permission to omit operations.

Named-global declarations are absent. Receiving-value widths and assignment-operand widths cannot establish global access widths, extension rules, or a complete byte-level overlap trace.

Embedded comments, prior prose, and reviewer feedback are not independent behavioral authority. No dependency inspection, implementation access, runtime execution, or runtime tests have been performed.
