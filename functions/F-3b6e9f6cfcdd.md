<!-- Trusted provenance: F-3b6e9f6cfcdd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b6e9f6cfcdd

## Purpose

F-3b6e9f6cfcdd performs an ordered sequence of indirect dependency invocations, transforms dependency-produced values, mutates fixed-address and computed-address memory, and makes a terminal four-argument invocation.

**Established static evidence:** the supplied representation declares no explicit parameters or result. Its behavior nevertheless depends on incoming machine state, dependency results, and mutable memory. The direct-access portion includes 8-bit, 16-bit, and 32-bit operations, overlapping accesses, retained loaded values, and repeated stores.

**Inferred intent:** some operations resemble field packing and pointer-like linking. No application-level algorithm, record schema, text format, or resource-management purpose is established. Character-like constants do not establish text output.

This contract describes the visible behavior without assuming that the recovered function boundary or dependency signatures constitute an ordinary callable interface.

## Interface

### Invocation and completion

- **Explicit inputs:** none declared.
- **Implicit inputs:** incoming control-related state used to derive the first indirect target; an incoming 32-bit data value; primary and secondary dependency results; memory contents and mappings.
- **Explicit result:** none declared.
- **Observable outputs:** dependency invocations and their arguments, memory reads and writes, and effects performed by dependencies.
- **Completion:** the visible path reaches a return after the terminal invocation, provided all preceding operations complete and that invocation returns normally.

Zero declared parameters does not mean arbitrary incoming machine state is valid.

### Widths and arithmetic

| Quantity | Established interpretation |
|---|---|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Paired dependency result | 64-bit representation, used as low and high 32-bit portions |
| Computed addresses | Formed from represented 32-bit quantities |
| Unsigned byte widened to word | Zero-extension |
| Value narrowed to byte | Low 8 bits retained |

Unsigned 32-bit arithmetic truncates modulo \(2^{32}\). A left shift performed at that width does not preserve discarded upper bits in a wider temporary.

Some shifts operate on an unsigned 64-bit result before narrowing. Shifts by 34, 36, and 44 therefore correspond to logical shifts of its high word by 2, 4, and 12, respectively. They are not oversized shifts of a 32-bit low word.

Other expressions use signed 32-bit operands. Signed overflow, signed left shifts outside their defined domain, conversion of out-of-range unsigned values to signed values, and negative signed right shifts are **documented_only**. This contract does not silently replace all such expressions with unsigned bit-vector arithmetic.

### Preconditions

Conditional successful execution requires valid indirect targets and valid storage for every executed access. No bounds, capacities, alignment guarantees, non-aliasing guarantees, or ownership annotations are supplied.

Low fixed addresses are accessed directly. A conventional environment in which those addresses are unmapped is not established as a valid execution environment. Some word accesses, including the read at `0xa9`, are not naturally aligned.

## Behavior

### Dependency-driven phase

The entry sequence begins with three indirect invocations having no recovered explicit arguments. The first target is derived from incoming control-related state by an addition of `0xffe`. The fourth invocation receives the incoming data value shifted left by 6.

A long straight-line invocation sequence follows. Its established properties are:

- Invocation order is significant.
- Sites expose different argument counts and result representations.
- Many primary results are forwarded after a left shift by 8.
- Paired results supply shifted low words and separately extracted high words.
- Arguments also include explicit zeroes, transformed secondary results, repeated copies, and values retained across intervening invocations.
- The caller exposes no success check, retry, rollback, or early-return policy.

“No recovered explicit arguments” does not mean all argument registers are zero or irrelevant. Secondary machine results contribute to later arguments even where no ordinary return assignment is recovered.

Transformations include low-24-bit and narrower masks, logical right shifts, and left shifts reaching bit positions near the upper end of a word. Mask widths visible in this phase include 22, 21, 20, 19, 18, 17, 16, 14, 13, 12, and 11 bits. Masks and shifts cannot be rearranged without preserving the original width and truncation points.

The complete invocation-by-invocation signature and argument schedule is **documented_only** here. These families are not a substitute API for the unresolved dependencies.

### Address vocabulary

The following descriptive terms identify arithmetic relationships, not recovered object types:

- **Packed base:** a retained dependency-produced low word shifted left by 25.
- **Shifted base:** a separate dependency result multiplied by \(2^{25}\).
- **Retained shifted value:** another result already shifted left by 8.
- **Selector:** a result shifted left by 8, masked to 15 bits, then logically shifted right by 6.
- **Small base:** twice the selector.
- **Retained link:** a word subsequently loaded through a byte-derived base.
- **Quadrupled link:** four times the retained link.
- **Adjusted quantity:** the quadrupled link minus `0x96`.
- **Working base:** four times the adjusted quantity.
- **External base:** the word loaded from fixed address `0x087fbbbc`.

Under a 32-bit bit-vector interpretation, the selector ranges from 0 to 508 in multiples of 4. The small base ranges from 0 to 1016 in multiples of 8. These are actual address-forming quantities, not established offsets into an allocation.

### Initial direct mutations

The final paired dependency result supplies a high-word-derived address and a low-word-derived scaled address.

The first mutations publish the packed base as a word at offset `0x14` from the high word, publish that high word’s low byte at packed-base offset `9`, and store the shifted base at offset `0x54` from 256 times the low word. Packed-base offset `9` is then overwritten with the packed base’s low byte.

Next, shifted-base offset `0x14` is cleared as a word before the byte at shifted-base offset `5` is read. Small-base offset `0x14` is cleared, and the read byte is zero-extended into a word at shifted-base offset `0x34`.

A fresh word read of shifted-base offset `0x14` follows. Small-base offset `0x14` is cleared again, followed by two distinct byte stores at small-base offset `0x15`, both receiving the low byte of that freshly read word.

The read after the clear remains an access obligation. Aliasing, asynchronous changes, or special memory behavior can distinguish it from substituting a constant.

### Fixed-address interleaving and link mutations

The byte at fixed address `0x05` is read and zero-extended for use as an address base. The small base’s low byte is then written to fixed address `0x01`.

Subsequent computed stores include:

- The selector’s low byte at byte-derived-base offset `0x15`.
- The retained shifted value’s low byte at shifted-base offset `0x11`.
- The retained shifted value as a word at offset `0x54` from the previously read shifted-base word.
- A zero byte overwriting shifted-base offset `0x11`.

The word at byte-derived-base offset `0x54` is then loaded as the **retained link**, before that location is overwritten with the earlier shifted-base word. The retained link remains available after this overwrite.

Further accesses traverse words at offset `0x74`, publish scaled dependency results, and perform repeated self-like and byte stores. Two reads contributing to the store through small-base offset `0x74` remain separate; their relative evaluation order within that single store expression is not established by this document.

Two fixed-address stores have explicit positions among these computed mutations:

- **Word at `0x54`:** after a word store to small-base offset `0x14` of the earlier dependency result shifted left by 26, fixed address `0x54` receives the retained link shifted left by one. Two word stores of the retained link to small-base offset `0x14` follow.
- **Byte at `0x15`:** after those word stores, two byte stores to small-base offset `0x15` receive the low byte of the quadrupled link. Fixed address `0x15` then receives the truncated small base. This fixed byte store precedes a word store of the retained shifted value at quadrupled-link offset `0x54`, and a word store of the retained link shifted left by two at small-base offset `0x34`.

These fixed addresses are not independent fields. They may overlap computed accesses.

The remaining traversal uses doubled, quadrupled, and eightfold retained-link quantities, words reached at offsets `4`, `0x64`, and `0x74`, and bytes reached at offset `5`. It includes distinct repeated reads of a byte through the word loaded at quadrupled-link offset `4`. Intervening stores may change those reads.

A later word loaded at offset `0x74` through such a byte-derived base supplies the **late traversal word**. Twice that word becomes the **final node base**; four times it becomes the **scaled late base**.

### Constants and repeated publication

Before the extended repetition, the late traversal word determines a byte store of `0x3a` at its fourfold address, a halfword store of `0x3a` at its eightfold address, and a word store of `0x3a` at offset `0x20` from its fourfold address.

Working-base offset `0x20` receives, in order:

- Word `0x20`.
- Word `0x2f`.
- A word containing the zero-extended byte read from working-base offset `4`.
- Word `0x20` again.

**The external-base word is then read from `0x087fbbbc`, before the word store of `0x20` to fixed address `0x60`.** The external base is retained across that store; it is not reloaded afterward.

The repeated region publishes a scaled link, the working base’s low byte at external-base offset `0x19`, the external base’s low byte at working-base offset `0x11`, and an eightfold quantity at external-base offset `0x34`.

The initial group uses eight times the original retained link for its link address and final publication. Subsequent groups use eight times the adjusted quantity. Their link payload is the scaled late base. Within each group, the word link store precedes the two byte stores, which precede the external-base word store.

The repetition is statically expanded, not a visible runtime loop. Its exact multiplicity is **documented_only** in this document. Repeated stores cannot be collapsed into one final-state update merely because payloads recur.

### Final mutations and retained reads

The beginning of the final region has this required ordering:

1. A 32-bit read at fixed address `0xa9`.
2. A 32-bit read at scaled-late-base offset `0x74`.
3. An 8-bit write at fixed address `0x3a`.
4. A 32-bit read at final-node-base offset `0x14`.
5. A 32-bit read at working-base offset `4`.

The `0x3a` byte is formed from the adjusted quantity’s low-byte contribution, multiplied by 8 and increased by `0x30`, then narrowed to 8 bits. The later two reads must not be moved before this write.

The word read at `0xa9` is the **retained fixed word**. The read at final-node-base offset `0x14` is retained for later restoration and the terminal invocation.

The function then publishes low working-base and external-base bytes through computed locations, writes `0x6c` at working-base offset `8`, and stores the final node base at its own offset `0x54`. Only afterward does fixed address `0x67` receive the low byte of the retained fixed word.

There are **three later, distinct byte reads of `0x67`**:

- The first follows the word read at final-node-base offset `0x44`; its value is written to fixed address `0x21`.
- The second follows computed word stores involving external-base offsets `0x54`, `0x74`, and `0x44`, and working-base offset `0x10`; its value is written to working-base offset `9`.
- The third follows another byte store at final-node-base offset `9`, a word store of the retained fixed word at working-base offset `0x44`, and a word read at working-base offset `0x54`; its value is again written to working-base offset `9`.

Each reads current memory. None is guaranteed to equal the original byte written to `0x67`: intervening computed stores can alias that location.

Other final obligations include:

- A signed word from final-node-base offset `0x44` is shifted right by the low byte of the working base, then narrowed for a byte store at final-node-base offset `1`.
- A word from working-base offset `0x40` is stored at working-base offset `0x10`.
- The retained fixed word is published as a word at working-base offset `0x44`, before that location is later overwritten with a newly loaded base.
- The retained word from final-node-base offset `0x14` is restored at final-node-base offset `0x54`.
- The low byte of the earlier scaled-late-base `0x74` read is published at computed offsets `0x0d` and `0x11`.

The variable shift has no established masked-count rule. Counts of 32 or greater and negative signed operands remain unresolved.

Finally, a newly read word at final-node-base offset `4` determines a halfword store of `0x65` at twice its value. Reads at that word’s offsets `0x64` and `0x74` supply a downstream base and a terminal argument. A computed offset `0x54` receives decimal `100`. A later read at final-node-base offset `0x64` supplies a value written both at downstream-base offset `0x54` and as a word at twice the downstream base.

The terminal invocation follows the final word write. Its four visible arguments are `0x72`, that downstream base, the retained final-node `0x14` word, and the later `0x74` word. No explicit result is consumed.

## Side effects

The access footprint includes computed byte offsets `1`, `5`, `8`, `9`, `0x0d`, `0x10`, `0x11`, `0x15`, and `0x19`; word offsets include `4`, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, and `0x74`. Halfwords are stored at scaled addresses.

Fixed-address effects include byte reads or writes at `0x01`, `0x05`, `0x15`, `0x21`, `0x3a`, and `0x67`; word stores at `0x54` and `0x60`; and word reads at `0xa9` and `0x087fbbbc`.

This footprint does not declare disjoint storage. A byte at offset `0x15` can overlap a word at offset `0x14`; endianness determines the affected bits.

No direct allocation or release is identified. Dependencies may allocate, release, mutate globals, or modify reachable storage. No ownership transfer, preservation guarantee, synchronization, atomicity, or lifetime policy is established.

No explicit validation, error result, exception handling, or rollback is visible. A failing access or invocation can prevent completion; earlier effects are not visibly undone.

## Dependencies

| Opaque reference ID | Role | Established limit |
|---|---|---|
| F-U001 | Initial incoming-state-derived indirect invocation | Target identity, signature, and effects unresolved |
| F-U002 | Remaining pre-mutation indirect invocation sites | Collection of sites, not one uniform callee |
| F-U003 | Terminal four-argument invocation | Target and returning-call semantics unresolved |

These reference IDs identify unresolved roles, not independently resolved implementations. No dependency is assumed to preserve globals or preserve, own, or release memory.

## Acceptance examples

These are conditional static obligations, not executed tests.

### Width and extraction

For a paired result with low word `0x12345678` and high word `0x9abcdef0`, a site forwarding the low word shifted left by 8 and the unchanged high word presents `0x34567800` and `0x9abcdef0`, conditional on 32-bit bit-vector shift semantics.

Masking `0xff123456` to 24 bits and shifting left by 6 yields `0x048d1580`.

A selector-producing result whose low seven bits are all set yields selector 508 and small base 1016 under the stated bit-vector interpretation.

### Byte widening and repeated stores

A byte read of `0xe3` widened for a word store contributes `0x000000e3`, not a sign-extended value.

Where stores are individually observable, both consecutive writes at small-base offset `0x15` occur. Later overlapping byte stores can alter a previously cleared word.

### External-base capture

If the read at `0x087fbbbc` returns one value and the subsequent store to `0x60` changes external state, repeated publications still use the captured external base. Reloading it after the `0x60` store changes the required access history.

### Positioned fixed writes

For a retained link of 3 and small base 8, the fixed word at `0x54` receives 6 between the specified small-base word stores. The fixed byte at `0x15` receives 8 after the two small-base byte stores receiving 12.

This example assumes valid noninterfering storage and defined signed arithmetic.

### Final ordering and fresh publication

An access observer must see the `0xa9` word read and scaled-late-base `0x74` read before the `0x3a` write, then the final-node `0x14` and working-base `4` reads.

If a computed store changes `0x67` after its first publication, subsequent reads forward its then-current byte. Substituting the retained fixed word’s original low byte is not equivalent.

### Terminal invocation

Conditional on completion, F-U003 occurs after the last direct word store with the four stated arguments. F-3b6e9f6cfcdd does not synthesize a success result.

## Uncertainties

The following remain **documented_only**:

- Function boundary, calling convention, and interpretation of secondary results.
- Dependency targets, complete call-site contracts, and hidden effects.
- Signed arithmetic and variable-shift semantics outside their defined domains.
- Endianness, address wrapping, alignment rules, and memory mappings.
- Aliasing and asynchronous or memory-mapped behavior.
- Exact repeated-region multiplicity as specified by this document.
- Operand-read ordering within expressions lacking an established evaluation order.
- Whether the terminal transfer returns normally.
- Domain meaning, ownership policy, concurrency guarantees, and error taxonomy.

These limits prevent this document from establishing a complete independent replacement interface.

## Evidence

**Established static evidence:** the supplied representation supports the access widths, retained-value relationships, positioned fixed-address writes, external-base capture before the `0x60` store, final-region ordering around `0x3a`, and three distinct later reads of `0x67`.

**Inferred intent:** field packing and state linking are plausible interpretations only.

**Unknowns:** unresolved invocation behavior and machine semantics remain explicit rather than replaced with preservation or ownership assumptions.

No runtime tests, execution traces, memory observations, or external implementation inspection have been performed. No snapshot hash or independently verified provenance is asserted.
