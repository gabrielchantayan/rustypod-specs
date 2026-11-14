<!-- Trusted provenance: F-9f8b412feb5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9f8b412feb5c

## Purpose

F-9f8b412feb5c performs a fixed sequence of indirect calls, address calculations, memory transfers, and global updates. It accepts one explicit 32-bit value and establishes no explicit return value.

Its observable behavior is primarily state mutation: call results and ambient machine values select destinations, supply stored values, and become arguments to later dependencies. The supplied static representation does not establish an application-level purpose, object model, allocation protocol, or success criterion.

The final explicit operation is a call to address zero with three zero arguments. A return follows only if that call returns. The evidence does not establish whether this target is executable, faults, invokes platform-specific machinery, or has another meaning.

This contract describes the supplied representation, not independently verified executable behavior. Its reconstructed calling conventions, secondary machine results, and low-address accesses remain subject to the uncertainties below.

## Interface

### Explicit input and output

- **Input:** one opaque 32-bit value.
- **Initial consumption:** forwarded unchanged as the sole explicit argument of the first indirect call.
- **Interpretation:** integer, address, handle, and encoded-value interpretations are all unestablished.
- **Output:** no explicit return value. Direct memory changes and dependency interactions are observable outputs.
- **Error interface:** no explicit error result, validation branch, recovery path, or rollback is shown.

### Ambient inputs

Execution also depends on:

- An incoming return-address-like value used to form the first call target.
- An incoming unsigned 32-bit value later shifted right by 22.
- Another incoming opaque 32-bit value used as the first stored word.
- Secondary machine-result values used as indices, repeated arguments, and a pointer.
- Two global storage regions, called **G1** and **G2** here.
- Bytes and words at absolute or calculated addresses.
- State changes caused by every dependency.

No ordinary caller-facing mechanism for supplying the ambient values is established.

### Widths and arithmetic

A **byte** is 8 bits, a **halfword** 16 bits, a **word** 32 bits, and a **wide result** 64 bits. Each of the three wide results is separated into low and high 32-bit halves.

Unsigned right shifts introduce zeros. Byte writes retain the least significant eight bits of their source. A word read followed by a byte write therefore transfers the word’s numerical low byte; it does not establish which physical byte of the source word contains those bits.

Unsigned 32-bit arithmetic retains 32-bit results. Some scaling and shifts instead use signed 32-bit intermediates. Their overflow behavior, and signed left shifts outside the supported domain, are not established. Address formulas involving those operations are conditional on their validity in the actual execution environment; they do not impose universal wrapping semantics.

The numerical ranges stated below follow from unsigned masks and shifts, not from an assumption that the resulting addresses are valid.

### Preconditions

Normal traversal of the complete represented path requires:

- Every reached indirect target to be callable using its actual calling convention.
- Every preceding dependency to return normally.
- Every direct access to be valid for its width and required alignment.
- Address arithmetic and signed operations to be valid under the actual execution model.
- Ambient and secondary machine values to support the uses attributed to them.

These are execution conditions, not checks performed by this routine. No local bounds, null, alignment, lifetime, or ownership checks are visible.

## Behavior

### Ordering model

The representation contains one straight-line path, with no explicit conditional branch, loop, or early return.

The phases below preserve the sequence of direct memory effects and the significant argument/result relationships. Intervening calls remain observable: they may modify globals, calculated destinations, selector bytes, or other machine state. They cannot be treated as pure, omitted, or reordered merely because an ordinary return value is unused.

Retained scalar values are distinguished from fresh memory reads. Retaining a scalar in the represented dataflow does not imply that a dependency preserves the memory from which it originated.

### 1. Initial forwarding and first global sample

The first dependency receives the explicit input unchanged. Its target is formed from the incoming return-address-like value and a fixed displacement.

A later call produces a word that is forwarded unchanged to the following dependency. Further calls produce an index used for a word destination:

\[
4(\text{index})+20.
\]

The incoming opaque ambient word is stored there. Immediately afterward, the routine reads G1 as a 32-bit value and retains that sample. The store precedes the sample; an overlapping store can therefore affect it.

The next relevant dependency receives four times the index, without the 20-byte destination displacement.

After more calls, two unsigned quantities are formed:

- The incoming unsigned ambient word shifted right by 22, ranging from 0 through 1,023.
- The retained G1 sample shifted right by 23, ranging from 0 through 511.

A secondary machine-result index selects another word destination at four times that index plus 20. The shifted ambient value is stored there.

A four-argument dependency then receives, in order, the secondary index twice, the shifted G1 sample, and the shifted ambient value.

### 2. Repeated stores and guaranteed-zero values

Another call-produced word is forwarded unchanged. The already shifted G1 sample is shifted right by 23 again. Because its current value cannot exceed 511, the second shift necessarily produces zero.

Following more calls:

1. A returned index selects a word destination at four times the index plus 20. The shifted ambient value is stored there.
2. A dependency receives that index multiplied by four.
3. A second secondary machine-result index selects another word destination at four times its value plus 20. Zero is stored there.
4. A dependency receives this secondary index.

The second secondary index remains retained for a later masked address calculation.

### 3. Derived word read and first selector-directed write

After further calls and result forwarding, a returned index selects another word destination at four times the index plus 20. The value stored is the initial G1 sample after two successive unsigned right shifts by 23; it is zero.

The routine then reads a word from:

\[
20+((\text{retained secondary index}\mathbin{\&}0x1fffffff)\gg19).
\]

The calculated address ranges from 20 through 1,043 inclusive. The expression does not guarantee word alignment.

A dependency receives four times the latest store index. Further calls intervene before the newly read word is shifted right by 22.

For the first selector-directed byte write, the previously established zero is increased by one to select a byte-read address. Thus the represented selector read is from absolute address 1. The unsigned selector byte is increased by 17 to form the byte-write destination. The low byte of the shifted word is written there.

The next significant dependency receives a secondary 32-bit value in each of its first three argument positions and zero in its fourth.

### 4. Byte clearing and repeated scaled arguments

A subsequent dependency is explicitly passed zero.

After intervening calls, the routine reads a selector byte from:

\[
1+4(\text{derived word}\gg22).
\]

Here “derived word” means the word read in the preceding phase, before its right shift. The selector-read address ranges from 1 through 4,093 in steps of four. A zero byte is written at the selector’s unsigned value plus 17.

A dependency receives a newly returned index shifted left by two. Later, another dependency receives a different newly returned index shifted left by two in both argument positions.

A returned word is forwarded unchanged and retained for a later right shift.

The routine then freshly reads the byte at absolute address 1 and clears the byte at that selector plus 17. A secondary machine-result value is retained and passed three times to the next significant dependency.

### 5. Retained byte base and initial G2 replacement

A new returned word is forwarded together with the preceding retained word shifted right by 23.

After more calls, the byte at absolute address 1 is read again. Its unsigned value is retained as a base, and the byte at that base plus 17 is cleared.

Subsequent dependencies receive newly produced scaled indices, including a two-argument call receiving the same shifted index twice.

Another dependency produces a word. The routine assigns that word to G2 before forwarding it to the following dependency. The forwarded value is the retained result, not a fresh read of G2.

After an intervening call, the routine reads a word at the retained byte-valued base plus 116 and writes its low byte at that same base plus 1.

The retained secondary machine-result value from the previous phase is then shifted left by 11 and passed to a dependency. This shift uses a signed intermediate in the representation, so its validity is conditional.

Changes to absolute address 1 do not change the already retained base.

### 6. First wide result and repeated field refreshes

The first distinct 64-bit call result is consumed by a three-argument dependency. Its arguments are:

1. The result’s low 32 bits.
2. Its high 32 bits.
3. The word previously assigned to G2 shifted right by 23.

The third argument is derived from the retained assigned value, not from a fresh G2 read.

After more calls, a returned index is multiplied by four to form a base. The routine reads the word at that base plus 116 and writes its low byte at that base plus 1. A dependency receives the base.

Further calls produce a different index, retained both unscaled and shifted left by two. The routine then repeats the previous base’s word-read/byte-write pair.

This repetition performs a fresh word read. If intervening dependencies changed the word at the previous base plus 116, the second byte update uses the changed value.

A four-argument dependency receives:

1. The earlier retained secondary value shifted left by 12.
2. The newly retained index shifted left by two.
3. That same scaled value again.
4. The new index shifted left by 18.

Signed-shift restrictions apply where displayed by the representation.

### 7. Full-word propagation and linking

Following several result-forwarding calls, another returned index is multiplied by four to form a new base.

The routine reads the word at that base plus 116, retains the full word for later propagation, and writes its low byte at the base plus 1. A dependency receives the base.

After additional calls, a returned index selects a word destination using a different stride:

\[
16(\text{index})+20.
\]

The retained full word is stored there.

A dependency then receives the earlier retained scaled index shifted unsigned-right by two and subsequently left by 16. The intermediate 32-bit scaling and shifts matter; this is not an unrestricted wider multiplication.

Further calls lead to another store of the retained full word, now at four times a newly returned index plus 20. A dependency receives that index multiplied by four.

Later, a new returned value is shifted left by two. The scaled result is stored as a word at:

\[
(\text{earlier retained scaled index}\mathbin{\&}0xfffffffc)+20.
\]

It is then passed twice to another dependency.

After another result-forwarding sequence, zero is stored at four times a returned index plus 20. A following dependency receives four times that index.

### 8. Second wide result and fresh G1 sampling

A second distinct 64-bit call result supplies low and high 32-bit halves.

The low half is interpreted through a signed 32-bit intermediate and shifted left by two. The resulting word is stored at four times the high half plus 20, also using the represented signed interpretation for that address calculation.

Immediately after the store, G1 is read again. This is a new snapshot, not reuse of the initial sample.

A four-argument dependency receives the scaled low half, the high half, and the scaled low half twice more.

After additional calls, the same scaled low-half value is stored at four times another returned index plus 20.

Next, a word is read at the freshly sampled G1 value plus 20. The preceding store occurs before this read; if the addresses overlap, the read can observe its effect. The base remains the retained G1 sample despite intervening dependency activity.

A following dependency receives four times the latest destination index.

### 9. Third wide result, final selector writes, and termination

A third distinct 64-bit call result is split into low and high halves.

The routine reads a selector byte from:

\[
1+(\text{retained propagated word}\gg23).
\]

This selector-read address ranges from 1 through 512. The byte at the selector’s value plus 17 receives the low byte of the word read through the second G1 sample.

A four-argument dependency receives the third wide result’s high half three times, followed by its low half shifted left by two.

Following further calls, the routine reads another selector byte from:

\[
1+((\text{third wide-result low half}\mathbin{\&}0x3fffffff)\gg21).
\]

This selector-read address also ranges from 1 through 512. The byte at the selector plus 17 receives the low byte of the G1-derived word shifted right by 22.

Subsequent dependencies receive fresh scaled indices, including one call with the same scaled value in both argument positions. Another produced word is forwarded. A final dependency produces a byte, with a secondary machine-result pointer associated with this terminal sequence.

The terminal direct effects occur in this order:

1. Write one zero byte through the secondary pointer.
2. Read the represented 24-bit quantity at absolute addresses 9 through 11 and assign G2 a word whose low eight bits are the final byte result and whose high 24 bits are that quantity.
3. Write a zero halfword at:
   \[
   (\text{third wide-result low half}\mathbin{\&}0x3fffffff)\gg7.
   \]
4. Call address zero with three zero arguments.
5. Return only if that call returns.

The halfword destination ranges from zero through `0x007fffff`, without guaranteed even alignment. The exact physical access sequence within the represented three-byte read is not established.

## Side effects

Direct effects include 8-, 16-, and 32-bit memory writes; byte and word reads; a represented three-byte read; two distinct G1 samples; and two G2 replacements.

The recurring access patterns establish only numerical relationships:

| Pattern | Established relationship |
|---|---|
| Word destinations | Frequently four times an index plus 20; once sixteen times an index plus 20 |
| Field refresh | Read a word at base plus 116, then write its low byte at base plus 1 |
| Selector-directed byte destination | Unsigned selector byte plus 17 |
| Final halfword destination | Masked and shifted low half of the third wide result |

Every selector-directed destination lies between 17 and 272 inclusive. This does not establish accessible storage or a valid object layout.

The supplied warning reports overlapping global symbols. G1, G2, absolute-address accesses, and calculated destinations must not be assumed independent. In particular, the terminal pointer write may affect the following three-byte read if their storage overlaps.

No allocation, deallocation, ownership transfer, reference counting, locking, or synchronization protocol is established. Dependencies may perform such operations, but none is assumed to do so.

No rollback is visible. If execution stops after earlier writes, those writes may remain observable.

## Dependencies

Only F-9f8b412feb5c has a supplied resolved opaque identity. The following labels are document-local opaque dependency IDs:

- **F-U001:** initial dynamically formed target receiving the explicit input.
- **F-U002:** terminal address-zero target receiving three zeros.
- **F-U003:** remaining indirect dependency interactions, including the three distinct wide-result producers.

F-U003 is a grouping label, not an assertion that these calls share a target or implementation.

Dependency interactions include unchanged result forwarding, repeated arguments, zeros, scaled indices, shifted values, three 64-bit result decompositions, and secondary machine results.

Concrete identities, calling conventions, and effects are **documented_only**. No dependency is assumed to preserve globals, preserve pointed-to memory, return normally, or own or release storage. Exact attribution of secondary machine results to individual calls remains unresolved.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Unchanged explicit input.** Given input `0x89abcdef`, a normally invoked F-U001 receives that same 32-bit value.

2. **Double shift.** If the first G1 sample is `0xffffffff`, shifting right by 23 produces 511. Shifting that result right by 23 produces zero. Both stores described as using the twice-shifted sample therefore write zero.

3. **Byte truncation.** If the derived word is `0xffffffff`, shifting right by 22 produces 1,023. The first selector-directed byte write stores `0xff`.

4. **Selector boundaries.** A selector byte of zero selects destination 17; a selector byte of 255 selects destination 272.

5. **Fresh field sampling.** If an intervening dependency changes the retained base’s word at offset 116, the repeated refresh writes the changed word’s low byte, not the earlier read’s low byte.

6. **Store before global sampling.** If the first word store overlaps G1, the first G1 snapshot observes the post-store contents. The analogous ordering applies before the second G1 snapshot.

7. **Three separate wide results.** If the first wide result has low half 3 and high half 5, its consumer receives 3 and 5 in its first two positions. If the second has the same halves and the signed calculations are valid, it causes a word containing 12 to be stored at address 40. If the third also has those halves, its four-argument consumer receives 5, 5, 5, and 12. These are distinct call results; equal example values do not imply reuse.

8. **Terminal overlap.** If the secondary pointer addresses byte 9, its zero-byte write precedes acquisition of the 24-bit quantity from addresses 9 through 11.

9. **Partial execution.** If a dependency does not return after earlier direct stores, no local rollback is represented.

10. **No guaranteed completion.** Reaching F-U002 does not establish a successful return. If F-U002 does not return, the routine’s displayed return is unreachable.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and success semantics.
- Concrete dependency identities and calling conventions.
- Origin and reliability of ambient and secondary machine values.
- Exact association of secondary results with particular calls.
- Validity of absolute low-address accesses and the address-zero call.
- Endianness, physical global overlap, and the detailed three-byte read mechanism.
- Actual address width, alignment requirements, and signed-overflow behavior.
- Object boundaries, storage lifetimes, ownership, concurrency, and synchronization.
- Fault handling, exceptions, and termination guarantees.

Repeated offsets may suggest structured storage, but that is inferred intent rather than an established object layout. Scaled values may represent addresses, handles, or encoded references; the evidence does not select among those interpretations.

## Evidence

Established evidence is limited to the supplied static representation: explicit widths, masks and shifts, direct memory-access ordering, indirect-call sequencing, repeated arguments, global assignments, and terminal actions.

There are **three distinct 64-bit call results decomposed into 32-bit halves**:

1. The result forwarded as low half, high half, and a shifted retained word.
2. The result whose scaled low half is stored using its high half as an index.
3. The result whose high half is repeated in a call and whose low half controls later selector and halfword addresses.

Range and guaranteed-zero conclusions follow from the displayed unsigned operations. Application intent, ownership, platform fault behavior, and successful execution are not established by those operations.

No runtime tests have been performed. No dependency implementations, independent execution observations, or external interface definitions were supplied.
