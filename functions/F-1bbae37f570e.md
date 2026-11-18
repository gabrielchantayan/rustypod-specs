<!-- Trusted provenance: F-1bbae37f570e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1bbae37f570e

## Purpose

F-1bbae37f570e performs an ordered sequence of indirect invocations, transforms captured primary and secondary outputs, mutates byte-addressed storage at several widths, and reaches an unresolved indirect control transfer.

**Established static evidence:** the supplied representation exposes no explicit parameters, returned payload, conditional branches, or iteration. Observable behavior includes dependency effects, direct reads and writes, and the final transfer.

**[INFERENCE] Intent:** recurring offsets, scaled addresses, and address-like stored values suggest construction or linking of state. They do not establish object types, a protocol, or a serialization format.

The contract distinguishes destination-address derivation from stored-value derivation. In particular, storing four times a captured root does not imply addressing through four times that root. Intermediate overwrites, repeated accesses, and capture boundaries remain significant when addresses overlap or storage has observable access effects.

## Interface

### Inputs and outputs

There are no represented explicit arguments. Effective inputs include:

- Incoming execution context selecting the initial indirect target.
- Primary results and separately consumed secondary execution-state outputs from dependencies.
- Fixed-location storage.
- Storage reached through computed addresses.
- The unresolved final transfer target.

Some invocation results are represented as 64-bit quantities whose lower and upper 32-bit words are consumed independently. Separately represented secondary outputs have an unresolved calling convention and lifetime.

No success indicator, error code, output buffer, output length, or explicit returned payload is established. Outputs are dependency effects, direct storage mutations, and control transfer.

### Widths and addressing

| Quantity or access | Established representation |
|---|---|
| Explicit byte accesses | 8 bits |
| Halfword stores | 16 bits |
| Explicit word accesses | 32 bits |
| Most scalar transformations and computed bases | 32 bits |
| Combined invocation results | 64 bits |
| Global read at absolute address 5 | Source width and signedness unresolved |

Byte and halfword stores retain the low eight and sixteen bits, respectively. Explicit unsigned-byte loads used as words are zero-extended.

The global read at absolute address 5 is different: its value is converted to unsigned 32 bits, but its declaration is absent. That conversion establishes neither an 8-bit source access nor a range of 0–255.

Offsets are byte displacements, not established structure fields. An explicitly loaded byte can itself supply an address base. No table interpretation, pointer decoding, or allocation relationship follows from that use.

Logical shifts and masks apply where explicitly represented. Extractions by 34, 36, and 44 bits operate on 64-bit results. Numerical simplifications below assume ordinary truncating 32-bit machine arithmetic where stated.

**documented_only:** signed overflow, signed left shifts, address conversion and wraparound, and unsupported variable-shift counts have no established machine-level semantics here.

### Preconditions and completion

Continuation requires executable indirect targets compatible with the consumed arguments and outputs, returning dependencies where subsequent behavior is expected, and supported accesses at all computed addresses.

No explicit null, bounds, alignment, or executable-target validation is represented. Low-address accesses, including a word read at absolute address 0xa9, are not visibly rejected.

Completion means reaching the final transfer. A return is represented afterward, but whether the target returns is unresolved.

## Behavior

### Invocation phase and retained values

The initial target derives from incoming execution context. Subsequent targets generally derive from represented execution locations plus 4,094 bytes. A common derivation does not establish a common dependency identity.

Invocation ordering is significant. Argument patterns include primary results shifted left by eight; independent transformations of lower and upper result words; values retained across intervening invocations; equal transformed values in several argument positions; explicit zeros; masked values; and separately consumed secondary outputs. Up to four arguments are represented.

Calls without represented arguments, and calls whose primary outputs are unused, still have unconstrained dependency effects. Arithmetic simplification cannot remove those effects or establish global preservation. For example, successive left shifts by 25 and 11 discard all input information under truncating 32-bit arithmetic, but intervening invocations remain observable.

The following descriptive names identify captured quantities, not source identifiers or object types:

| Name | Derivation |
|---|---|
| Shift-derived base | An earlier combined result’s lower word shifted left by 25 |
| Multiplication-derived base | A separately captured primary result multiplied by 33,554,432 |
| Small index | An earlier primary result shifted left by eight, masked to fifteen low bits, then logically shifted right by six |
| Indexed base | Twice the small index |
| Retained shifted value | Another captured primary result shifted left by eight |
| Final combined result | The last combined result before direct storage mutations |

Under ordinary 32-bit operations, the small index ranges from 0 through 508 in increments of four. The indexed base ranges from 0 through 1,016 in increments of eight.

### Opening mutations and cleared-word capture

The final combined result’s upper word supplies a destination base for a word store of the shift-derived base at offset 0x14. Shift-derived offset 9 receives the upper word’s low byte. A word store then places the multiplication-derived base at offset 0x54 from the final lower word multiplied by 256. Shift-derived offset 9 subsequently receives the shift-derived base’s low byte.

Multiplication-derived offset 0x14 is cleared as a word before an unsigned-byte load from its offset 5. Indexed offset 0x14 is cleared before that loaded byte is stored, zero-extended, as a word at multiplication-derived offset 0x34.

The word at multiplication-derived offset 0x14 is then captured. Indexed offset 0x14 is cleared again, followed by two distinct byte stores of the captured word’s low byte at indexed offset 0x15.

Under ordinary byte-addressed memory without external interference or access side effects, the captured word is zero. The intervening indexed clear cannot introduce nonzero bytes through ordinary overlap. The same-base word store at offset 0x34 is disjoint from the cleared word at offset 0x14.

The read remains an observable event. A nonzero observation requires an additional unresolved condition, such as external interference, storage access side effects, or nonordinary address aliasing.

### Global-derived base and saved root

After the duplicate indexed byte stores, the global at absolute address 5 is read and converted to unsigned 32 bits. This capture is the **global-derived base**.

The capture precedes a byte store of the indexed base’s low byte to absolute address 1. Global-derived offset 0x15 then receives the small index’s low byte.

Multiplication-derived offset 0x11 receives the retained shifted value’s low byte. The retained shifted value is stored as a word at offset 0x54 from the previously captured cleared-word value. Multiplication-derived offset 0x11 is then cleared.

The word at global-derived offset 0x54 is captured as the **saved root**, before that location is overwritten with the earlier cleared-word capture. Later behavior separately uses the saved root, four times the saved root, and eight times the saved root.

Unlike bases obtained from explicit unsigned-byte loads, the global-derived base has no established 8-bit bound.

### Indexed destinations versus scaled-root destinations

Two word stores at indexed offset 0x34 have distinct values and retain the same destination-base derivation:

| Relative position | Destination derivation | Stored word |
|---|---|---|
| Earlier | Indexed base plus 0x34 | Small index |
| Later | Indexed base plus 0x34 | Four times the saved root |

The earlier store follows the saved-root capture and replacement. The later store occurs after the first retained-shifted-value store at four-times-saved-root offset 0x54 and before the subsequent indexed offset-0x74 word load used to form a doubled value.

Neither store is established as a mutation at four-times-saved-root offset 0x34. Possible runtime equality or aliasing between bases does not change either store’s address derivation.

The intervening region follows the word at multiplication-derived offset 0x74. That followed base receives a transformed earlier primary value at offset 0x54, then later the retained shifted value there.

Another word store consumes two separate reads of indexed offset 0x74: one supplies its destination base, the other its stored value. Their relative operand-evaluation order is unresolved. A self-value word store at saved-root offset 0x24, followed by a byte store of the saved root’s low byte at the multiplication-followed base’s offset 9, occurs twice, separated by a word store.

Indexed offset 0x14 receives an intermediate shifted word and then two separate stores of the saved root. Absolute address 0x54 receives twice the saved root as a word. Two separate byte stores at indexed offset 0x15 receive the low byte of four times the saved root. Absolute address 0x15 receives the indexed base’s low byte.

### Scaled-root and followed-link mutations

Four-times-saved-root addressing supports mutations at offsets 9, 0x0d, 0x14, 0x15, and 0x54 in this region. Offset 0x34 is not part of that destination list.

A doubled word obtained from indexed offset 0x74 supplies another base and stored value. Saved-root offset 0x44 receives that doubled value.

An explicit unsigned-byte load from four-times-saved-root offset 5 supplies the base of a word load at offset 0x74. The resulting followed base receives the doubled value at offset 0x44, later replaced by a zero-extended byte observation.

Four-times-saved-root offset 0x15 receives several distinct bytes, including the low bytes of eight times the saved root, four times the saved root, and another followed base. Its offset 0x0d receives the first byte-derived followed word’s low byte. Its offset 0x14 receives its own base value and is later replaced with eight times the saved root. Offset 0x54 also receives repeated word updates.

The word at four-times-saved-root offset 0x64 supplies a base receiving a byte at offset 0x0d and a self-value word at offset 0x14. The word at four-times-saved-root offset 4 supplies a separate base whose offset-5 byte is loaded twice, with intervening writes.

The later byte observation supplies another offset-0x74 word load, captured as the **link value**. Twice that value is retained as the **doubled-link base**. Intervening mutations include byte and word updates at offsets 9, 0x14, 0x15, 0x44, and 0x54. Separate observations and overwrites are not interchangeable with final-state assignments.

### Constants and recurring updates

The numeric value 0x3a is stored at three widths:

| Destination | Width |
|---|---:|
| Four times the link value | 8 bits |
| Eight times the link value | 16 bits |
| Four times the link value plus 0x20 | 32 bits |

The **adjusted root** is four times the saved root minus 150. The **work base** is four times the adjusted root.

Work-base offset 0x20 receives four ordered word stores: 0x20, 0x2f, a zero-extended byte observed at work-base offset 4, and 0x20 again. The byte observation occurs after the second store and before the third.

A fixed-location 32-bit read captures the **recurring base** before a word store of 0x20 to absolute address 0x60. Recurring destinations continue using that capture, not refreshed storage.

The recurring region contains explicitly repeated four-store groups:

| Position within each group | Effect |
|---|---|
| First | Word store of four times the link value at a scaled base’s offset 0x54 |
| Second | Byte store of the work base’s low byte at recurring-base offset 0x19 |
| Third | Byte store of the recurring base’s low byte at work-base offset 0x11 |
| Fourth | Word store of the scaled base at recurring-base offset 0x34 |

The exceptional first group uses eight times the saved root as its scaled base. Subsequent groups use eight times the adjusted root.

**documented_only:** this contract does not enumerate the full repetition count. The representation contains explicit repetitions, not a visible runtime loop; that documentation limit does not permit collapsing their access trace.

### Mutation-separated final captures

The final region first captures the word at absolute address 0xa9, then the word at offset 0x74 from four times the link value.

A byte write to absolute address 0x3a follows. Under ordinary truncating arithmetic, its value is the low byte of eight times the adjusted root plus 0x30.

Only afterward are words captured from doubled-link offset 0x14 and work-base offset 4. Hoisting these later captures ahead of the 0x3a write can change their values through overlap.

Doubled-link offset 0x0d then receives the work base’s low byte. Work-base offset 8 receives byte 0x6c; offsets 0x0d and 9 receive the recurring base’s low byte. Doubled-link offset 0x54 receives the doubled-link base as a word.

### Three observations of absolute address 0x67

Absolute address 0x67 receives the low byte of the captured absolute-address-0xa9 word. Doubled-link offset 0x44 is read as a word before the first subsequent observation of 0x67 supplies the byte store to absolute address 0x21.

Before the second observation, the captured offset-0x44 word supplies a byte store at work-base offset 0x10, a word store at recurring-base offset 0x54, and a right-shift-derived byte store at doubled-link offset 1. The shift count is the work base’s low byte. Signed right-shift behavior and unsupported counts remain unresolved.

Recurring-base offset 0x54 is then replaced with the work base. A word is captured from work-base offset 0x40. Recurring-base offset 0x74 receives the earlier work-base offset-4 capture; work-base offset 0x10 receives the full offset-0x40 capture as a word; recurring-base offset 0x44 receives the doubled-link base.

The second observation of 0x67 supplies a byte store at work-base offset 9.

Before the third observation, doubled-link offset 9 receives the work base’s low byte, work-base offset 0x44 receives the absolute-address-0xa9 capture, and a word is captured from work-base offset 0x54. The third observation again supplies a byte store at work-base offset 9.

Each observation uses storage at its own position. Computed writes that alias 0x67, or external interference, can change later observations.

### Remaining mutations and final transfer

The work-base offset-0x54 capture supplies another base. Its offset 0x0d receives the low byte of the earlier four-times-link offset-0x74 capture. Doubled-link offset 0x54 receives the doubled-link offset-0x14 capture taken after the 0x3a write.

A word is captured from work-base offset 0x14. Doubled-link offset 0x11 receives the link-related capture’s low byte. Work-base offset 9 receives the work-base offset-0x54 capture’s low byte, and work-base offset 0x44 receives that full capture.

A word loaded from doubled-link offset 4 supplies the **final followed base**. A halfword store writes 0x65 at twice that base, before its offset-0x64 word is captured.

The earlier work-base offset-0x14 capture supplies a byte store at offset 0x11 from the work-base offset-0x54 capture. That latter base receives decimal 100 as a word at offset 0x54.

The final followed base’s offset-0x74 word and doubled-link offset-0x64 word are captured separately, in that order. Doubled-link offset 9 receives the final followed base’s low byte. The doubled-link offset-0x64 capture supplies a destination at offset 0x54 for the earlier final-followed-base offset-0x64 capture. Twice that earlier capture supplies the final direct word-store destination, receiving the doubled-link offset-0x64 capture.

The final indirect transfer has four represented arguments:

| Position | Value |
|---|---|
| First | Numeric 0x72 |
| Second | Final followed base’s offset-0x64 capture |
| Third | Doubled-link offset-0x14 capture taken after the 0x3a mutation |
| Fourth | Final followed base’s later offset-0x74 capture |

No explicit returned payload is established if the target returns.

## Side effects

Offsets do not identify independent fields. A word at offset 0x14 spans offset 0x15; a word at offset 0x10 spans offset 0x11. Fixed and computed destinations can also overlap. Endianness is unresolved, so overlapping byte values generally require an environmental assumption.

Dependency effects are unconstrained. Global reads are observations at their represented positions, not entry-time snapshots.

No explicit allocation, deallocation, retention, release, or ownership transfer appears. Address-like values are not proven allocated objects. Passing a value to a dependency establishes neither borrowing nor consumption.

There is no represented rollback, error branch, or recovery protocol. Failure to return or an unsupported access can prevent later effects after earlier effects have occurred. Atomicity, idempotence, reentrancy, and thread safety are not established.

## Dependencies

F-1bbae37f570e is the only resolved opaque function ID supplied. No resolved opaque IDs are available for indirect dependencies; none are invented.

Unresolved dependencies comprise the incoming-context-derived initial target, execution-location-derived targets, producers of combined or separately consumed secondary outputs, and the final transfer target.

Their identities, ABI details, storage effects, ownership behavior, failure behavior, and return guarantees remain **documented_only**. No dependency is assumed to preserve globals or own or release memory.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Independent result words.** A combined result with lower word 1 and upper word 2 supplies 256 through the lower-word left shift by eight, independently of forwarding upper word 2.

2. **32-bit truncation.** Under truncating 32-bit arithmetic, successive left shifts by 25 and 11 yield zero. Widening intermediates and retaining discarded bits does not satisfy that interpretation.

3. **Unbounded converted global.** If the absolute-address-5 observation converts to unsigned value 0x1234, its offset-0x15 destination is 0x1249 under ordinary addressing. Restricting the base to its low byte is unsupported. Whether the undeclared global can produce this value remains unresolved.

4. **Cleared-word observation.** Ordinary memory without interference or access effects yields zero at the multiplication-derived offset-0x14 capture. Overlapping zero stores alone do not justify a nonzero result.

5. **Indexed offset-0x34 attribution.** If the small index is 4 and the saved root is 0x100, the indexed base is 8. The two relevant word stores target address 0x3c, first storing 4 and later 0x400. Redirecting the latter store to address 0x434 because its value is four times the saved root violates the established destination derivation. This example assumes supported accesses and ordinary arithmetic.

6. **Byte-derived addressing.** An explicit byte load returning 7 supplies address 0x7b for the subsequent offset-0x74 word access. No table-entry interpretation is established.

7. **Recurring capture.** If the recurring-base capture yields 0x1000, recurring destinations retain that base even if later writes affect its backing storage.

8. **Mutation-separated captures.** If absolute address 0x3a overlaps doubled-link offset 0x14, the later word load observes the preceding byte write under ordinary memory semantics. The absolute-address-0xa9 capture remains earlier.

9. **Separate 0x67 observations.** If an intervening aliased write changes 0x67 from 0x41 to 0x42, the next observation supplies 0x42. A later observation need not match either earlier value.

10. **Intermediate effects.** The four work-base offset-0x20 stores retain their order and intervening byte observation. A single final store does not reproduce that access trace.

11. **Interrupted execution.** If a dependency does not return, later effects are not guaranteed and earlier effects are not specified to be undone.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object types, and fidelity of the supplied representation.
- Indirect dependency identities, calling conventions, and secondary-output lifetimes.
- Source width and signedness of the global at absolute address 5.
- Endianness, alignment support, address mapping, volatility, and storage access effects.
- Signed overflow and shifts, address conversion and wraparound, and unsupported shift counts.
- Actual aliasing, external interference, and whether low addresses represent ordinary memory.
- Whether the final target returns.

The invocation phase and recurring-group count are not exhaustively enumerated here. These documentation limits prevent a complete standalone access-trace implementation contract; they do not weaken the established widths, destination derivations, or capture boundaries.

## Evidence

**Established static evidence:** the supplied representation supports the invocation patterns, explicit access widths, scaled addressing, constants, repeated mutations, and final four-argument transfer described above.

The two relevant offset-0x34 word stores derive their destinations from twice the small index. Their values are, respectively, the small index and four times the saved root. Four-times-saved-root destination mutations in that region use offsets 9, 0x0d, 0x14, 0x15, and 0x54, not offset 0x34. Potential runtime aliasing does not alter this attribution.

The absolute-address-5 conversion establishes an unsigned 32-bit converted value only. The cleared-word interval contains no ordinary intervening write capable of introducing nonzero bytes into that cleared word.

The recurring-base capture precedes the absolute-address-0x60 store. The 0x3a byte mutation separates two earlier captures from two later word loads. Absolute address 0x67 is observed at three separate positions.

**[INFERENCE]:** linking intent remains speculative. Range calculations and arithmetic simplifications depend on the stated machine-arithmetic assumptions.

**Verification boundary:** only the supplied textual representation was analyzed. No tools, independent implementation access, runtime execution, or runtime tests were used. Acceptance examples express conditional obligations, not observed outcomes.
