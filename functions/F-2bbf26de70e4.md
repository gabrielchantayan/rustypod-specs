<!-- Trusted provenance: F-2bbf26de70e4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2bbf26de70e4

## Purpose

F-2bbf26de70e4 performs a dependency-driven sequence of packed-value transformations, ordered memory mutations, and a final unresolved invocation carrying retained memory snapshots.

**Established static evidence:** the supplied representation accepts three 32-bit arguments, forwards the first argument to its opening invocation, extracts bits from the third argument, and later accesses memory at fixed and computed addresses. Access widths include 8, 16, and 32 bits. Several reads capture values that remain in use after their source locations may have changed.

**Inferred intent:** recurring masks, scaling operations, and offsets suggest packed state and related memory structures. They do not establish an application-level purpose or complete object layout.

This contract does not classify the operation as a parser, formatter, allocator, serializer, or initializer. It describes static constraints, not demonstrated executable behavior.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | Passed unchanged as the opening invocation’s sole explicit argument. |
| Second argument | 32 bits | No explicit consumption appears in the supplied body. |
| Third argument | Unsigned 32 bits | Bits 4–13 supply a field-derived argument to an early invocation. |

The first argument has no established pointer, handle, or ownership interpretation. It is not visibly dereferenced before forwarding.

The second argument’s apparent nonuse does not prove that it is unobservable under the original calling convention. Implicit register inputs and invocation conventions are incomplete.

### Implicit inputs and preconditions

Execution also depends on incoming control-flow context, indirect targets, dependency results, register-derived values, and accessible memory.

Normal progression requires compatible executable targets and valid storage for every reached access, including its full width. Applicable alignment, address-arithmetic, and byte-order rules must also permit those accesses. No visible validation enforces these conditions.

The three explicit arguments alone do not determine the operation.

### Outputs and completion

No return value, status code, output buffer, or output length is declared. Observable outputs consist of invocations and memory effects.

The final unresolved invocation receives four explicit 32-bit values:

| Position | Value |
|---|---|
| First | 0x72 |
| Second | A late snapshot from offset 0x64 of a memory-loaded base. |
| Third | An earlier snapshot from offset 0x14 of the terminal linked base. |
| Fourth | A later snapshot from offset 0x74 of the same loaded base used for the second argument. |

These arguments are retained values, not reads performed at invocation time. Their capture points differ.

A return is represented afterward. Whether the final invocation returns normally, redirects execution, or follows another convention remains **documented_only**.

## Behavior

### Execution and arithmetic model

The visible body contains no explicit loop, conditional branch, input rejection, or early return. Progression is sequential, provided invocations return and accesses complete. Dependencies may introduce additional control flow.

Most values are 32 bits. Some results occupy 64-bit containers whose low and high 32-bit halves are consumed separately. Byte and halfword stores retain the low 8 and 16 bits, respectively; byte reads used as word values are zero-extended.

Unsigned 32-bit shifts discard bits beyond bit 31. Intermediate truncation matters: a left shift followed by a right shift need not recover the original value. Unsigned 64-bit right shifts by 32, 34, 36, and 44 extract the high half shifted right by zero, two, four, and twelve bits, respectively.

Some scaling, shifts, and address additions are represented with signed 32-bit operands. Signed overflow, left shifts involving negative values, and negative signed right shifts are not fully resolved. A wrapping machine interpretation must not be silently substituted for portable signed arithmetic.

A terminal signed right shift takes its count from the low eight bits of an address-related value. No visible restriction limits that count to 0–31. Oversized counts remain **documented_only**.

### Opening invocations

The opening invocation receives the first argument unchanged. Two invocations without explicit arguments follow. The latter supplies a 32-bit result to an invocation receiving:

- That result shifted left by eight.
- Bits 4–13 of the third input relocated to bits 18–27.

The field-derived value ranges from zero through 0x0FFC0000, in increments of 0x00040000. Other bits of the third argument do not contribute to this particular argument; they are masked rather than rejected.

The long subsequent invocation region uses zero through four explicit arguments. Its transformations include masks, left and right shifts, duplicated values, zero arguments, and low/high result-half extraction. Some saved values survive across intervening invocations.

No-argument invocations and invocations with ignored ordinary results remain potentially consequential. Neither locally zero arithmetic nor unused results justify eliminating those calls. Their memory effects and ownership behavior are unknown.

### Address roles

The following names describe dataflow, not object types:

- **Opening scaled base:** a dependency-derived value scaled by 0x02000000, used by the opening clears and byte reads.
- **Small base:** twice an early masked-and-shifted dependency-derived value, used for repeated accesses at offsets 0x14 and 0x15.
- **Fixed-read base:** the unsigned 32-bit value captured from fixed address 0x05.
- **Retained word:** the 32-bit word captured from fixed-read-base offset 0x54 before that location is overwritten.
- **Working base:** the computed base used by the later repeated offset-0x11 byte stores.
- **Fixed-state base:** the 32-bit value captured from fixed address 0x087FBBBC.
- **Terminal linked base:** a doubled, memory-derived address used near completion at offsets including 0x0D, 0x14, 0x44, and 0x54.

The working base is formed by scaling the retained word by four, subtracting 0x96, and scaling the result by four again. Equivalence to alternative algebraic forms is conditional on the arithmetic model.

### Opening memory effects

Direct memory mutation begins after the last invocation in the long transformation region.

A word store through the latest paired result’s high half at offset 0x14 precedes a byte store at another computed base’s offset 9. A word store through the paired result’s low half scaled by 256, at offset 0x54, precedes another byte store at that offset-9 destination.

The opening scaled base’s offset-0x14 word is cleared before its offset-5 byte is read. A clear at small-base offset 0x14 then precedes a word store of the zero-extended byte at opening-scaled-base offset 0x34.

The subsequent word read from opening-scaled-base offset 0x14 occurs after intervening writes. It cannot be assumed to return zero without excluding aliasing.

Further clears and duplicate byte stores remain distinct represented effects.

### Fixed read at 0x05 and retained-word capture

The fixed-address read at 0x05 occurs before the byte store at fixed address 0x01. Its loaded value is converted to unsigned 32 bits and saved as the fixed-read base. The 0x01 store receives the small base’s low byte.

Later, fixed-read-base offset 0x15 receives the low byte of the value whose doubling formed the small base. Other computed writes intervene before the word at fixed-read-base offset 0x54 is captured as the retained word.

That offset-0x54 location is then overwritten with the previously captured opening-scaled-base offset-0x14 word. Subsequent calculations use the retained word, not a reload of the overwritten location.

Neither the 0x01 store nor other intervening writes cause the representation to reread 0x05. The original storage width of the symbol at 0x05 is absent; its read and conversion to 32 bits are established.

### Fixed stores at 0x54 and 0x15

These are separate global mutations, not merely computed accesses with matching offsets.

**Fixed address 0x54 receives a 32-bit store of the retained word shifted left by one.** The shift is represented with a signed operand, so its meaning for negative or overflowing values remains subject to the signed-arithmetic uncertainty above.

Its ordering is established:

- It follows a computed word store at small-base offset 0x14 containing a dependency-derived value shifted left by 26.
- It precedes two further computed word stores of the retained word at that same small-base offset 0x14.
- Those stores precede two byte stores at small-base offset 0x15, each containing the low byte of four times the retained word.

**After those computed writes, fixed address 0x15 receives the small base’s low byte.** This is the low byte of the doubled computed base used by the adjacent offset-0x15 byte stores. It is not the byte value those two stores write, and it is not a read from either computed destination.

Both fixed stores may affect later accesses if computed destinations or sources overlap their storage. The retained word itself remains a saved value even if its original source is subsequently changed.

### Chained updates and constant stores

The following region combines snapshots, overwrites, byte-derived addresses, and offsets including 4, 0x14, 0x24, 0x34, 0x44, 0x54, 0x64, and 0x74.

Scaled forms of the retained word serve as address bases and stored values. Reads through offset 0x74 and through zero-extended byte-derived bases establish further linked addresses. Saved words remain available after their source locations change.

Some expressions contain more than one access to the same computed offset. Independent operand-evaluation order within a single expression is not established. Such accesses must not be assumed collapsible where storage observability matters.

A later group stores 0x3A through three widths: a byte at one scaled address, a halfword at another, and a word at offset 0x20 of the first scaled base.

At working-base offset 0x20, four word stores occur in order: 0x20, 0x2F, the zero-extended byte read from working-base offset 4, and 0x20 again. The byte read occurs after the 0x2F store and before its value is stored. These constants do not establish character-output semantics.

### Fixed-state capture and repeated updates

After the offset-0x20 sequence, a 32-bit read at 0x087FBBBC captures the fixed-state base. It precedes the 32-bit store of 0x20 at fixed address 0x60 and all repeated update groups.

The ranges 0x60–0x63 and 0x087FBBBC–0x087FBBBF are distinct. No alternative mapping between them is established.

Each repeated group stores, in order:

- The working base’s low byte at fixed-state-base offset 0x19.
- The fixed-state base’s low byte at working-base offset 0x11.
- A scaled 32-bit value at fixed-state-base offset 0x34.

The first group’s last value is eight times the retained word. Later groups use eight times the intermediate value obtained after quadrupling the retained word and subtracting 0x96.

Computed word stores at offset 0x54 precede and separate groups. Their stored value is a saved scaled memory-derived base. The first such destination uses a different scale source from the later repeated destination.

The fixed-state base is not reread between groups. If a later computed destination overlaps 0x087FBBBC, subsequent groups still use the captured base.

This is repeated mutation of the same represented destinations, not traversal of successive records. Individual represented accesses must not be reduced to one final write when accesses are observable. Correspondence to original instruction multiplicity remains **documented_only**.

### Terminal snapshots and fixed-address copies

Immediately after the repeated region, a 32-bit read at fixed address 0xA9 is captured. A word at offset 0x74 of another previously scaled base is captured next. Only afterward does fixed address 0x3A receive a byte derived from the earlier intermediate value multiplied by eight and increased by 0x30.

That early offset-0x74 snapshot supplies later byte stores. It is not the final invocation’s fourth argument.

After the 0x3A store, two more word snapshots are captured:

- Terminal-linked-base offset 0x14, retained for the final invocation’s third argument.
- Working-base offset 4, later stored at fixed-state-base offset 0x74.

These captures precede terminal mutations including byte stores at terminal-linked-base offset 0x0D and working-base offsets 8, 9, and 0x0D. Working offset 8 receives 0x6C. A word store places the terminal linked base at its own offset 0x54.

The accesses to fixed address 0x67 are temporally distinct:

| Access | Ordering and value |
|---|---|
| Initial byte store | Receives the low byte of the saved 0xA9 word. |
| Copy to fixed 0x21 | Uses a fresh byte read from 0x67, after a word read at terminal-linked-base offset 0x44. |
| First copy to working offset 9 | Uses another fresh byte read after further computed accesses. |
| Second copy to working offset 9 | Uses another fresh read after a byte store at linked offset 9, a word store of the saved 0xA9 value at working offset 0x44, and a word read at working offset 0x54. |

Computed writes may overlap 0x67. These copies therefore need not agree with one another or with the initially stored byte.

Other terminal effects include a working-offset-0x40 word read and a later word store of that value at working offset 0x10. The saved working-offset-4 word is stored at fixed-state offset 0x74 before the first computed copy from 0x67.

### Final argument capture and remaining mutations

The earlier linked-offset-0x14 snapshot is written to linked offset 0x54 and remains retained for the third argument.

A word read at linked offset 4 supplies the late loaded base. A halfword store of 0x65 through twice that base precedes the word read at loaded-base offset 0x64. That word becomes the final invocation’s second argument.

Further byte and word stores follow, including a word store of decimal 100 at offset 0x54 through another captured base. The word at late-loaded-base offset 0x74 is then captured for the fourth argument.

Effects still follow this final capture:

- A word read at terminal-linked-base offset 0x64.
- A byte store at terminal-linked-base offset 9 containing the late loaded base’s low byte.
- A word store of the second-argument snapshot at offset 0x54 through the newly read base.
- A word store of that newly read base through twice the second-argument snapshot.

Even if these writes overlap snapshot sources, the final invocation receives the captured values.

## Side effects

### Widths, layout, and overlap

Offsets establish access locations, not complete records.

| Width | Established access classes |
|---|---|
| 8 bits | Computed byte fields and fixed stores at 0x01, 0x15, 0x21, 0x3A, and 0x67; fresh reads at 0x67. |
| 16 bits | Scaled-address stores of 0x3A and 0x65. |
| 32 bits | Computed word fields; fixed stores at 0x54 and 0x60; fixed reads at 0xA9 and 0x087FBBBC. |

The fixed read at 0x05 has unresolved original storage width.

Byte and word views may overlap. For example, offset 0x15 lies within the four-byte span beginning at offset 0x14. Endianness affects the numeric consequence, not the overlap itself.

No atomic record replacement is established. Fixed-address storage is not identified as ordinary memory, device storage, or invalid storage.

### Ownership, errors, and concurrency

No allocation, release, ownership transfer, reference-count operation, or lifetime guarantee is identified in the visible body. Unknown dependencies may allocate, retain, release, or mutate memory.

No explicit error status, recovery path, rollback, synchronization, or transactional boundary appears. A fault or non-returning dependency may leave earlier effects in place.

Reentrancy, thread safety, interrupt safety, and suitability for volatile or device storage remain **documented_only**.

## Dependencies

Only F-2bbf26de70e4 has a supplied opaque function ID. No opaque IDs resolve the indirect targets; their identities must not be invented.

Established constraints concern invocation shape and dataflow: zero through four explicit arguments, 32-bit results, paired result halves, and additional register-derived values. The final invocation receives the four values specified above.

No dependency is assumed to preserve globals, preserve reachable memory, return normally, or own or release any supplied value.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Input-field boundaries

Assuming the early invocations return normally:

| Third input | Field-derived argument |
|---:|---:|
| 0x00000000 | 0x00000000 |
| 0x0000000F | 0x00000000 |
| 0x00000010 | 0x00040000 |
| 0x00003FF0 | 0x0FFC0000 |
| 0xFFFFFFFF | 0x0FFC0000 |

Equal field-derived arguments do not imply equal complete executions.

### Width and fixed-store values

Under unsigned 32-bit arithmetic, shifting 0x12345678 left by eight yields 0x34567800. If the opening offset-5 byte read yields 0xE1, its word-valued copy is 0x000000E1.

If the retained word is 0x00000100, the fixed 32-bit store at 0x54 carries 0x00000200; this positive, representable shift does not require an overflow assumption.

If the small base is 0x128 and the retained word is 0x100, the two adjacent computed offset-0x15 byte stores carry 0x00, while the later fixed-address 0x15 store carries 0x28. These are different value sources.

### Captured fixed state

If a subsequent computed destination equals or overlaps 0x087FBBBC, its write may change the stored base, but later repeated groups continue using the previously captured value.

The fixed store at 0x60 does not itself overlap that capture location under ordinary address-range interpretation.

Similarly, changing storage at 0x05 after its read does not replace the saved fixed-read base.

### Fresh copies and final snapshots

If 0x67 initially receives 0xAB, the copy to 0x21 uses the byte present at its own read point. A computed write changing 0x67 to 0xCD before a later copy makes that copy consume 0xCD, subject to accessible storage and the stated overlap.

If linked offset 0x14 yields 0x11223344 at capture, later overwrites do not change the third argument. If the late loaded-base offset-0x74 read yields 0x55667788, subsequent writes do not change the fourth argument; writes before that read may affect it.

### Interrupted execution

A non-returning dependency prevents any requirement that later accesses occur. A fault establishes no recovery result or restoration of earlier writes.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended whole-state transition.
- Original calling convention, function boundary, and final-invocation semantics.
- Indirect dependency identities, effects, and ownership policies.
- Original storage width of the fixed read at 0x05.
- Valid addresses, alignment, byte order, and storage classes.
- Signed overflow, problematic signed shifts, and oversized shift counts.
- Complete alias relationships, concurrency behavior, and original instruction multiplicity.
- Fidelity of unusual low-address effects to executable machine behavior.

No supplied evidence establishes alternative storage mapping between the fixed locations 0x60 and 0x087FBBBC.

## Evidence

This contract derives from static inspection of the supplied representation. Embedded comments, prior prose, and reviewer feedback are not independent behavioral evidence.

Explicit access widths and dataflow establish the fixed stores at 0x54 and 0x15, their distinct value sources, their position among computed writes, fixed-base capture ordering, fresh reads from 0x67, and separate final-argument capture points.

No dependency bodies, implementation access, machine-instruction verification, or runtime observations were available. No runtime tests have been performed.
