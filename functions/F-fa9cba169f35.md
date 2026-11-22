<!-- Trusted provenance: F-fa9cba169f35; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa9cba169f35

## Purpose

**Established static evidence:** F-fa9cba169f35 has two explicit 32-bit inputs and no explicit return value. Its visible behavior is an ordered sequence of dependency calls, computed-address memory accesses, indexed stores, and updates to two global storage regions. The final operations include a 16-bit zero store and an attempted call to address zero with three zero arguments.

No application-level purpose is established. In particular, names and repeated access patterns do not establish allocation, construction, parsing, cleanup, or service dispatch.

**Inferred intent:** Repeated selector writes and indexed word updates suggest a state-manipulation protocol. Some accesses resemble descriptor fields. Neither interpretation establishes object boundaries, valid indices, ownership, or dependency equivalence.

This is a static behavioral contract, not a guarantee that execution succeeds in an ordinary address space.

## Interface

### Explicit inputs and output

| Item | Width | Established use |
|---|---:|---|
| First input | Unsigned 32 bits | Logically shifted right by 19 to form the fourth argument of the first dependency call. |
| Second input | 32 bits; interpretation unknown | Passed unchanged as both the first and second arguments of that call. |
| Return | No explicit value | Normal return is reached only if the final zero-address call returns. |

The first call’s arguments, in order, are the second input, the second input again, zero, and the first input shifted right by 19. The extracted argument ranges from 0 to 8191. No other direct use of either explicit input is visible.

### Implicit inputs and preconditions

Execution also depends on incoming machine state supplying:

- An address-bearing base and an incoming byte.
- A control-flow value from which the first call target is derived by adding 0xFFE.
- Existing memory and global contents.
- Primary, secondary, and occasionally 64-bit dependency results.

The complete entry convention and dependency conventions are **documented_only**. Two explicit arguments alone cannot reproduce execution.

Reaching an operation requires every preceding call to return and every preceding access to succeed. Necessary mappings can include low addresses, computed byte locations, potentially unaligned halfword and word locations, global storage, stack-relative storage, and executable call targets. There are no visible bounds, null, alignment, overflow, or executable-target checks.

### Ownership and errors

No allocation, release, ownership transfer, or lifetime extension is established. A dependency receiving an address may modify or retain it; its allocation and release behavior is unknown. One dependency receives the same stack-relative address four times. No safe retained lifetime is established.

There is no explicit status result, error code, recovery branch, or rollback. Faults, traps, nonreturning dependencies, and nonlocal exits remain environment-dependent.

## Behavior

### Ordering, snapshots, and arithmetic

The visible body has no data-dependent branch or loop. Dependencies may have internal control flow and unrestricted unknown side effects.

Calls are sequencing boundaries. Fresh memory reads after calls must not be replaced with earlier reads. Conversely, an explicitly saved value remains the value used by its subsequent visible consumers: a later global or selector change does not turn that saved value into a fresh read.

Widths are significant:

- Selector bytes explicitly treated as unsigned range from 0 through 255.
- Byte stores retain only the low eight bits.
- A 32-bit load subsequently truncated for a byte store remains a four-byte load.
- A halfword load or store accesses two bytes.
- A 64-bit result is sometimes separated into low and high 32-bit halves.

Unsigned 32-bit arithmetic retains the low 32 bits where results exceed that width. Some displayed arithmetic instead has signed intermediates, including left shifts. Portable overflow and negative-value behavior for those expressions is **documented_only**; universal wrapping must not be imposed on them without target evidence.

Multi-byte access widths and numerical values are established, but byte order is not.

### Entry and initial dependency phases

Before any call, an unsigned byte is read at the incoming base plus 1. Its numerical value plus 0x11 selects the destination for the incoming byte. This destination lies between 0x11 and 0x110; it is not relative to the incoming base.

The first dependency receives the four arguments specified above. After further calls, the procedure rereads the byte at the incoming base plus 1 and clears the newly selected destination. Intervening calls may change that selector, so this need not clear the entry-write destination.

Initial call sequences also consume results multiplied by four, sometimes duplicated within one call. An unsigned returned word supplies a saved base equal to its upper 14 bits multiplied by 1024. Two selector-based zero writes use fresh reads at this base plus 1, with calls between them.

Another dependency receives four occurrences of the 32-bit quantity at address 0x30. The representation does not establish the physical read count for those repeated argument expressions.

### Recurring layouts

These are address forms, not established object definitions.

| Address form | Access | Use |
|---|---|---|
| Base + 1, then selector + 0x11 | Byte read, byte write | Clear or store a truncated value through an unsigned selector. |
| 4 × index + 0x14 | 32-bit access | Store results, indices, extracted fields, scaled values, or zero; occasionally read fields. |
| 16 × index + 0x14 | 32-bit store | Commonly store four times that index. |
| Base + 0x24 | 32-bit store | Store zero or an extracted field. |
| Base + 0x74 | 32-bit read | Supply calls, new bases, or truncated byte copies. |
| Base + 0x0D | Byte store | Store a truncated doubled base, then later 4. |
| Base + 9 | Byte store | Store zero or a truncated address-bearing value. |
| Base + 0x15 | Byte read | Forward byte values to dependencies. |

A scale-16 store and a scale-4 read with the same index and offset are distinct accesses. Neither their addresses nor their contents may be conflated.

### Indexed and field-processing phases

The first substantial indexed-store phase writes a secondary result both as the index and as the stored 32-bit value. The following call receives that saved result and a word-derived field: a word is read at a base formed from another result’s upper 14 bits multiplied by 1024, plus 0x14, then logically shifted right by 19.

Further indexed stores preserve earlier results or store fourfold-scaled results. Some scale-16 stores are followed by scale-4 reads, rather than reads back from the just-written location.

A later unsigned result supplies several distinct quantities:

- Four times the result, truncated for a selector-based byte write.
- A 30-bit-masked value shifted right by 20, also truncated for a byte write.
- A 30-bit-masked value shifted right by 17, ranging through 8191.
- A 30-bit-masked value shifted right by 7, ranging through 8,388,607.

A seven-argument call repeats one result four times and also receives the latter two fields and a fixed control-related value. A subsequent 64-bit result supplies separately used low and high halves.

Later operations include:

- Fresh selector reads and zero writes using shifted result-derived bases.
- A byte read at twice a saved unsigned result, forwarded four times.
- A word read at a saved unsigned result plus a high-bit contribution derived from its bit 19.
- A byte read at a secondary-result-derived base plus 0x0E, forwarded four times.
- Four occurrences of one stack-relative address passed to a dependency.
- An unsigned byte read at a saved low word plus a secondary result.
- A halfword read at that byte value plus a separately extracted field, forwarded four times.

The bit-19-derived contribution is a high-address-bit calculation, not a bounded array offset. A signed intermediate in that calculation prevents assuming portable wrapping semantics throughout.

### Descriptor-phase reads at offset 0x44

After halfword processing, two indexed word stores receive the same extracted unsigned field. A saved base from a 64-bit result’s low half is retained for two byte writes at offset 0x0D.

The first byte write supplies the truncated doubled saved base. A later returned unsigned word determines two different word-read addresses:

1. **Before the overwrite with 4:** A 32-bit word is read at four times the returned unsigned word plus 0x44. This word is the sole displayed argument of the following dependency call.
2. **After the overwrite with 4:** Following that call and further calls, the saved base plus 0x0D is written with 4. The following call receives a fourfold-scaled result as its first argument and a second 32-bit read as its second argument. This read uses the returned unsigned word masked to 30 bits, shifted right by 20, then increased by 0x44.

The second read therefore occurs after the byte overwrite, not alongside the first read. Its unscaled field ranges from 0 through 1023, so its address ranges from 0x44 through 0x443. Alignment is not established.

Subsequent updates include a word store at scale 256 plus 0x14 containing four times that saved unsigned word; call arguments scaled by 64; and an indexed store of a 26-bit-masked field shifted right by 13. Repeated scale-16 and scale-4 stores follow, together with high-bit contributions formed by shifts of 24 or 22.

A later scaled base receives a word-sized zero at offset 0x24. A word read at that base plus 0x74 becomes a new saved base.

### Extracted fields and low-address processing

An unsigned result shifted right by 20 is saved and written at the new base plus 0x24. This field is at most 4095. Subsequent arguments using that saved field shifted right by 21 are therefore zero, regardless of later memory changes.

Three distinct byte reads at dependency-derived bases plus 0x15 occur later. Calls and selector writes separate them. The third byte is also stored, unsigned, into an indexed word location. These are not three guaranteed reads from one stable object.

Further indexed phases propagate extracted fields, scaled indices, saved results, and zeros. The first global region is explicitly cleared with a 32-bit store during these phases. Many calls occur before either subsequent explicit read of that region.

Other late operations include a byte read at a returned base plus 1, followed, after calls, by a byte read at the saved byte value plus 1. The latter address is between 1 and 256. It is not a pointer-sized link traversal.

Unsigned bounds establish several zero values without eliminating associated operations. In particular, a 30-bit-masked word shifted right by 19 is at most 2047; shifting that saved field right by 22 yields zero.

### First snapshot of the cleared global

The first explicit post-clear read of the first global is a 32-bit snapshot. It occurs after an indexed word store and before a call receiving that store’s scaled index.

Five dependency calls follow the snapshot before the saved global value is shifted right by 23. In this interval, another saved word is separately shifted right by 22 and stored into an indexed word location.

The next four-argument call receives:

- A secondary-result-derived value twice.
- The first global snapshot shifted right by 23.
- The separately extracted field.

A following dependency supplies another result. The already shifted global snapshot is then shifted right by 23 again. Because its first shifted value is at most 511, this second shift yields zero.

That zero is retained for later selector addressing at zero plus 1, meaning a fresh selector read at address 1. The first shifted snapshot also later supplies an indexed store after another dependency sequence.

These are uses of the saved first snapshot. They are not fresh reads of the global, and they are not necessarily zero merely because the global was cleared earlier.

### Late address-1 snapshot and first publication

Later, after additional calls and a selector clear using a fresh address-1 read, the procedure makes a distinct unsigned byte snapshot of address 1.

The snapshot immediately selects a zero store at its value plus 0x11. It is retained across the following call sequences and across publication of a newly returned 32-bit word into the second global region.

That publication value comes from a dependency result, not from the byte snapshot. The published word is then passed to a dependency, followed by another call.

Only then is the saved byte snapshot reused as a base: a 32-bit word is read at the snapshot value plus 0x74, and its low byte is written at the snapshot value plus 1. Thus:

- The word-read address ranges from 0x74 through 0x173.
- The byte-write address ranges from 1 through 256.
- Changes to address 1 during intervening calls do not replace the saved base.
- This word read is fresh; it was not performed when the selector was saved.

Other late bases receive the same word-at-offset-0x74 to byte-at-offset-1 pattern. One saved scaled base receives that pattern twice, with calls between the occurrences. Each occurrence performs its own word read.

### Second snapshot of the cleared global

Still later, a 64-bit dependency result supplies a low half scaled by four and a high half used as an index. The scaled low half is stored at four times that high half plus 0x14.

The procedure then takes its **second** explicit 32-bit snapshot of the first global.

A four-argument dependency call follows, then four further dependency calls. An additional indexed word store writes the earlier scaled low half. **After that store**, a 32-bit word is read at the second global snapshot plus 0x14.

The dependent read is therefore delayed across five calls and occurs after another indexed store. It must not be moved to the global-snapshot point or replaced with a read based on the global’s current value.

The loaded word is retained across further calls. Its low byte later supplies a selector-based write. After additional calls, its value shifted right by 22 supplies another truncated selector-based byte write.

### Final publication and termination

The final dependency sequence supplies a byte-valued result and a secondary pointer result. The byte location designated by that pointer is cleared first.

The second global region is then written with a 32-bit combination whose low eight bits are the byte result and whose upper 24 bits come from a three-byte quantity beginning at address 9. The byte-to-bit ordering within that three-byte quantity is **documented_only**.

Next, a 16-bit zero is stored at an address formed from a saved unsigned word masked to 30 bits and shifted right by 7.

Finally, address zero is called with three zero arguments. Normal return occurs only if that call returns.

## Side effects

Direct effects include computed byte, halfword, and word writes, the first global’s word-sized clear, and two publications to the second global.

Dependencies may independently alter any accessible memory, globals, machine state, or external resources. No preservation, purity, pointer-retention, allocation, or release contract is established.

Earlier effects are not undone if a later access faults or a call fails to return. No synchronization, atomicity, thread-safety, or reentrancy guarantee is visible. Potentially overlapping global storage prevents treating the two regions as independent objects without further evidence.

## Dependencies

Only the subject’s opaque identity is resolved. The following are documentary references, not verified identities:

| Opaque ID | Scope | Established interface |
|---|---|---|
| F-000000000001 | Entry-state-derived first call | Four arguments specified in Interface. |
| F-000000000002 | Remaining unresolved computed-target calls | Ordered consumers and producers of the values described above. This ID denotes a collection, not one common implementation. |
| F-000000000003 | Separately designated late direct dependency | Receives the same unsigned word twice. |
| F-000000000004 | Final zero-address call | Three zero arguments; execution and return unknown. |

Displayed argument counts do not establish complete machine-level signatures. Secondary results and wide-result conventions require additional evidence. No dependency is assumed to preserve globals or manage memory in a particular way.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input extraction:** An all-ones first input supplies 8191 as the first call’s fourth argument. Changing only its lower 19 bits does not change that argument.

2. **Fresh selectors:** If the entry selector is 7 and a dependency changes it to 9 before the later clear, the writes target 0x18 and 0x1A respectively.

3. **Distinct descriptor reads:** If the descriptor-phase unsigned word is 2, the first offset-0x44 word read targets 0x4C. The second targets 0x44 and occurs only after the saved-base-plus-0x0D overwrite with 4.

4. **First global snapshot:** If the first snapshot is 0xFF800000, its right-shift-by-23 call argument is 511. The subsequent right shift by 23 is zero. Later global changes do not alter either saved calculation.

5. **Saved address-1 selector:** If the late snapshot is 7, it selects a zero store at 0x18. Even if calls change address 1 to 9, the later word read uses 0x7B and the truncated byte write uses address 8. Publication to the second global occurs between these uses.

6. **Delayed second-global-snapshot consumer:** If the second snapshot of the first global is 0x1000, the dependent word read uses 0x1014 after the intervening calls and indexed store. If those operations change memory at 0x1014, the read observes the contents present when it occurs.

7. **Word-to-byte truncation:** A word read yielding 0x123456AB supplies 0xAB to its associated byte store. This does not reduce the original read width.

8. **Terminal fault:** If address zero is not executable, normal return is not required. Previously completed publications and the final halfword clear are not rolled back.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and intended consumer.
- Complete calling conventions and origins of implicit state.
- Individual dependency identities, signatures, and secondary-result provenance.
- Valid memory mappings, alignment requirements, and index ranges.
- Exact global addresses, overlap relationships, and byte order.
- Target semantics for signed overflow and signed shifts.
- Physical read counts for repeated memory expressions within argument lists.
- Dependency retention or release of stack and other addresses.
- Fault handling, nonlocal exits, and zero-target execution behavior.

No unconditional success condition or final global value is established.

## Evidence

This contract uses the supplied static representation’s declared widths, access widths, arithmetic, saved-value dataflow, call order, global accesses, and terminal call.

The two post-clear global snapshots, the retained late address-1 selector, and the two distinct offset-0x44 reads are supported by explicit dataflow and ordering, not by inferred purpose. Comments, labels, and the prior draft are not independent behavioral evidence. Omitted unreachable material supplies no additional contract.

**No runtime tests have been performed.** All acceptance examples are conditional static deductions.
