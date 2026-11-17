<!-- Trusted provenance: F-1aca81bc95af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1aca81bc95af

## Purpose

F-1aca81bc95af performs an ordered chain of indirect invocations, transforms dependency results, mutates fixed and computed memory at several widths, and supplies four explicit arguments to an unresolved continuation.

No application-level purpose is established. Repeated offsets and character-like constants do not establish strings, records, collections, or initialized objects.

This contract distinguishes **established static evidence**, **conditional arithmetic interpretations**, and **documented_only** behavior whose semantics or complete trace are not specified here. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established direct use |
|---|---:|---|
| First argument | 32 bits | No explicit consumption. |
| Second argument | 32 bits | No explicit consumption. |
| Third argument | Signed 32 bits | Supplies an early indirect invocation after a four-bit left shift. |

The lack of direct consumption does not establish that the first two arguments are irrelevant: dependency interfaces and exposure of incoming machine state are incomplete.

No argument is established as a validated address, buffer, size, object reference, or ownership token.

### Implicit inputs and outputs

Execution depends on incoming continuation state, indirect destinations, dependency results, existing memory, and a 32-bit global sampled later as an address base.

Some dependency results occupy represented 64-bit containers with separately consumed low and high words. Whether these are conventional 64-bit returns or combined machine-state results is **documented_only**.

The first indirect destination is incoming continuation state plus 0xFFE. Subsequent destinations use represented code-location anchors with the same displacement. Calls without explicit arguments may consume implicit state.

No explicit return value is established. Observable outputs include dependency effects, ordered memory accesses, mutations, and the final continuation invocation. A return is represented afterward; actual continuation return behavior and the accuracy of that function boundary remain **documented_only**.

### Preconditions and failures

Completion requires executable indirect destinations, dependency-compatible state, and accessible storage for every reached access at its actual width and required alignment.

No local validation, error result, recovery branch, rollback, or cleanup path is represented. Null, low, overlapping, and misaligned addresses are not explicitly rejected. A fault or dependency failure may stop execution after earlier effects.

## Behavior

### Widths, arithmetic, and layout

Bytes, halfwords, and words are respectively 8, 16, and 32 bits. Byte stores retain the low eight bits. Unsigned byte samples widened to words range from 0 through 255. Halfword and word stores affect two and four bytes, even when storing character-like constants.

Most arithmetic and address components are represented at 32 bits. Certain unsigned extractions shift an entire 64-bit container right by 34, 36, or 44 bits before narrowing; these are not shifts of its truncated low word.

**Conditional interpretation:** under 32-bit bit-vector arithmetic, multiplication and left shifts retain only the low 32 bits. Successive shifts can erase every original bit. Signed overflow, signed left shifts, and address wraparound are not established as portable mathematical operations.

Documentary bases below do not imply separate allocations or validated fields. Computed accesses can overlap one another, fixed addresses, or the global’s backing storage. Captures and intermediate writes therefore cannot generally be replaced by final-state assignments.

### Dependency phase and retained values

A long sequential chain of indirect invocations precedes the explicit mutation phase. No local conditional branch or loop is represented.

Explicit arguments include shifted primary results, high words of result containers, transformed secondary results, masked values, unsigned right shifts, zeros, and repeated arguments. The third input is used after its four-bit left shift. Calls remain ordered effects even when their results are discarded.

Complete per-invocation identities, arguments, and secondary-result provenance are **documented_only**; this overview is not a complete dependency-call trace. Similar argument patterns do not establish equal destinations or interchangeable effects. No dependency preservation guarantee is assumed.

The following descriptions identify retained values, not source identifiers:

| Documentary value | Relationship |
|---|---|
| Selector | A late result shifted left eight bits, masked with 0x7FFF, then shifted unsigned right six bits. |
| Doubled selector | Twice the selector. |
| Scaling source | An earlier dependency result used for initial scaling and a later 26-bit shift. |
| Initial scaled base | Scaling source multiplied by 0x02000000. |
| Carried word | An earlier dependency result shifted left eight bits. |
| Carried tag | A retained result-derived value also used as the base for initial plus-0x09 byte stores. |
| Sampled word | A word captured at a byte-derived base plus 0x54. |
| Four-times base | Four times the sampled word. |
| Adjusted quantity | Four-times base minus 0x96. |
| Adjusted base | Four times the adjusted quantity. |
| Late loaded word | A later word captured at a freshly sampled byte-derived base plus 0x74. |
| Working base | Twice the late loaded word. |
| Captured global base | The value retained from the later single read of the global. |

Under the bit-vector interpretation, the selector is a multiple of four from 0 through 508; its double is a multiple of eight from 0 through 1016.

### Initial mutation and fixed byte 0x01

The last result-producing dependency provides a represented 64-bit container. Its high word supplies a destination base for a word store of the carried tag at offset 0x14. Next, the high word’s low byte is stored at carried-tag plus 0x09.

The initial scaled base is stored as a word at 256 times the container’s low word plus 0x54. Carried-tag plus 0x09 then receives the carried tag’s own low byte.

Initial-scaled-base plus 0x14 is cleared before its plus-0x05 byte is sampled. Doubled-selector plus 0x14 is cleared before that unsigned sample is stored as a word at initial-scaled-base plus 0x34.

The word at initial-scaled-base plus 0x14 is then captured. It need not remain zero if intervening accesses overlap it. Another doubled-selector plus-0x14 clear precedes two byte stores of this capture’s low byte at doubled-selector plus 0x15.

Fixed byte 0x05 is sampled and widened unsigned as a base. **After that sample**, fixed byte 0x01 receives the doubled selector’s low byte.

Subsequently, the byte-derived base’s plus-0x15 byte receives the selector’s low byte. Initial-scaled-base plus 0x11 receives the carried word’s low byte; the earlier plus-0x14 capture supplies the base for a carried-word store at offset 0x54; initial-scaled-base plus 0x11 is then cleared.

The byte-derived base’s plus-0x54 word is captured as the sampled word, then overwritten with the earlier plus-0x14 capture. Doubled-selector plus 0x34 receives the selector as a word.

### Fixed word 0x54 and fixed byte 0x15

A word loaded from initial-scaled-base plus 0x74 supplies a base whose plus-0x54 word first receives the scaling source shifted left 26 bits.

A compound operation reads doubled-selector plus 0x74 twice, using one read as the destination base for a plus-0x54 word store and the other as its value. Their internal evaluation order is unresolved; they are not established as one stable sample.

Next come a self-valued word store at sampled-word plus 0x24 and a sampled-word low-byte store at the loaded base plus 0x09. That base’s plus-0x54 word receives the carried word, followed by another occurrence of each preceding self-valued and byte store.

Doubled-selector plus 0x14 receives the scaling source shifted left 26 bits. **Fixed address 0x54 then receives twice the sampled word as a word.** Two subsequent word stores at doubled-selector plus 0x14 each supply the sampled word.

Two byte stores at doubled-selector plus 0x15 supply the four-times base’s low byte. **Only afterward** does fixed byte 0x15 receive the doubled selector’s low byte.

Computed mutation resumes with a carried-word store at four-times-base plus 0x54 and a four-times-base word store at doubled-selector plus 0x34.

### Intermediate computed memory

A word read from doubled-selector plus 0x74, multiplied by two, establishes an **intermediate doubled base**. Four-times-base plus 0x54 receives the four-times base itself.

The next ordering is significant:

- Four-times-base plus 0x15 receives the low byte of eight times the sampled word.
- **Sampled-word plus 0x44 receives the intermediate doubled base as a 32-bit word.**
- Only afterward is the unsigned byte at four-times-base plus 0x05 sampled. That sample supplies the address base for a plus-0x74 word read defining a **loaded destination**.

The sampled-word plus-0x44 store can overlap the subsequent byte sample and thereby change the loaded destination. It is not interchangeable with the later plus-0x44 mutations.

The loaded destination’s plus-0x44 word receives the intermediate doubled base. Four-times-base plus 0x15 then receives the four-times base’s low byte. The loaded destination’s low byte supplies four-times-base plus 0x0D, and four-times-base plus 0x14 receives the four-times base as a word. Four-times-base plus 0x54 receives the four-times base again.

A word loaded at four-times-base plus 0x64 supplies another base: its plus-0x0D byte receives the loaded destination’s low byte, and its plus-0x14 word receives that base itself.

A word loaded at four-times-base plus 0x04 establishes a **byte-sampling base**. The intermediate doubled base’s plus-0x09 byte receives the loaded destination’s low byte, and its plus-0x44 word receives eight times the sampled word.

The first byte sample at byte-sampling-base plus 0x05 precedes an intermediate-doubled-base low-byte store at four-times-base plus 0x09. The unsigned sample then replaces the loaded destination’s plus-0x44 word. Four-times-base plus 0x14 receives eight times the sampled word.

The intermediate doubled base’s plus-0x15 byte receives the byte-sampling base’s low byte, and its plus-0x44 word receives the four-times base. A **second, distinct** byte sample at byte-sampling-base plus 0x05 follows. Four-times-base plus 0x15 receives the low byte of eight times the sampled word before this second sample supplies the plus-0x74 word read defining the late loaded word.

Subsequent updates place the byte-sampling base’s low byte at four-times-base plus 0x15, the four-times base at its plus-0x54 word, the working base’s low byte at four-times-base plus 0x09, and eight times the sampled word at four-times-base plus 0x14.

The constant 0x3A is then stored, in order, as a byte at four times the late loaded word, a halfword at eight times that word, and a word at four times that word plus 0x20.

### Adjusted-base mutation and global capture

Adjusted-base plus 0x20 receives four successive words: 0x20, 0x2F, an unsigned byte sampled at adjusted-base plus 0x04, and 0x20 again. The sample occurs between the second and third stores.

**After all four stores**, the global base is read once. **The fixed word store of 0x20 at 0x60 follows that capture.**

A repeated segment retains that global value. Each repeated relationship orders these effects:

- A scaled base’s plus-0x54 word receives four times the late loaded word.
- Captured-global-base plus 0x19 receives the adjusted base’s low byte.
- Adjusted-base plus 0x11 receives the captured global base’s low byte.
- Captured-global-base plus 0x34 receives the scaled base as a word.

The first scaled base is eight times the sampled word; subsequent repetitions use eight times the adjusted quantity. Exact repetition count remains **documented_only** here, not permission to collapse repeated effects.

### Fixed byte 0x3A and retained captures

After the repeated segment, a word is captured at fixed address 0xA9, followed by a word at four-times-late-loaded-word plus 0x74.

Fixed byte 0x3A next receives the low byte of eight times the adjusted quantity plus 0x30. Only afterward are words captured at working-base plus 0x14 and adjusted-base plus 0x04. The working-base plus-0x14 capture is retained through the final invocation.

Working-base plus 0x0D receives the adjusted base’s low byte. Adjusted-base plus 0x08 receives byte 0x6C; its plus-0x0D and plus-0x09 bytes receive the captured global base’s low byte.

A self-valued word store at working-base plus 0x54 precedes storing the fixed 0xA9 capture’s low byte at fixed byte 0x67.

### Fresh reads of fixed byte 0x67

A signed-represented word is read at working-base plus 0x44. A fresh byte read at 0x67 then supplies fixed byte 0x21.

The signed-represented capture supplies an adjusted-base plus-0x10 byte store and a captured-global-base plus-0x54 word store. Its right shift by the adjusted base’s low eight bits supplies the byte stored at working-base plus 0x01. Negative-operand behavior and counts of at least 32 are unresolved.

Captured-global-base plus 0x54 is overwritten with the adjusted base. The next access order is:

1. Capture the word at adjusted-base plus 0x40.
2. Store the retained adjusted-base plus-0x04 word at captured-global-base plus 0x74.
3. Store the just-captured plus-0x40 word at adjusted-base plus 0x10.
4. Store the working base at captured-global-base plus 0x44.

**The plus-0x74 store is interposed between the plus-0x40 capture and the plus-0x10 replacement.** The destinations may overlap; reversing the two stores can change subsequent observations. The replacement uses the retained plus-0x40 sample, not a reread after the interposed store.

Another fresh read of 0x67 supplies adjusted-base plus 0x09. Working-base plus 0x09 receives the adjusted base’s low byte; adjusted-base plus 0x44 receives the fixed 0xA9 capture.

The word at adjusted-base plus 0x54 is captured as the **newer destination base**, before a third fresh read of 0x67 supplies another adjusted-base plus-0x09 byte store. None of these three fixed-byte reads can generally be replaced with the original 0xA9 capture’s low byte.

### Final captures and continuation

The newer destination’s plus-0x0D byte receives the earlier four-times-late-loaded-word plus-0x74 capture’s low byte. Working-base plus 0x54 receives the retained working-base plus-0x14 capture.

**After that store**, a separate word is captured at adjusted-base plus 0x14: the **final adjusted-field capture**. It precedes the working-base plus-0x11 byte update using the earlier plus-0x74 capture.

Adjusted-base plus 0x09 and plus 0x44 then receive the newer destination base at byte and word widths. A word read at working-base plus 0x04 establishes the **final loaded base**.

A halfword store of 0x65 at twice the final loaded base precedes capturing its plus-0x64 word. Only afterward does the newer destination’s plus-0x11 byte receive the final adjusted-field capture’s low byte. Its plus-0x54 word next receives decimal 100.

Words are then captured at final-loaded-base plus 0x74 and working-base plus 0x64. Working-base plus 0x09 receives the final loaded base’s low byte. The latter plus-0x64 capture supplies a destination whose plus-0x54 word receives the final-loaded-base plus-0x64 capture. A word store at twice that stored value receives the destination base.

The final continuation receives, in order:

1. 0x72.
2. The captured word from final-loaded-base plus 0x64.
3. The retained word from working-base plus 0x14.
4. The captured word from final-loaded-base plus 0x74.

The third argument and final adjusted-field byte source are retained captures, not fresh handoff reads.

## Side effects

Explicit writes affect computed byte, halfword, and word locations, including sampled-word plus 0x44, and fixed storage at 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67.

Offsets do not establish independent fields. Byte order, alignment requirements, atomicity, and device-memory behavior are unresolved. No synchronization, isolation, rollback, thread-safety, reentrancy, or idempotence guarantee is established.

No explicit allocation or release is identified in the memory phase. Dependencies may allocate, release, retain references, or change lifetimes. Ownership and continued address validity are **documented_only**.

## Dependencies

| Opaque function ID | Documentary role | Unresolved contract |
|---|---|---|
| F-unknown-01 | Initial continuation-relative invocation | Destination, implicit inputs, results, effects. |
| F-unknown-02 | Subsequent anchor-relative invocations | Individual identities, interfaces, secondary results, memory and global effects. |
| F-unknown-03 | Final indirect continuation | Destination, ownership, failure and return behavior. |

These IDs describe categories, not proof of one actual function per category. Unknown callees are not assumed to preserve globals or memory, or to own or release referenced storage.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input width:** under 32-bit bit-vector shifting, third input 3 supplies 48 at its shifted use; 0x10000000 supplies zero.

2. **Selector boundary:** selector source 0x7F produces selector 508 and doubled selector 1016. Fixed bytes 0x01 and 0x15 receive 0xF8 at their separate ordered positions. Source 0x80 produces selector zero.

3. **Unsigned widening:** byte sample 0xE1 supplies word 0x000000E1, not a sign-extended value.

4. **Fixed scaling:** sampled word 0x80000001 supplies 0x00000002 to fixed word 0x54 under modular arithmetic.

5. **Intermediate overlap:** if sampled-word plus 0x44 overlaps four-times-base plus 0x05, the subsequent byte sample observes the intervening word store’s effect under the applicable byte order. Sampling before that store is not equivalent.

6. **Global retention:** if the fixed 0x60 store alters the global’s backing bytes, the repeated segment still uses the earlier captured base.

7. **Interposed stores:** if captured-global-base plus 0x74 equals adjusted-base plus 0x10, the retained plus-0x04 word is stored first, then replaced with the retained plus-0x40 word. Absent further overlapping writes, the latter remains.

8. **Fixed-byte conversion:** adjusted quantity 0x21 supplies 0x38 to fixed byte 0x3A before the two following word captures.

9. **Fresh fixed samples:** a computed write overlapping 0x67 can change a later adjusted-base plus-0x09 store. Each such update uses its own fixed-byte sample.

10. **Final retention:** final adjusted-field capture 0x123456E1 supplies byte 0xE1 to newer-destination plus 0x11, despite later changes to its original field. Capture occurs after working-base plus 0x54 is updated; consumption occurs after the 0x65 halfword store and plus-0x64 capture, before the decimal-100 store.

11. **Unresolved operations:** negative right-shift operands and oversized counts have no uniquely prescribed result here. An inaccessible destination does not require rollback or a clean error return.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention, dependency identities, per-call trace, and function boundaries.
- Secondary-result origins and interpretation of 64-bit result containers.
- Signed overflow, signed shifts, oversized counts, and target address arithmetic.
- Byte order, alignment, atomicity, external memory behavior, and concurrency.
- Address validity, allocation boundaries, ownership, and lifetimes.
- Exact repeated-store multiplicity and evaluation order within compound nested reads.
- Fault delivery, resumption, and continuation return behavior.

**Inferred intent:** result transformation, memory mutation, and continuation transfer. No higher-level application semantics are adopted.

## Evidence

The supplied static body supports the represented widths, value relationships, access ordering, retained captures, and final invocation described above.

In particular, it establishes both the sampled-word plus-0x44 word store before the subsequent byte-derived destination read, and the captured-global-base plus-0x74 store between the adjusted-base plus-0x40 capture and plus-0x10 replacement. Their positions are observable under overlap.

Prior prose, reviewer feedback, comments, and warnings are not independent behavioral authority. No snapshot identifier is asserted. No dependency inspection, machine-level validation, runtime observation, or runtime testing has been performed.
