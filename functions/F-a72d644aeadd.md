<!-- Trusted provenance: F-a72d644aeadd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a72d644aeadd

## Purpose

F-a72d644aeadd performs a finite, ordered chain of indirect calls, transforms represented results and implicit entry values, and then accesses fixed and computed memory locations using mixed widths. It ends with an unresolved indirect transfer represented as a call.

No application-level purpose is established. Constants resembling characters and recurring address displacements do not establish text processing, object initialization, a container, or a protocol.

This contract distinguishes:

- **Established static evidence:** represented calls, explicit access widths, constants, captured-value relationships, and ordering.
- **Conditional deductions:** consequences requiring the stated arithmetic or memory assumptions.
- **documented_only:** behavior insufficiently resolved here for an executable acceptance criterion.

The nominal interface does not establish a portable, independently callable routine.

## Interface

### Inputs and entry state

No explicit parameters are declared. Execution nevertheless consumes:

- A represented 32-bit entry value shifted left seven positions and supplied twice to the first call.
- Another represented 32-bit entry value shifted left three positions.
- An entry control-address value whose sum with `0xffe` determines the first indirect destination.

These are implicit machine-state inputs, not established conventional arguments.

Later calls consume ordinary results, combined 64-bit results, and secondary machine-result values. The calling convention and validity of secondary results remain **documented_only**.

The direct-access phase also consumes fixed storage associated with address `0x05`. The representation converts that global value to an unsigned 32-bit value. Its declaration is absent, and the operation is not an explicit byte dereference. **Its actual load width and original signedness are unresolved.** The converted value is not established to lie between zero and 255.

### Outputs and completion

There is no declared return value. Observable behavior includes dependency invocations, arguments, dependency effects, direct memory operations, and the final transfer.

The final transfer receives four represented arguments:

1. `0x72`.
2. A word captured from a late-computed base plus `0x64`.
3. A word captured earlier from the tail base plus `0x14`.
4. A later word captured from the same base as argument two, at offset `0x74`.

Argument two also determines a doubled destination address for the final direct word store. It is not an established status code.

A return follows in the representation. Whether the final transfer returns, branches permanently, or uses another continuation convention is **documented_only**.

### Widths, arithmetic, and layout

Explicit byte, halfword, and word accesses represent 8, 16, and 32 bits respectively. Stores of narrower values retain the corresponding low bits. Explicit unsigned byte loads are zero-extended when widened. These rules do not establish the width or signedness of the undeclared global at `0x05`.

Ordinary arithmetic and computed addresses are represented with 32-bit values. Combined results have 64 bits; their high portion means bits 32–63, independently of memory endianness. Logical shifts of a combined result by 34, 36, and 44 select its high portion shifted right by two, four, and twelve.

Signed-looking shifts, multiplication, and subtraction do not establish portable overflow semantics. Arithmetic bounds and equivalences below are conditional on **32-bit truncating bit-vector arithmetic**. In that model, address arithmetic wraps modulo \(2^{32}\).

Offsets identify access displacements, not proven object fields. No complete structure layout, allocation size, endianness, alignment capability, or memory mapping is established.

### Preconditions

Every operation requires preceding calls to return and preceding accesses to complete. Full execution requires compatible implicit entry state, executable indirect destinations, usable results, and accessible storage supporting the represented widths.

No visible validation establishes these conditions.

## Behavior

### Indirect-call phase and retained values

The first call presents the first numeric entry value shifted left seven positions in argument positions one and two, and the second entry value shifted left three positions in position three.

Two more calls follow; the third supplies a result subsequently shifted left eight positions and forwarded. Much of the remaining chain derives destinations from represented code-location values plus `0xffe`, including locations represented as code addresses.

The chain includes calls without represented arguments, discarded ordinary results, duplicate shifted arguments, explicit zeros, retained results reused after intervening calls, split combined results, masks, logical right shifts, and secondary machine results. Each call remains a separate event. Discarded results do not establish purity, and similar surrounding arithmetic does not establish dependency identity.

This is not a represented loop, retry policy, or search. The complete intermediate call-by-call argument trace is **documented_only** here; the description does not authorize removing, merging, or reordering calls. Dependencies may access memory throughout.

For the direct-access phase, the following descriptive names identify retained values:

| Name | Meaning |
|---|---|
| **Anchor** | A call result multiplied by `0x02000000`. |
| **Expanded anchor** | That same originating result shifted left 26 positions. |
| **Shifted base** | A different retained result shifted left 25 positions. |
| **Selector** | A result shifted left eight positions, masked with `0x7fff`, then logically shifted right six positions. |
| **Doubled selector** | Twice the selector. |
| **Payload** | A separately captured result shifted left eight positions. |
| **Final low / final high** | Low and high words of the last combined result before direct accesses. |

Under the conditional arithmetic model, selector ranges from zero through 508 in multiples of four. Doubled selector plus `0x14` therefore ranges from `0x14` through `0x40c` in increments of eight. Expanded anchor equals twice anchor in that model; the distinct expressions are retained because their signed-looking arithmetic is otherwise unresolved.

Other conditional deductions include: an eight-position left shift followed by an unsigned eight-position right shift retains the original low 24 bits; successive left shifts totaling at least 32 positions erase all bits; and shifted base has zero low 25 bits.

### Opening direct accesses

The opening effects are ordered as follows:

- A word store places shifted base at final high plus `0x14`. A byte store then places the low byte of final high at shifted base plus nine.
- A word store places anchor at 256 times final low plus `0x54`. The byte at shifted base plus nine is then overwritten with the low byte of shifted base—conditionally zero.
- A zero word is stored at anchor plus `0x14`, followed by an unsigned byte read at anchor plus five. A zero word is then stored at doubled selector plus `0x14`, and the captured byte is widened and stored as a word at anchor plus `0x34`.
- Only afterward is the word at anchor plus `0x14` captured as **anchor readback**. Another zero word store at doubled selector plus `0x14` follows, then two distinct byte stores place the low byte of anchor readback at doubled selector plus `0x15`.

Anchor readback cannot unconditionally be replaced with zero: intervening accesses may overlap, and storage semantics are unknown.

### Fixed-global conversion and seed capture

The global associated with fixed address `0x05` is evaluated and converted to unsigned 32 bits. Call the converted, captured value the **global-derived base**. This name deliberately imposes no byte-sized range.

The next fixed byte store, at `0x01`, receives the low byte of doubled selector. Subsequent byte stores place the low byte of selector at global-derived base plus `0x15`, and the low byte of payload at anchor plus `0x11`.

A word store then places payload at anchor readback plus `0x54`; the byte at anchor plus `0x11` is subsequently replaced with zero. **The access through anchor readback plus `0x54` is a store, not a word read.** No loaded result from that address is established.

The word at global-derived base plus `0x54` is then captured as **seed**, before that same nominal location is overwritten with anchor readback. Thus seed is the value observed after all preceding stores, including any aliases, but before this overwrite.

A word store at doubled selector plus `0x34` next receives selector.

### Mutation through selector-derived and seed-derived addresses

A fresh word read at anchor plus `0x74` supplies the **replacement base**. Its offset `0x54` receives expanded anchor.

The following word store uses two separate reads of doubled selector plus `0x74`: one supplies the destination base, whose offset `0x54` is written, and the other supplies the stored word. Their relative evaluation order is unresolved; neither is established as a shared snapshot.

The next effects preserve this sequence:

- Seed is stored as a word at seed plus `0x24`; replacement base plus nine receives seed’s low byte; replacement base plus `0x54` receives payload.
- Those first two seed stores recur: seed at seed plus `0x24`, then its low byte at replacement base plus nine.
- Doubled selector plus `0x14` receives expanded anchor. Fixed address `0x54` then receives the 32-bit value **seed shifted left one position**.
- Doubled selector plus `0x14` receives seed twice as separate word stores. Its offset `0x15` then receives the low byte of four times seed twice as separate byte stores.
- Fixed address `0x15` receives the low byte of doubled selector.

Define **scaled seed** as four times seed, and **eightfold seed** as seed shifted left three positions. Multiplication by four and shifting left two positions are equivalent only under the conditional arithmetic model; stores explicitly derived by shifting are identified below.

Scaled seed plus `0x54` first receives payload. Doubled selector plus `0x34` receives seed shifted left two positions. A fresh word from doubled selector plus `0x74`, doubled arithmetically, becomes the **bridge base**.

Scaled seed plus `0x54` is then overwritten with scaled seed. Its offset `0x15` receives the low byte of eightfold seed, and seed plus `0x44` receives bridge base.

An explicit unsigned byte read at scaled seed plus five selects a word-read address by adding `0x74` to that byte’s numeric value. The resulting word is the **first selected word**. Its offset `0x44` receives bridge base.

Next, scaled seed plus `0x15` receives the low byte of scaled seed; scaled seed plus `0x0d` receives the low byte of first selected word; scaled seed plus `0x14` receives scaled seed; and scaled seed plus `0x54` receives seed shifted left two positions.

### Fresh selection and transition to the tail bases

The word at scaled seed plus `0x64` is now captured as the **intermediate base**. Its offset `0x0d` receives the low byte of first selected word, and its offset `0x14` receives intermediate base itself.

A later word read at scaled seed plus four supplies the **lookup base**. Bridge base plus nine receives the low byte of first selected word; bridge base plus `0x44` receives eightfold seed.

An unsigned byte from lookup base plus five is captured. Between this read and its use as a word value, scaled seed plus nine receives the low byte of bridge base. First selected word plus `0x44` then receives the captured byte, zero-extended.

Scaled seed plus `0x14` receives eightfold seed. Bridge base plus `0x15` receives the low byte of lookup base, and bridge base plus `0x44` is overwritten with scaled seed.

A **fresh** unsigned byte read from lookup base plus five follows those stores. Scaled seed plus `0x15` then receives the low byte of eightfold seed. A word read at the newly captured byte’s numeric value plus `0x74` produces the **selected word**. It need not equal first selected word.

Before the constant-store region, scaled seed plus `0x15` receives lookup base’s low byte; scaled seed plus `0x54` receives seed shifted left two positions; scaled seed plus nine receives the low byte of twice selected word; and scaled seed plus `0x14` receives eightfold seed.

These reads and writes cannot be replaced with a final-memory-image description: aliasing can change the later lookup results and destinations.

### Constants and repeated stores

Define:

- **Adjusted seed:** scaled seed minus `0x96`.
- **Working base:** four times adjusted seed.
- **Tail base:** twice selected word.
- **Repeated payload:** four times selected word.

A byte `0x3a` is stored at repeated payload. Adjusted-seed arithmetic occurs before a halfword `0x3a` store at eight times selected word. Repeated payload plus `0x20` then receives word `0x3a`.

Working base plus `0x20` receives word stores in this order: `0x20`, `0x2f`, an unsigned byte freshly read from working base plus four and widened, then `0x20`.

A represented 32-bit value from fixed code-region storage is captured as **dispatch base before** fixed address `0x60` receives the represented word value `0x20`. Subsequent operations use the captured dispatch base, not a reread of its source.

The first repeated-store group has four ordered effects:

1. Repeated payload as a word at eight times seed plus `0x54`.
2. Working base’s low byte at dispatch base plus `0x19`.
3. Dispatch base’s low byte at working base plus `0x11`.
4. Eight times seed as a word at dispatch base plus `0x34`.

Later groups retain the word–byte–byte–word order but use eight times adjusted seed for the scaled destination and the final stored word. Dispatch base and both byte values remain captured throughout.

Exact group multiplicity remains **documented_only** in this contract. The representation establishes many distinct finite groups; this limitation does not permit collapsing them to one group.

### Tail captures and fixed-byte rereads

The tail first captures a represented word from fixed address `0xa9`, then a word from repeated payload plus `0x74`. Call them **fixed captured word** and **selected captured word**.

Fixed byte address `0x3a` receives the low eight bits of eight times adjusted seed plus `0x30`. Only afterward are words captured from tail base plus `0x14` and working base plus four, becoming **saved tail word** and **saved working word**.

Subsequent byte stores place working base’s low byte at tail base plus `0x0d`, `0x6c` at working base plus eight, and dispatch base’s low byte at working base plus `0x0d` and then plus nine. Tail base plus `0x54` receives tail base as a word.

The low byte of fixed captured word is written to `0x67`. A word from tail base plus `0x44` is then captured as **shift operand**. Only after that read does a fresh evaluation of fixed byte storage at `0x67` supply the byte written to `0x21`.

Working base plus `0x10` receives shift operand’s low byte; dispatch base plus `0x54` receives the complete shift operand. Tail base plus one receives a byte derived from a signed-looking right shift of shift operand, with a count equal to working base’s low eight bits. Dispatch base plus `0x54` is then overwritten with working base.

A word from working base plus `0x40` is captured. Dispatch base plus `0x74` receives saved working word; working base plus `0x10` receives the newly captured word; dispatch base plus `0x44` receives tail base.

A second fresh read of `0x67` supplies a byte store at working base plus nine. Tail base plus nine then receives working base’s low byte, and working base plus `0x44` receives fixed captured word.

A word from working base plus `0x54` becomes **late base**. A third fresh read of `0x67` subsequently supplies another byte store at working base plus nine. Equality among the three fixed-byte read results is not guaranteed.

### Final stores and transfer

Late base plus `0x0d` receives selected captured word’s low byte. Tail base plus `0x54` receives saved tail word.

A word from working base plus `0x14` is captured **after** those stores. Tail base plus `0x11` then receives selected captured word’s low byte; working base plus nine receives late base’s low byte; working base plus `0x44` receives late base.

A word from tail base plus four becomes **final object base**. A halfword `0x65` is stored at twice final object base. Only afterward is **final transfer value** captured from final object base plus `0x64`.

The earlier capture from working base plus `0x14` supplies a byte at late base plus `0x11`. Late base plus `0x54` then receives word decimal 100.

A fresh word from final object base plus `0x74` supplies the fourth final argument. A word from tail base plus `0x64` becomes **final destination base**.

The last direct effects are ordered: final object base’s low byte at tail base plus nine; final transfer value as a word at final destination base plus `0x54`; final destination base as a word at twice final transfer value.

The unresolved transfer then receives `0x72`, final transfer value, saved tail word, and the captured fourth argument.

## Side effects

Direct accesses are not confined to caller-provided storage. Computed addresses can overlap other computed locations, fixed globals, executable storage, or device registers. The represented word capture at fixed address `0xa9` uses an odd address.

Access width and order matter independently of final contents:

- Captured values are retained, not implicit rereads.
- Explicit rereads observe storage at their own positions in the sequence.
- Duplicate stores remain distinct events.
- Narrow stores can affect later wider reads.
- Read-sensitive storage cannot be excluded.

No atomicity, synchronization, reentrancy, or thread-safety guarantee is established.

No identifiable allocation or release operation is established. Unknown dependencies may allocate, retain, transfer, or release memory; no ownership or lifetime preservation is assumed.

There is no represented validation branch, rollback, cleanup path, or explicit error result. Faults and nonreturning dependencies can leave earlier effects in place.

## Dependencies

The supplied identity map establishes only **F-a72d644aeadd**. No opaque IDs are available for its indirect dependencies; none are invented.

Unresolved dependencies include the entry-derived first destination, code-location-derived destinations, and final transfer destination. Their identities, signatures, calling conventions, global preservation, memory effects, ownership effects, error behavior, and continuation behavior are **documented_only**.

## Acceptance examples

These are conditional obligations, not executed tests.

1. **Initial arguments:** With numeric entry values one and two, the first call presents 128, 128, and 16, provided destination execution is valid.

2. **Truncation:** Under 32-bit bit-vector arithmetic, shifting `0x01000000` left eight positions yields zero.

3. **Undeclared global:** If the conversion of the global at `0x05` yields `0xfffffffe`, global-derived addressing uses that converted value, not an assumed byte value of 254. Whether the global can yield it is unresolved.

4. **Explicit byte selection:** An explicit unsigned byte read of `0xfe` at lookup base plus five selects word address `0x172`. This does not characterize the undeclared global at `0x05`.

5. **Fixed stores:** If selector is `0x1fc` and seed is three, fixed address `0x01` receives byte `0xf8`; later fixed address `0x54` receives word six; still later fixed address `0x15` receives byte `0xf8`. Intervening accesses remain required.

6. **Seed snapshot:** If global-derived base plus `0x54` contains a particular word when seed is captured, the later overwrite with anchor readback does not retroactively change seed. Earlier aliasing stores may affect the captured value.

7. **Fresh lookup:** If intervening writes change lookup base plus five, the later selection uses the fresh byte, not the byte previously stored as a widened word through first selected word.

8. **Dispatch capture:** Changing dispatch-base source storage after its capture does not change the retained dispatch base.

9. **Tail interleaving:** If the byte store at `0x3a` changes storage included in the later saved-tail-word read, that capture reflects the intervening write according to the actual memory model.

10. **Independent fixed-byte reads:** Different results from the three reads of `0x67` supply their respective destinations independently.

11. **Partial execution:** A nonreturning dependency prevents later operations. No compensating cleanup is established.

12. **Final argument retention:** The third final argument remains saved tail word even if subsequent stores alter its original source location.

## Uncertainties

The following remain **documented_only**:

- Function-boundary accuracy and the nominal zero-argument interface.
- Application intent and dependency contracts.
- The undeclared global’s load width and original signedness at `0x05`.
- Signed arithmetic, overflow, and oversized runtime shift-count behavior.
- Endianness, alignment, mappings, device effects, and fault handling.
- Relative ordering of the two reads within the self-derived word-store expression.
- The complete intermediate call trace and exact repeated-group multiplicity specified here.
- Ownership, lifetime, termination, and final continuation semantics.

Unknown aliasing does not erase established ordering. Unknown callees do not imply preserved globals or retained ownership.

## Evidence

This contract uses only the supplied static representation and identity map. Prior prose, comments, and reviewer feedback are not independent behavioral evidence.

The direct-access relationships distinguish explicit unsigned byte dereferences from the undeclared global conversion at `0x05`. The mutation description preserves the seed capture before overwrite, fixed-store data relationships, intervening scaled-seed mutations, and fresh lookup reads. The access at anchor readback plus `0x54` is established as a payload store, not a read.

Arithmetic bounds and equivalences are conditional deductions under the stated 32-bit model. Application semantics remain unestablished.

No implementation access, runtime execution, instrumentation, or tests have been performed.
