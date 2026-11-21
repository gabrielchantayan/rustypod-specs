<!-- Trusted provenance: F-7c8bcf8896ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c8bcf8896ed

## Purpose

F-7c8bcf8896ed performs an ordered sequence of indirect dependency calls, transformed-value propagation, and memory reads and writes. The supplied static representation establishes these interactions, but not an application-level purpose.

This contract does not classify the function as an initializer, parser, allocator, dispatcher, or cleanup routine. Repeated offsets and scaling factors do not establish an object model.

Every effect is conditional on execution reaching it. A dependency may not return, and a computed address or call target may be invalid. No runtime tests have been performed.

## Interface

### Explicit interface

- Explicit parameters: none.
- Explicit return value: none.
- Explicit error result: none.
- Terminal interaction: a call to address zero with three zero arguments.
- Normal completion: conditional on that terminal call returning and execution reaching the represented return.

No explicit validation, recovery branch, or error-status handling is represented.

### Implicit inputs and storage

Implicit inputs include incoming control-link state, two incoming address bases, primary and secondary dependency-result state, and memory selected through those values.

Three global storage roles are used below:

| Role | Represented access |
|---|---|
| **G-A** | A 32-bit zero write, followed much later by two separate 32-bit reads |
| **G-B** | An intermediate 32-bit write and a final composed 32-bit write |
| **G-C** | A logical 24-bit read contributing bits 31–8 of the final G-B value |

These are descriptive roles, not assertions of disjoint storage. The supplied representation warns that global symbols overlap.

Other inputs include the byte at absolute address 1 and an address identifying a stack-resident location, passed to a dependency.

### Widths and arithmetic

The representation includes 8-, 16-, and 32-bit memory accesses **and a logical 24-bit read of G-C**. The final composition consumes all 24 represented G-C bits. The physical load or loads implementing that read, their decomposition and order, and their atomicity are not established. A logical 24-bit read must not be interpreted as proof of one physical 24-bit load.

Paired dependency results are represented as 64-bit quantities split into low and high 32-bit portions. This does not establish a 64-bit memory access. Secondary result state is distinct from either portion unless explicitly identified otherwise.

Unsigned byte reads produce values from 0 through 255. Byte writes retain the low eight bits. Other eight-bit quantities and the 16-bit quantity passed to dependencies have unspecified argument-extension conventions.

Unsigned right shifts introduce zero bits. Masks and unsigned extractions have their represented 32-bit meaning. Some multiplication and left-shift expressions have signed 32-bit representations; portable overflow semantics are not established. Simplifications using wraparound are conditional on 32-bit modular machine arithmetic.

Endianness, physical address width, alignment requirements, and calling convention remain documented_only.

### Interaction terminology

The following relationships describe addresses without asserting valid objects or bounds:

- **Indexed word destination selected by a value:** four times that value, plus 20.
- **Sixteen-scaled word destination:** sixteen times that value, plus 20.
- **Selected-byte write:** read an unsigned selector byte at the specified selector address, then write a byte at the loaded selector plus 17.
- **Field copy at a base:** read a 32-bit word at base plus 116, then write its low byte at base plus 1.

Selected-byte destinations range from addresses 17 through 272 inclusive. Their selector addresses need not lie in that range.

A **cycle** consists of a call with the specified ordered arguments followed by exactly four no-argument calls. Its retained result, when needed, comes from the fourth no-argument call.

A **relay cycle** has one additional preceding no-argument call whose primary result becomes the cycle’s first argument. Any additional cycle arguments are stated explicitly.

These terms preserve call multiplicity. They do not permit merging calls, removing producers of unused results, or assuming equivalent dependency behavior.

## Behavior

### Ordering and dependency identity

The represented body has no explicit branches or loops. The regions below describe its observable order.

Each call site has an opaque identifier of the form F-D01-001, with the middle number identifying the region and the final number counting calls in execution order within that region. Cycles and relay cycles expand into their constituent calls for this numbering. These identifiers distinguish sites, not proven distinct callees.

Dependencies have unknown effects on memory and globals. A retained value is therefore not interchangeable with a later memory read or later secondary result.

### 01. Opening propagation

Two no-argument calls occur first. The first target is incoming control-link state plus 4094; the second is derived from a fixed internal anchor. The secondary value available after the second call is the **opening secondary value**.

That value is written as a word at its own indexed destination. A word is then read at the first incoming base plus 20.

Exactly four propagation cycles follow:

1. The first receives the opening secondary value and the loaded unsigned word shifted right by 19. Its result selects an indexed destination receiving the retained opening secondary value, not a newly obtained secondary result.
2. The second receives four times the preceding selector. Four times its result becomes the **opening scale**, written at the result’s sixteen-scaled destination.
3. The third receives the opening scale, a fresh unsigned word at opening scale plus 20 shifted right by 19, and zero. Its result selects an indexed destination receiving the saved opening scale.
4. The fourth receives four times that selector and supplies the **early seed**.

There is no fifth opening propagation cycle.

### 02. Early-seed extraction and seven-argument interaction

A selected-byte write using the second incoming base plus 1 stores the low byte of four times the early seed.

A cycle receives that four-times value twice, followed by the opening scale. The selector at the second incoming base plus 1 is then reread. Its selected destination receives the low byte of the early seed masked to 30 bits and shifted right by 20.

Another cycle receives four times the preceding result. A subsequent call receives two copies of four times the new result. One no-argument call supplies the **repeated value**.

The next call receives seven ordered arguments:

- Four copies of the repeated value.
- The early seed masked to 30 bits and shifted right by 17.
- The same masked seed shifted right by 7.
- An internal anchor value.

The final argument is the anchor itself, not the displaced call target. Both seed extractions remain retained. One no-argument call supplies the **first paired result**.

### 03. First paired result and byte consumption

A selected-byte zero write uses the saved right-by-7 extraction plus 1. A call receives the first paired high portion twice, followed by the repeated value; a relay cycle follows.

The same selector address is reread for another zero write. A cycle receives four times the preceding result. A call receives four times the new result, followed by one no-argument call. The next call receives that no-argument call’s secondary result twice, then four times the repeated value.

After another no-argument call, a cycle receives the first paired low portion shifted right by 28. A selected-byte zero write uses that low portion shifted right by 9, plus 1.

A cycle receives four times the preceding result. A call receives four times its result and the saved four-times-repeated value shifted left by 31. After one no-argument call, an unsigned byte is read at twice the first paired low portion. Four copies are passed to a call, followed by one no-argument call whose secondary result is retained.

The right-by-9 selector is reread for a zero write. A cycle receives the retained secondary result and zero. A selected-byte zero write then uses absolute address 1.

A cycle receives four times the preceding result. A call receives two copies of four times its result and the loaded byte shifted left by 31. One no-argument call supplies secondary state whose unsigned right-by-18 extraction is passed to a separately displaced dependency. Its primary result is passed in four positions to another call.

One no-argument call supplies the **second paired result**.

### 04. Second and third paired-result regions

A selected-byte zero write uses the retained secondary value from region 03 shifted right by 9, plus 1. The second paired high portion is passed to a call, followed by a relay cycle.

A selected-byte zero write uses the second paired low portion plus 1. A cycle receives four times the preceding result. A call receives two copies of four times its result, followed by one no-argument call.

A word is read at the second paired low portion plus a displacement formed by shifting that low portion unsigned-right by 19 and then left by 31. Under 32-bit modular arithmetic, the displacement is either zero or $0x80000000$, not unrestricted index multiplication.

Four copies of the word are passed to a call. One no-argument call supplies the **saved secondary base**. The subsequent ordered interactions are:

- Selected-byte zero write using the second paired low portion shifted right by 9, plus 1.
- Cycle receiving the saved secondary base.
- Selected-byte zero write using that base plus 1.
- Cycle receiving four times the preceding result.
- Call receiving two copies of four times its result, then one no-argument call.
- Eight-bit read at saved secondary base plus 14, passed in four positions.
- No-argument call supplying the **third paired result**.

A selected-byte zero write uses the saved secondary base shifted right by 9, plus 1. The third paired high portion is passed to a call, followed by a relay cycle.

A selected-byte zero write uses the third paired low portion plus 1. A cycle receives four times the preceding result. A call receives two copies of four times its result. Its following no-argument call produces a primary value passed in four positions to another call. One more no-argument call supplies the **secondary address value**.

A selected-byte zero write uses 4096 times the third paired low portion, plus 1. A cycle receives the secondary address value in positions one, two, and four, with its unsigned right-by-19 extraction in position three.

A selected-byte zero write uses that address value plus 1. A cycle receives four times the preceding result. A call receives two copies of four times its result, followed by one no-argument call.

The same stack-location address is then passed in four positions to a call. One no-argument call supplies the **fourth paired result**.

### 05. Halfword consumption

A selected-byte zero write uses one plus the secondary address value’s right-by-19 extraction multiplied by $0x80000000$. Under 32-bit modular arithmetic, only the extraction’s low bit affects the displacement.

The fourth paired high portion is passed to a call, followed by a relay cycle. A selected-byte zero write uses the fourth paired low portion shifted right by 19, plus 1.

A cycle receives four times the preceding result. A call receives two copies of four times its result, followed by one no-argument call. An unsigned byte is read at the fourth paired low portion plus the resulting secondary displacement. Four copies are passed to a call. One no-argument call supplies the **halfword base**.

Before consumption of that base, a selected-byte zero write uses the fourth paired low portion shifted right by 9, plus 1. A cycle receives the halfword base. A selected-byte zero write uses halfword base plus 1. Another cycle receives four times the preceding result.

A call receives the halfword base shifted left by 31 first and four times the new result second. After one no-argument call, a halfword is read at the halfword base’s unsigned right-by-19 extraction plus the previously loaded unsigned byte. Four copies are passed to a call. One no-argument call supplies the **fifth paired result**.

Its high portion selects an indexed destination receiving the halfword-base extraction. A cycle receives the fifth paired low portion, high portion, and zero. Its result selects another indexed destination receiving the same extraction.

A cycle receiving four times that selector supplies the **table seed**. The fifth paired low portion remains the **field base**.

### 06. Table reads and scaled-word propagation

The byte at field base plus 13 first receives the low byte of twice the field base. A word at four times the table seed plus 68 is passed to a cycle.

The same byte is overwritten with 4. A cycle receives four times the preceding result and a word read at 68 plus the table seed masked to 30 bits and shifted right by 20.

This cycle supplies the **wide selector**. A word at 256 times that selector plus 20 receives four times the table seed. A cycle receives two copies of 64 times the wide selector and the table seed’s masked right-by-17 extraction shifted left by 31.

Its result selects an indexed destination receiving the wide selector masked to 26 bits and shifted right by 13. A cycle receives four times that selector.

Three scaled-word segments follow. Each writes four times its incoming result at that result’s sixteen-scaled destination, obtains another selector through a cycle, writes the saved four-times value at the selector’s indexed destination, and finishes with a cycle receiving four times that selector.

Their first-cycle arguments differ:

| Segment | Ordered arguments |
|---|---|
| First | Four-times value; original result shifted left by 24 |
| Second | Four-times value twice; original result shifted left by 24 |
| Third | Four-times value twice; zero; original result shifted left by 22 |

The third segment’s original result and left-by-22 transformation remain retained.

A word zero is written at sixteen times that original result plus 36. A word at sixteen times the same result plus 116 is read as the **middle field base**.

A separately displaced call receives the middle field base, four times the last cycle result, and the retained left-by-22 transformation. One no-argument call supplies the **middle extraction seed**.

### 07. Middle-field updates and offset-21 reads

A cycle receives the middle extraction seed. Its unsigned right-by-20 extraction, at most 4095, is stored as a word at middle field base plus 36. A fresh word at middle field base plus 116 is passed to a cycle.

A selected-byte write using middle field base plus 1 stores the earlier result retained at the end of the scaled-word chain, masked to 30 bits and shifted right by 20.

A cycle receives the middle extraction shifted right by another 21, followed by the saved left-by-22 transformation. Its first argument is zero. A selected-byte zero write rereads middle field base plus 1.

Three separate unsigned byte reads at secondary bases plus 21 follow. Each is preceded by a cycle receiving four times the preceding result, a two-argument call, and a no-argument call. The first two two-argument calls receive duplicate four-times results; the third receives zero first and its four-times result second.

For the first byte, a consuming call and no-argument call precede a selected-byte zero write using 1024 times the middle extraction plus 1. A separately displaced call receives the saved byte. A relay cycle follows with zero as its second argument. The extraction-based selector is reread for zero.

For the second byte, a consuming call and no-argument call precede another extraction-based zero write. That no-argument call’s secondary state supplies the **bounded selector**, masked to 29 bits and shifted right by 18. A cycle receives the second byte, followed by another extraction-based zero write.

The third byte is passed to a call followed by one no-argument call. Its secondary result selects an indexed destination receiving the zero-extended byte.

A cycle receives this selector and propagates the selector itself to another indexed destination. A cycle receiving four times the new selector supplies a sixteen-scaled four-times store.

The bounded selector shifted left by 13 passes through a cycle and is stored at its result’s indexed destination. A cycle receiving four times that selector supplies another sixteen-scaled four-times store.

A cycle receives the saved four-times value and propagates it to an indexed destination. A final cycle receives four times that selector and supplies the **later seed**.

### 08. Later-seed propagation and G-A reset

A selected-byte write using 1024 times the bounded selector plus 1 stores the low byte of four times the later seed. A separately displaced call receives that four-times value, followed by a relay cycle.

A selected-byte write using absolute address 1 stores the later seed’s masked right-by-20 extraction. A cycle receives four times the preceding result. A call receives two copies of four times its result; a no-argument call supplies a primary value passed once to another call, followed by another no-argument call.

A selected-byte zero write uses one plus the four-times-later-seed value masked to 29 bits and shifted right by 6. The latest secondary result is retained as the **shift source**.

A separately displaced call receives the shift source, followed by a relay cycle. The same selector is reread for zero. A cycle receives four times the preceding result. A duplicate-four-times call and no-argument call supply a primary value passed once to another call, followed by one no-argument call.

The latest secondary result selects an indexed destination receiving that saved primary value. A cycle receives the secondary selector and propagates it to another indexed destination. A cycle receiving four times the new selector supplies a sixteen-scaled four-times store.

A cycle receives this saved four-times value. Its result selects an indexed destination receiving that saved value masked to 30 bits and shifted right by 19, at most 2047. This stored extraction is not the source of region 10’s zero-valued shifts.

A cycle receives four times the selector and supplies another sixteen-scaled four-times store. A fresh word at that result’s indexed destination, unsigned-shifted right by 21, is passed to a cycle.

Its result selects an indexed destination receiving the shift source cumulatively shifted left by 7. A cycle receives four times that selector. Only after its four no-argument calls is zero written to G-A.

### 09. Post-reset propagation and zero working state

A cycle receives two zeros and selects an indexed destination receiving word zero. A cycle receiving four times that selector supplies a sixteen-scaled four-times store.

A cycle receives this four-times value twice and a fresh unsigned word at that value plus 20 shifted right by 21. Its result selects an indexed destination receiving the saved four-times value. A cycle receives four times that selector.

The new result supplies another sixteen-scaled four-times store. A working value is explicitly replaced with zero. A cycle receives three zeros and a fresh word from the result’s indexed destination masked to 29 bits and shifted right by 18. Its result selects an indexed destination receiving zero.

Further propagation comprises a cycle on four times that selector, a sixteen-scaled four-times store, a cycle on the saved four-times value, an indexed store of that value, and another cycle on four times the selector.

The next result supplies a sixteen-scaled four-times store. Its left-by-6 transformation passes through a cycle and is stored at the resulting indexed destination. A cycle receiving four times that selector supplies the **low-address seed**.

A selected-byte write using absolute address 1 stores the low byte of four times this seed. A separately displaced call receives that value, followed by one no-argument call.

The retained zero working value’s masked right-by-19 extraction is passed to a cycle as zero. A selected-byte write using absolute address 1 stores the low-address seed’s masked right-by-20 extraction.

A cycle receives four times the preceding result. A duplicate-four-times call and no-argument call supply the **saved byte base**. A separately anchored call receives this base twice, followed by one no-argument call and a selected-byte zero write using absolute address 1.

A cycle receives the latest secondary result and the zero working value’s extraction. Another selected-byte zero write uses absolute address 1. A cycle receives four times the preceding result; a duplicate-four-times call and no-argument call follow.

An unsigned byte at saved byte base plus 1 is passed to a call, followed by one no-argument call. A selected-byte zero write uses the earlier secondary selector shifted right by 9, plus 1.

A fresh masked right-by-19 extraction of the retained zero working value supplies the **late zero extraction**. A cycle receives the latest secondary result. A selected-byte write using absolute address 1 stores the saved byte base masked to 22 bits and shifted right by 12.

A cycle receives four times the preceding result. A call receives zero and four times its result, followed by one no-argument call. An eight-bit read at the previously loaded unsigned byte plus 1 is passed to a call, followed by one no-argument call.

A selected-byte zero write uses the prior secondary selector shifted right by 9, plus 1. A cycle receives the latest secondary result. A selected-byte write using absolute address 1 stores zero: its masked right-by-12 extraction comes from an unsigned byte.

A cycle receives four times the preceding result. A call receives four times its result and zero, followed by one no-argument call. The unsigned byte at absolute address 1 is read, passed to a call, and retained across one more no-argument call. That last secondary result selects an indexed destination receiving the retained byte as a word.

### 10. Zero-derived propagation and shifted-byte result

A cycle receives the latest secondary selector and propagates it to another indexed destination. A cycle receiving four times the new selector supplies a sixteen-scaled four-times store.

A cycle receives that four-times value. Its result selects an indexed destination receiving the late zero extraction shifted right by 22: zero. A cycle receiving four times the selector supplies another sixteen-scaled four-times store.

A separately displaced call receives the saved four-times value, followed by a relay cycle. Its result selects an indexed destination receiving that saved value. A cycle receiving four times that selector supplies another sixteen-scaled four-times store.

The late zero extraction is shifted right by 1, then by 23, and supplied to a cycle as zero. Its result selects an indexed destination receiving the preceding saved four-times value. These zero consequences derive from region 09’s explicit replacement.

A cycle receives four times the selector. Its result supplies a saved four-times value and a separate left-by-3 transformation. The four-times value is stored at its sixteen-scaled destination. A cycle receives two copies of that value masked to 31 bits and shifted right by 21.

Its result selects an indexed destination receiving the saved four-times value. A cycle receiving four times that selector supplies another sixteen-scaled four-times store.

A cycle receives the new four-times value and the saved left-by-3 transformation shifted right by 22. Its result selects an indexed destination receiving the four-times value. A cycle receiving four times that selector supplies an eight-bit result.

A selected-byte write uses half the saved unsigned four-times value, plus 1, and stores the eight-bit result shifted left by 2, truncated to a byte. Signed interpretation of that eight-bit result remains uncertain.

A separately displaced call receives the right-by-22 transformation, its unshifted source, and another copy of the right-by-22 value. A relay cycle follows with two additional copies of that shifted value.

### 11. Offset-9 accesses and first G-A snapshot

A selected-byte zero write uses the saved right-by-22 value plus 1. A cycle receives four times the preceding result. A call receives four times its result; one no-argument call supplies the **late extraction seed**.

That seed is passed once to a call. One no-argument call supplies the **late field base**. A byte zero is written at 8192 times the saved four-times result from the preceding cycle, plus 9.

A separately displaced call receives the late field base, the late extraction seed shifted right by 22, and the unshifted seed. A relay cycle supplies a paired result.

The byte at late field base plus 9 receives the low byte of that base. A cycle receives four times the paired low portion, the high portion, and a word read at late field base plus 52.

Its final secondary result selects an indexed destination receiving the late field base. A word is read at that unscaled secondary selector plus 20.

A separately displaced call receives the secondary selector, followed by a relay cycle. Its result selects an indexed destination receiving the late field base. A word is read at 20 plus the preceding saved word shifted right by 22.

A cycle receives four times the selector. Four times its result is stored at the indexed destination selected by half the newly loaded unsigned word. A further word is read at the newly loaded word plus 20.

A separately displaced call receives the half-word selector, followed by a relay cycle. Its result selects an indexed destination receiving the saved four-times result.

Immediately after that store, G-A is read as 32 bits and retained as the **first G-A snapshot**, before the next cycle.

### 12. First snapshot consumption and initial G-B write

A cycle receives four times the preceding destination selector. Its final secondary result selects an indexed destination receiving the previously loaded offset-20 word shifted right by 22.

A separately displaced call receives that secondary selector twice, the first G-A snapshot shifted right by 23, and the shifted offset-20 word. One no-argument call supplies a relay value.

The saved snapshot extraction is shifted right by 23 again. This second extraction is zero because the first is at most 511. A cycle receives the relay value and selects an indexed destination receiving the shifted offset-20 word.

A cycle receives four times that selector. Its final secondary result selects an indexed destination receiving zero. A separately displaced call receives this secondary selector, followed by a relay cycle. Its result selects an indexed destination receiving the twice-shifted first snapshot, also zero.

A word is read at 20 plus the preceding secondary selector masked to 29 bits and shifted right by 19. A cycle receives four times the indexed destination selector.

The loaded word shifted right by 22 is stored through a selected-byte write using absolute address 1. A separately displaced call receives three copies of the latest secondary result and zero, followed by one no-argument call.

A cycle receives zero. A selected-byte zero write uses four times the shifted loaded word plus 1. A cycle receives four times the preceding result. A duplicate-four-times call and no-argument call supply a primary result passed once to another call, followed by one no-argument call.

A selected-byte zero write uses absolute address 1. The latest secondary result is the **late shift source**.

A separately displaced call receives three copies of this source. A relay cycle follows with the earlier saved primary result shifted right by 23 as its second argument.

The byte at absolute address 1 is read and retained as the **low base**. A zero byte is written at low base plus 17. A cycle receives four times the preceding result. A duplicate-four-times call and no-argument call follow.

That primary result is written to G-B, retained, and passed once to another call, followed by one no-argument call.

### 13. Repeated field copies and second G-A snapshot

A field copy occurs at low base. A separately displaced call receives late shift source shifted left by 11. One no-argument call supplies a paired result.

A cycle receives its low portion, high portion, and the saved value written to G-B shifted right by 23. Four times its result becomes the **repeated-copy base**.

A field copy occurs at this base. A cycle receives the base. After its four no-argument calls, another field copy rereads the same source word and rewrites the same destination byte.

Four times the latest result is retained as the **late scale**. A separately displaced call receives late shift source shifted left by 12, late scale twice, and the unscaled result shifted left by 18.

One no-argument call, a separately displaced consumer of its primary result, and a relay cycle follow. Four times the final result becomes a new field-copy base. Its word at offset 116 is retained as the **retained copied word**, and its low byte is written at offset 1.

A cycle receives this base. Its result’s sixteen-scaled destination receives the retained copied word.

A separately displaced call receives late scale shifted right by 2 and then left by 16. Another no-argument call, displaced consumer, and relay cycle follow. Its result selects an indexed destination receiving the retained copied word.

A cycle receives four times that selector. Four times its result is stored at 20 plus late scale with its low two bits cleared.

A separately displaced call receives this stored four-times value twice. Another no-argument call, displaced consumer, and relay cycle follow. Its result selects an indexed destination receiving zero.

A cycle receives four times that selector and supplies a paired result. Four times its low portion is stored at the indexed destination selected by its high portion.

Immediately after this store, G-A is read again as 32 bits and retained as the **second G-A snapshot**. This is a separate observation, not reuse of the first snapshot.

### 14. Second snapshot consumption and termination

A cycle receives four times the preceding paired low portion, the high portion, and two further copies of the four-times low value. Its result selects an indexed destination receiving the saved four-times low value.

Only after this intervening cycle and store is a word read at second G-A snapshot plus 20. A cycle receives four times the destination selector and supplies another paired result.

A selected-byte write uses the retained copied word shifted right by 23, plus 1, and stores the low byte of the snapshot-selected word.

A separately displaced call receives three copies of the newest paired high portion and four times its low portion. A relay cycle follows.

A selected-byte write uses one plus the newest paired low portion masked to 30 bits and shifted right by 21. It stores the low byte of the snapshot-selected word shifted right by 22.

A cycle receives four times the preceding result. A duplicate-four-times call and no-argument call follow. A separately displaced no-argument call produces a value passed once to another call.

The next no-argument call supplies an eight-bit result and secondary pointer state. Final visible effects occur in this order:

1. Write a zero byte through the secondary pointer.
2. Consume G-C as a logical 24-bit value and write G-B as a 32-bit composition: G-C contributes bits 31–8; the final eight-bit result contributes bits 7–0.
3. Write a 16-bit zero at the newest paired low portion masked to 30 bits and shifted right by 7.
4. Call address zero with three zero arguments.
5. Return only if that call returns.

The halfword destination is at most $0x007FFFFF$. No alignment check is represented. The logical G-C consumption precedes the composed G-B write; the representation does not specify physical load decomposition or atomicity.

## Side effects

Effects are incremental dependency interactions and memory accesses. No rollback is represented if execution stops partway through.

Repeated observations remain distinct:

- Selected-byte writes reread their selectors unless the selector is explicitly retained as low base.
- The repeated-copy source word is reread after intervening calls.
- G-A’s reset does not establish either later snapshot’s value.
- The first G-A snapshot feeds an argument extraction; the second supplies a later word-read base.
- The final pointer-byte write precedes the logical G-C read and can affect its contribution if storage aliases.

The 24-bit composition specifies bit significance, not byte order or a physical three-byte layout. No indivisible snapshot of G-C, indivisible G-B publication, or synchronization relationship is established.

No explicit bounds checks, null checks, alignment checks, synchronization, or error handling appear.

No allocation, release, or ownership transfer is established. Dependencies may nevertheless allocate, release, retain pointers, or mutate relevant storage. Passing a stack-location address does not establish its initialization, extent, or permissible retention lifetime.

## Dependencies

Only F-7c8bcf8896ed has a supplied verified function identity. F-D01-001 through F-D14-* designate the dependency sites defined by the ordered regions above.

Anchor-derived and separately displaced sites are not collapsed into one dependency. The separately anchored two-argument interaction in region 09 is also distinct. The terminal dependency targets address zero.

For every dependency, application semantics, ownership, memory and global preservation, exception behavior, nonlocal exits, reentrancy, and thread safety remain documented_only. Primary and secondary result associations are established only at the supplied representation’s level.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Opening saved value.** If the opening secondary value is 9 and the first propagation cycle selects 12, its following word store writes 9 at address 68. A different secondary result from that cycle does not replace the retained 9.

2. **Opening argument order.** If the opening secondary value is 9 and the word at the first incoming base plus 20 is $0x00380000$, the first cycle receives 9 first and 7 second.

3. **Shared seed provenance.** An early seed of $0x00060000$ gives masked right-by-17 and right-by-7 extractions of 3 and 3072. If the repeated value is 6, the seven arguments are four copies of 6, then 3, then 3072, then the anchor value.

4. **Selector reread.** If intervening dependency effects change a selector byte from 3 to 7, successive selected-byte writes target addresses 20 and 24. Reusing the first selector changes behavior.

5. **Byte truncation.** An early seed of $0x00000123$ produces $0x0000048C$ when multiplied by four; the corresponding byte write stores $0x8C$.

6. **Zero provenance.** Region 08’s masked right-by-19 store may be nonzero. Region 09 nevertheless explicitly replaces the relevant working value with zero. Region 10’s right-by-22 store and right-by-1/right-by-23 argument consequently remain zero.

7. **First G-A snapshot.** A first snapshot of $0xAB800000$ produces 343 after unsigned right shift by 23. The later four-argument call receives 343 in position three despite intervening changes to G-A. Shifting that retained extraction right by 23 again produces zero.

8. **Second G-A snapshot.** A second snapshot of $0x1000$ selects the later word read at $0x1014$, conditional on valid addressing. Subsequent G-A changes do not replace the retained base. The intervening cycle and propagation store still precede the read.

9. **Repeated field copying.** If a dependency changes the repeated-copy source word between copies, the second copy uses the later word read.

10. **Logical 24-bit composition.** If the final logical G-C read yields $0x123456$ and the final eight-bit result is $0xAB$, G-B receives $0x123456AB$. This does not establish whether G-C was physically loaded in one access or several, nor whether concurrent modification can affect the observation.

11. **Aliasing before composition.** If the final secondary pointer aliases storage contributing to G-C, the zero-byte write occurs before G-C is consumed. Computing the composition from a G-C value cached before that write would not preserve the represented order.

12. **Terminal failure.** If address zero is not executable, normal completion is not promised. The preceding pointer-byte, composed G-B, and halfword writes have already occurred if execution reached the terminal call.

## Uncertainties

The following remain documented_only:

- Application purpose, intended data structures, and object extents.
- Recovered signature validity and machine-level result conventions.
- Argument extension and register-preservation rules.
- Signed overflow, signed left shifts, and signedness of the late eight-bit result.
- Endianness, alignment, address-space mapping, and low-address validity.
- Physical implementation of the logical 24-bit G-C read: load widths, decomposition, constituent-load order, and atomicity.
- Atomicity and synchronization properties of other represented memory accesses.
- Complete aliasing and global-overlap relationships.
- Stack-location initialization, extent, and permissible pointer retention.
- Dependency ownership, allocation, release, mutation, and control-transfer behavior.
- Reachability of the terminal call and represented return.

No guaranteed final memory snapshot follows from the explicit stores alone. Aliasing and dependency effects can alter later observations.

## Evidence

**Established static evidence:** the supplied representation supports the ordered calls, argument multiplicity, paired-result splits, retained-value flows, global observations, absence of explicit branches and loops, and terminal zero-address call described here.

The access-width inventory includes ordinary 8-, 16-, and 32-bit accesses and the logical 24-bit G-C input to the final composition. That composition places the complete represented 24-bit contribution above the final eight-bit result in a 32-bit G-B value. It does not establish the physical instructions or atomicity of G-C’s acquisition.

The opening post-cycle store uses the retained opening secondary value. Exactly four propagation cycles follow the initial two calls.

Region 08’s stored bounded extraction is not region 10’s retained shift source. An explicit zero replacement in region 09 feeds the later zero-valued derivatives.

Both seven-argument seed extractions derive from the same masked early seed.

The two later G-A reads have distinct positions and consumers. The first supplies a saved right-by-23 argument extraction. The second supplies a word-read base consumed after intervening calls and a propagation store.

**Inferred intent:** transformed-value propagation through dependency interactions and memory. No application-specific meaning is assigned.

**Validation limit:** static analysis only. No runtime tests, smoke runs, execution traces, or independent machine-level validation have been performed.
