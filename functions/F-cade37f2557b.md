<!-- Trusted provenance: F-cade37f2557b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cade37f2557b

## Purpose

F-cade37f2557b performs a sequence of indirect invocations, retains and transforms their results, mutates absolute and computed memory locations, repeatedly publishes retained values, and ends with an indirect transfer carrying four represented arguments.

No application-level purpose is established. Character-like constants do not establish text formatting; recurring offsets do not establish objects, collections, or validated pointer structures.

This contract distinguishes:

- **Established static evidence:** explicit access types, value conversions, constants, retained-value relationships, and represented sequencing.
- **Inferred intent:** possible interpretations, not behavioral requirements.
- **documented_only:** properties not resolved by the supplied representation.

Every later effect depends on execution reaching it. No runtime tests have been performed.

## Interface

### Entry and observable results

The presented interface has no explicit parameters and no explicit scalar result. Effective inputs include incoming control state, an incoming unsigned 32-bit quantity, indirect-invocation results, additional machine outputs, existing memory, and named globals whose declarations are absent.

The entry convention is **documented_only**. A conventional no-argument invocation is not established as sufficient to provide the required incoming state.

Observable results include:

- The indirect-invocation trace and represented arguments.
- Separate memory and global evaluations.
- Computed and absolute-address mutations.
- The terminal indirect transfer.

A return is represented after that transfer, but its reachability and the transfer’s continuation behavior are unresolved.

### Widths, conversions, and layouts

Explicitly typed indirect accesses establish the following represented widths:

| Access or retained quantity | Width |
|---|---:|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Explicit word access | 32 bits |
| Combined invocation result | 64 bits |

A combined result’s lower word is bits 0–31; its upper word is bits 32–63. This numerical decomposition does not establish memory byte order.

Explicit unsigned byte reads are zero-extended when converted to unsigned words. Explicit byte and halfword destinations receive their corresponding narrow conversions. These facts do **not** establish the storage types of undeclared named globals.

For every named global below, storage width, signedness, accessed extent, and declaration-dependent conversion behavior remain **documented_only**:

| Global location or role | Established use |
|---|---|
| Absolute 0x05 | Evaluated and converted to an unsigned 32-bit address-building value |
| Absolute 0x01 | Assigned an explicitly converted 8-bit value |
| Absolute 0x15 | Assigned an explicitly converted 8-bit value |
| Absolute 0x54 | Assigned an expression formed by shifting a retained signed word left by one |
| Publication-base source | Evaluated into a signed 32-bit retained value |
| Absolute 0x60 | Assigned numeric value 0x20 |
| Absolute 0xA9 | Evaluated into a 32-bit retained value |
| Absolute 0x3A | Assigned a character-converted arithmetic expression |
| Absolute 0x67 | Assigned an explicitly converted 8-bit value; evaluated separately three times afterward |
| Absolute 0x21 | Assigned the result of the first subsequent evaluation of global 0x67 |

Consequently, the 0x05 evaluation is not established as an unsigned byte load; the 0xA9 evaluation is not established as a four-byte load; and evaluations of 0x67 are not established as byte loads. Likewise, an 8-bit right-hand value does not prove an 8-bit global store. The global assignment at 0x54 has no established four-byte extent.

Offsets are byte displacements. Decimal offset 100 means 0x64; the late stored value decimal 100 means numeric value 0x00000064.

Unsigned operations support fixed-width interpretation. Signed left shifts, multiplication, overflow, address arithmetic, and variable signed right shifts require environmental semantics. Conditional examples using 32-bit modular arithmetic do not assert portable signed arithmetic.

### Preconditions and failures

Continuation requires compatible calling conventions, executable indirect targets, valid computed addresses, accessible storage, and appropriate alignment support.

No visible null, bounds, alignment, or target validation is represented. There is no explicit error result, diagnostic, retry, or recovery path. Fault behavior is environment-dependent and can expose an executed prefix of the mutations.

## Behavior

### Sequencing model

The visible body has no explicit application-level branch or loop. Repeated publication is unrolled.

Separate calls, reads, global evaluations, and stores remain distinct. Their apparent final memory state is not an adequate substitute where aliasing or external observation distinguishes intermediate events.

Sequencing here describes the supplied representation, not hardware ordering, atomicity, synchronization, or volatility. Operand evaluation order within a compound access is not established merely by the ordering of its written operands.

### Indirect invocation phase

The first target is derived from incoming control state plus 0xFFE. Subsequent targets are formed from embedded control-location values with the same displacement.

Invocations have zero through four represented explicit arguments. An invocation displaying no arguments is not proved to have no machine-level inputs. Combined results and additional machine outputs do not establish a complete return convention.

Established argument relationships include:

- The incoming unsigned word shifted right by 12, then left by seven, supplies two equal arguments.
- An early unsigned result contributes its low 11 bits shifted left by ten.
- Another result contributes a left shift by eight, bits 4–23 shifted left by 13, and later bits 19–23.
- Combined results repeatedly supply a lower word shifted left by eight and a separately used upper word.
- Other arguments select 16 bits from a masked 24-bit value, 12 bits from another masked 24-bit value, seven bits from an 18-bit selection, and 13 bits from a 17-bit selection.
- Single-bit selections appear at bit 16 or bit 31.
- Several invocations duplicate a retained expression in multiple argument positions.
- Additional machine outputs supply arguments directly or after shifting.

Intermediate widths matter. Under 32-bit bit-vector semantics, a left shift by 31 followed by a left shift by 30 yields zero.

Several retained quantities are explicitly reset to zero. Their later derivatives remain zero where the represented operations preserve zero. This does not imply that intervening dependencies preserve globals or unrepresented machine state.

### Initial direct mutations

The final combined invocation result participates in an initial ordered cluster:

- A word is stored at upper-word-plus-0x14.
- The upper word’s byte conversion is stored at an earlier retained base plus nine.
- A retained scaled word is stored at lower-word-times-256-plus-0x54.
- A second byte value replaces the same offset-nine location.
- A computed offset-0x14 word is cleared.

The two offset-nine stores are separate effects.

The following interval interleaves clears, explicit byte reads, word reads, and stores. An unsigned byte at a retained base plus five supplies a zero-extended word at that base plus 0x34. A small computed base’s offset-0x14 word is cleared twice. Another offset-0x14 word is read after its earlier clear, and the retained result’s byte conversion is written twice at small-base offset 0x15.

The global associated with 0x05 is then evaluated and converted to an unsigned 32-bit value used as an address base. Its source width and extension behavior are unresolved. The next global assignment, associated with 0x01, receives the small base’s explicitly narrowed value.

Further operations include:

- A byte store at the global-derived base plus 0x15.
- Successive nonzero and zero byte stores at a retained base plus 0x11.
- A word read at global-derived-base offset 0x54 before replacement of that location.
- Word stores and address chains involving offsets 0x54 and 0x74.
- Repeated stores of a retained word and its byte conversion at recurring destinations.

One nested operation contains two word evaluations at small-base offset 0x74, using them as destination-building and payload inputs. Their relative operand evaluation order is not established. They must not silently be treated as a single retained read.

A clear followed by a read does not guarantee zero without additional assumptions about aliasing, ordinary memory, and external modification.

### Global 0x54 assignment and address chains

Define:

- **Field value:** the retained word read at offset 0x54 through the global-derived base.
- **Small base:** twice the retained unsigned field extracted near the invocation phase’s end.

The following interval preserves four distinct assignments in this order:

| Order | Destination | Supplied value |
|---|---|---|
| First | Small base plus 0x14, explicit word | Earlier retained result shifted left by 26 |
| Second | Global associated with 0x54 | Field value shifted left by one |
| Third | Small base plus 0x14, explicit word | Field value |
| Fourth | Same explicit word location | Field value again |

No intervening memory access is represented. The second assignment’s destination width and declaration-dependent conversion are **documented_only**; its arithmetic also carries signed-shift uncertainty.

Subsequent accesses use four-times and eight-times-scaled forms of the field value. They include duplicated byte stores at small-base offset 0x15, the global assignment associated with 0x15, and word publications at offsets 0x34, 0x44, and 0x54.

Address chains follow both explicit unsigned bytes at offset five and words at offsets four, 0x64, and 0x74. Some byte-source locations are read again after intervening stores. These are fresh reads, not guaranteed repetitions of an earlier value.

Recurring offsets describe access sites, not independent fields. In particular, byte offsets 0x15 and 0x11 can overlap words beginning at 0x14 and 0x10. Absolute and computed accesses can also overlap. Numerical consequences depend on byte order and the actual access extents.

### Constants and repeated publication

A later interval writes numeric 0x3A through three explicitly sized accesses: a byte at a four-times-scaled address, a halfword at the corresponding eight-times-scaled address, and a word at the four-times-scaled address plus 0x20.

Define the **adjusted value** as the earlier four-times-scaled field value minus 0x96, and the **peer base** as four times that adjusted value.

Peer-base offset 0x20 receives successive words 0x20, 0x2F, a zero-extended byte, and 0x20. The explicit unsigned byte source is peer-base offset four, evaluated after the 0x2F store and before the third word store.

Next, the publication-source global is evaluated into the retained **publication base**, before global 0x60 is assigned 0x20. Neither global’s storage width is established. If the assignment affects the publication source, it does not retroactively change the retained base.

The repeated region consists of four-store groups:

1. A retained four-times-scaled word payload is stored at a computed offset-0x54 word location.
2. The peer base’s byte conversion is stored at publication-base offset 0x19.
3. The publication base’s byte conversion is stored at peer-base offset 0x11.
4. A scaled word is stored at publication-base offset 0x34.

The first group uses the original field value’s eight-times-scaled form for its first destination base and final payload. Subsequent groups use eight times the adjusted value instead. The first store’s payload remains the retained four-times-scaled word.

These are fixed, repeated address expressions, not advancing array positions. Every represented group remains a separate group; collapsing their multiplicity changes the access trace. No intervening calls are represented.

### Terminal chronology

Define:

- **Terminal anchor:** twice the loaded word whose four-times-scaled form supplies the repeated word payload.
- **Saved fixed value:** the 32-bit retained conversion of global 0xA9.
- **Saved anchor field:** the word subsequently read at terminal-anchor offset 0x14.
- **Late destination base:** the later word read at peer-base offset 0x54.
- **Final chain base:** the later word read at terminal-anchor offset four.

#### Global evaluations and initial terminal publications

Global 0xA9 is evaluated first. Its storage width and extension behavior are unresolved. An explicit word is then read at repeated-payload offset 0x74.

Global 0x3A is next assigned an expression involving the character-converted adjusted value, multiplication by eight, and addition of 0x30. Its destination width and full character-conversion semantics are unresolved.

Only afterward are the saved anchor field and peer-base offset-four word read and retained.

The following publications include the peer base’s byte conversion at terminal-anchor offset 0x0D, byte 0x6C at peer-base offset eight, the publication base’s byte conversion at peer-base offsets 0x0D and nine, and the terminal anchor as a word at its own offset 0x54.

Global 0x67 then receives the saved fixed value’s explicit 8-bit conversion. A word at terminal-anchor offset 0x44 is read before a fresh evaluation of global 0x67 supplies the assignment to global 0x21.

The offset-eight byte store of 0x6C therefore precedes both global assignments. Neither global assignment has an established byte-wide extent.

#### Computed updates and repeated global evaluations

The retained offset-0x44 word supplies:

- A byte at peer-base offset 0x10.
- A word at publication-base offset 0x54.
- A byte at terminal-anchor offset one, through a signed right shift whose count is the peer base’s low eight bits.

The shift can have a count outside 0–31; its interpretation is **documented_only**. Publication-base offset 0x54 is then overwritten with the peer base.

A word read at peer-base offset 0x40 precedes publication of the earlier offset-four word at publication-base offset 0x74. The offset-0x40 result is stored as a word at peer-base offset 0x10, followed by the terminal anchor at publication-base offset 0x44.

A second evaluation of global 0x67 supplies an explicit byte store at peer-base offset nine. Subsequent accesses publish the peer base’s byte conversion at terminal-anchor offset nine, store the full saved fixed value at peer-base offset 0x44, and read the late destination base.

A third evaluation of global 0x67 supplies another explicit byte store at peer-base offset nine.

These are three separate global evaluations overall. The latter two have established 8-bit **destinations**, not established 8-bit sources. Intervening aliasing mutations can affect each evaluation.

#### Final mutations and transfer

The earlier payload-relative offset-0x74 word supplies a byte at late-destination-base offset 0x0D. The saved anchor field is restored as a word at terminal-anchor offset 0x54.

A word read at peer-base offset 0x14 becomes the **late peer field**. Further stores publish the earlier offset-0x74 value’s byte conversion at terminal-anchor offset 0x11, the late destination base’s byte conversion at peer-base offset nine, and the full late destination base at peer-base offset 0x44.

After the final chain base is read, halfword 0x0065 is stored at twice that base.

The terminal argument dependencies require this chronology:

- Final-chain-base offset 0x64 is read and retained as the **first final link**.
- The late peer field’s byte conversion is stored at late-destination-base offset 0x11.
- Word value decimal 100 is stored at late-destination-base offset 0x54.
- Final-chain-base offset 0x74 is then read as the **fourth transfer argument**.

A word at terminal-anchor offset 0x64 becomes the **second final link**. The final chain base’s byte conversion is stored at terminal-anchor offset nine. The first final link is stored at second-final-link offset 0x54; the final direct word store writes the second final link at twice the first final link.

The terminal transfer receives, in order: 0x72, the first final link, the saved anchor field, and the fourth transfer argument.

## Side effects

Direct effects include explicit byte, halfword, and word accesses, plus named-global evaluations and assignments of unresolved storage widths.

No transaction, rollback, synchronization, allocation, release, or ownership protocol is established. Dependencies may nevertheless allocate, release, retain values, or mutate arbitrary state.

Address-building values are not proved to be owned pointers, borrowed references, handles, or validated storage. Ownership transfer remains **documented_only**.

## Dependencies

| Opaque ID | Established role | Unresolved properties |
|---|---|---|
| F-indirect-001 | Initial entry-derived invocation | Complete inputs, convention, results, preservation, effects |
| F-indirect-002 | Collective label for intermediate invocations | Individual identities, target equivalence, complete contracts, ownership, termination |
| F-indirect-003 | Terminal four-argument transfer | Target, transfer mechanism, continuation, return behavior |

The collective label does not identify one implementation or permit merging calls. No dependency is assumed to preserve globals, own passed memory, or release memory.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Incoming extraction.** Incoming unsigned 0x12345000 produces 0x12345 after shifting right by 12; the duplicated arguments are both 0x0091A280.

2. **Mask boundary.** An early unsigned result of 0xFFFFFFFF supplies 0x001FFC00 from its low 11 bits shifted left by ten.

3. **Established unsigned byte.** An explicit offset-five unsigned byte read of 0xFE supplies word 0x000000FE. This rule does not establish the conversion of global 0x05.

4. **Unresolved global extension.** If global 0x05 were a signed 8-bit value of −2, its unsigned 32-bit conversion would be 0xFFFFFFFE. If instead it were unsigned 8-bit 0xFE, the result would be 0x000000FE. The supplied evidence does not select either declaration.

5. **Global 0x54 ordering.** With field value three and defined shifting, six is supplied to the global assignment between the preceding computed word store and both subsequent stores of three. The stored extent and final declaration-converted value require the missing global declaration.

6. **Retained publication base.** If the global 0x60 assignment aliases publication-source storage, publication still uses the value retained before that assignment.

7. **Fresh global evaluations.** If a later evaluation of global 0x67 yields 0x99, the corresponding explicit byte destination receives its 8-bit conversion, regardless of the originally saved fixed value. This does not imply a byte-wide source read.

8. **Dispatch alias.** With late destination base 0x1020 and final chain base 0x1000, offsets 0x54 and 0x74 both designate 0x1074. Under stable ordinary memory and valid accesses, the decimal-100 word store supplies fourth argument 0x00000064.

9. **Retained link.** An intervening mutation of the first final link’s original storage does not change its retained second transfer argument. The fourth argument instead comes from its later required read.

10. **Terminal obligation.** Returning without the terminal transfer omits represented behavior. Reaching it requires first argument 0x72 and the order-dependent fourth argument.

## Uncertainties

The following remain **documented_only**:

- Application purpose and useful outcome.
- Entry convention, implicit inputs, dependency contracts, and target validity.
- Named-global declarations, storage widths, signedness, accessed extents, and extension or assignment conversions.
- Signed arithmetic, character signedness, overflow, address wrapping, and variable-shift behavior.
- Byte order, alignment, mappings, object boundaries, and low-address interpretation.
- Ownership, allocation, release, concurrency, volatility, and atomicity.
- Operand evaluation order not explicitly sequenced.
- Terminal transfer mechanism, successful completion, faults, and return behavior.

Formatting and structural-linking interpretations remain inference only.

## Evidence

The supplied static representation supports explicit indirect-access widths, masks, retained-value relationships, distinct global evaluations, and the sequencing described above.

Named-global declarations are absent. Casts and receiving-local widths establish value conversions, not source access widths. Narrow right-hand expressions likewise do not establish global destination widths.

The terminal sequence specifically places two stores between the final offset-0x64 and offset-0x74 reads. That later read supplies the fourth transfer argument and is alias-sensitive.

Prior prose, comments, and reviewer feedback are not independent behavioral evidence. No dependency implementation, runtime mapping, memory trace, successful execution, or runtime test has been established.
