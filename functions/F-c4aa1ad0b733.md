<!-- Trusted provenance: F-c4aa1ad0b733; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4aa1ad0b733

## Purpose

F-c4aa1ad0b733 performs an ordered series of unresolved indirect calls, transforms their results, modifies memory through computed and fixed addresses, and invokes a final unresolved target.

No application-level purpose is established. The representation does not support a contract identifying the function as a parser, formatter, allocator, serializer, or initializer. Character-compatible constants and recurring offsets are insufficient to establish strings or an object model.

This contract distinguishes:

- **Established static evidence:** widths, operations, data dependencies, and ordering displayed in the supplied representation.
- **Conditional consequences:** behavior that follows when those operations correspond to executable machine behavior and the relevant accesses complete.
- **documented_only:** unresolved semantics that must not be replaced by assumptions.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits, uninterpreted | No explicit use in the body. |
| Second argument | 32 bits, uninterpreted | No explicit use in the body. |
| Third argument | Signed-represented 32 bits | Shifted left by six positions and supplied to an early indirect call. |

The third argument receives no local validation. The representation establishes no accepted range, enumeration, or pointer interpretation.

The absence of explicit uses of the first two arguments does not establish their irrelevance at the machine interface. Argument recovery is incomplete, and unresolved targets may observe ambient register or memory state.

### Ambient inputs

Processing depends on:

- Incoming return-link state, used to derive the first call target.
- Further executable addresses derived from represented code locations.
- Primary and secondary result channels from unresolved calls.
- Results represented as 64-bit aggregates.
- Memory reached through calculated addresses and loaded address-like values.
- A fixed-location value at 0x05.
- A 32-bit base captured from 0x087fbbbc.
- A 32-bit value captured from 0xa9.
- Three separate later byte reads from 0x67.
- An unresolved final transfer target.

These fixed-location inputs have different capture times. They are not interchangeable with an entry-time snapshot of global memory.

### Outputs and completion

No explicit return value is represented. Outputs are dependency interactions, memory modifications, and the final indirect transfer.

The final target is represented as a call followed by a return. Normal return is conditional on that target returning and all preceding operations completing. No status result, error code, or explicit failure-reporting channel is established.

### Preconditions and ownership

Completion requires each reached target to support execution and every reached memory access to be valid at its represented width.

No complete object extent, buffer size, or allocation boundary can be established. Some bases are obtained from call results, some from 32-bit memory reads, and some from byte-valued reads.

No ownership transfer is established. No dependency is identified as an allocator or release operation. Unknown targets must not be assumed to preserve globals, retain memory, release memory, or leave storage untouched.

## Behavior

### Ordered dependency interactions

The call-driven portion is represented as straight-line processing without a local loop or conditional branch. It contains many indirect calls, including calls with no displayed arguments.

Early ordering establishes that:

- Several calls occur before the third explicit argument is consumed.
- A primary result is shifted left by eight positions for a subsequent call.
- The third argument shifted left by six is supplied alongside a secondary result and that secondary result shifted left by six.
- Later interactions consume shifted results, extracted fields, secondary outputs, and explicit zero values.
- Captured results sometimes survive intervening calls and are reused later.

A call with no displayed arguments is still an interaction. An unused result does not establish that its producer is removable. The body cannot be treated as a pure calculation over the three explicit arguments.

Exact target identities, complete calling conventions, and dependency behavior remain **documented_only**. The displayed call order and result dependencies are static evidence, not a recovered executable API for those targets.

### Widths and arithmetic

The dominant scalar width is 32 bits. Some results are represented as 64-bit quantities, with low and high 32-bit portions extracted separately.

The transformations include:

- Fixed left shifts with counts between one and 31.
- Logical right shifts of unsigned-represented values.
- Low-field masks of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, and 11 bits, plus single-bit selection.
- Selection of the low 20 bits followed by a right shift of ten.
- Low-24-bit selections followed by further shifts.
- Logical extraction after shifting a 64-bit result by 34, 36, or 44 positions.

The last three extractions correspond to shifting the high 32-bit portion right by two, four, or twelve positions.

Widening intermediate arithmetic can change the represented behavior. Under 32-bit bit-vector semantics, shifting a value left by 28 and then left by another 28 produces zero; it does not preserve a 56-bit result.

Signed-represented shifts, signed multiplication, and overflow do not establish portable arithmetic semantics. Address scaling and offset addition likewise must not silently become unbounded arithmetic. Their effective overflow and address-conversion rules remain **documented_only**.

### Entry into direct memory mutation

The transition from calls to direct memory updates establishes this initial order:

1. A 32-bit shifted value is stored at offset 0x14 from the high portion of the last aggregate result.
2. A byte at another derived base plus nine receives the low byte of that high portion.
3. A 32-bit value is stored at offset 0x54 from the last aggregate’s low portion scaled by 256.
4. The earlier byte destination is overwritten with the low byte of another captured value.
5. A derived 32-bit field at offset 0x14 is cleared.
6. A byte at offset five from that base is captured.
7. Another 32-bit destination is cleared.
8. The captured byte is zero-extended into a 32-bit store at offset 0x34.

A subsequent 32-bit read of the first cleared field occurs only after these operations. Aliasing may therefore affect its value; the earlier clear alone does not justify replacing the read with a constant.

The memory phase interleaves reads and writes. It is not equivalent to reading every input first and then committing final outputs.

### Capture at 0x05

After two consecutive byte writes to the same computed offset-0x15 destination, the function reads the fixed-location value at **0x05** and represents the captured result as an unsigned 32-bit quantity. This read precedes the byte write at **0x01**.

The captured value subsequently serves as a memory base:

- A byte is written at captured base plus 0x15, after the write at 0x01.
- A 32-bit value is later read at captured base plus 0x54.
- That same offset-0x54 destination is then overwritten with another captured 32-bit value.

Other computed writes intervene between the first use of this base and its offset-0x54 read. If they alias the offset-0x54 storage, the read observes their effects. If they instead change storage at 0x05, that does not refresh the already captured base.

The underlying storage width of the named value at 0x05 is not explicitly declared in the supplied material. Its conversion to an unsigned 32-bit working quantity is established; the original read width remains **documented_only**.

### Mixed-width and linking updates

The remaining direct-memory phase includes byte, 16-bit, and 32-bit accesses.

| Width | Established access shapes |
|---|---|
| 8 bits | Offsets including 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, and 0x19 |
| 16 bits | Scaled computed addresses |
| 32 bits | Offsets including 4, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74 |

These offsets do not establish one structure layout. Address-like values are stored relative to themselves, fields are overwritten repeatedly, and reads traverse previously loaded values. In one traversal, a byte read supplies the base for a subsequent 32-bit read at offset 0x74.

Byte writes retain only the low eight bits. Byte reads converted to unsigned 32-bit values are zero-extended. Numeric 16-bit and 32-bit stores do not establish byte order.

Literal updates include:

- 0x3a stored at 8-, 16-, and 32-bit widths to different scaled destinations.
- Successive 32-bit values 0x20, 0x2f, a zero-extended byte, and 0x20 at one offset-0x20 destination.
- A byte value 0x6c.
- A 16-bit value 0x65.
- A 32-bit value of decimal 100.

The byte used in the third offset-0x20 write is read from the corresponding base plus four after the first two writes and before the third. It is not an entry-time input.

### Capture at 0x087fbbbc and repeated writes

The 32-bit read at **0x087fbbbc** occurs after all four successive offset-0x20 writes and before the fixed 32-bit write of 0x20 at **0x60**.

Call its captured value the **repetition base**. This descriptive label denotes a captured value, not a guaranteed object or allocation.

The repeated-write region reuses that captured base. Its recurring interaction shape is:

- A 32-bit store at a computed destination plus 0x54.
- A byte store at repetition base plus 0x19.
- A byte store at another computed base plus 0x11.
- A 32-bit store at repetition base plus 0x34.

The byte at repetition base plus 0x19 receives the low byte of the other base. The byte at the other base plus 0x11 receives the low byte of the repetition base.

The initial setup differs from the stabilized repetitions in both the first word destination and the value supplied to repetition base plus 0x34. The stabilized region repeats the same four-write pattern explicitly.

No re-read of 0x087fbbbc is displayed during this region. A computed write that changes that fixed storage does not redirect later uses of the captured repetition base. Conversely, the capture occurs late enough to include effects of preceding calls and potentially aliasing writes.

Identical or overwritten stores remain separate represented operations. Collapsing the region to final memory contents does not preserve its displayed interaction trace.

### Late fixed-global sequence

For the following description:

- **Working base** means the unsigned-represented scaled base used throughout the repeated region.
- **Link base** means a previously captured doubled value from an offset-0x74 traversal.
- **Fourfold base** means twice that link base.
- **Repetition base** retains its captured meaning above.

After the repeated region, the function captures a **32-bit value from 0xa9**, then reads a **32-bit value at fourfold base plus 0x74**, then writes a **byte at 0x3a**. The byte written at 0x3a is derived from a scaled intermediate plus 0x30, truncated to eight bits.

The 0xa9 capture therefore precedes both that indirect read and the write at 0x3a. Neither later operation refreshes the captured value.

The body then captures words at link base plus 0x14 and working base plus four. Further writes update link base plus 0x0d, working base plus eight, 0x0d, and nine, and link base plus 0x54. Only afterward does **0x67 receive the low byte of the captured 0xa9 word**.

The subsequent uses of 0x67 are distinct memory reads:

| Read | Relative timing | Consumer |
|---|---|---|
| First byte read at 0x67 | After a 32-bit read at link base plus 0x44 | Byte assignment to 0x21 |
| Second byte read at 0x67 | After intervening byte and word updates, including a word read at working base plus 0x40 and updates through the repetition base | Byte store at working base plus nine |
| Third byte read at 0x67 | After a byte write at link base plus nine, a 32-bit store of the captured 0xa9 word at working base plus 0x44, and a 32-bit read at working base plus 0x54 | A second byte store at working base plus nine |

These are not three consumers of one cached byte. Intervening computed writes can alias 0x67. Each consumer uses the value obtained by its own read.

The captured 0xa9 word also remains available independently of 0x67: its full 32 bits are later stored at working base plus 0x44. That store does not use a refreshed read of either fixed location.

### Final updates and transfer

The late phase also includes a byte write derived from a signed-represented 32-bit right shift. Its count comes from the low eight bits of the working base. No local rejection of counts at or above 32 is displayed; effective shift behavior remains **documented_only**.

Further traversals load address-like values, perform the 16-bit store of 0x65 and the 32-bit store of decimal 100, and finish with a 32-bit store through a value scaled by two.

The final unresolved target receives four displayed arguments, in order:

1. The 32-bit constant 0x72.
2. A recently loaded 32-bit value from an offset-0x64 access.
3. The 32-bit value captured earlier from link base plus 0x14.
4. A recently loaded 32-bit value from an offset-0x74 access.

The fourth argument is a later capture, not the earlier fourfold-base offset-0x74 value. The target’s identity, additional machine inputs, effects, and return behavior are unresolved.

## Side effects

### Fixed and computed storage

Established fixed-location writes are:

| Location | Width |
|---|---:|
| 0x01, 0x15, 0x21, 0x3a, 0x67 | 8 bits |
| 0x54, 0x60 | 32 bits |

Meaningful fixed-location reads include 0x05, the 32-bit captures at 0x087fbbbc and 0xa9, and the three separate byte reads at 0x67.

These addresses are not proof of conventional globals. Low addresses, unaligned accesses, overlapping storage, and unusual pointer-like traversals may reflect actual storage or limitations of the supplied representation.

No disjointness guarantee exists between fixed and computed destinations. No atomicity, synchronization, thread-safety, reentrancy, or signal-safety guarantee is established.

### Errors and partial completion

No explicit local checks handle invalid addresses, unsupported alignment, invalid call targets, dependency failure, arithmetic overflow, or out-of-range shift counts.

No rollback path is displayed. If an operation prevents further execution, earlier effects are not explicitly undone. The representation does not establish a particular exception mechanism or a clean error return.

## Dependencies

The only mapped opaque function ID is **F-c4aa1ad0b733** itself. No other dependency has an established opaque identity, and no recursive call to this function is established.

The dependency surface consists of the entry-state-derived indirect call, subsequent code-location-derived indirect calls, their primary and secondary result channels, aggregate-represented outputs, and the final unresolved transfer.

Dependency identities and contracts remain **documented_only**. Unknown targets may modify memory and globals, fail to return, or affect subsequent observations. No purity, preservation, or memory-ownership guarantee is inferred.

## Acceptance examples

These examples are conditional criteria, not executed tests.

### Explicit input and field widths

If the third argument is 3 and processing reaches its displayed use, its six-position left shift supplies 192.

If an unsigned intermediate is 0x000abcde, selecting its low 20 bits and shifting right ten yields 0x2af. Shifting that result left six yields 0x0000abc0.

If a 64-bit result has high portion 0x80000010, logical extraction after a 36-position right shift yields 0x08000001, not a sign-extended result.

### Capture at 0x05

If the fixed read at 0x05 produces 0x40, the later base-relative accesses use the captured base 0x40. The read must precede the write at 0x01. A subsequent change to storage at 0x05 does not by itself redirect those accesses.

### Repetition-base capture

If 0x087fbbbc contains 0x2000 when read after the four offset-0x20 writes, the repetition base is 0x2000. Its offset-0x19 and offset-0x34 destinations use that capture even if later writes change 0x087fbbbc. Reading the base only after the write at 0x60 would not preserve the displayed capture time.

### Captured word versus refreshed byte

If the word read at 0xa9 is 0x123456ab, the later explicit write to 0x67 supplies 0xab, while the full-word store at working base plus 0x44 supplies 0x123456ab.

The assignment to 0x21 and the two subsequent computed byte stores nevertheless consume three separate reads of 0x67. If an intervening aliasing write changes that byte, the affected later consumer uses the changed value. If no intervening operation changes it, the reads may agree; that agreement does not authorize caching.

### Overwrites and failures

For observable writes, the offset-0x20 sequence retains 0x20, then 0x2f, then the captured byte zero-extended to 32 bits, then 0x20. Keeping only the final value loses ordering.

If a reached address is inaccessible, no clean error result or restoration of earlier writes is required by the available evidence.

If the final target never returns, normal return from F-c4aa1ad0b733 is not required.

## Uncertainties

The following remain **documented_only**:

- Application purpose and complete public calling convention.
- Dependency identities, semantics, termination, and effects on globals.
- Reliability of secondary result channels and aggregate-result recovery.
- Signed arithmetic, overflow, large-count shifts, and address-conversion rules.
- Original storage width of the fixed-location input at 0x05.
- Byte order, alignment rules, mapped address ranges, and object extents.
- Aliasing relationships, ownership, and lifetimes.
- Whether every displayed repeated store corresponds to an executed machine access.
- Final-target identity and return behavior.

**[INFERENCE]** The unusual addressing and unresolved control flow limit confidence in reconstructing a runnable high-level API. They do not justify replacing the displayed interactions with a more plausible algorithm.

## Evidence

The supplied static representation establishes the three displayed 32-bit inputs, ordered indirect interactions, width-sensitive transformations, mixed-width memory updates, capture timing at 0x05 and 0x087fbbbc, and the late sequence involving 0xa9, 0x3a, 0x67, and 0x21.

In particular, it distinguishes values retained across later mutations from separately refreshed memory reads. That distinction governs both the reused repetition base and the three consumers of 0x67.

No dependency bodies, external implementation, memory mappings, or executable calling convention were inspected. No machine execution or runtime tests were performed. Acceptance examples describe conditional consequences of static evidence, not experimentally verified behavior.
