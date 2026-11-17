<!-- Trusted provenance: F-f2b4d3c0d4a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2b4d3c0d4a8

## Purpose

F-f2b4d3c0d4a8 performs a fixed represented sequence of indirect invocations and width-sensitive transformations, followed by ordered memory mutations and a final unresolved indirect transfer.

No domain-level purpose is established. The available material does not justify identifying the function as a serializer, allocator, constructor, formatter, or device initializer. Character-like constants and recurring offsets do not establish text output or a record format.

This contract distinguishes:

- **Established static evidence:** represented inputs, operations, access widths, data dependencies, and ordering.
- **[INFERENCE]:** deductions conditional on an explicitly stated execution model.
- **documented_only:** unresolved behavior, not an invented implementation requirement.

No runtime tests have been performed.

## Interface

### Explicit inputs and output

| Input | Represented width | Established use |
|---|---:|---|
| First argument | Signed 32-bit quantity | Shifted left by 11 and supplied twice to the first indirect invocation. |
| Second argument | Uninterpreted 32-bit quantity | No explicit use in the represented body. |
| Third argument | Signed 32-bit quantity | Shifted left by four and supplied to an early indirect invocation. |

The explicit arguments are not directly dereferenced as buffers. No capacity, count, string-termination rule, or accepted argument range is established.

The represented interface provides no explicit return value. Observable outputs are dependency effects, memory accesses, and the final transfer. An ordinary return is represented afterward, but whether the final destination returns under the applicable calling convention is **documented_only**.

The absence of an explicit use of the second argument does not establish that its incoming machine-register contents are irrelevant to incompletely represented dependency interfaces.

### Implicit inputs and preconditions

Execution also depends on an incoming execution-context value, computed indirect targets, dependency results, additional register outputs, existing memory, and a global word sampled late as a memory base.

Completion requires all reached targets to be executable and all reached addresses to support the indicated accesses. Memory-derived values used as addresses must remain usable when accessed. Low-address mappings and potentially unaligned word accesses are required by the represented path.

No checks enforce these conditions. Arithmetic, narrowing, and shifts require defined target semantics. There is no established null convention, validation result, ownership-transfer rule, or rollback guarantee.

### Widths and layout

Explicit data-memory accesses are 8-bit bytes, 16-bit halfwords, and 32-bit words. Narrow stores retain the low eight or 16 bits. Unsigned byte loads used as word values are zero-extended.

Offsets are byte offsets. Repeated offsets do not establish a structure layout: bytes can overlap words, loaded bytes become address bases, and scaled integers become addresses. Endianness, alignment behavior, and access atomicity remain **documented_only**.

## Behavior

### Invocation and transformation phase

A long straight-line invocation phase precedes the first explicit data-memory store. No explicit loop or conditional branch is represented in that phase; dependencies may themselves branch, mutate memory, or fail to return.

The opening invocation receives two copies of the first argument shifted left by 11. Subsequent invocations include an explicit pair of zeros, a prior result shifted left by eight alongside zero, and another result shifted left by 15 supplied twice. Later, the third argument shifted left by four is supplied alongside an unresolved register output and an explicit zero third argument.

Similar argument shapes do not establish identical destinations or interchangeable helpers. Invocations with unused results remain potentially observable.

The phase repeatedly combines:

- Results shifted left by eight, sometimes reused after intervening invocations.
- Duplicate arguments and explicit zeros.
- Low and high 32-bit halves of represented 64-bit results.
- Saved older results rather than exclusively the newest result.
- Masked fields and additional register outputs with unavailable producing contracts.

Masks select low fields of widths including 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Subsequent shifts must retain their represented widths and sequencing. For example, selecting the low 20 bits and shifting logically right by ten produces 0–1,023; doing so with 19 bits produces 0–511.

Unsigned 64-bit right shifts by 34, 36, and 44 extract the high half shifted logically right by two, four, and 12 respectively. They are not shifts of the low half alone.

**[INFERENCE]:** Under 32-bit bit-vector arithmetic, scaling and left shifts discard bits beyond bit 31. Sequential left shifts by 28 and 28 therefore produce zero, not a retained 56-bit shift result. This model must not be silently substituted for portable signed-language semantics: several represented operands are signed, and overflow and negative left shifts remain target-dependent.

### Analytical memory roles

The following names describe dataflow, not recovered objects or source identifiers:

| Role | Definition |
|---|---|
| Primary base | A saved invocation result multiplied by \(2^{25}\). |
| Link base | A separate saved result shifted left by 25. |
| Saved shifted value | A later invocation result shifted left by eight. |
| Extracted field | Another result shifted left by eight, masked to 15 bits, then shifted logically right by six. |
| Secondary base | Twice the extracted field. |
| Final result pair | Low and high 32-bit halves of the last represented 64-bit result before explicit stores. |

**[INFERENCE]:** With 32-bit bit-vector arithmetic, the primary base depends on at most seven input bits. The extracted field is a multiple of four in 0–508; the secondary base is a multiple of eight in 0–1,016.

These ranges permit accesses near address zero and do not support ordinary heap-object assumptions.

### Initial mutations and the early reload

The initial link publication stores the link base as a word at the final pair’s high half plus 0x14, then stores that high half’s low byte at link base plus 0x09. A word at 256 times the pair’s low half plus 0x54 receives the primary base. The link-base byte is then overwritten with the link base’s own low byte.

The primary-base word at 0x14 is cleared before its byte at 0x05 is read. The secondary-base word at 0x14 is then cleared, and the saved byte is zero-extended into a word at primary base plus 0x34. Only afterward is primary base plus 0x14 reloaded.

That reload remains a distinct represented access. Under ordinary, quiescent 32-bit memory semantics, it reads zero: the intervening secondary-base store writes only zeros, and the word at primary base plus 0x34 cannot overlap the word at plus 0x14. Aliasing among these explicit stores alone cannot introduce a nonzero result. A different value requires separately unresolved device, concurrency, or memory semantics.

After the reload, secondary base plus 0x14 is cleared again. Two distinct identical byte stores then place the reload’s low byte at secondary base plus 0x15.

### Fixed-location interleavings and scaled links

The unsigned byte at absolute 0x05 is sampled next and retained as a small address base. This read precedes the byte write at absolute 0x01, which receives the secondary base’s low byte. The following computed byte write, at the sampled base plus 0x15, receives the extracted field’s low byte.

The primary-base byte at 0x11 receives the saved shifted value’s low byte. A word at the early reload value plus 0x54 receives the full saved shifted value; primary base plus 0x11 is then cleared.

The word at the sampled small base plus 0x54 is loaded and saved as the **scaled-link seed**, then replaced by the early reload value. Secondary base plus 0x34 receives the extracted field.

Further accesses derive bases from primary base plus 0x74 and secondary base plus 0x74. They publish words at those loaded bases plus 0x54 and repeat stores at scaled-link seed plus 0x24 and another loaded base plus 0x09. The secondary-base word at 0x74 is represented as being read twice for one link operation; it is not established as a single reusable snapshot.

The word at secondary base plus 0x14 first receives the saved primary-producing result shifted left by 26. The subsequent absolute word write at 0x54 receives twice the scaled-link seed. Secondary base plus 0x14 then receives the unscaled seed twice. Two byte stores at secondary base plus 0x15 receive the low byte of four times the seed. Only afterward does absolute byte location 0x15 receive the secondary base’s low byte.

Thus, the absolute word write at 0x54 precedes the absolute byte write at 0x15, with computed writes between them. Neither fixed write may be moved across those computed accesses on an assumption of disjoint addresses.

### Computed linking and constant publication

Let the **quadrupled base** mean four times the scaled-link seed, and the **octupled value** mean eight times that seed. Subsequent linking uses both values, a doubled word loaded from secondary base plus 0x74, and further bases obtained through byte and word loads.

The represented effects include self-links, low-byte address publication, repeated writes, and replacement of previously stored words. Important reads include:

- An unsigned byte at quadrupled base plus 0x05, followed by a word at that byte-derived base plus 0x74.
- Words at quadrupled base plus 0x64 and plus 0x04.
- Two separate byte reads at the latter loaded base plus 0x05, with computed writes between them.

The second of those byte samples selects another word at offset 0x74. Call that loaded word the **terminal link seed**, and twice it the **terminal base**. The two samples are not guaranteed equal when intervening writes alias their source.

Numeric 0x3A is subsequently written, in order, as a byte at four times the terminal link seed, a halfword at eight times that seed, and a word at four times that seed plus 0x20.

The **adjusted base** is the quadrupled base minus 0x96; the **working base** is four times the adjusted base. At working base plus 0x20, the ordered word writes are 0x20, 0x2F, an unsigned byte sampled from working base plus 0x04, and 0x20 again.

The publication-base global is read after these writes. Its saved value becomes the **publication base**. This global read precedes the absolute word write of 0x20 at 0x60, which in turn precedes the repeated publication region. If the global aliases 0x60, the publication base is the value sampled before that write.

Each publication group orders a word link, a byte at publication base plus 0x19 containing the working base’s low byte, a byte at working base plus 0x11 containing the publication base’s low byte, and a word at publication base plus 0x34.

The initial word link is at octupled value plus 0x54 and contains four times the terminal link seed; the group’s last word contains the octupled value. Later groups instead use eight times the adjusted base for both the link’s address base and the final word value. Saved byte values are reused; no intervening invocation is represented.

The region is finite and explicitly repeated. Its exact multiplicity is **documented_only** here; this contract does not authorize collapsing it to one group or claim complete trace equivalence.

### Post-publication snapshots and absolute 0x67

After publication, a word is saved from absolute 0xA9, followed by a word from four times the terminal link seed plus 0x74. Absolute byte location 0x3A is then written with the low byte of eight times the adjusted base plus 0x30.

Only after that byte write are words saved from terminal base plus 0x14 and working base plus 0x04. This interleaving matters if computed accesses alias 0x3A. The terminal-base snapshot at 0x14 is retained for the final transfer.

Initial post-snapshot updates place the working base’s low byte at terminal base plus 0x0D, numeric 0x6C at working base plus 0x08, and the publication base’s low byte at working base plus 0x0D and plus 0x09. Terminal base plus 0x54 receives the terminal base.

Absolute byte location 0x67 then receives the low byte of the saved absolute-0xA9 word. Its subsequent behavior involves **three distinct reads**, not continued reuse of that saved byte:

1. A word is loaded from terminal base plus 0x44. Absolute 0x67 is then read for the byte copy to absolute 0x21.
2. Computed writes follow: a byte at working base plus 0x10, publication-base word updates, a variable-shift-derived byte at terminal base plus 0x01, and replacement of the working-base word at 0x10 using a word read at 0x40. After a word update at publication base plus 0x44, absolute 0x67 is read again for a byte store at working base plus 0x09.
3. A byte at terminal base plus 0x09 receives the working base’s low byte; working base plus 0x44 receives the saved absolute-0xA9 word; and a word is loaded from working base plus 0x54. Absolute 0x67 is then read a third time for another byte store at working base plus 0x09.

Each read samples memory at its own position. Intervening computed writes may alias 0x67, so the two later samples need not equal either the original saved low byte or each other.

The variable shift uses the signed word loaded from terminal base plus 0x44 and a count selected from the working base’s low eight bits. Counts can exceed 31; target behavior remains **documented_only**.

### Final updates and transfer

The word loaded from working base plus 0x54 becomes the **late linked base**. Its byte at 0x0D receives the low byte of the earlier post-publication 0x74 snapshot. Terminal base plus 0x54 is restored from its saved 0x14 snapshot.

A word is then saved from working base plus 0x14. Further byte and word publications update terminal base plus 0x11 and working base plus 0x09 and plus 0x44.

A word loaded from terminal base plus 0x04 becomes the **final record base**. A halfword at twice that base receives 0x65. The word at final record base plus 0x64 is loaded before the late linked base receives the working-base 0x14 snapshot’s low byte at 0x11 and decimal 100 as a word at 0x54.

The word at final record base plus 0x74 is then saved. Another base is loaded from terminal base plus 0x64. A byte write at terminal base plus 0x09 precedes a word link at that newly loaded base plus 0x54, followed by a word write at twice the saved final-record 0x64 value.

The final unresolved transfer receives four represented arguments:

1. 0x72.
2. The late word loaded from final record base plus 0x64.
3. The earlier snapshot from terminal base plus 0x14.
4. The late word loaded from final record base plus 0x74.

The third argument is a retained snapshot, not a final reload.

## Side effects

Explicit memory effects include fixed-location reads and writes, computed links, overlapping narrow stores, overwritten intermediate values, and repeated publication. A final-memory-state-only implementation does not preserve all established ordering.

The absolute word read at 0xA9 is not naturally four-byte aligned. No assumption of successful unaligned access is justified.

No recognizable allocation or release contract is established. Unknown dependencies may modify globals, access or release memory, allocate resources, or fail to return. Their effects are not bounded by this function’s explicit accesses.

There is no explicit validation branch, error return, rollback, transaction, or recovery path. Faulting accesses, invalid destinations, unsupported arithmetic, and dependency failures may terminate execution after partial mutation.

## Dependencies

F-f2b4d3c0d4a8 is the only supplied opaque function identity. No opaque dependency IDs are established; assigning fabricated identities would imply unsupported resolution.

| Dependency relationship | Established evidence | Unresolved contract |
|---|---|---|
| Initial indirect destination | Target incorporates incoming execution context and a displacement of 4,094 bytes | Identity, signature, effects |
| Intermediate destinations | Computed targets invoked in fixed represented order | Identity equivalence, register conventions, global and memory effects |
| Result producers | Supply word results, paired halves, or additional register outputs | Meaning, validity, ownership, preservation |
| Final destination | Receives the four arguments specified above | Transfer convention, effects, return behavior |

No dependency is assumed to preserve globals or to own, retain, or release any memory.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Opening arguments:** With first argument 1 and defined shifts, the first invocation receives 2,048 twice. Third argument 3 produces 48 in the relevant early argument position.

2. **Extraction boundary:** Selecting bits 10–19 from 0x000FFC00 yields 1,023; an input containing only bit 20 yields zero.

3. **Early reload:** In ordinary quiescent memory, the reload at primary base plus 0x14 returns zero even if the secondary-base zero store overlaps it. A nonzero expectation requires additional memory or concurrency semantics.

4. **Fixed-location ordering:** The absolute-0x05 byte sample occurs before the absolute-0x01 write and the following sampled-base write. The absolute-0x54 word receives twice the scaled-link seed before absolute byte 0x15 receives the secondary base’s low byte.

5. **Global snapshot:** If the publication-base global overlaps absolute 0x60, the publication stores use the value read before 0x20 is written there.

6. **Post-publication interleaving:** The absolute-0xA9 and computed-0x74 word snapshots precede the absolute-0x3A byte write. The terminal-base 0x14 and working-base 0x04 snapshots follow it.

7. **Resampling 0x67:** If a computed write changes 0x67 between its first and second reads, the first later working-base byte store uses the changed sample. If another intervening write changes it again, the next store uses the third sample.

8. **Narrowing and zero-extension:** A sampled unsigned byte 0xFE becomes word value 254. A byte store of a word ending in 0xAB modifies only the addressed byte.

9. **Saved final argument:** Changing terminal base plus 0x14 after its snapshot does not replace the represented third final argument.

10. **Partial execution:** A dependency that does not return imposes no requirement to perform later stores or complete normally.

## Uncertainties

The following remain **documented_only**:

- Domain intent and external API meaning.
- Dependency identities, complete interfaces, and additional register-output provenance.
- Target overflow, address arithmetic, signed-shift, and oversized-shift semantics.
- Endianness, mappings, alignment, atomicity, device behavior, and concurrency.
- Exact repeated-publication multiplicity.
- Ownership, allocation, and release behavior.
- Final-transfer return semantics.
- Whether the unusual represented addresses and repetitions faithfully describe executable behavior.

These limitations do not erase established access order or permit treating resampled memory as saved values.

## Evidence

The contract is based on the supplied static body, independently of the prior draft, comments, warnings, and reviewer assertions.

Established evidence includes represented argument widths, indirect invocation relationships, masks and shifts, result-half extraction, explicit memory widths, fixed-location interleavings, the three reads of absolute 0x67, snapshot reuse, and the final four-argument transfer.

Arithmetic range and non-overlap conclusions are conditional on the stated ordinary 32-bit memory model. No successful execution, runtime output, test result, dependency ownership, or domain interpretation is asserted.
