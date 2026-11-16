<!-- Trusted provenance: F-aebe7567dbc5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aebe7567dbc5

## Purpose

F-aebe7567dbc5 performs an extended sequence of unresolved indirect calls, transforms returned and implicit machine values, mutates memory through literal and computed addresses, and finishes with an unresolved four-argument dispatch.

**Established static evidence:** the supplied representation establishes call ordering, selected argument relationships, access widths, dependent address calculations, and ordered memory effects. It does not establish an application-level purpose or a meaningful success result.

**Inferred intent:** some dependent reads and writes resemble updates to interconnected records. That resemblance does not establish a particular data structure, object model, initialization operation, or ownership protocol.

This contract describes the visible behavior without substituting a conventional higher-level operation. Unresolved behavior is marked **documented_only**. Acceptance examples are conditional obligations, not observed executions. No runtime tests have been performed.

## Interface

### Invocation and implicit inputs

The represented interface has no explicit parameters and no declared return value. Its behavior nevertheless depends on incoming machine state:

- An incoming control-link value determines the first indirect target through addition of 4094.
- An incoming 32-bit general-purpose value supplies two equal arguments to an early call, each formed by shifting that value left seven bits.
- Secondary machine-result values are consumed after calls whose complete result conventions are unavailable.
- Call results and memory contents supply subsequent values and addresses.
- A global 32-bit word supplies a base captured before the repeated-store region.
- The terminal indirect target is unresolved.

The lack of explicit parameters does not establish that arbitrary entry state is valid.

### Outputs and completion

There is no established output buffer, scalar result, success flag, or recoverable error value. Observable outputs comprise the dependency invocations, their represented arguments, the direct memory accesses, and dependency effects.

A return follows the terminal dispatch in the representation. Reaching that return requires every preceding call and access, and the terminal dispatch itself, to complete. An unresolved dependency is not guaranteed to return.

### Widths and storage

| Quantity or access | Width | Contract |
|---|---:|---|
| Unsigned-byte read | 8 bits | Widening to 32 bits introduces zero upper bits. |
| Byte store | 8 bits | Stores the low eight bits of the supplied value. |
| Halfword store | 16 bits | Writes two bytes, including for constants 0x003A and 0x0065. |
| Word read or store | 32 bits | Used for values and address-bearing quantities. |
| Composite call result | 64 bits | Low and high 32-bit portions are consumed separately. |

Composite results do not establish a public 64-bit numeric return convention; they may represent multiple machine results.

All relative offsets below are byte offsets. Endianness, alignment requirements, executable-address conventions, and the memory map are **documented_only**.

### Preconditions and ownership

Successful completion requires each reached indirect target to be callable and each reached memory range to permit the indicated access width. Returned and loaded address-bearing values must be usable when their dependent accesses occur.

No validation of these requirements is visible. In particular, zero-extended bytes and small computed integers become address bases without visible range checks.

No allocation, release, retention, borrowing, or ownership-transfer contract is established. Unknown dependencies must not be assumed to preserve globals, leave memory unchanged, allocate storage, or release storage.

## Behavior

### Execution phases and arithmetic

The visible operation has three ordered phases: indirect calls and result transformations; direct memory reads and mutations; terminal indirect dispatch.

No explicit conditional branch, retry, validation exit, rollback, or cleanup path appears in the body. Dependencies may branch internally, modify state, fault, or fail to return.

Ordinary values and address-bearing arithmetic are represented predominantly at 32-bit width. For explicitly unsigned operations, right shifts are logical. Under a wrapping 32-bit interpretation, additions, subtractions, multiplications, and left shifts retain their low 32 bits.

Some operations instead use signed quantities. Signed overflow, signed left shifts, and correspondence to the underlying machine behavior remain **documented_only**. Address formulas in this document identify relationships, not unbounded mathematical arithmetic.

A late signed right shift uses a count derived from a computed base’s low byte. Behavior for counts outside 0–31 is unresolved; the count must not silently be masked, saturated, or rejected.

### Indirect calls and transformations

The first target is derived from incoming control state plus 4094. Subsequent targets use represented control-location values with the same displacement. This does not establish target equivalence or a shared signature.

Calls have zero through four represented arguments. Visible relationships include:

- Low result words shifted left eight bits.
- Unchanged high result words paired with transformed low words.
- Equal transformed values in multiple argument positions.
- Literal zero arguments and retained values reused after intervening calls.
- Masked fields repositioned by left shifts.
- Logical right shifts of unsigned values and portions of composite results.

Fixed left shifts reach 31 bits. Masks include field widths of 11, 12, 13, 14, 16, 17, 18, 19, 20, 21, 22, and 24 bits. These transformations are not established bounds checks, size calculations, or unit conversions.

Several retained quantities are explicitly zero before additional shifts. Under fixed-width bit-vector interpretation they remain zero, but their associated calls still occur.

### Access layout and aliasing

No complete record layout is established. Relative access sites include:

| Offset | Access widths |
|---:|---|
| 1 | Byte |
| 4 | Byte and word |
| 5 | Unsigned byte |
| 8, 9, 13, 17, 21, 25 | Byte |
| 16 | Byte and word |
| 20, 32, 36, 52, 64, 68, 84, 100, 116 | Word |

These offsets belong to different computed bases and must not be combined into one inferred structure.

Literal-address effects also occur, including byte accesses at 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, and word accesses at 0x54, 0x60, and 0xA9. A literal address is not interchangeable with an equal-numbered offset.

Bases derive from call results, scaling, word reads, and unsigned-byte reads. No disjointness guarantee exists. Earlier stores may affect later reads, and equal stores may remain separately observable.

### Initial direct-memory effects

The memory phase begins with a word store at offset 20 from the high word of the final composite call result. The stored value is a retained transformed result.

A byte at offset 9 from another computed base is written twice, with a separate word store between those writes. The first byte derives from the composite result’s high word; the second derives from the retained transformed result.

The function then clears an offset-20 word, reads an unsigned byte at offset 5 from that base, clears an offset-20 word through a second computed base, and stores the zero-extended byte at offset 52 from the first base.

For the remainder of this contract, **second-clear base** means that second computed base. It is twice a quantity formed by shifting a preceding 32-bit result left eight bits, retaining its low 15 bits, and logically shifting right six bits.

The first offset-20 word is actually reloaded. The second-clear base’s offset-20 word is cleared again, followed by two byte stores at its offset 21, each receiving the low byte of the reload.

The reload cannot be replaced by a presumed zero: intervening accesses may overlap the previously cleared storage.

Next, an unsigned-byte read at literal address 0x05 supplies another base. A byte is written at literal address 0x01 from the second-clear base’s low eight bits. Dependent writes follow through the byte-derived and other bases. A word at offset 84 from the byte-derived base is captured before replacement; this **captured word** drives later scaling and repeated stores.

### Dependent updates, including literal address 0x15

Further updates use words obtained at offsets 4, 84, 100, and 116 as address-bearing quantities. An unsigned byte read at offset 5 also supplies a base for an offset-116 word read.

Effects include self-relative word stores, duplicate stores, byte stores derived from address-bearing values, and offset-84 stores through values loaded at offset 116. An offset-116 location is read again around a dependent store; substitution of the earlier read is not justified where aliasing may change the later result.

Literal address 0x54 receives a word equal to twice the captured word. The second-clear base’s offset-20 word is then written with the captured word twice.

A later pair of byte stores occurs at **second-clear base plus decimal 21**, each storing the low eight bits of four times the captured word. **Immediately after this later pair, the function performs a one-byte store to literal address 0x15, storing the low eight bits of the second-clear base itself.** The following direct memory effect is a word store at offset 84 from four times the captured word, using a retained transformed call result.

These are three distinct address relationships:

- The duplicate computed-base stores target second-clear base plus decimal 21.
- The intervening fixed store targets literal address 0x15, which is decimal 21.
- A much later fixed store targets literal address 0x21, which is decimal 33.

The fixed 0x15 store is neither a third computed-offset store nor the later 0x21 store. It retains its position between the later duplicate pair and the following offset-84 word store. Aliasing may cause it to affect subsequent storage contents or address-bearing reads. Later overlapping stores may replace it, so no unconditional final value at 0x15 is promised.

Subsequent dependent updates include offset-52, offset-68, offset-84, and offset-20 word stores and offset-9, offset-13, and offset-21 byte stores. Some values are replaced later; the contract concerns the ordered accesses, not merely final field contents.

### Constants and preparation for repetition

A later loaded word supplies three related destinations:

- A byte containing 0x3A at four times that word.
- A halfword containing 0x003A at eight times that word.
- A word containing 0x0000003A at offset 32 from four times that word.

These are separate access-width obligations.

The captured word supplies two additional descriptive quantities:

- **Adjusted quantity:** four times the captured word minus decimal 150.
- **Scaled base:** four times the adjusted quantity.

At scaled-base offset 32, the function writes 0x20 and then 0x2F, reads an unsigned byte at scaled-base offset 4, stores that byte’s zero-extended value at offset 32, and writes 0x20 there again.

The global base word is then captured. A word containing 0x20 is written at literal address 0x60 afterward. There is no explicit reload of the global base because of that store.

### Repeated-store region

The **payload** is four times the later word obtained through an unsigned-byte-derived offset-116 read.

Each repeated group has four ordered stores:

| Position | First group | Subsequent groups |
|---:|---|---|
| 1 | Word payload at offset 84 from eight times the captured word | Word payload at offset 84 from eight times the adjusted quantity |
| 2 | Low byte of scaled base at captured-global-base offset 25 | Same destination and value |
| 3 | Low byte of captured global base at scaled-base offset 17 | Same destination and value |
| 4 | Eight times captured word at captured-global-base offset 52 | Eight times adjusted quantity at captured-global-base offset 52 |

The first group differs in both its first word destination and its fourth stored value. Equality or overlap with later destinations requires an actual arithmetic or alias relationship.

Captured quantities are not visibly reloaded within these groups. Overwriting the global’s original storage does not itself change the captured base.

Duplicate stores remain part of the represented trace. The region is not established as copying, allocation, appending, or initialization. Exact group multiplicity is **documented_only** here; no numeric count is asserted.

### Post-repetition capture ordering

After repetition, the function first captures the word at literal address 0xA9 and then a word at offset 116 from the payload.

It next writes one byte at literal address 0x3A. That byte is eight times the adjusted quantity plus 0x30, narrowed to eight bits.

Only afterward are two more words captured: an offset-20 word through a base equal to twice the loaded word from which the payload was formed, followed by the word at scaled-base offset 4.

The offset-20 and offset-4 captures therefore observe storage after the fixed 0x3A byte store if their ranges overlap it. The earlier fixed-word and payload-relative captures retain their pre-store values.

### Remaining mutations and dispatch

Later effects include byte 0x6C at scaled-base offset 8, byte updates at offsets 9 and 13, and a self-valued word store at offset 84 from the doubled-word base.

The low byte of the captured 0xA9 word is stored at literal address 0x67. Storage at 0x67 is subsequently read for copying to literal address 0x21 and for additional byte stores. These reads are distinct accesses; intervening computed stores may alias 0x67.

A word read at offset 68 from the doubled-word base supplies further mutations and the variable signed right shift. The shift result is narrowed to a byte at offset 1 from that base. Its count is the scaled base’s low byte.

Under wrapping 32-bit arithmetic, the scaled base equals sixteen times the captured word minus 600. Its low byte is therefore one of 8, 24, 40, through 248 in steps of 16. Many possible counts exceed 31; their machine semantics remain unresolved.

Further effects include:

- Reading scaled-base offset 64 and storing that captured word at offset 16.
- Storing the earlier scaled-base offset-4 capture at captured-global-base offset 116.
- Additional byte and word mutations at offsets 9, 17, 68, and 84.
- A halfword store of 0x0065 at twice a newly loaded address-bearing word.
- A word store of decimal 100 at offset 84 through another loaded base.
- Additional offset-100 and offset-116 reads and a word store at twice a loaded value.

The terminal dispatch receives:

| Argument | Origin |
|---:|---|
| 1 | 0x72 |
| 2 | Word captured at offset 100 from a late address-bearing word loaded through an offset-4 relationship |
| 3 | Offset-20 word captured after the literal 0x3A byte store |
| 4 | Word later captured at offset 116 from the same late address-bearing word used for argument 2 |

Arguments 2 and 4 arise from distinct reads separated by mutations. Argument 3 retains its earlier capture rather than reloading its original storage at dispatch.

## Side effects

The function is not pure or established as atomic. Direct effects include mixed-width memory accesses, literal low-address mutations—including the byte store at 0x15—dependent-address writes, repeated stores, global-base capture, and indirect calls.

No transaction, synchronization primitive, memory barrier, rollback, or cleanup mechanism is visible. If execution stops at a call or access, earlier effects have no represented undo path.

Dependency effects, global preservation, reentrancy, thread safety, and idempotence are **documented_only**. Repeated equal stores do not establish idempotence where reads, aliases, calls, or access-sensitive storage are involved.

## Dependencies

Only F-aebe7567dbc5 has a supplied resolved function identity. Unresolved dependencies use opaque documentation IDs:

| Opaque ID | Scope |
|---|---|
| F-opaque-001 | Initial entry-state-derived indirect call |
| F-opaque-002 | Subsequent unresolved indirect-call sites |
| F-opaque-003 | Terminal four-argument indirect dispatch |

F-opaque-002 denotes a collection of sites, not one established callee or signature. These IDs imply neither recursion nor target equivalence.

For each dependency, valid argument domains, result conventions, memory and global effects, ownership rules, failure modes, and return guarantees are **documented_only**.

## Acceptance examples

These examples are conditional and have not been executed.

1. **Narrowing and widening.** A byte store supplied with 0x123456AB writes 0xAB through one byte. An unsigned-byte read yielding 0xFE widens to 0x000000FE for an associated word store.

2. **Shift truncation.** Under wrapping 32-bit shifts, shifting 0x12345678 left eight bits and then logically right eight bits yields 0x00345678.

3. **Early ordered overwrites.** An access-observing environment must distinguish the two early offset-9 byte stores and their intervening word store. Recording only the final byte is insufficient.

4. **Literal 0x15 versus computed offset 21.** If the second-clear base is 0x128 and the captured word is decimal 100, the later duplicate stores each write byte 0x90 at address 0x13D. They are followed by a one-byte store of 0x28 at literal address 0x15, then the following word store at address 0x1E4. This fixed byte store must not be redirected to 0x13D or to literal address 0x21. Later aliasing may overwrite any of these bytes.

5. **Literal-store aliasing.** If subsequent accesses reach storage containing address 0x15, they must respect the fixed byte mutation and any intervening overlapping effects. Its final value cannot be inferred solely from the low byte originally stored there.

6. **Repeated-region distinction.** For captured word 100 without overflow, the adjusted quantity is 250. The first offset-84 destination is address 884; subsequent groups use address 2084. Their fourth stores contain 800 and 2000 respectively.

7. **Post-repetition aliasing.** If the later offset-20 read spans address 0x3A, it observes memory after the fixed byte store there. Endianness determines the affected numeric byte position. The two earlier captures retain their pre-store values.

8. **Access widths.** The related 0x3A stores require byte, halfword, and word accesses. Equal numeric constants do not make those accesses interchangeable.

9. **Capture lifetime and partial execution.** Later changes to the original offset-20 storage do not alter dispatch argument 3’s captured value. If an earlier dependency never returns, neither later memory effects nor terminal dispatch are reached; no rollback is promised.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose, object model, and complete record layouts.
- Complete entry-state requirements and calling conventions.
- Completeness and machine fidelity of the supplied representation.
- Resolved dependency identities and complete per-site contracts.
- Complete enumerated call trace and exact repeated-store multiplicity.
- Memory permissions, endianness, alignment, and actual alias relationships.
- Signed arithmetic, overflow, and variable-shift machine semantics.
- Allocation, ownership, lifetime, and release rules.
- Fault reporting, recoverable errors, and normal-return guarantees.
- Terminal-target selection and effects.

These uncertainties do not authorize omitting visible effects, inventing validation, assuming disjoint storage, or conflating literal addresses with computed offsets.

## Evidence

Established behavior derives from static inspection of the supplied representation, not from the prior draft or reviewer feedback as independent authority.

The literal 0x15 mutation is directly represented as a byte-width store of the second-clear base’s low eight bits. Its position follows the later duplicate byte stores at that base’s decimal offset 21 and precedes the following offset-84 word store. The later literal 0x21 mutation is a separate copy from literal 0x67.

The repeated-region distinction follows from different origins for the first and subsequent offset-84 destinations and offset-52 values. Post-repetition capture timing follows the represented fixed-word read, payload-relative read, fixed byte store, and subsequent offset-20 and offset-4 reads.

Arithmetic simplifications are conditional where machine semantics are unresolved. Structural interpretations remain inference.

**No runtime tests have been performed.** No dependency bodies, executable trace, or machine-state observations were available.
