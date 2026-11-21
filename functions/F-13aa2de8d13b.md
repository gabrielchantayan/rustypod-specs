<!-- Trusted provenance: F-13aa2de8d13b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-13aa2de8d13b

## Purpose

F-13aa2de8d13b performs an ordered sequence of indirect invocations, transforms selected result words, and then accesses memory through computed addresses and fixed numerical addresses. Its final represented action is an unresolved indirect invocation with four arguments.

No application-level purpose is established. The constants, recurring offsets, and byte writes do not establish text formatting, object initialization, collection management, or character emission.

This contract distinguishes:

- **Established static evidence:** behavior directly represented in the supplied material.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior for which no definitive execution contract is available.

All effects are conditional on execution reaching them. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits, presented as signed | Passed unchanged to the initial invocation and also shifted left by 14 bits |
| Second input | 32 uninterpreted bits | Passed unchanged to the initial invocation |
| Third input | 32 uninterpreted bits | Passed unchanged to the initial invocation |

The initial invocation receives four represented argument words: the three inputs in order, then the shifted first input.

No direct dereference of an explicit input is represented in this function. The initial dependency may nevertheless interpret any input as an address; its contract is unknown.

### Implicit inputs

Execution also depends on:

- An incoming control-link value used to form the initial invocation target.
- Two incoming unsigned register-state words, each later shifted right by 15 bits.
- Secondary output state associated with earlier invocations.
- Single-word and paired-word invocation results.
- Memory at fixed addresses and computed addresses.
- A global word loaded during the direct-memory region.
- An unresolved final invocation target.

The ABI, validity requirements, and responsibility for establishing this state are **documented_only**. These dependencies must not be replaced with assumed zero values.

### Output and preconditions

No explicit return value is represented. Observable outputs are dependency invocations, memory accesses, and the final four-argument invocation.

A return is represented after the final invocation. Whether its target returns, and whether the enclosing function consequently returns normally, is unknown.

There are no visible argument, address, alignment, range, or target checks. Successful execution requires every reached target and memory access to be valid in the actual environment, including accesses to small numerical addresses. No conventional protected-memory environment is established as supported.

## Behavior

### Ordering and arithmetic model

The represented body has three regions: indirect invocations and result transformations; direct-memory updates including repeated publication; and final updates followed by indirect dispatch. No explicit conditional branch, recovery path, or rollback appears. Dependencies may branch, modify memory, fail, or not return.

Calls with no represented arguments do not establish parameterless interfaces. Their complete machine-state inputs are unknown.

The representation distinguishes 8-, 16-, 32-, and 64-bit quantities. Conditional examples below use a **32-bit bit-vector interpretation**: arithmetic retains the low 32 bits, byte writes retain the low 8 bits, and halfword writes retain the low 16 bits. This model does not establish actual signed-overflow, pointer, or processor semantics.

Several shifts and multiplications use values presented as signed. Unbounded mathematical arithmetic is not an established substitute. A late signed right shift takes its count from the low byte of an address-like value; negative operands and counts of 32 or greater remain **documented_only**.

Splitting a represented 64-bit result into low and high words establishes significance, not memory byte order.

### Invocation and transformation region

The initial target is formed from incoming control state plus 0xFFE. Subsequent targets use represented location-related values plus the same offset. Dependency identities and contracts are unavailable.

The invocation order is significant independently of result calculations. The region includes:

- Invocations receiving explicit zero words.
- Invocations receiving earlier results shifted left by 8 bits.
- Paired results divided into low and high 32-bit words.
- Four-argument combinations of unshifted high words, shifted low words, and retained transformed values.
- Arguments derived from secondary output state rather than explicit function inputs.

The following extraction properties are established under the stated bit-vector model:

| Transformation | Result |
|---|---|
| Retain low 24 bits, then unsigned shift right by 8 | Original bits 8–23; range 0–0xFFFF |
| Retain low 18 bits, then unsigned shift right by 11 | Original bits 11–17; range 0–0x7F |
| Retain low 17 bits, then unsigned shift right by 4 | Original bits 4–16; range 0–0x1FFF |
| Retain low 24 bits, then unsigned shift right by 12 | Original bits 12–23; range 0–0xFFF |
| Shift left by 8, retain low 15 bits, then unsigned shift right by 6 | Multiples of four from 0 through 508 |

Other represented shifts include counts 6, 7, 9, 13–17, 22–26, and 30. Intermediate truncation matters. For example, a 32-bit value shifted left by 14 and subsequently left by 23 has no surviving bits under this model.

Some retained intermediates are explicitly zero and remain zero through subsequent masks and shifts. This does not remove the effects of surrounding invocations.

### Initial direct-memory transition

The following names describe dataflow roles, not objects:

- **Final pair:** low and high words of the last paired result before direct memory access.
- **Shifted base:** an earlier result word shifted left by 25.
- **Scaled base:** another result word multiplied by 0x02000000.
- **Small index:** the bounded multiple-of-four extraction above.
- **Small base:** twice the small index.

The first memory effects preserve this order:

| Position | Access |
|---:|---|
| 1 | Write the shifted base as a word at final-pair high word plus 0x14 |
| 2 | Write the low byte of that high word at shifted-base offset 9 |
| 3 | Write the scaled base as a word at final-pair low word multiplied by 256, plus 0x54 |
| 4 | Overwrite shifted-base offset 9 with the low byte of the shifted base |
| 5 | Write a zero word at scaled-base offset 0x14 |
| 6 | Read a byte at scaled-base offset 5 |
| 7 | Write a zero word at small-base offset 0x14 |
| 8 | Write the zero-extended byte from position 6 at scaled-base offset 0x34 |
| 9 | Read a word at scaled-base offset 0x14 |
| 10 | Write another zero word at small-base offset 0x14 |
| 11 | Write the low byte of the word from position 9 twice consecutively at small-base offset 0x15 |

A byte at offset 0x15 can overlap a word at offset 0x14. Repeated writes are part of the access sequence, even when their final-state contribution is redundant.

For ordinary memory without interfering aliasing, position 9 observes the zero from position 5. Intervening aliases or special-memory behavior can invalidate that conclusion.

### Intermediate address formation

The next region reads a byte from fixed address 0x05 and uses its zero-extended value as a numerical base. It also writes a byte at fixed address 0x01.

A word read at offset 0x54 from the byte-selected base supplies the **central scaling word**. That location is subsequently overwritten; later use of the captured scaling word is not a fresh read.

The region performs word and byte updates through the small base, scaled base, and further loaded values. Relevant properties include:

- Word reads at offset 0x74 supply further address-like values.
- The central scaling word participates in address calculations multiplied by two, four, and eight.
- Fixed address 0x54 receives a word derived by doubling the central scaling word.
- Fixed address 0x15 receives a byte derived from the small base.
- Byte selectors obtained from offset 5 are used directly for word reads at selector plus 0x74.
- A byte read at offset 5 is performed again later; it is not established as a cached selector.
- Self-valued word writes and repeated writes to the same destinations remain observable accesses.

The recurring offsets do not establish object layouts or disjoint storage. Nested address expressions involving multiple reads do not, by themselves, establish a total machine-level order among their constituent reads.

### Literal writes and publication preparation

A later selector-based word read supplies the **selected word**. Define these contract-level relationships under the bit-vector model:

- **Selected quadruple:** four times the selected word.
- **Selected double:** twice the selected word.
- **Adjusted scaling word:** four times the central scaling word, minus 0x96.
- **Derived base:** four times the adjusted scaling word.

The literal 0x3A is written with three different widths: a byte at selected quadruple, a halfword at eight times the selected word, and a word at selected-quadruple offset 0x20.

Derived-base offset 0x20 receives four word writes, in order: 0x20, 0x2F, a zero-extended byte read from derived-base offset 4, and 0x20. The byte read occurs after the first two writes and before the last two.

**The publication global base is loaded only after the final 0x20 write at derived-base offset 0x20. Its word load precedes the fixed-address word write of 0x20 to 0x60.** It is not an entry-time value. Earlier writes or dependency effects may affect the loaded value if storage aliases.

### Repeated publication

The publication block uses the captured global base, derived base, and selected quadruple.

Each group preserves this four-access order:

1. A word write of the selected quadruple at an eight-times-scaled address plus 0x54.
2. A byte write of the derived base’s low byte at global-base offset 0x19.
3. A byte write of the global base’s low byte at derived-base offset 0x11.
4. A word write of the eight-times-scaled value at global-base offset 0x34.

The first group uses the central scaling word. Subsequent groups use the adjusted scaling word. The same retained operands recur; there is no established iteration over distinct objects and no represented reload of the global base within these groups.

Every represented repetition is significant. The exact numerical repetition count is **documented_only** in this contract; collapsing the block into one group or inventing a collection length is not supported.

### Final-region capture timing

The opening accesses after publication have the following interleaving:

1. Read a word from fixed address 0xA9; retain it as the **saved fixed word**.
2. Read a word at selected-quadruple offset 0x74; retain it as the **saved selected word**.
3. Write a byte at fixed address 0x3A, derived from eight times the adjusted scaling word plus 0x30.
4. Read and retain a word at selected-double offset 0x14; this becomes the third dispatch argument.
5. Read and retain a word at derived-base offset 4.

The fixed-address 0x3A write therefore precedes the last two captures. It must not be moved after them: either captured address could overlap that byte.

The function then publishes low bytes of the derived and global bases at selected-double offset 0x0D and derived-base offsets 0x0D and 9, writes 0x6C at derived-base offset 8, and writes the selected double itself as a word at its offset 0x54.

### Live fixed-address byte reads

Fixed address 0x67 is not merely a cached representation of the saved fixed word.

First, the low byte of the saved fixed word is written to 0x67. A word is then read at selected-double offset 0x44. Only afterward is 0x67 read as a byte to supply the byte write at fixed address 0x21.

The captured offset-0x44 word next supplies a byte write at derived-base offset 0x10, a word write at global-base offset 0x54, and the signed-right-shift value written as a byte at selected-double offset 1. The shift count is the derived base’s low byte.

The global-base offset-0x54 word is subsequently overwritten with the derived base. A word read at derived-base offset 0x40 is later stored at derived-base offset 0x10. Between that read and store, the previously saved derived-base offset-4 word is written at global-base offset 0x74. Global-base offset 0x44 then receives the selected double.

After those effects, a **new byte read from 0x67** supplies a write at derived-base offset 9. The following intervening accesses occur before another read from 0x67:

- A byte write of the derived base’s low byte at selected-double offset 9.
- A word write of the saved fixed word at derived-base offset 0x44.
- A word read at derived-base offset 0x54, retained as the **late destination base**.

A **further byte read from 0x67** then supplies another write at derived-base offset 9.

These are three distinct reads from 0x67: one for 0x21 and two for derived-base offset 9. No equality among their results is guaranteed. Aliasing, special memory, or external changes may distinguish them.

### Remaining updates and dispatch

The saved selected word’s low byte is written at late-destination offset 0x0D. The saved selected-double offset-0x14 word is restored at selected-double offset 0x54. A word is then captured from derived-base offset 0x14.

Subsequent updates publish the saved selected word’s low byte at selected-double offset 0x11, overwrite derived-base offset 9 with the late destination’s low byte, and overwrite derived-base offset 0x44 with the late destination as a word.

A word read at selected-double offset 4 selects the **dispatch base**. A halfword value 0x65 is written at twice that base. The function then captures a word from dispatch-base offset 0x64 as the second dispatch argument.

Before reading the fourth dispatch argument from dispatch-base offset 0x74, it writes the low byte of the captured derived-base offset-0x14 word at late-destination offset 0x11 and writes the word 100 at late-destination offset 0x54.

After capturing the fourth argument, it reads a further word at selected-double offset 0x64. The final stores publish the dispatch base’s low byte at selected-double offset 9, the second argument at offset 0x54 from that further word, and that further word at twice the second argument.

The final unresolved invocation receives, in order:

1. 0x72.
2. The captured dispatch-base offset-0x64 word.
3. The earlier saved selected-double offset-0x14 word.
4. The captured dispatch-base offset-0x74 word.

Saved values are not interchangeable with fresh reads after intervening writes.

## Side effects

### Widths and storage

| Width | Observed locations |
|---|---|
| 8 bits | Computed offsets including 1, 5, 8, 9, 0x0D, 0x10, 0x11, 0x15, 0x19; fixed addresses including 0x01, 0x05, 0x15, 0x21, 0x3A, 0x67 |
| 16 bits | Scaled locations receiving 0x3A or 0x65 |
| 32 bits | Computed offsets including 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74; fixed addresses 0x54, 0x60, 0xA9; the publication-global load |

Overlapping byte and word accesses make endianness relevant. Alignment, endianness, address wraparound, low-address accessibility, and special-memory semantics are not established.

### Ownership, failure, and concurrency

No explicit allocation, release, reference-count operation, or ownership transfer is established for the direct-memory region. Dependencies may allocate, release, retain, or invalidate storage; global preservation and lifetime guarantees must not be assumed.

No status output, handler, cleanup protocol, or rollback is visible. An invalid access or target may prevent later effects. The failure mechanism is **documented_only**, and earlier completed effects are not visibly undone.

Atomicity, synchronization, reentrancy, and thread safety are unknown.

## Dependencies

F-13aa2de8d13b is the only supplied opaque function identity. No opaque IDs are supplied for its indirect dependencies; none are invented here.

Unresolved dependencies comprise the entry-state-derived invocation, location-derived invocations, invocations supplying single-word or paired-word results and secondary state, and the final four-argument invocation.

For each, complete arguments, global effects, ownership behavior, failure behavior, result validity, and normal-return guarantees are **documented_only**. A represented result width is not a complete dependency signature.

## Acceptance examples

These are conditional static examples, not executed tests.

- **Initial forwarding:** With first input 1, the fourth initial argument is 0x00004000 under the bit-vector model. The first three argument bit patterns remain unchanged.
- **Extraction boundary:** Retaining the low 24 bits of 0xAB123456 and shifting right by 8 gives 0x1234.
- **Small-address boundary:** An extraction source of 0x7F produces small index 508 and small base 1016. Substituting an allocated-object address does not preserve that calculation.
- **Overwritten byte:** Both initial writes at shifted-base offset 9 remain required accesses. Without interference, the second leaves the shifted base’s low byte.
- **Global-load timing:** If preceding derived-base writes affect the publication-global storage, the global load observes storage after those writes. It still precedes the write to 0x60.
- **Interleaved capture:** If selected-double offset 0x14 overlaps address 0x3A, its saved word must reflect the preceding byte write according to the actual memory model. A pre-write capture is not equivalent.
- **Live byte reads:** If the three reads from 0x67 return 0x11, 0x22, and 0x33, they supply the write to 0x21 and the two later writes to derived-base offset 9, respectively. Reusing the saved fixed word’s low byte is not equivalent.
- **Repeated publication:** Where individual writes are observable, identical repeated publications remain distinct effects.
- **Dispatch:** On reaching the final invocation, its first argument is 0x72. No character-processing interpretation follows.

## Uncertainties

The following remain **documented_only**:

- Actual ABI, implicit state, dependency identities, and complete dependency contracts.
- Fidelity of the represented control flow and recovered widths.
- Signed arithmetic, oversized shifts, pointer conversion, and address wraparound.
- Memory mapping, alignment, endianness, aliasing, and special-memory behavior.
- Exact publication repetition count asserted by this contract.
- Object layouts, ownership, lifetime, synchronization, and concurrent interference.
- Final target behavior and normal-return reachability.

“Publication” describes recurring writes, not an established synchronization protocol. Character-like interpretations of constants are inferred intent only.

## Evidence

This contract uses static inspection of the supplied restricted representation. The earlier draft and reviewer feedback are not independent evidence.

Established evidence includes explicit input forwarding, ordered indirect invocations, width-specific transformations and accesses, the publication-global load’s position, the interleaved 0x3A write and saved reads, three separate final-region reads from 0x67, and the final four-argument invocation.

No runtime execution, tests, dependency inspection, memory-map verification, or external provenance verification was performed. No snapshot hash is asserted.
